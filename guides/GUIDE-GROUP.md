# Groups: User Guide

**Status**: Active
**Audience:** Application developers and operators provisioning collective identities (organizations, families, communities, federations) on Entity Core peers. Assumes familiarity with identity (GUIDE-IDENTITY.md) — group reuses identity's four-layer model directly. Familiarity with roles (GUIDE-ROLE.md) is helpful for the in-group authority sections.
**Spec reference:** `EXTENSION-GROUP.md` (current draft), `EXTENSION-IDENTITY.md` (current draft), `EXTENSION-ROLE.md` (current draft), `ENTITY-CORE-PROTOCOL.md` §5 (capability system).
**Related guides:** GUIDE-IDENTITY.md, GUIDE-ROLE.md, GUIDE-CAPABILITIES.md, GUIDE-MULTISIG.md.

---

## A note on naming used in this guide

Two terms recur. Pin them once now:

- **The operator key** is the day-to-day signing key for an identity. The identity extension calls it "Op." For a group, there's a group operator key (the group's day-to-day key); for an individual, there's a personal operator key. In examples below, when you see `alice_op` or `Op_alice`, that's Alice's operator key — her individual signing key, the one her own identity extension manages. When you see `acme_op`, that's Acme the group's own operator key, distinct from any member's personal operator key.
- **The public identity** is what external contacts see. For an individual, `Public_alice`. For a group, `Public_acme`. It's a separate keypair from the operator key in the default configuration.

If you haven't read GUIDE-IDENTITY.md, the four-layer identity model (quorum / operator / public identity / runtime peers) is worth skimming before §2 of this guide, because group reuses it directly.

---

## 1. What groups are (and what they aren't)

A **group** is an identity that represents multiple individuals collectively. Acme as an employer; the partnership of Acme and Globex; a family; a small open-source project; a community garden's mailing list. The group has its own external face (`Public_acme`), its own operator key (`acme_op`), its own quorum, its own runtime peers. Members aren't the group; they belong to it.

The core protocol gives you per-peer authorization. Identity (the extension) gives you "Alice across multiple devices, with rotation and recovery." Group (this extension) gives you "Acme as a multi-user authority, where membership and roles within Acme are managed."

**A group is structurally an identity.** Its quorum, operator, public face, and runtime peers are exactly the same primitive types as in the identity extension, scoped under the group's namespace. What the group extension adds is membership management, lifecycle, governance patterns, and composition seams.

**The fall-back-to-individual property holds.** A user who doesn't need group infrastructure doesn't install this extension. The core protocol plus identity plus role gives them everything they need for personal authority. Groups are opt-in. A peer that doesn't run the group extension can still receive cross-peer caps from group-extension peers — cap chains stay clean; group attestations are non-cap entities that group-unaware peers ignore (the same two-mechanisms invariant the identity extension uses).

What the group extension does:
- Defines the group entity and its identity stack under the group's namespace.
- Provides governance dispatch: founder K-of-N, admin set, all-members, on-chain, hierarchical.
- Manages membership (`add_member`, `remove_member`, `change_role`).
- Manages subgroups (`add_subgroup`, `remove_subgroup`).
- Lifecycle ops (`form`, `dissolve`, `merge`, `split`) composed as identity-extension primitives.
- Acting-on-behalf-of attestations (when a member sends a message "from the group").
- Federation patterns (per-member individual sharing + bulk attestation sync).

What the group extension does NOT do:
- Modify the core protocol's chain verification.
- Introduce new identity primitives — reuses the identity extension directly.
- Introduce new capability mechanisms — uses caps + multi-sig as needed.
- Coordinate runtime peers as a service cluster — that's a separate planned extension.
- Multi-user shared individual identities — the identity extension is single-user-per-identity by design; group is the multi-user surface.

Terminology pin: in this guide, **group** is the multi-user identity. **Cluster** is reserved for a separate planned extension (HA, replication, leader election among runtime peers). They are different concepts despite the colloquial overlap.

---

## 2. The model — a group is a type of identity

The group's identity stack mirrors the identity extension directly. Re-using the same four layers:

| Layer | Individual identity (Alice) | Group identity (Acme) |
|---|---|---|
| **Quorum** | Alice's backup keys (paper, second device) | Constituents drawn from members or admins (founders' operator keys, the admin set's operator keys, all members' operator keys, on-chain oracle, etc.) |
| **Operator** (`Op`) | Alice's day-to-day operational key | Acme's day-to-day operational key (`acme_op`), distinct from any individual member's operator key |
| **Public** | `Public_alice` — Alice's external face | `Public_acme` — Acme's external face |
| **Runtime peers** | Alice's devices and servers | Machines hosting Acme (`partnership_server`, member-volunteered hardware, etc.) |

What's structurally the same:
- Operator-delegation entities (the quorum signs K-of-N to install the operator's authority).
- Public-quorum-attestation (contact-visible record of constituents and threshold).
- Public-identity-rotation entities (handoff or compromise-recovery).
- Per-peer runtime-peer-attestations (multi-modal: internal / public / per-relationship / embedded).
- Bootstrap exemption (the group's `form` op, like identity's `configure`, runs via the SDK's L0 direct-store path during initial setup).

What changes vs. an individual quorum:

- **Constituents are policy-driven** (per the `governance` field), not personal. Founders, admins, all members, oracles, parent-group references — not "my paper backup key."
- **Constituents can be dynamically derived.** An `admin_set` quorum recomputes when a member becomes / stops being an admin. The group handler chains `change_role` into `rotate_quorum` to keep the quorum entity in sync.
- **Multiple groups can run on the same hardware.** A host machine MAY operate as multiple runtime peers — one for `Public_alice`, one for `Public_garden`, etc. Each runtime peer has its own peer-config; peer-configs MUST NOT share state across identities.

### 2.1 The reuse principle

Read carefully: there are **no parallel `system/group/quorum`, `system/group/operator-delegation`, etc. types.** The group extension uses identity's existing types under the group's namespace:

| Type | Lives at (in group's namespace) |
|---|---|
| `system/identity/quorum` | `system/group/{group_id}/identity/quorum/{quorum_id}` |
| `system/identity/operator-delegation` | `system/group/{group_id}/identity/quorum/{quorum_id}/operator-delegation/{hash}` |
| `system/identity/public-identity-attestation` | `system/group/{group_id}/identity/internal/public-identity-attestation/{hash}` |
| `system/identity/runtime-peer-attestation` | `system/group/{group_id}/identity/{mode-path}/runtime-peer-attestation/{peer_id}` |
| `system/identity/public-quorum-attestation` | `system/group/{group_id}/identity/public/public-quorum-attestation/{hash}` |
| `system/identity/public-identity-rotation` | `system/group/{group_id}/identity/public/public-identity-rotation/{hash}` |
| `system/identity/peer-config` | per-runtime-peer local; one per identity served |

Use the identity-handler operations to manage these. Group extension types layer above them.

### 2.2 What's in the group extension proper

Four group-specific entity types beyond the reused identity types:

| Type | Path | Signed by | Purpose |
|---|---|---|---|
| `system/group` | `system/group/{group_id}` | Not signed (structural) | Group entity: name, public_identity, governance, created_at |
| `system/group/member` | `system/group/{group_id}/members/{member_peer_id}/{hash}` | K-of-N group quorum | Member entry: who's in the group, primary role, joined-at, optional `derived_from_parent` |
| `system/group/subgroup` | `system/group/{parent_id}/subgroups/{subgroup_id}/{hash}` | K-of-N parent group quorum | Subgroup attestation: parent acknowledges this subgroup |
| `system/group/acting-on-behalf-of-attestation` | `system/group/{group_id}/acting-on-behalf-of/{delegate_id}/{hash}` | `Public_group` | Authorizes a member to act as the group's official voice |

Eleven handler operations: `form`, `dissolve`, `merge`, `split`, `add_member`, `remove_member`, `change_role`, `add_subgroup`, `remove_subgroup`, `attest_acting_on_behalf`, `revoke_acting_on_behalf`.

---

## 3. Governance patterns

The group's quorum constituents and threshold are determined by the `governance` field on the group entity. Five patterns are recognized.

A note on signer identifiers in the examples below: the `signers` (or `founders`) list contains specific keypair identifiers. In the default mode (concrete-peer-ID; see §4), each entry is a hash of a specific keypair — typically each founder's *individual* operator key (`alice_op`, `bob_op`, `carol_op`). That's because each founder needs to sign group operations using a key they actually control day-to-day, and their own operator key is exactly that. **The group's own operator key (`acme_op`) is delegated by the quorum AFTER `form`; it is NOT a constituent of the quorum.** The constituents are the founders' personal keys; the group-level operator key is the authority that flows downstream from K-of-N constituents-signed delegation.

### 3.1 Founder K-of-N

```
governance := {
  type:      "founder_k_of_n",
  founders:  [alice_op, bob_op, carol_op],
  threshold: 2
}
```

A fixed set of founder keys (each founder's individual operator key) with a static threshold. Operations on the group require K-of-N founders to sign. Founders rarely change; most operations go through the group's operator key (delegated by the founder quorum after `form`).

Suitable for: **small businesses, families, partnerships, joint accounts.** When you have a small number of well-known principals and you want their authority to be load-bearing.

Trade-off: replacing a founder requires `rotate_quorum`. Routine in a stable group; cumbersome if founders churn.

### 3.2 Admin set

```
governance := {
  type:                "admin_set",
  threshold_default:   1,
  threshold_sensitive: 3,
  sensitive_ops:       ["dissolve", "rotate_quorum", "merge"]
}
```

The admin set is *dynamic* — admins are added or removed via `change_role` operations. The group's quorum entity has signers derived from "members with role admin" (concretely, those members' operator keys). Threshold can vary per operation type.

When the admin set changes, the group handler chains `change_role` into `rotate_quorum` to update the quorum entity. This chain is normatively pinned by validation atomicity, ordered writes, and a `409 chain_failed` recovery story — see §6.2.

Suitable for: **organizations with rotating leadership, project teams, communities with elected admins.** When the set of authoritative principals evolves but the threshold doesn't.

### 3.3 All-members

```
governance := {
  type:                "all_members",
  threshold_routine:   1,
  threshold_protected: "<integer or fraction>"
}
```

Constituents are every member's operator key. Threshold configurable: 1-of-N for inclusive (any member can act), K-of-N for protected operations.

Suitable for: **very small groups (couples, partnerships) or consensus-driven groups (small co-ops).** When membership is small and equal authority is the design intent.

Trade-off: scales poorly. A group of 100 members with `threshold_protected: 51` requires 51 signatures for protected operations.

### 3.4 On-chain governance

```
governance := {
  type:             "on_chain",
  chain:            "ethereum",
  contract:         "0x...",
  oracle:           <oracle_group_id_or_peer_id>,
  oracle_threshold: K
}
```

Constituents are an oracle (or oracle quorum) that attests outcomes of on-chain proposals. The on-chain mechanism is opaque to the protocol; the oracle is a trusted bridge.

**Recommended: K-of-N oracle quorum.** A single oracle is a single point of trust. Use K-of-N oracle services run by independent operators.

Suitable for: **DAOs, blockchain-anchored governance.**

**Validation status:** designed, not validated; pilot scope. Walks at the conceptual level (e.g., the CommunityDAO walkthrough in `EXPLORATION-DEPLOYMENT-SHAPES.md §14.6`) explore the pattern but assume bridge infrastructure not specified here. Treat as pilot until the bridge layer is exercised in production.

### 3.5 Hierarchical

```
governance := {
  type:                "hierarchical",
  parent_group:        <parent_id>,
  internal_governance: { type: "admin_set", threshold_default: 1, ... },
  parent_required_ops: ["rename", "dissolve", "merge"]
}
```

A subgroup with degraded governance — some operations require parent-group approval. Engineering as a subgroup of Acme: Engineering's admins manage internal Engineering matters, but renaming or dissolving Engineering requires Acme's quorum.

Composes with subgroup membership (§7).

**Authority walk fail-closed.** When resolving authority for a hierarchical-governance op, the implementation walks the parent chain from the subgroup upward. If the walk detects a cycle or exceeds the depth bound (32; SHOULD), the walk MUST fail closed and return `403 governance_walk_cycle`. This is structurally enforced by §7's cycle prevention on `add_subgroup`.

### 3.6 Choosing a governance pattern

| Pattern | Use when | Avoid when |
|---|---|---|
| Founder K-of-N | Small principal set; stable; coordinated | Frequent principal churn |
| Admin set | Rotating leadership; configurable thresholds | Very small groups (overhead) |
| All-members | Consensus-driven; small | Large membership |
| On-chain | DAO; on-chain proposals authoritative | Need finality faster than chain block time; no bridge infrastructure |
| Hierarchical | Subgroup with parent-imposed constraints | Independent subgroups |

### 3.7 Choosing the signer keys

The governance pattern fixes WHO can sign group operations (founders, admins, all members, oracle, parent group). What actually goes in the `signers` list — the keypair identifiers — is a *separate* question with privacy implications. This is orthogonal to the governance pattern AND to the resolution mode (§4 concrete vs. identity-resolved).

In concrete mode (the default), each entry in `signers` is a hash of a specific keypair. That keypair is what the constituent will sign Acme's operations with. Three patterns are common; you choose per-deployment.

**Option A: Each founder's personal operator key.** The constituent IS the founder's individual `alice_op` from their personal identity stack — the same hot key Alice uses to sign role assignments and other internal management on her own peers.

- Acme's `public-quorum-attestation` (visible to any contact of Acme) lists `alice_op` as a constituent. Anyone who interacts with Acme as an external contact learns that `alice_op` exists and is one of Acme's authorities. This is **metadata leakage**, not key compromise — Bob can't sign as Alice without her private key — but it does break the identity model's "Op is internal to Alice's fleet" property at the group seam.
- When Alice rotates her personal Op (yearly hygiene, compromise recovery, etc.), Acme's quorum has to be updated via `quorum-update`. Personal identity rotation entangles with group governance.
- **Suitable for:** public-facing organizations, partnerships, or any group where the founders' participation is itself public knowledge. Acme's CEOs being publicly known signers of Acme isn't a leak — it's what you'd expect.

**Option B: A dedicated per-group key.** Alice generates a separate signing key specifically for her Acme participation — call it `alice_acme_op`. The constituent in Acme's quorum is this dedicated key; her personal `alice_op` stays internal to her fleet.

- Acme's `public-quorum-attestation` lists `alice_acme_op`. This key has no other purpose; its exposure carries no metadata about Alice's other activities.
- When Alice rotates `alice_acme_op` (the hygiene cadence for the Acme key), only Acme's quorum-update fires. Her personal Op rotation is independent.
- When Alice rotates her personal Op, Acme's quorum is unaffected.
- **Suitable for:** private groups, partnerships where Alice doesn't want her participation publicly tied to her individual identity, families, support networks, or any deployment where the founder values clean separation between their personal identity and their group affiliations. **This is the right default for most deployments.**

**Option C: Identity-resolved (Public_alice as the constituent).** The `signers` list contains `Public_alice` — Alice's external face — rather than any concrete signing key. Acme's quorum entity uses `signer_resolution: "identity-resolved"`. Validation walks Public_alice's operator-delegation chain at signature-verification time to find Alice's current Op and verifies the signature against it. See §4.2 for the full mechanics.

- `Public_alice` is already public by design, so listing it publicly leaks nothing new.
- Alice can rotate her Op privately; Acme adapts transparently. No quorum-update needed for Op rotations.
- **Visibility constraint (formalized in v1.1 as shared-infrastructure-only):** for the verifier (e.g., `partnership_server`) to resolve `Public_alice` → current Op, Alice's `operator-delegation` must be reachable from the verifying peer. Operator-delegations are internal-mode in the identity extension (per `EXTENSION-IDENTITY.md §3.3`). The v1.1 spec formalizes ONE reachability mechanism: shared infrastructure (the group's verifying peer runs on the constituent's hardware, with native access to the operator-delegation tree). Two other mechanisms — per-relationship publication of operator-delegation, or embedded resolution proof attached to signatures — are convention-only in v1.1 and not yet standardized for cross-impl interop. See `EXTENSION-GROUP.md §5.6` "Operational requirement for identity-resolved mode" for the deployment-level guidance, and `PROPOSAL-IDENTITY-INFRASTRUCTURE.md §13.4` for the tracked future-amendment item if production deployments need cross-fleet identity-resolved validation.
- **Suitable for:** groups with frequent member rotation, deployments where members manage identity hygiene independently and the group should not have to coordinate, or environments where the verifier and the constituent share infrastructure.

**Default recommendation.** For most deployments, **Option B (dedicated per-group key)** is the right default. It preserves the identity model's separation, keeps personal Op rotation independent of group governance, and avoids the operator-delegation reachability question entirely. Reserve Option A for genuinely public groups where the keys' exposure is part of the design. Reserve Option C for deployments that explicitly want identity-layer coupling and have validated the visibility resolution.

**Notation in this guide.** The worked examples below use `alice_op`, `bob_op`, `carol_op` as shorthand for "each founder's signing key into the group." Depending on the deployment's choice, these might be personal Ops (Option A), dedicated per-group keys (Option B — and in that case `alice_acme_op` would be the more accurate name), or references to public identities under identity-resolved mode (Option C). Where the distinction matters for a specific example, the prose calls it out explicitly.

---

## 4. Signer reference style — concrete vs. identity-resolved

Orthogonal to the governance pattern: how the quorum's `signers` field is interpreted. Two modes, both supported via the `signer_resolution` field on the (reused) identity quorum schema.

### 4.1 Concrete-peer-ID style (default)

The `signers` list contains specific keypair identifiers — concrete hashes of the actual keypairs that will sign the group's operations. Validation looks up each constituent's identity entity by hash and verifies the signature against that identity.

The choice of WHICH keypair goes in the list (a member's personal Op vs. a dedicated per-group key) is covered in §3.7 above. Concrete mode is agnostic to that choice — both are concrete hashes.

When the member rotates whatever signing key they're using for the group, the group's quorum no longer recognizes their signatures (the new key isn't in `signers`). The group MUST issue a `quorum-update` to swap the old keypair for the new one.

**Rigid but simple.** Each signer is a specific keypair; rotation is explicit; audit trail is rich (every key change visible in the group's history).

### 4.2 Identity-resolved style (advanced)

The `signers` list contains references to public identities (`Public_alice`, `Public_bob`, etc.) rather than specific keypairs. Validation resolves each constituent through its identity layer to find the current operator-delegation, then verifies the signature against the current operator key for that identity.

When a member rotates their operator key, their own operator-delegation chain reflects the rotation. The group's quorum dereferences `Public_X` → current operator key transparently. **No `quorum-update` needed for the group.**

**Flexible but coupled.** Members rotate independently; the group survives without coordination — but compromise of a member's identity affects the group automatically (the attacker rotating the member's operator key gains the member's quorum slot).

**Operator-delegation reachability (formalized as shared-infrastructure-only in v1.1).** For the verifier to walk `Public_alice` → current Op, Alice's operator-delegation needs to be reachable from the verifying peer (e.g., from `partnership_server`). Operator-delegations are internal-mode by default in the identity extension — they don't propagate to contacts. The v1.1 spec (`EXTENSION-GROUP.md §5.6` and `EXTENSION-IDENTITY.md §3.3`) formalizes ONE reachability mechanism:

1. **Shared infrastructure (v1.1; supported).** The group's verifying peer runs on the constituent's hardware. Native access to the operator-delegation tree. This is the only mechanism guaranteed to interoperate across implementations in v1.1.

Two further mechanisms are convention-only in v1.1 (deployments may adopt them with their own documentation, but cross-impl interop is not guaranteed):

2. **Per-relationship publication of operator-delegation** — Alice promotes her operator-delegation to a per-relationship publication mode for the group. Not yet formalized in the identity extension's publication-mode set (only `runtime-peer-attestation` has explicit per-relationship support).
3. **Embedded resolution proof** — the constituent attaches a current-operator-delegation snapshot to each signature or to a periodic refresh entity.

A future spec amendment may formalize one of these — most likely (2), which mirrors the per-relationship pattern already in use for `runtime-peer-attestation`. Tracked in `PROPOSAL-IDENTITY-INFRASTRUCTURE.md §13.4` as a known follow-up.

**Until then,** deployments needing cross-fleet identity-resolved validation (where the verifier and constituent do NOT share infrastructure) SHOULD prefer dedicated per-group keys in concrete mode (per §3.7 Option B). See §3.7 Option C for the broader trade-off.

### 4.3 Trade-offs

| Property | Concrete-peer-ID | Identity-resolved |
|---|---|---|
| Verification cost | Direct lookup + signature verify | Identity resolution + lookup + verify |
| Member rotation | Each rotation requires group `quorum-update` | Transparent through member's identity |
| Trust model | Group trusts specific keypairs | Group trusts members' identity layers |
| Compromise blast radius | Group unaffected (different keypair) | Group affected — attacker controlling member's identity has the slot |
| Coordination cost | High over time (many quorum-updates) | Low (members rotate independently) |
| Audit trail | Every member rotation in group's history | Member rotations not in group's history |

### 4.4 Choosing

Use **concrete** when:
- Member rotation is rare AND coordination is cheap.
- Audit requires group-side visibility into each member's key changes.
- Trust is anchored to specific keypairs (regulated environments).

Use **identity-resolved** when:
- Member rotation is frequent.
- Members manage their own identity hygiene independently.
- The group wants to delegate identity-rotation responsibility to members.

### 4.5 Multi-operator constituents

In identity-resolved mode, a member running multiple operator keys concurrently (the multi-Op pattern from the identity extension) contributes one signature per quorum slot. Any of their currently-live operator keys satisfies that slot. Which key produces it is the constituent's choice.

### 4.6 Composition with the multi-sig primitive (no core dependency)

The multi-sig primitive in the core protocol is keypair-based and stays that way. `signer_resolution` is purely an extension-level interpretation; it does NOT change the core protocol's multi-sig verifier.

Identity-resolved mode resolves to concrete keypair hashes at cap issuance time; the multi-sig verifier sees concrete keypairs regardless of mode and applies the local-peer-in-signers rule unchanged.

What this means for you:
- A multi-sig cap with `signers` populated from a group's quorum constituents always carries concrete keypair hashes.
- Identity-resolved mode at the entity-attestation layer (operator-delegations, member entries, etc.) is independent of multi-sig at the cap layer.
- The two layers compose without modifying multi-sig core.

See GUIDE-MULTISIG.md for the cap-layer mechanics; the bridging happens at issuance, not at verification. **For K-of-N decisions within group operations, see GUIDE-MULTISIG.md §6.2 / GUIDE-QUORUM.md §9** for the arbiter rule on cap-level vs entity-level mechanisms.

---

## 5. Forming a group

`form` is the bootstrap-exempt op (parallel to identity's `configure`). The first call on a fresh group runs via the SDK's L0 direct-store path on the runtime peer hosting the group.

### 5.1 What `form` produces

A single `form` call sets up:
1. **The group entity** at `system/group/{group_id}` with `name`, `governance`, and a reference to `Public_group`.
2. **The group's quorum entity** at `system/group/{group_id}/identity/quorum/{quorum_id}` with the initial signer set, threshold, and `signer_resolution` mode.
3. **The initial operator-delegation** (K-of-N signed by the initial quorum) authorizing the group's operator key (`acme_op` for a group named Acme).
4. **`Public_group`'s keypair** (handled by the initiator).
5. **The initial public-quorum-attestation** (K-of-N signed by the quorum itself, contact-visible).
6. **Member entries** for each initial member.

After `form`, the group's identity stack mirrors an individual identity's: subsequent operations go through dispatched EXECUTEs against `system/group:*` ops.

### 5.2 The bootstrap exemption

Like identity's `configure`, the first `form` call cannot go through dispatched EXECUTE because there's no caller capability authorized over the group's namespace yet. It runs via L0 (peer-owner authority on the runtime peer hosting the group). After bootstrap, all group operations require dispatched EXECUTE under the group's quorum or operator authority.

Conformance is observed via post-state (the group entity, quorum, and initial operator-delegation exist) — same convention as identity bootstrap.

### 5.3 The form-request shape

```
form-request := {
  name:           "Acme",
  initial_quorum: {
    signers:           [alice_acme_op, bob_acme_op, carol_acme_op],
    threshold:         2,
    signer_resolution: "concrete"      ; or "identity-resolved"
  },
  governance: {
    type:      "founder_k_of_n",
    founders:  [alice_acme_op, bob_acme_op, carol_acme_op],
    threshold: 2
  },
  initial_members: [
    {member: Public_alice, role: "founder"},
    {member: Public_bob,   role: "founder"},
    {member: Public_carol, role: "founder"}
  ]
}
```

The `signers` and `founders` fields here use **dedicated per-group keys** (Option B from §3.7) — each founder has generated a separate signing key specifically for Acme. This is the recommended default. Replace `alice_acme_op` with `alice_op` if the deployment is choosing Option A (personal Op as constituent — fine for genuinely public groups), or with `Public_alice` if choosing Option C (identity-resolved; set `signer_resolution: "identity-resolved"` accordingly).

The group's *own* operator key (`acme_op`) is generated and delegated by the quorum AFTER `form` runs; it is not in `signers`. `acme_op` is what drives Acme's day-to-day operations downstream of the founder quorum.

The handler produces `{group_id, quorum_id, public_group_id}` in the form-result.

### 5.4 Worked-example walkthrough — Acme founders form Acme

Setup decision (per §3.7): Acme's founders choose **Option B** (dedicated per-group keys). Each founder generates `alice_acme_op`, `bob_acme_op`, `carol_acme_op` — distinct from their personal Ops. Their personal `alice_op`, `bob_op`, `carol_op` stay internal to their fleets.

1. Alice initiates `system/group:form` via L0 on her runtime peer, passing the dedicated keys in `signers` and `founders`.
2. The handler validates the initial_quorum, signs the operator-delegation (K=2 of [`alice_acme_op`, `bob_acme_op`, `carol_acme_op`]) authorizing `acme_op` as Acme's day-to-day operator, generates `Public_acme`, signs the public-quorum-attestation listing the three Acme keys, persists the three founder member entries.
3. The form-result returns `{acme_id, quorum_id, public_acme_id}`.
4. Bob and Carol's runtime peers sync the group's namespace and recognize themselves as founders (their member entries propagate).
5. Subsequent group operations go through dispatched EXECUTE.

After form, three things are true:
- **`acme_op` drives day-to-day operations on Acme.** Adding members, role assignments within Acme, attest_acting_on_behalf — all dispatched through `acme_op`'s authority (via Acme's local peer→Op cap on its runtime peers).
- **The founders' Acme keys (`alice_acme_op`, `bob_acme_op`, `carol_acme_op`) re-emerge only when K-of-N is needed** — rotating Acme's operator, adding admins (in `admin_set` governance), dissolving the group, etc.
- **The founders' personal Ops are never touched by Acme.** Alice can rotate her personal `alice_op` whenever; Acme's quorum stays intact. Acme's `public-quorum-attestation` (visible to contacts) lists the Acme-specific keys, not personal Ops.

If the deployment had chosen Option A instead (personal Op as constituent), the walkthrough is identical except every `*_acme_op` is replaced with the personal `*_op`, and Alice's personal Op rotation would force a `quorum-update` on Acme. If the deployment had chosen Option C (identity-resolved), the `signers` would list `[Public_alice, Public_bob, Public_carol]` with `signer_resolution: "identity-resolved"`, and signature verification would walk Public_X → current Op at validation time — provided the operator-delegation reachability question (§4.2) is solved for the deployment.

---

## 6. Membership management

After `form`, the group has founders. Adding new members, changing roles, removing members all go through dispatched EXECUTE under group authority.

### 6.1 Add a member

```
add-member-request := {
  group_id:       Acme_id,
  member_peer_id: Public_dave,    ; the new member's public identity
  role:           "engineer",
  expires_at:     <optional ms>,
  metadata:       <optional>
}
```

Authority is per governance:
- Founder K-of-N: K founders sign.
- Admin set: K admins sign (per the operation's threshold).
- All-members: K members sign (per threshold).
- On-chain: oracle attests the on-chain proposal outcome.
- Hierarchical: per the subgroup's `internal_governance` for member ops; parent for `parent_required_ops`.

Implementations MAY relax the threshold for routine operations (e.g., `threshold_default: 1` for adding low-privilege members) and require a higher threshold for sensitive ones.

The result is a member entry at `system/group/{group_id}/members/{Public_dave}/{hash}` signed by K-of-N quorum.

### 6.2 Change a role (and the chained quorum-update)

```
change-role-request := {
  group_id:       Acme_id,
  member_peer_id: Public_dave,
  new_role:       "admin"
}
```

For governance types where role affects quorum membership (admin_set, all_members), the handler chains into `rotate_quorum` to update the quorum entity. This chain is normatively pinned by three rules:

1. **Validation atomicity (MUST).** Validate both the member-entry update AND the chained quorum-update end-to-end **before persisting either entity**. Includes the assigner-coverage check, K-of-N signature presence, and membership consistency. Failure → `409 chain_failed`; persist nothing.
2. **Write order (MUST).** When validation succeeds, persist **the member-entry update first, then the quorum-update**. The order matters:
   - Member-entry persists, quorum-update fails → member is recognized as admin at the role layer; not yet in K-of-N. They can call admin-scoped role ops; cannot sign quorum-level decisions. **Safe.** Caller retries; per-entity content-addressing makes the quorum-update idempotent.
   - Quorum-update persists, member-entry fails (rejected order) → member is in K-of-N without admin role recognition. **Privilege escalation surface. Unsafe.**
3. **Recovery (MUST).** The chained `quorum-update` MUST be signed by the current quorum (the K-of-N that validates the change). On second-leg failure, return `409 chain_failed`. Caller retries the full chain.

**Implementations MUST NOT require a tree-level transactional-bundle primitive.** Per-entity atomicity (already provided by the entity store) plus validation atomicity plus ordered writes is sufficient.

**Quorum-update supersedes chain (MUST).** When `change_role` chains into `rotate_quorum`, the resulting `quorum-update` entity MUST set its `supersedes` field to the content_hash of the immediately-previous `quorum-update` (or the original quorum entity's hash if first). The chain is what lets contacts validating compromise-recovery rotations walk back to a known-cached `public-quorum-attestation`. **Don't skip `supersedes`** — cross-peer attestation validation depends on a continuous chain.

### 6.3 Remove a member

```
remove-member-request := {
  group_id:       Acme_id,
  member_peer_id: Public_dave
}
```

For `remove_member`, the handler:
1. Deletes the member entry.
2. **Deletes any role assignments scoped to the group's context** (`system/role/{group_id}/assignment/{member_peer_id}/...`). This is the bridge between membership and role: removing the member entry removes everything role-scoped.
3. **Federation cascade walk.** Walks all `system/group/*/members/*` entries whose `derived_from_parent` field points at this group AND whose `member` matches the removed member, and removes them in cascade. **Independent memberships (no `derived_from_parent` set) are unaffected** (default; safer semantics).
4. **If the member was a quorum constituent** (admin_set / all_members), chains into `rotate_quorum` to remove them — same three-rule ordered-writes logic as `change_role`, same supersedes-chain MUST.

### 6.4 The member entry vs role assignment seam

Two ways to record "Alice is an admin in Acme":

- **Member entry** (`system/group/{acme_id}/members/{Public_alice}/...` with `role: "admin"`).
- **Role assignment** (`system/role/{acme_id}/assignment/{Public_alice}/admin`).

These overlap. The convention pinned by the spec:
- **Member entry records the PRIMARY role and the membership record itself.** Revoking the member entry = removing from the group entirely.
- **Role assignments record ADDITIONAL roles within the group** (orthogonal grants beyond the primary role).
- Removing a member removes both the member entry AND all role assignments scoped to the group.

For everyday "Alice is a member with role X," the member entry is sufficient. For complex multi-role situations (Alice is admin + finance + technical-lead), role assignments add finer granularity.

Multi-role per peer per context is supported — see GUIDE-ROLE.md §4.5. A member can hold multiple role assignments under the group's context concurrently.

---

## 7. Subgroups

A group can have subgroups — Engineering as a subgroup of Acme; Regional Office as a subgroup of Acme. The subgroup is a full group in its own right (own quorum, own identity stack, own members) plus an attestation from the parent acknowledging the relationship.

### 7.1 Forming a subgroup

Two-step:

1. **The subgroup is formed** with its own `system/group:form` call (its own quorum, identity stack, initial members). Hierarchical governance is typical — the subgroup's `internal_governance` covers internal ops; `parent_required_ops` lists ops requiring parent approval.
2. **The parent attests the relationship** via `system/group:add_subgroup`:
   ```
   add-subgroup-request := {
     parent_group_id: Acme_id,
     subgroup_id:     Engineering_id,
     relationship:    "department"       ; optional
   }
   ```

The subgroup entity is persisted at `system/group/{acme_id}/subgroups/{engineering_id}/{hash}`, K-of-N signed by Acme's quorum.

Implementations MAY combine these into a higher-level `form_subgroup` convenience, but the underlying ops are these two.

### 7.2 Graph hygiene (MUST)

`add_subgroup` MUST enforce:

- **Cycle prevention (MUST).** `add_subgroup(parent, candidate)` MUST reject if a transitive walk from `candidate` upward reaches `parent` (a cycle would result). Returns `403 governance_walk_cycle`.
- **Self-reference rejection (MUST).** `add_subgroup(A, A)` MUST be rejected.
- **Depth bound (SHOULD).** Bound subgroup-graph traversal at 32. Authority resolution exceeding the depth bound MUST fail closed.

The hierarchical-governance authority walk (§3.5) MUST also fail closed on cycle detection — a defense-in-depth for the case where cycle prevention misses something.

**Mutual subgroup-side consent is informative-only and deferred.** The parent unilaterally declares the subgroup; the subgroup's own quorum doesn't need to acknowledge. Making consent symmetric would require a new entity type (subgroup-acknowledgment) and a new dispatch flow. Document the asymmetry to subgroup operators out-of-band.

### 7.3 Subgroup identity stack

The subgroup has its own group entity at `system/group/{subgroup_id}` and its own identity stack at `system/group/{subgroup_id}/identity/...`. The subgroup entity at `system/group/{parent_id}/subgroups/...` is the *parent's attestation* that the subgroup exists in this hierarchy.

If the subgroup migrates to a new parent (organizational restructuring), the new parent issues a new subgroup attestation; the old parent removes theirs (`remove_subgroup`). The subgroup's own identity stack is unchanged — it's the relationship that's mutating, not the subgroup itself.

---

## 8. Lifecycle — form, dissolve, merge, split

The four lifecycle ops handle the group's beginning, end, and transitions. Dissolve / merge / split compose with identity-extension primitives (no new mechanism).

### 8.1 Dissolve

Quorum-signed (the group's quorum). Process:

1. Revoke all member entries (they're no longer authorized within the group).
2. **For each of the group's runtime peers**, invoke `EXTENSION-IDENTITY:revoke_peer(runtime_peer: <peer_id>, scope: 'all')`. This deletes attestations across all four publication modes and revokes local caps from the group's identity to that runtime peer. Contacts processing the revocations through standard sync invalidate cap-chains rooted at those runtime peers.
3. Archive (don't delete) the group entity, marking dissolved-at timestamp. Useful for historical record and audit.
4. Optionally hand off resources to a successor group or to individual members.

Dissolution is structural; it does not automatically destroy the group's data. Application policy decides what happens with shared resources.

**Verifier cache-miss dependency (cross-reference identity §10.1).** Dissolve's effect on outstanding caps depends on the verifying peer successfully resolving "the group's runtime peer's grant is no longer present" via the `is_revoked` chain-walk. If the verifier has the grant cached or sync is stalled, caps appear valid until cache invalidation. Verifier cache-miss policy (fetch-on-demand / reject-and-escalate / embedded-only) is deployment-configurable. Dissolve's revocation propagation is therefore bounded by the deployment's chosen cache-miss policy plus sync latency. **Tight dissolve windows MUST configure aggressive cache invalidation; loose deployments accept longer windows.**

### 8.2 Merge

Quorum-signed by BOTH source and target. Process:

1. Members from source are added to target (their roles may need translation if names differ; the optional `role_translation` field maps).
2. **Compose with `public-identity-rotation` (handoff form):** emit a `Public_source` → `Public_target` rotation entity signed by both quorums (or by source's quorum and acknowledged by target's). Contacts of the source process the rotation via the standard identity-attestation pipeline.
3. **For each of source's runtime peers**, invoke `EXTENSION-IDENTITY:revoke_peer(scope: 'all')` to invalidate caps issued by those runtime peers.
4. Source group is archived (`reason: "merged_into_{target}"`).
5. Subgroup memberships of source flow up to target.

The rotation + revocation composition is what makes merge actually mechanical: contacts of the source group learn of the handoff via the rotation entity through their normal sync pipeline; outstanding caps signed by source's runtime peers are invalidated via the revocation flow.

### 8.3 Split

Quorum-signed by source. The current scope is bounded: **form children + dissolve parent + manual cap re-issuance under children.**

1. Form one or more new groups (per `member_distributions`) — each child is a separate `system/group:form` call.
2. Distribute members per spec.
3. Distribute resources (application territory).
4. Dissolve the parent (per dissolve flow above, including `revoke_peer` for each parent runtime peer).
5. Caps that contacts held against the parent are invalidated by the dissolve revocations. Children must manually re-issue caps to contacts under their own runtime peers.

A 1→N rotation primitive that automatically redirects contacts from parent to appropriate child is **deferred** to a future amendment. Current deployments accept the manual-reissuance limitation.

### 8.4 Migration (out of scope for this version)

Migrating a group from one governance structure to another (e.g., founder K-of-N → admin set as the group grows) requires defining the new governance, updating the quorum entity, possibly re-issuing operator-delegations. Composes with `rotate_quorum` + `change_role`; the current spec doesn't normatively define `migrate_governance` — candidate for a future amendment.

---

## 9. Acting on behalf of — two patterns

When a member of a group sends a message that involves the group, two patterns apply.

### 9.1 Pattern 1 — Member acting independently with group context

The member's runtime peer signs the message (standard cap mechanics). The recipient's peer recognizes the member (their runtime peer is in their `Public_X`'s runtime-peer-attestations). The recipient's peer also sees a separate attestation that the member is a member of the group.

**Implementation.** Alice's outgoing message envelope includes (or references) the `system/group/{acme_id}/members/{Public_alice}` entry. The recipient's peer fetches Acme's group entity for context. Application UI shows "Alice from Acme."

**Use case:** casual; Alice mentions she's at Acme but she's writing as herself. Most everyday messaging.

The recipient's address book entry: `"Alice (member of Acme)"` — both Alice's individual identity and her group membership are visible.

### 9.2 Pattern 2 — Group acting via member as messenger

The member's runtime peer signs the message, but the message is "from the group" semantically. Alice is the technical signer; Acme is the semantic author.

This requires Acme's `Public_acme` to attest Alice's runtime peer as authorized to act on Acme's behalf. The cap chain root is Alice's runtime peer (the rule that root capabilities must be granted by the local peer is unchanged); the message envelope includes `acting_on_behalf_of: Public_acme` plus an `acting-on-behalf-of-attestation` entity authorizing Alice for this action.

**Implementation.** Needs an `acting_on_behalf_of` field in the message envelope plus the attestation entity. The receiving peer validates: the delegate is recognized (Alice is a member of Acme), the scope covers the action, the attestation isn't expired, and the attestation has not been revoked.

**Use case:** formal; the message is officially from the group. Press release; official announcement; contract; treasury operation. The recipient treats the message as the group's official statement.

The recipient's address book entry: `"Acme (via Alice)"` — Acme is the semantic author; Alice is just the messenger.

### 9.3 Lifecycle (open-ended-with-revocation)

Acting-on-behalf-of attestations have an **open-ended-with-revocation** lifecycle:
- An attestation is issued (optionally with `expires_at`).
- It is valid until either expiry or explicit revocation via `revoke_acting_on_behalf`.
- It covers any number of messages within its `scope`.

**Per-act mode (a fresh attestation per message, with `target_message_hash`) is rejected.** If a deployment needs per-act semantics — i.e., the group authorizing exactly one specific message — use a separate cap from the group's runtime peer covering that one message. Cleaner spec; less surface; aligns with the role extension's lifecycle model.

### 9.4 Concurrency

Multiple delegates MAY hold acting-on-behalf-of attestations with overlapping scopes; both attestations are independently valid. **Conflict resolution between contradictory statements (two delegates publishing within the same scope) is application-level, not protocol-level.** The protocol does not legislate "whose voice wins" when two delegates publish contradictory statements.

### 9.5 Operator recourse

`revoke_acting_on_behalf` removes matching attestations from the group's tree. Recipients processing subsequent messages from the delegate fail validation if the revoked attestation is referenced. This gives operators recourse against abuse without waiting for `expires_at` to roll over.

---

## 10. Federation — cross-group identity sharing

When two groups partner (Acme and Globex form a joint partnership group), the partnership group's runtime peers need to verify caps used by members of either parent. Verification requires the verifying peer to have the member's `runtime-peer-attestation` for the relevant `Public_X`.

### 10.1 Two patterns

**Per-member individual sharing (default; MUST).** Each member of the partnership individually shares their runtime-peer-attestations with `Public_partnership` via the per-relationship namespace (`relationships/{public_partnership_id}/runtime-peer-attestation/...`). Public_partnership's runtime peers sync these and can verify the relevant members' caps.

- Pros: privacy-preserving — each member chooses what to expose.
- Cons: scales linearly. 100 members → 100 individual configurations. Coordination overhead.

**Bulk attestation sync (convenience; SHOULD for federations of meaningful size).** A parent group (Acme) provides its members' attestations in bulk to the partnership. When a member is added to the partnership, an Acme-side mechanism automatically copies that member's runtime-peer-attestations into Public_partnership's tree.

- Pros: scales with group size, not membership churn.
- Cons: requires parent group to have a sync convention; trusts the parent's bulk-attestation flow; less granular per-member control.

### 10.2 Recommended default for federation deployments

1. Each member who joins the partnership opts in via a configuration choice ("share my runtime peers with this partnership").
2. Their parent group's infrastructure (Acme's) propagates their attestations to the partnership.
3. New members of the partnership are auto-handled by the same mechanism.
4. When a member leaves, their attestations are revoked (deleted from the partnership's tree).

This gives members opt-in control plus operational scaling for the federation.

### 10.3 Cascading vs. independent membership

When a member leaves a parent group, what happens to their membership in groups the parent participates in? Two patterns, selected per partnership-membership entry via the optional `derived_from_parent` field on `system/group/member`:

- **Independent (default; absent `derived_from_parent`).** Partnership membership is independent of parent membership — Carol stays in the partnership unless explicitly removed. Failing-safe default.
- **Cascade (opt-in; `derived_from_parent` set to parent group hash).** Partnership membership is gated on parent membership — Carol leaving Acme automatically removes her from any group whose `derived_from_parent` points at Acme.

The cascade walk runs in `remove_member` (§6.3): the handler walks `members` entries across all groups whose `derived_from_parent` points at this group AND whose `member` matches the removed member, and removes them in cascade. **Real federations use both patterns; the field is optional with independence as the default.**

### 10.4 Federation trust

In bulk-sync deployments, the partnership group trusts the parent groups' bulk-sync flows. If a parent's sync flow is compromised, attacker-controlled attestations could be propagated to the partnership.

Mitigations:
- Per-member individual sharing as the foundation (always supported, MUST).
- Bulk sync as opt-in convenience.
- **Partnership-side validation that bulk-synced attestations are signed by the expected `Public_X` keys.** The bulk-sync mechanism handles transport; signature validation MUST happen at the partnership's runtime peers.

---

## 11. Group infrastructure on a member's hardware

A common pattern in informal groups: the group's infrastructure runs on a member's individual hardware. Carol hosts `garden_calendar_server` on her home machine; Alice hosts `family_server` on her desktop. The member doesn't transfer ownership of their hardware to the group; they just *operate* the group's runtime peer on it.

### 11.1 Key separation

The group's runtime peer has its own keypair, signed for in `Public_group`'s `runtime-peer-attestation`. **That keypair is not the member's individual keypair.** Even though the member's machine holds the keypair material, the keypair acts on behalf of the group, not on behalf of the member.

### 11.2 Per-runtime-peer peer-config (MUST)

A host machine operating as multiple runtime peers (e.g., `carol_desktop` for `Public_carol` AND `garden_calendar_server` for `Public_garden`) MUST keep peer-configs strictly separate per the per-runtime-peer-identity rule from the identity extension.

- Each runtime peer has its own peer-config in its own namespace.
- Each trusts its own quorum.
- Each has its own `operator_grants`.
- Each participates in its own public identities.
- **Peer-configs do not share state across identities.**

Implementations MUST treat each runtime peer as a separately-configured identity participant. No cross-identity inheritance of `trusts_quorum`, `operator_grants`, or `public_identities` bindings.

### 11.3 Trust model

Members of the group trust the host to:
- Run the keypair on the group's behalf (sign things the group expects, not unauthorized ones).
- Keep the machine secure (so the keypair isn't extracted).
- Cooperate with migration if they leave the group or hand off hosting.

This is a real trust assumption, but no different from any "someone has to host it" scenario in distributed systems. The group can mitigate by:
- Replicating the runtime peer across multiple members' hardware (multiple runtime peers in `Public_group`'s set; the group survives any single host's compromise).
- Limiting what the host can do via cap scoping (the group's runtime peer holds caps with specific scopes, not blanket admin).
- Auditing: the group's tree records all operations the runtime peer signs.

### 11.4 Hosting transition (Carol → David)

1. The group's quorum (or whoever has add-runtime-peer authority) signs a new `runtime-peer-attestation` for `david_host_keypair`.
2. The new keypair takes over operations (members redirect EXECUTE targets, services migrate state).
3. Optionally, the group's quorum revokes the old `runtime-peer-attestation` for `carol_host_keypair` (via `revoke_peer`).

Standard runtime-peer rotation. No new mechanism.

### 11.5 Compromise scenario

Carol's machine is compromised; the group's keypair is exfiltrated. The group's quorum signs revocation of the compromised runtime-peer-attestation; members' verifiers reject signatures from the compromised keypair on next sync; the group provisions a new runtime peer on different hardware. Compromise window is sync-latency-bounded; **the group survives** (its identity is the quorum, not any one runtime peer).

### 11.6 Hostile-host scenario

Carol stops cooperating but won't relinquish hosting. The group's quorum signs revocation of `carol_host`'s runtime-peer-attestation (the quorum must reach threshold without Carol if Carol is the one being removed); the group provisions a new host; members redirect.

Carol's old machine still has the old keypair, but signatures from it are rejected by the group's verifiers (attestation revoked). **The architecture cannot prevent Carol from running her old keypair; it can only prevent the rest of the group from trusting it.** That's sufficient — the group continues without her.

---

## 12. Composition with other extensions

### 12.1 With identity (the structural reuse)

Per §2.1: group's identity stack is the identity extension, used directly under the group's namespace. All entity types, all publication modes, all rotation flows carry over.

The two-mechanisms invariant from the identity extension applies in full: identity-extension and group-extension attestations are validated **only at the entity level** by the K-of-N signature helper; cap-chain machinery MUST NOT process them.

When the group's runtime peer issues a cap to an external party (e.g., `partnership_server` issues a cap to an external contractor), the cap is signed by the group's runtime-peer keypair (the rule that root capabilities must be granted by the local peer is unchanged — root caps are signed by the local peer). The group's quorum and operator key never sign caps; they sign attestation entities at the entity level.

### 12.2 With role (group_id as role context)

A group is a natural role-context. See GUIDE-ROLE.md §11 for the role-side seam. Quick summary:

- `system/role/{group_id}/...` is where group-scoped roles live.
- Multi-role per peer per context is supported (a member can be both `manager` and `auditor`).
- Member entry records primary role; role assignments add additional roles.
- `remove_member` removes both member entry AND all role assignments scoped to the group.

### 12.3 With multi-sig (cap-level K-of-N joint authority)

The multi-sig primitive is optional. A group can use multi-sig caps where it needs **cap-level K-of-N for joint authority** (e.g., a partnership group's runtime peer issues a joint-authority cap to an external contractor where 2-of-3 founders must sign).

The cap is a multi-sig cap with `signers` populated from the group's quorum constituents. **The cap's `signers` field always contains concrete keypair hashes regardless of `signer_resolution` mode** (per §4.6). The multi-sig verifier validates against keypairs.

See GUIDE-MULTISIG.md for the cap-layer mechanics. The bridging between identity-resolved mode and multi-sig caps happens at issuance time — concrete keypairs are baked into the cap's `signers` field.

### 12.4 With registry (group-name resolution)

A group has a name (per §2.2's group entity field). Names appear in the registry layer:

- `acme.example` resolves to Acme's `Public_acme` peer ID + bootstrap_endpoints.
- `acme.example/engineering` resolves to Engineering's `Public_engineering` (subgroup).

The registry resolver returns the group's identity attestations and bootstrap_endpoints, just like for an individual user.

**Member-of-group addressing.** `alice@acme.example` resolves to Alice's identity + Acme membership context. The resolver returns Alice's `Public_alice` peer ID and bootstrap endpoints, plus an attestation that Alice is a member of Acme.

A receiving peer caches this. Future messages from Alice can be displayed as "Alice from Acme" (Pattern 1 of §9).

**Federation.** Cross-org addressing (`alice@globex.example` reaching `bob@acme.example`) works via the registry layer's federation hops. No new mechanism in the group extension.

---

## 13. What gets stored where (and who can write it)

| Path | Who writes | Mutation gates | Sync exposure |
|---|---|---|---|
| `system/group/{group_id}` | `system/group:form` | Bootstrap exemption (L0) | Per the deployment's sync policy |
| `system/group/{group_id}/identity/quorum/...` | Group quorum (entity-level K-of-N) via identity handler | Identity-extension rules | Internal sync only — never crosses fleet boundary |
| `system/group/{group_id}/identity/internal/...` | Group operator key (via identity handler) | Identity-extension rules | Internal sync only |
| `system/group/{group_id}/identity/public/...` | Public_group (via identity handler) | Identity-extension rules | All contacts (via two-tier sync and/or registry) |
| `system/group/{group_id}/identity/relationships/{contact_id}/...` | Public_group | Identity-extension rules | Only the named contact |
| `system/group/{group_id}/members/...` | `system/group:add_member` / `change_role` / `remove_member` | K-of-N per governance | Per deployment policy (privacy of group membership; see §13.2) |
| `system/group/{group_id}/subgroups/...` | `system/group:add_subgroup` / `remove_subgroup` | K-of-N parent quorum | Typically public |
| `system/group/{group_id}/acting-on-behalf-of/...` | `system/group:attest_acting_on_behalf` / `revoke_acting_on_behalf` | K-of-N quorum (per governance) | Embedded in messages or co-synced for active delegations |

**Application-level capability grants should target handler operations, not raw tree paths.** Per GUIDE-CAPABILITIES.md §2.2:

| Want to allow | Grant this | NOT this |
|---|---|---|
| Adding members | `system/group:add_member` | `system/tree:put` on `system/group/{group_id}/members/*` |
| Removing members | `system/group:remove_member` | (no good `tree:put` equivalent — group lifecycle composition needs handler) |
| Adding subgroups | `system/group:add_subgroup` | direct `tree:put` to subgroups subtree |
| Acting on behalf of | `system/group:attest_acting_on_behalf` | direct `tree:put` to acting-on-behalf-of subtree |
| Group lifecycle (dissolve / merge) | `system/group:dissolve` / `merge` | direct `tree:put` (would skip identity-primitive composition) |

### 13.1 Reserved path segments within `system/group/{group_id}/`

`identity`, `members`, `subgroups`, `acting-on-behalf-of` are reserved. Implementations MUST reject group-id values that would collide with these segments at content validation time.

### 13.2 Privacy of group membership

Should it be public who's in a group? Some yes (a public organization's employees); some no (a private support network).

Per the identity extension's per-peer attestation publication modes, member entries can be in different publication modes:
- **Public:** "Alice is publicly known to be a member of Acme." Lives in `system/group/{acme_id}/identity/public/members/...` (or just `members/`).
- **Internal-to-group:** "Alice's membership is visible within Acme but not externally." Lives in `system/group/{acme_id}/identity/internal/members/...`.
- **Per-relationship:** "Bob knows Alice is in Acme; Carol doesn't." Lives in `system/group/{acme_id}/identity/relationships/{contact_id}/members/...`.

Same publication-mode discipline as for runtime-peer-attestations. Deployments choose per-group what's public, internal, or per-relationship.

---

## 14. Pitfalls and antipatterns

### 14.1 Treating a group as a separate primitive from identity

**The group extension reuses identity types, not parallel definitions.** If you find yourself writing code that handles `system/group/quorum` and `system/identity/quorum` differently, you've drifted from the spec. The group's quorum lives at `system/group/{group_id}/identity/quorum/...` but it IS a `system/identity/quorum` entity; the same handler validates it.

### 14.2 Conflating group quorum with `system/cluster`

`cluster` is reserved for a separate planned extension (HA, replication, leader election among runtime peers). The group's quorum is NOT a cluster — it doesn't run, isn't online, doesn't process requests. It's a delegation root that occasionally re-emerges to sign new entities.

### 14.3 Granting raw `tree:put` to group namespaces

```
; ANTIPATTERN
{handlers: {include: ["system/tree"]},
 operations: {include: ["put"]},
 resources: {include: ["system/group/{group_id}/members/*"]}}
```

Allows the holder to fabricate member entries (and thereby bypass governance). The group handler validates K-of-N signatures, supersedes chains, lifecycle ordering. Raw `tree:put` doesn't.

```
; COHERENT
{handlers: {include: ["system/group"]},
 operations: {include: ["add_member", "remove_member", "change_role"]}}
```

Routes through the handler. Governance authority is enforced; lifecycle composition (e.g., `change_role` chaining into `rotate_quorum`) fires correctly.

### 14.4 Skipping the supersedes chain on quorum-update

When `change_role` chains into `rotate_quorum`, the resulting `quorum-update` MUST set `supersedes` to the previous `quorum-update`'s content_hash. **Don't skip it** — cross-peer attestation validation depends on a continuous chain; compromise-recovery rotations walk the chain back to a known-cached `public-quorum-attestation`.

A discontinuous chain breaks contact-side validation. The supersedes field is load-bearing.

### 14.5 Ignoring the federation cascade walk

`remove_member` MUST walk for `derived_from_parent` cascade entries and remove them. **Implementations that skip the walk leave stale memberships** — Carol leaves Acme; she's still a member of the Acme+Globex partnership; her caps in the partnership context still pass chain verification (she's still authorized at the partnership runtime peer level until *that* membership is removed).

If your deployment uses `derived_from_parent`, the cascade walk is the cleanup mechanism.

### 14.6 Pre-1.0 atomicity wording

Earlier prose used "per-assignee atomic" and "before either is persisted" wording that read literally as requiring a multi-entity transactional-bundle primitive. **No implementation provides this and the current spec doesn't require it.** The correctness intents are achievable with two cheap disciplines: validation atomicity (compose plan, validate end-to-end, fail fast with `409 chain_failed`) plus ordered writes (pin order so failure between legs leaves the tree in the safe direction).

If a future proposal asks for tree-level transactional bundling, that's a real architectural change and should be flagged. The current stack is intentionally without it.

### 14.7 Treating identity-resolved mode as universally better

Identity-resolved mode is more flexible (members rotate independently) but couples the group to members' identity-layer security. **In `concrete` mode, member compromise affects only the compromised member's keypair; in `identity-resolved` mode, member compromise can give the attacker the member's quorum slot.**

Frozen-cap horizon: outstanding multi-sig caps signed before a constituent's individual compromise inherit the constituent's compromise — frozen keypairs are forgeable by the attacker.

Mitigations (SHOULD): short-cap-TTL when identity-resolved + multi-sig is the pattern; subscribe to constituents' `runtime-peer-attestation` revocations as out-of-band signal.

Choose the mode for the deployment's threat model.

### 14.8 Overspecifying acting-on-behalf-of

Per-act mode (a fresh attestation per message with `target_message_hash`) is **rejected by the spec**. Open-ended-with-revocation is the only supported lifecycle.

If a deployment needs per-act semantics, use a separate cap from the group's runtime peer covering that one message — don't try to extend the acting-on-behalf-of attestation to per-act usage. That approach has been ruled out.

### 14.9 Catastrophic governance loss

If a group's founders all become unavailable (death, departure, etc.), the group's quorum drops below threshold. **The group can't be operated.** Mitigations are deployment policy:
- A "recovery quorum" outside the operational quorum.
- Hierarchical: parent group can rescue subgroup if subgroup's quorum fails.
- On-chain: governance-mandated successor.

For voluntary-participation groups (open-source projects, neighborhoods) — set thresholds proportional to expected attrition. Tightly-coupled groups (employer / employee) tolerate tight ratios (2-of-3, 3-of-5); loosely-coupled voluntary groups need looser ratios (2-of-5, 3-of-7) to tolerate drift. Consider the "stand-in" pattern — additional pre-promoted constituents whose authority activates on attestation that primary constituents are unreachable. Composes via existing role + caveat machinery; doesn't need new primitives.

### 14.10 Putting your personal Op in a group's quorum without thinking about exposure

If you list `alice_op` (Alice's personal day-to-day signing key from her individual identity stack) as a quorum constituent of a group in concrete mode, anyone who pulls the group's `public-quorum-attestation` learns that `alice_op` exists and is one of the group's authorities. This is a **metadata leak**, not a key compromise — the attacker can't sign as `alice_op` without the private key — but it breaks the identity model's "Op is internal to Alice's fleet" property and creates correlation surface (the attacker now knows one of Alice's keys to target).

For groups whose constituent identities aren't already publicly known (private partnerships, family groups, support networks, anything other than a public-facing organization), generate dedicated per-group signing keys instead — see §3.7 Option B. The dedicated key has only the one purpose; its exposure carries no metadata about your other activities, and rotating it is independent of your personal Op rotation.

Public-facing groups (employer organizations, public partnerships, well-known projects) where the founders' participation is itself public knowledge can reasonably accept the leak — see §3.7 Option A. The choice is a deployment decision, but it should be a deliberate one, not a default.

### 14.11 Forgetting the core-protocol-only fallback

Group is opt-in. **A peer that doesn't run the group extension can still receive cross-peer caps from group-extension peers.** Cap chains stay clean; group attestations are non-cap entities that group-unaware peers ignore.

If your application code requires every peer to "understand groups," you've broken the additivity property. Cross-peer interop with core-protocol-only peers must work — see `EXPLORATION-DEPLOYMENT-SHAPES.md §14.8` for the validation surface.

---

## 15. Worked examples

### 15.1 Forming Acme — founder K-of-N

Setup choice: dedicated per-group keys (Option B from §3.7). Each founder has generated `alice_acme_op` / `bob_acme_op` / `carol_acme_op` for Acme participation.

1. Alice (one of Acme's founders) initiates `system/group:form` via L0 on her runtime peer:
   ```
   form-request: {
     name: "Acme",
     initial_quorum: {
       signers:           [alice_acme_op, bob_acme_op, carol_acme_op],
       threshold:         2,
       signer_resolution: "concrete"
     },
     governance: {
       type:      "founder_k_of_n",
       founders:  [alice_acme_op, bob_acme_op, carol_acme_op],
       threshold: 2
     },
     initial_members: [
       {member: Public_alice, role: "founder"},
       {member: Public_bob,   role: "founder"},
       {member: Public_carol, role: "founder"}
     ]
   }
   ```
2. Handler creates the group entity, the group quorum, the operator-delegation (signed K=2 of [`alice_acme_op`, `bob_acme_op`, `carol_acme_op`]) authorizing `acme_op` as Acme's day-to-day operator, generates `Public_acme`, signs the `public-quorum-attestation` (which lists the three Acme keys — Alice's personal `alice_op` is not exposed by this), persists three founder member entries.
3. Form-result returns `{acme_id, quorum_id, public_acme_id}`.

### 15.2 Adding Dave as engineer

```
add-member-request: {
  group_id:        Acme_id,
  member_peer_id:  Public_dave,
  role:            "engineer"
}
```

Handler verifies governance: founder_k_of_n requires K=2 signatures. Alice's signature alone (using `alice_acme_op`) is insufficient; Bob co-signs (using `bob_acme_op`). Dave's member entry is persisted at `system/group/{acme_id}/members/{Public_dave}/{hash}`.

Caps for Dave are derived from his role: `system/role/{acme_id}/engineer` is the role definition; the role extension's `assign` op derives capability tokens for Dave per role-extension mechanics. See GUIDE-ROLE.md §14.1.

### 15.3 Acme issues a press release (Pattern 2 acting on behalf of)

Acme issues a press release. Alice drafts and sends; the message is officially "from Acme."

1. Acme's quorum (K=2 founders) signs an `acting-on-behalf-of-attestation` for Alice with scope "press releases":
   ```
   attest-acting-on-behalf-request: {
     group_id:   Acme_id,
     delegate:   Public_alice,
     scope:      [<grant-entry covering 'press releases' resources/operations>],
     expires_at: <optional ms>
   }
   ```
2. Alice's runtime peer drafts the message and signs it (standard cap mechanics).
3. The message envelope includes `acting_on_behalf_of: Public_acme` plus a reference to (or embedded copy of) the attestation.
4. The recipient's peer:
   - Verifies the message signature (Alice's runtime peer; recognized via Public_alice's runtime-peer-attestation).
   - Validates the attestation: signed by Public_acme (recognized via Public_acme's `public-identity-attestation` and registry resolution); scope covers the action; not expired; not revoked.
   - Recognizes the message as "from Acme (via Alice)."

### 15.4 Engineering as a subgroup of Acme

1. Alice (as Acme founder) initiates `system/group:form` for Engineering. Engineering's initial quorum is the engineering admins; governance is `admin_set`.
2. Engineering is created as its own group with its own identity stack (its own operator key `engineering_op`, its own `Public_engineering`).
3. Acme's quorum signs `system/group:add_subgroup`:
   ```
   add-subgroup-request: {
     parent_group_id: Acme_id,
     subgroup_id:     Engineering_id,
     relationship:    "department"
   }
   ```
4. The subgroup entity is persisted at `system/group/{acme_id}/subgroups/{engineering_id}/{hash}`.
5. Members can now be cross-group: Alice is a member of Acme AND Engineering.

If Engineering's `internal_governance` is `admin_set` and Engineering's `parent_required_ops` is `["dissolve", "merge", "rename"]`, then Engineering's admins manage internal Engineering matters but renaming/dissolving Engineering requires Acme's quorum (per §3.5 hierarchical).

### 15.5 Acme + Globex partnership (federation)

Acme and Globex form a joint partnership group:

1. The partnership group is formed with constituents drawn from both Acme's and Globex's quorums (e.g., 1-of-Acme-quorum + 1-of-Globex-quorum, threshold 2 — requiring sign-off from both sides).
2. Members from both parent groups join the partnership.
3. Each member runs an opt-in configuration to share their `runtime-peer-attestation` with `Public_partnership` via the per-relationship namespace.
4. (Optionally, per §10's SHOULD) Acme's and Globex's group infrastructure run bulk-sync flows that propagate opted-in members' attestations to the partnership.
5. When a Globex employee accesses `partnership_server`:
   - `partnership_server` validates the cap chain (root = `partnership_server`, locally rooted).
   - Checks that the EXECUTE signer is one of `Public_dave_at_globex`'s runtime peers per the cached attestation.

When a member leaves the partnership, the bulk-sync flow propagates the deletion; `partnership_server` updates its caches.

### 15.6 Open-source project — voluntary-participation group

A small open-source project. Three role tiers:
- `proposer` (anonymous-allow default).
- `contributor` (assigned by maintainers).
- `maintainer` (admin).

Governance: `admin_set` with `threshold_default: 1` (any maintainer can act for routine ops), `threshold_sensitive: 3` for `dissolve` / `rotate_quorum`.

**Quorum drift mitigation.** Maintainers may drift away (open-source attrition is common). With 7 maintainers and `threshold_sensitive: 3`, the project tolerates 4 maintainers going inactive before sensitive ops can't reach threshold. Consider the "stand-in" pattern: pre-promoted constituents whose authority activates on attestation of unreachability.

**Forking via group-creation.** If the project forks, the fork is a new group (`system/group:form`); the fork's members may include or exclude original maintainers; the fork has its own quorum, identity stack, and authority. The original group is unaffected.

### 15.7 Group hosted on a member's hardware — community garden

Carol hosts `garden_calendar_server` on her home machine. The setup:

- `Public_garden` is the garden group's external face.
- `garden_op` is the garden group's operator key.
- Carol's home machine runs as TWO runtime peers: `carol_desktop` (runtime peer of `Public_carol`) AND `garden_calendar_server` (runtime peer of `Public_garden`).
- Each runtime peer has its own peer-config (per §11.2 MUST).
- The garden's `runtime-peer-attestation` for `garden_calendar_server` is signed by Public_garden, authorized by `garden_op`.

If Carol's machine is compromised:
- Both `carol_desktop`'s keypair AND `garden_calendar_server`'s keypair are at risk.
- The garden's quorum signs revocation of the compromised runtime-peer-attestation (`revoke_peer(scope: 'all')`).
- The garden survives — its identity is the quorum, not any one runtime peer.
- Members redirect to a new host; Carol's identity is recovered separately via her own quorum.

If Carol stops cooperating (hostile-host scenario):
- The garden's quorum reaches threshold without Carol (assuming Carol's not the only constituent — voluntary groups should set thresholds accordingly).
- Quorum signs revocation of `garden_calendar_server`'s attestation.
- Carol's old keypair still exists but is no longer trusted.
- The garden provisions a new host on different hardware.

---

## 16. Reference

### 16.1 Entity types (group-specific)

| Type | Path | Signed by | Purpose |
|---|---|---|---|
| `system/group` | `system/group/{group_id}` | Not signed (structural) | Group entity |
| `system/group/member` | `system/group/{group_id}/members/{member_peer_id}/{hash}` | K-of-N group quorum | Member record |
| `system/group/subgroup` | `system/group/{parent_id}/subgroups/{subgroup_id}/{hash}` | K-of-N parent quorum | Parent's attestation of subgroup |
| `system/group/acting-on-behalf-of-attestation` | `system/group/{group_id}/acting-on-behalf-of/{delegate_id}/{hash}` | `Public_group` | Authorizes member to act as group |

Reused identity-extension types (per §2.1): `system/identity/quorum`, `system/identity/quorum-update`, `system/identity/operator-delegation`, `system/identity/public-identity-attestation`, `system/identity/runtime-peer-attestation`, `system/identity/public-quorum-attestation`, `system/identity/public-identity-rotation`, `system/identity/peer-config`. All under `system/group/{group_id}/identity/...`.

### 16.2 Handler operations (`system/group`)

| Operation | Authorized by | Async? | Purpose |
|---|---|---|---|
| `form` | Bootstrap exemption (L0); subsequent dispatched ops require operator key via local cap | No | Create new group; initial quorum + identity stack + initial members |
| `dissolve` | Quorum-signed | Per-runtime-peer revoke flows | Archive group; revoke runtime peers; trigger contact-side cap invalidation |
| `merge` | K-of-N source AND target quorums | Per-runtime-peer rotation + revoke | Merge source into target; compose with `public-identity-rotation` (handoff) |
| `split` | K-of-N source quorum | Per child group form + parent dissolve | Form children + dissolve parent; manual cap re-issuance |
| `add_member` | Per governance (founder K-of-N / admin K / etc.) | No | Add a member with a primary role |
| `remove_member` | Per governance | Yes if quorum constituent (chains into rotate_quorum) | Remove member; cascade role assignments + federation cascade |
| `change_role` | Per governance | Yes if affects quorum (admin_set / all_members) | Update member's primary role; chains into `rotate_quorum` (ordered writes) |
| `add_subgroup` | Parent K-of-N | No | Parent attests subgroup relationship |
| `remove_subgroup` | Parent K-of-N | No | Parent rescinds subgroup attestation |
| `attest_acting_on_behalf` | Per governance | Yes (signature gathering) | Issue acting-on-behalf-of attestation (Pattern 2) |
| `revoke_acting_on_behalf` | Per governance | No | Remove acting-on-behalf-of attestation |

### 16.3 Status codes

| Code | When |
|---|---|
| 200 | Operation succeeded |
| 400 | Malformed request |
| 403 | Per-governance authority insufficient |
| 403 `governance_walk_cycle` | Hierarchical authority walk detected cycle / exceeded depth bound |
| 409 `chain_failed` | `change_role` ↔ `rotate_quorum` chained operation failed validation or write |
| 401 (downstream verifying peers) | Cap chain root revoked due to dissolve / runtime-peer revocation |

### 16.4 The seam map

```
Core protocol capability system (chain verification) ← unchanged
   ↑ used by
EXTENSION-IDENTITY (four-layer model)                ← reused under group's namespace
   ↑ structurally embedded in
EXTENSION-GROUP (this guide)                         ← membership + lifecycle + governance
   ↑ composes with
EXTENSION-ROLE (in-group authority)                  ← group_id as role context
Multi-sig core primitive (cap-level K-of-N)          ← optional; for joint authority caps
Registry layer                                       ← group name resolution; federation hops
```

### 16.5 Spec section pointers (for the reader who wants to dig deeper)

**EXTENSION-GROUP.md:**
- §1.5 — Relationship to identity (structural reuse)
- §1.6 — Relationship to role (group_id as context)
- §2 — The group as identity (four-layer mapping)
- §3 — Type definitions; §3.6 reused identity-extension types
- §4 — Tree path conventions; §4.2 audience and sync; §4.3 reserved path segments
- §5 — Governance patterns; §5.6 signer reference style (concrete vs. identity-resolved)
- §6 — Group handler; §6.2 form; §6.3 member management with the three ordered-writes rules; §6.4 subgroup management with graph hygiene; §6.5 lifecycle composition; §6.6 acting-on-behalf-of
- §7 — Composition with other extensions
- §7.4 — Federation; cross-group identity sharing (per-member + bulk sync)
- §7.5 — Group infrastructure on a member's hardware
- §8 — Acting-on-behalf-of patterns
- §10 — Security considerations; §10.6 quorum drift in identity-resolved mode; §10.7 frozen-cap horizon

**Other:**
- ENTITY-CORE-PROTOCOL.md §5 — capability system; the rule that root capabilities must be granted by the local peer
- EXTENSION-IDENTITY.md §3.10 — `verify_k_of_n_signatures` (entity-level; distinct from cap-chain)
- EXTENSION-IDENTITY.md §5.7 — `rotate_quorum` (composed by `change_role` and `remove_member`)
- EXTENSION-IDENTITY.md §5.9 — `revoke_peer(scope: 'all')` (composed by `dissolve` and `merge`)
- EXTENSION-ROLE.md §4 — role handler (used for group-scoped role context)
- GUIDE-IDENTITY.md — the four-layer model in detail; rotation flows; pitfalls
- GUIDE-ROLE.md §11 — role-side composition with group
- GUIDE-MULTISIG.md — cap-level K-of-N (composes for joint authority caps)
- GUIDE-CAPABILITIES.md — capability foundation; coherent capability principle

---

## 17. Document history

- **v0.4:** Operator-delegation reachability for identity-resolved mode is now formalized in the specs themselves rather than only flagged as an open question in the guide. **Spec changes:** `EXTENSION-GROUP.md §5.6` gains an "Operational requirement for identity-resolved mode" subsection naming three reachability mechanisms (shared infrastructure — formalized; per-relationship publication of operator-delegation — convention only in v1.1; embedded resolution proof — convention only in v1.1) plus a recommended-scope paragraph telling deployments that need cross-fleet identity-resolved validation to prefer dedicated per-group keys in concrete mode until a mechanism is formalized. `EXTENSION-IDENTITY.md §3.3` gains an "Audience and publication mode" note documenting that `operator-delegation` is internal-mode only by default and cross-references the group-extension constraint. **Meta-tracker change:** `PROPOSAL-IDENTITY-INFRASTRUCTURE.md §13.4` tracks the constraint as a known follow-up with a future-amendment trigger (formalize per-relationship publication of operator-delegation if production deployments need cross-fleet identity-resolved validation). **Guide changes (this doc):** §3.7 Option C and §4.2 updated to reference the spec text (not describe the issue as open) and point readers to the spec sections + meta-tracker for the future-amendment context.

- **v0.3:** Documented the "what key goes in `signers`" question — orthogonal to both governance pattern (§3) and resolution mode (§4). New §3.7 "Choosing the signer keys" enumerates three options (personal Op as constituent / dedicated per-group key / identity-resolved Public_X reference) with privacy implications spelled out. New caveat in §4.2 flagging the operator-delegation reachability question for identity-resolved mode (operator-delegations are internal-mode by default; identity-resolved verification needs them reachable, which the spec doesn't currently formalize). §5.3 form-request example switched to the recommended-default notation (`alice_acme_op` etc.) with prose explaining how to flip to Option A or C. §5.4 walkthrough rewritten to make the choice explicit and trace the consequences. §15.1 worked example aligned with the new framing. New pitfall §14.10 calling out accidental personal-Op exposure. Default recommendation: dedicated per-group keys (Option B) for most deployments; personal Op (Option A) for genuinely public groups; identity-resolved (Option C) for deployments that have validated operator-delegation reachability. The architecture supports all three; the documentation now makes the choice deliberate rather than implicit.

- **v0.2:** Cleanup pass for new-reader clarity. Removed proposal-tracking acronyms (IA15/IA16/IA17/IA18/IA19/IA20/IA21, G3.7, G12.4, M6, "per Amendment 8") from body prose in favor of descriptive language. Replaced "V7" with "core protocol" or "the protocol" throughout (the spec file is `ENTITY-CORE-PROTOCOL.md`; "V7" was its version, not a name new readers would recognize). Added an upfront "A note on naming used in this guide" section explaining the operator-key terminology — since group examples freely use `alice_op`, `bob_op`, `acme_op`, etc., new readers need to know upfront that these are operator keys (the day-to-day signing keys the identity extension calls "Op"), and that the *group's* operator key (`acme_op`) is distinct from any *member's* operator key (`alice_op`). Clarified throughout §3 and §5 that the founders' individual operator keys are the quorum constituents; the group's operator key is delegated by that quorum after `form` runs (it is NOT a quorum constituent). Dropped pedantic line-number references. No content changes; same sections, same examples, same mechanics — just legible to a reader who hasn't tracked every proposal.

- **v0.1:** Initial draft. Walks the model (group as type of identity; structural reuse of identity types under group's namespace), the four-layer mapping (quorum/operator/Public_group/runtime peers), governance patterns (founder K-of-N, admin set, all-members, on-chain, hierarchical) with use-case mapping, signer reference style (concrete vs. identity-resolved with multi-operator constituents; multi-sig composition without core dependency), forming a group with bootstrap exemption, membership management with `change_role` ↔ `rotate_quorum` ordered writes, subgroup graph hygiene (cycle prevention + depth bound + hierarchical authority-walk fail-closed), lifecycle composition (dissolve as `revoke_peer` per runtime peer; merge as `public-identity-rotation` handoff + revoke; split as form-children + dissolve-parent), acting-on-behalf-of patterns with open-ended-with-revocation lifecycle (per-act mode rejected), federation patterns with per-member + bulk attestation sync and cascading vs independent membership, group infrastructure on member's hardware with per-runtime-peer-identity peer-config MUST, composition seams (identity / role / multi-sig / registry), per-path write authority, pitfalls, worked examples lifted from validation surfaces. Companion to GUIDE-IDENTITY.md, GUIDE-ROLE.md, GUIDE-CAPABILITIES.md, GUIDE-MULTISIG.md.
