# Group Extension — Normative Specification

**Version**: 1.4
**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.35+); EXTENSION-IDENTITY.md (v3.0+) — group reuses identity's three primary entity types (`quorum`, `peer-config`, `attestation`) and six attestation kinds under the group's namespace (one `cert` kind with `function`-discriminated standard + app-defined values; cert lifecycle events; quorum events); EXTENSION-ROLE.md (v1.5+) — group composes with role for in-group authority via R6 multi-role and R10 reserved names
**Optional**: PROPOSAL-MULTISIG-CORE-PRIMITIVE.md — used for cap-level K-of-N joint authority where deployments need it; no compile-time dependency
**Related**: PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE.md — registry layer used for group-name resolution and federation hops; EXTENSION-NETWORK.md — sync and bootstrap conventions
**Synthesis**: EXPLORATION-DEPLOYMENT-SHAPES.md (validated the use cases this extension serves)
**Encoding**: ENTITY-CBOR-ENCODING.md (ECF)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/group/{group_id}` means `/{local_peer_id}/system/group/{group_id}`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

> **Terminology.** This extension uses **group** for an identity that represents multiple individuals collectively. The word **cluster** is reserved for `system/cluster` (a separate planned extension for runtime-peer coordination — HA, replication, leader election). Groups are an identity-level concept; clusters are an infrastructure-level concept. They are different.

## 1. Overview

A **group** is an identity that represents multiple individuals collectively. The group has its own identity stack — a peer graph of quorum, controller's key, agents, and (optionally in the multi-key advanced shape) a identifier key — using the unified-attestation key-graph model defined by `EXTENSION-IDENTITY.md` v2.0. The group extension layers membership management, lifecycle operations, subgroups, acting-on-behalf-of patterns, and composition with the role and registry extensions on top.

The group extension is **additive**. It composes with what already exists in the in-flight stack:
- Identity provides the structural primitive (peer graph: quorum + controller's key + agents, with optional identifier for hygiene rotation).
- Role provides per-group authority management (member-internal grants).
- Multi-sig provides cap-level K-of-N for joint authority when needed.
- Registry provides group-name resolution.

This extension does NOT introduce:
- New core protocol mechanisms.
- A new identity primitive — uses the identity extension's three primary entity types (`quorum`, `peer-config`, `attestation`) and six attestation kinds directly.
- A new capability mechanism — uses V7 caps + multi-sig as needed.
- A new K-of-N validation — uses the identity extension's `verify_k_of_n_signatures` helper (`EXTENSION-IDENTITY.md` v3.0 §3.5) and `verify_attestation` dispatcher (§3.6) for entity-level group decisions.

### 1.1 Scope

This extension defines:

- **Group entity** — the structural definition of a group, with `name`, `governance`, and a reference to the group's published handle (the identifier key in the four-key shape; the controller's key in the three-key default).
- **Membership entities** — `system/group/member` (peer X is a member of group Y, with role) and `system/group/subgroup` (group X is a subgroup of group Y).
- **Acting-on-behalf-of attestation** — the entity used when a group member acts as the group's official voice (Pattern 2 of §8).
- **Group handler** — `system/group` with operations: `form`, `dissolve`, `merge`, `split`, `add_member`, `remove_member`, `change_role`, `add_subgroup`, `remove_subgroup`, `attest_acting_on_behalf`.
- **Tree path conventions** — under `system/group/{group_id}/...` with the identity stack (quorum + attestations + peer-config) mirrored at `system/group/{group_id}/identity/...`.
- **Governance patterns** — five well-trodden patterns (founder K-of-N / admin set / all-members / on-chain / hierarchical).
- **Signer reference style** — concrete-peer-ID (default) and identity-resolved (advanced) modes for group quorums.
- **Composition seams** — explicit cross-extension contracts with identity, role, multi-sig, and registry.

This extension does **not** define:

- New identity-stack primitives — `EXTENSION-IDENTITY.md` does this.
- New cap-chain mechanics — V7 + multi-sig primitive do this.
- Specific governance implementations beyond the recognized patterns.
- Federation conventions — those fall out of group composition + registry; documented as patterns here, but the wire-level federation hops are the registry layer's concern.
- Multi-user shared identities at the *individual* level — `EXTENSION-IDENTITY.md` is single-user-per-identity by design; group is the multi-user surface.

### 1.2 Design Principles

- **A group is a type of identity.** It has the same peer-graph structure as an individual identity: a quorum at the structural root; one or more controllers certified by the quorum; agents attested by a controller (or by the identifier in the multi-key advanced shape). What differs is that a group's quorum constituents are drawn from members or admins rather than personal backup keys; what's the same is the unified `attestation` entity type with all six kinds (`cert` with `function: controller | agent | identifier | <app-defined>`; `cert-rotation-handoff`; `cert-rotation-recovery`; `cert-retirement`; `quorum-update`; `quorum-publish`) — same types from `EXTENSION-IDENTITY.md` v3.0, just with the group's quorum doing the K-of-N signing.
- **Reuse rather than duplicate.** Group reuses identity's three primary entity types (`quorum`, `peer-config`, `attestation`) under the group's namespace. There are no parallel `system/group/quorum`, `system/group/attestation`, etc. types — the identity-extension types apply directly, scoped by path.
- **Membership is the new layer.** What the group extension adds beyond identity is membership management (member entries, subgroup entries) and lifecycle (form, dissolve, merge, split). The structural identity-stack mechanics are inherited.
- **Governance is presentational, not assumed.** The four basic patterns (founder K-of-N, admin set, all-members, on-chain) are recognized; deployments can compose them and add their own variants. The group extension dispatches signature-checking by governance type.
- **Composition over new primitives.** Where the role extension or registry already does something, the group extension reuses rather than reinvents. For instance, group-scoped role assignments use role v1.3's existing `system/role/{group_id}/...` namespace; group names use the registry layer's pluggable resolver.

### 1.3 Coherent Capability

`system/group` operations are the proper create/update path for group entities, member entries, subgroup references, and acting-on-behalf-of-attestations. Direct `tree:put` to `system/group/{group_id}/members/*`, `/subgroups/*`, etc. is permitted (V7 kernel exposes `tree:put` by design) but bypasses the group handler's validation. Application-level capability grants SHOULD cover `system/group:add_member`, `:remove_member`, `:form`, etc. rather than raw `tree:put` to group-managed namespaces.

This follows the Coherent Capability principle from PROPOSAL-COHERENT-CAPABILITY-AUTHORITY.md §2 (R0).

### 1.4 Relationship to V7 and Multi-Sig Primitive

```
V7 (cap chains)  +  Multi-sig primitive (K-of-N root caps)
                          ↑
                      used by
                          ↓
              EXTENSION-IDENTITY (key-graph + attestation kinds)
                          ↑
                      reused by
                          ↓
              EXTENSION-GROUP (this spec)
                          ↑
                      consumed by
                          ↓
              Application code (group-aware UIs, etc.)
```

The group extension is at the top of the identity stack. It depends on identity for the structural model, and on role for in-group authority management. Multi-sig is optional — used when a deployment needs cap-level K-of-N for joint authority (e.g., a partnership group's agent issues a joint-authority cap to an external contractor); the group extension does not require multi-sig to be installed for membership and lifecycle operations.

### 1.5 Relationship to the Identity Extension

A group **is** an identity (mechanically). Its quorum entity, attestation entities (across all six kinds: `cert` with controller/agent/identifier/app-defined functions; `cert-rotation-handoff`; `cert-rotation-recovery`; `cert-retirement`; `quorum-update`; `quorum-publish`), and peer-config entity are the same types defined in `EXTENSION-IDENTITY.md` v3.0, scoped under `system/group/{group_id}/identity/...`. All publication modes for `cert(function=agent)` attestations (internal / public / per-relationship / embedded), all rotation flows, all per-agent peer-config mechanics carry over. Group-specific functions (e.g., `function="moderator"`, `function="treasurer"`, `function="federation-bridge"`) compose via the v3.0 app-defined-function mechanism without needing new attestation kinds.

What the group extension layers on:
- **Membership** — who is in the group; their role; how they join and leave.
- **Subgroups** — nested groups for organizations, federations.
- **Lifecycle** — form, dissolve, merge, split.
- **Governance dispatching** — the group handler reads `governance` and dispatches signature-checking accordingly.
- **Acting-on-behalf-of** — when a member sends a message "from the group."

What it explicitly does not change:
- The identity-extension's key-graph model and unified attestation entity.
- The path tier semantics (quorum / internal / public / relationships are about external sync exposure, not which peers hold what; per `EXTENSION-IDENTITY.md` v3.0 §5.2).
- The two-mechanisms invariant (cap chains and identity attestations stay separate; per `EXTENSION-IDENTITY.md` v3.0 §3.6 / §12.3).
- The verifier cache-miss policy for agent certs.

### 1.6 Relationship to the Role Extension

A group is a natural role-context. `system/role/{group_id}/...` is where group-scoped role definitions, assignments, and exclusions live. Role v1.3 R6 (multi-role per peer, context) means a member can have multiple roles within a group (admin + finance, manager + IC, etc.). Role v1.3 R10 reserves `assignment` and `excluded` as path-segment names; the group extension does not introduce additional reserved names beyond those.

Member entries (`system/group/{group_id}/members/...`) record the member's PRIMARY role and the membership record itself. Role assignments (`system/role/{group_id}/assignment/...`) record additional roles within the group. Removing a member removes both the member entry AND all role assignments scoped to that group's context. See §7.2.

---

## 2. The Group as Identity

A group's identity stack mirrors `EXTENSION-IDENTITY.md` v2.0 directly. The key-graph layers map:

| Layer | Individual identity (`EXTENSION-IDENTITY` v3.0) | Group identity (this extension) |
|---|---|---|
| **Quorum** | User's backup keys (paper, second device) | Constituents drawn from members/admins (founder controller's keys, admin controller's keys, or all members) |
| **Operational key** | User's day-to-day controller's key | Group's day-to-day controller's key (one per group; multiple in concurrent multi-controller deployments) |
| **Contact-face key** (multi-key advanced; optional) | A separately-attested key contacts trust as the user's handle | A separately-attested key contacts trust as the group's handle (e.g., `K_group_face`) |
| **Runtime peers** | The user's devices and servers | The machines hosting the group (forum_server, partnership_server, etc.) |

What the group quorum changes vs. an individual quorum:
- Constituents are policy-driven (per the group's `governance` field), not personal.
- Constituents can be statically fixed (founder K-of-N) or dynamically derived (admin_set: "members with role admin").
- Multiple groups can run on the same agent (per `EXTENSION-IDENTITY.md` v3.0 §3.2 per-runtime-peer-identity peer-config rule); a host machine may operate as multiple agents, one per identity it serves.

Most of the structural mechanics inherited from identity (referencing the unified `attestation` entity and its kinds):
- `cert(function=controller)` attestations (signed K-of-N by quorum; install controller authority).
- `quorum-publish` attestations (contact-visible record of constituents + threshold; previous quorum signs supersedes — per `EXTENSION-IDENTITY.md` v3.0 §4.6).
- `cert-rotation-handoff` and `cert-rotation-recovery` attestations (handle rotation patterns).
- Per-peer `cert(function=agent)` attestations (multi-modal: internal / public / per-relationship / embedded).
- Bootstrap exemption (the group's `form` op, like identity's `configure`, runs via L0 direct-store).

---

## 3. Type Definitions

### 3.1 Group Entity

```
system/group := {
  fields: {
    name:               {type_ref: "primitive/string"}     ; human-readable label
    published_handle:   {type_ref: "system/hash"}          ; the group's contact-side handle:
                                                            ; the group's controller's key in
                                                            ; three-key default; the group's
                                                            ; identifier key in multi-key
                                                            ; advanced (per EXTENSION-IDENTITY
                                                            ; v3.0 §1.1 / §11.4)
    governance:         {type_ref: "system/group/governance-spec"}
    created_at:         {type_ref: "primitive/uint"}       ; ms since Unix epoch
    created_by:         {type_ref: "system/hash"}          ; identity hash of the
                                                            ; peer that initiated `form`
    metadata:           {type_ref: "primitive/any", optional: true}
  }
}
```

Stored at `system/group/{group_id}` (where `group_id` is the entity's content_hash; see §4).

The group entity is structural and is not itself signed. Authority for the group flows from operations on the group's quorum (defined per the identity extension), governed by the `governance` field.

### 3.2 Governance Spec

```
system/group/governance-spec := {
  ; one of:
  oneof: [
    {type_ref: "system/group/governance-founder-k-of-n"},
    {type_ref: "system/group/governance-admin-set"},
    {type_ref: "system/group/governance-all-members"},
    {type_ref: "system/group/governance-on-chain"},
    {type_ref: "system/group/governance-hierarchical"}
  ]
}
```

See §5 for the per-pattern field shapes.

### 3.3 Member Entity

```
system/group/member := {
  fields: {
    group:              {type_ref: "system/hash"}          ; group entity hash
    member:             {type_ref: "system/hash"}          ; member's published handle
    role:               {type_ref: "primitive/string"}     ; primary role within group:
                                                            ; admin, member, child, guest, etc.
    joined_at:          {type_ref: "primitive/uint"}       ; ms since Unix epoch
    not_before:         {type_ref: "primitive/uint", optional: true}
    expires_at:         {type_ref: "primitive/uint", optional: true}
    derived_from_parent: {type_ref: "system/hash", optional: true}
                                                            ; parent group whose membership gates
                                                            ; this one (federation cascading; per
                                                            ; IA21 / §7.4). Absent (default) =
                                                            ; independent membership; present =
                                                            ; cascade-on-parent-removal semantics.
    metadata:           {type_ref: "primitive/any", optional: true}
  }
}
```

Stored at `system/group/{group_id}/members/{member_peer_id}/{hash}`.

K-of-N signed by the group's quorum (per governance — admins or founders, depending on pattern).

The `role` field is the member's PRIMARY role within the group. Additional roles are recorded as role-extension assignments (per §7.2). Removing a member entry removes both the member entry AND all role assignments scoped to that group's context.

The optional `derived_from_parent` field supports federation cascading membership: when a member is removed from the parent group referenced here, the group handler walks `members` entries pointing at that parent and removes them in cascade. Absent `derived_from_parent` = independent membership (default; safer semantics — failing-safe). See §7.4 for the federation patterns and §6.3 for the cascade walk.

### 3.4 Subgroup Entity

```
system/group/subgroup := {
  fields: {
    parent_group:       {type_ref: "system/hash"}          ; parent group
    subgroup:           {type_ref: "system/hash"}          ; child group
    relationship:       {type_ref: "primitive/string", optional: true}
                                                            ; e.g., "department", "team",
                                                            ; "regional_office"
    created_at:         {type_ref: "primitive/uint"}       ; ms since Unix epoch
    metadata:           {type_ref: "primitive/any", optional: true}
  }
}
```

Stored at `system/group/{parent_group_id}/subgroups/{subgroup_id}/{hash}`.

K-of-N signed by parent group's quorum.

The subgroup itself has its own group entity at `system/group/{subgroup_id}` and its own identity stack at `system/group/{subgroup_id}/identity/...`. The subgroup entity at `system/group/{parent_id}/subgroups/...` is the parent's attestation that the subgroup exists in this hierarchy.

**Graph hygiene (per IA16).** The subgroup graph MUST be acyclic. Cycle prevention, self-reference rejection, and depth bounds are normative on `add_subgroup` (§6.4); the hierarchical-governance authority walk (§5.5) MUST fail closed if a cycle is detected. Mutual subgroup-side consent is informative-only and deferred — v1.0 deployments document the parent-unilateral asymmetry to subgroup operators out-of-band.

### 3.5 Acting-on-Behalf-of Attestation

```
system/group/acting-on-behalf-of-attestation := {
  fields: {
    group:              {type_ref: "system/hash"}          ; the group acting
    delegate:           {type_ref: "system/hash"}          ; the member acting on its behalf
    scope:              {array_of: {type_ref: "system/capability/grant"}}
                                                            ; what this delegate is authorized
                                                            ; to do for the group; reuses V7's
                                                            ; existing grant shape so cross-impl
                                                            ; interop is mechanical (per IA20)
    not_before:         {type_ref: "primitive/uint", optional: true}
    expires_at:         {type_ref: "primitive/uint", optional: true}
  }
}
```

Stored at `system/group/{group_id}/acting-on-behalf-of/{delegate_id}/{hash}`.

Signed by the group's published handle — the group's controller's key in three-key default; the group's identifier key in multi-key advanced (per `EXTENSION-IDENTITY.md` v3.0 §4.5 / §11.4). Single-sig (the published handle authorizes the delegation; the quorum is upstream of the published handle and does not sign these directly).

**Why this is a group-extension entity, not a kind on `system/identity/attestation`.** v2's unified `system/identity/attestation` covers structural-cryptographic key-graph edges — `cert(function=controller)`, `device`, `identifier`, `quorum-publish`, `rotation-*`, `retirement` — each saying "this key holds role X in the identity." `acting-on-behalf-of` is conceptually different: it is a **scoped delegation** ("this delegate may speak with the group's voice within these constraints"), carrying a `scope: array of system/capability/grant` field that no structural attestation kind has. It is also entity-level validated (not chain-walked by `verify_capability_chain`), so it is not a V7 capability either. It is its own thing — an attestation entity in the GROUP namespace, distinct from both identity attestations and V7 caps. Recipients validate it at the entity level by checking the signature is from a currently-authoritative published handle of the group, the delegate is a recognized member, the scope covers the action, and the attestation has not been superseded or revoked or expired.

Used in Pattern 2 of acting-on-behalf-of (§8.2): when a member sends a message "from the group" with the group as the semantic author. Recipients validate the attestation: the delegate is recognized, the scope covers the action, the attestation isn't expired.

### 3.6 Reused Identity-Extension Types

The group's identity-stack entities are the same three primary types defined in `EXTENSION-IDENTITY.md` v2.0, used directly under the group's namespace. There are no parallel definitions in this extension.

| Type | Group namespace path (kind, where applicable) | Defined in |
|---|---|---|
| `system/identity/quorum` | `system/group/{group_id}/identity/quorum/{quorum_id}` | `EXTENSION-IDENTITY.md` v3.0 §3.1 |
| `system/identity/attestation` (`kind=quorum-update`) | `system/group/{group_id}/identity/quorum/{quorum_id}/cert/{hash}` | v2.0 §4.3 |
| `system/identity/attestation` (`kind=cert, function=controller`) | `system/group/{group_id}/identity/quorum/{quorum_id}/cert/{hash}` | v2.0 §4.2 |
| `system/identity/attestation` (`kind=retirement`) | `system/group/{group_id}/identity/quorum/{quorum_id}/cert/{hash}` | v2.0 §4.9 |
| `system/identity/attestation` (`kind=identifier`) | `system/group/{group_id}/identity/internal/cert/{hash}` | v2.0 §4.5 |
| `system/identity/attestation` (`kind=cert, function=agent`) | `system/group/{group_id}/identity/{mode-path}/cert/{hash}` (mode-path varies per publication mode) | v2.0 §4.4 |
| `system/identity/attestation` (`kind=quorum-publish`) | `system/group/{group_id}/identity/public/cert/{hash}` | v2.0 §4.6 |
| `system/identity/attestation` (`kind=cert-rotation-handoff` / `kind=cert-rotation-recovery`) | `system/group/{group_id}/identity/public/cert/{hash}` | v2.0 §4.7, §4.8 |
| `system/identity/peer-config` | per-runtime-peer local; one per identity served | v2.0 §3.2 |

All attestation entities share the same type (`system/identity/attestation`); the `kind` field discriminates structural meaning. Storage path is determined by kind (and, for `device`, by publication mode).

The `signer_resolution` field on `system/identity/quorum` (concrete vs. identity-resolved; per `EXTENSION-IDENTITY.md` v3.0 §3.1) is what controls the group-quorum dispatch behavior described in §5.6 of this spec. Group is one of the two consumers of that field (the other being individual-identity quorums).

### 3.7 K-of-N Validation Helper (reused)

The group extension uses `verify_k_of_n_signatures` (`EXTENSION-IDENTITY.md` v3.0 §3.5) and the kind-dispatched `verify_attestation` wrapper (§3.6) for all group-quorum entity-level signature validation across all attestation kinds (cert with controller/agent/identifier/app-defined functions, quorum-update, quorum-publish, cert-retirement, etc.) plus member entries and subgroup entries. The group extension does NOT introduce a parallel helper.

The two-mechanisms operational invariant from `EXTENSION-IDENTITY.md` v3.0 §3.6 / §12.3 applies in full: identity-extension and group-extension attestations are validated only at the entity level by `verify_attestation` / `verify_k_of_n_signatures`; cap-chain machinery (`verify_capability_chain`) MUST NOT process them.

---

## 4. Tree Path Conventions

### 4.1 Path Layout

```
system/group/
   {group_id}                                              ← group entity (§3.1)
   {group_id}/identity/                                    ← group's identity stack
       quorum/{quorum_id}                                  ← group quorum entity
       quorum/{quorum_id}/cert/{hash}               ← K-of-N attestations under this quorum
                                                             (kind = cert (function=controller),
                                                              quorum-update,
                                                              retirement)
       internal/
           attestation/{hash}                              ← identifier attestations (group op
                                                             → group identifier) + device
                                                             attestations (mode = internal)
       public/
           attestation/{hash}                              ← quorum-publish attestations
                                                             + cert-rotation-handoff / cert-rotation-recovery
                                                             attestations + agent certs
                                                             (mode = public)
       relationships/
           {contact_id}/cert/{hash}                 ← agent certs
                                                             (mode = per-relationship)
       peer-config                                          ← per-runtime-peer local config
   {group_id}/members/                                     ← member entries (§3.3)
       {member_peer_id}/
           {hash}                                          ← latest member state
   {group_id}/subgroups/                                   ← subgroup references (§3.4)
       {subgroup_id}/{hash}
   {group_id}/acting-on-behalf-of/                         ← G9 attestations (§3.5)
       {delegate_peer_id}/{hash}
```

The `system/group/{group_id}/identity/...` subtree mirrors `EXTENSION-IDENTITY.md §4.1` exactly, scoped under the group's namespace.

### 4.2 Audience and Sync

**Tier separation describes external sync exposure, not which peers hold what.** All of the group's agents (forum_server, partnership_server, host machines, etc.) hold the FULL group subtree. The `quorum/` / `internal/` / `public/` / `relationships/` split describes what the group's outward-facing infrastructure exposes externally. This is the same clarification as `EXTENSION-IDENTITY.md §4.2`, applied to the group's namespace.

| Path subtree | Held by | Visible externally to | External propagation mechanism |
|---|---|---|---|
| `system/group/{group_id}/identity/quorum/...` | All of the group's agents | None (members may sync if they need quorum visibility) | Internal sync only |
| `system/group/{group_id}/identity/internal/...` | All of the group's agents | None | Internal sync only |
| `system/group/{group_id}/identity/public/...` | All of the group's agents | All contacts (via two-tier sync and/or registry resolution) | Standard sync-scope cap covering `public/`, optionally surfaced via registry |
| `system/group/{group_id}/identity/relationships/{contact_id}/...` | All of the group's agents | Only the named contact | Three-tier sync — contact-specific sync-scope cap |
| `system/group/{group_id}/members/...` | All of the group's agents (full membership visible internally) | Per the group's deployment policy (see §10.4 privacy of group membership) | Same publication-mode discipline as identity extension's per-peer attestations |
| `system/group/{group_id}/subgroups/...` | All of the group's agents | Typically public (subgroup hierarchies are usually structural and visible) | Public sync; deployments may opt to make per-relationship |
| `system/group/{group_id}/acting-on-behalf-of/...` | All of the group's agents | Recipients of messages where the attestation is referenced | Embedded in messages or co-synced for active delegations |

Members of a group typically sync the group's tree under appropriate sync-scope caps. Contacts (entities outside the group) typically only sync `public/` and any `relationships/{their_id}/` subtrees the group has shared.

### 4.3 Reserved Path Segments

Within `system/group/{group_id}/`, the path segments `identity`, `members`, `subgroups`, and `acting-on-behalf-of` are reserved for the group extension. Implementations MUST reject group-id values that would collide with these segments at content validation time.

---

## 5. Governance Patterns

The group's quorum constituents and threshold are determined by the group's `governance` field. Five patterns are recognized.

### 5.1 Founder K-of-N

Constituents: a fixed set of founder controller's keys.

```
governance := {
  type:        "founder_k_of_n",
  founders:    [K_alice_op, K_bob_op, K_carol_op],
  threshold:   2
}
```

- Operations on the group (member additions, role changes, infrastructure changes) require K-of-N founders to sign.
- Founders rarely change; most operations go through the group's controller's key (certified by the founder quorum).
- Suitable for small businesses, families, partnerships.

### 5.2 Admin Set

Constituents: members with the "admin" role.

```
governance := {
  type:                  "admin_set",
  threshold_default:     1,
  threshold_sensitive:   3,
  sensitive_ops:         ["dissolve", "quorum_update", "merge"]
}
```

- The admin set is dynamic — admins are added or removed via `change_role` operations.
- Operations require K-of-N admins (per the operation's threshold).
- Threshold can vary per operation type (low for routine, high for sensitive).
- The group's quorum entity has `signers` derived from "members with role admin"; the admin set change triggers the unified rotation flow (`change_role` chains into a `kind="quorum-update"` attestation plus a fresh `kind="quorum-publish"` attestation per `EXTENSION-IDENTITY.md` v3.0 §4.3 / §4.6 — see §6.3 below).

### 5.3 All-Members

Constituents: every member's controller's key.

```
governance := {
  type:                   "all_members",
  threshold_routine:      1,
  threshold_protected:    "<integer or fraction>"
}
```

- Threshold is configurable: 1-of-N for inclusive (any member can act), K-of-N for protected operations (require multiple members).
- Membership changes update the quorum (similar to admin-set).
- Useful for very small groups (couples, partnerships) or for groups that want consensus-driven decisions.

### 5.4 On-Chain Governance

Constituents: an oracle (or oracle quorum) that attests outcomes of on-chain proposals.

```
governance := {
  type:                "on_chain",
  chain:               "ethereum",
  contract:            "0x...",
  oracle:              <oracle_group_id_or_peer_id>,
  oracle_threshold:    K
}
```

- The on-chain mechanism is opaque to the protocol (a smart contract, a voting system).
- The oracle is a trusted entity that signs off on outcomes.
- **Recommended: K-of-N oracle quorum.** A single oracle is a single point of trust; if compromised, the DAO's authority is forgeable. K-of-N oracles (e.g., 2-of-3 oracle services run by independent operators) mitigates this. Mechanically: the oracles form their own sub-group with K-of-N quorum; the DAO's quorum references the oracle group's identity.
- Single-oracle deployments are allowed but SHOULD be flagged as carrying single-point-of-trust risk.

This pattern requires bridge infrastructure (oracle service, on-chain contract). Application-layer; the group extension just provides the composition point.

**Validation status (designed, not validated; pilot scope).** The on-chain governance pattern is recognized and documented but has not been validated end-to-end in production deployments. Walks at the conceptual level (e.g., `EXPLORATION-DEPLOYMENT-SHAPES §14.6` CommunityDAO) explore the pattern but assume bridge infrastructure (reliable on-chain event watchers, gas costs, finality semantics, multi-DAO federation) that is not specified here. Deployments using `on_chain` governance SHOULD treat it as pilot scope and validate the bridge layer before production use. This is a layering note; the pattern composes correctly with the rest of the spec, but operational concerns beyond the protocol surface remain to be exercised.

### 5.5 Hierarchical

A subgroup may have a degraded governance — some operations require parent-group approval.

```
governance := {
  type:                  "hierarchical",
  parent_group:          <parent_id>,
  internal_governance:   { type: "admin_set", threshold_default: 1, ... },
  parent_required_ops:   ["rename", "dissolve", "merge"]
}
```

E.g., Engineering as a subgroup of Acme: Engineering's admins manage internal Engineering matters, but renaming or dissolving Engineering requires Acme's quorum.

Composes with subgroup membership (§3.4).

**Authority walk fail-closed (MUST; per IA16).** When resolving authority for a hierarchical-governance operation, the implementation walks the parent chain from the subgroup upward. If the walk detects a cycle (which §3.4's cycle prevention should make impossible, but defensively), or if it exceeds the SHOULD-tier depth bound (32; §6.4), the walk MUST fail closed and return `403 governance_walk_cycle`. The walk MUST NOT loop unboundedly.

### 5.6 Signer Reference Style — Concrete vs. Identity-Resolved

Orthogonal to the governance pattern: how the quorum's `signers` field is interpreted. Two styles, both supported via the `signer_resolution` field on the quorum entity (per `EXTENSION-IDENTITY.md` v3.0 §3.1).

**Concrete-peer-ID style (default).** The `signers` list contains specific peer-ID hashes. Each entry is a fixed keypair. Validation: at signature-verification time, look up each constituent's identity entity by hash; verify the signature against that identity.

When a member rotates their individual controller's key, the group's quorum no longer recognizes their signatures because the new controller's key's keypair isn't in `signers`. The group MUST issue a `quorum-update` attestation to swap the old keypair for the new one.

This is rigid but simple. Each signer is a specific keypair; rotation is explicit.

**Identity-resolved style (advanced).** The `signers` list contains references to member identities (the published handles of P_a, P_b, etc.) rather than specific keypairs. Validation: at signature-verification time, resolve each constituent through its identity layer to find the current `cert(function=controller)` attestation; verify the signature against the currently-certified controller's key for that identity.

When a member rotates their controller's key, the member's own `cert(function=controller)` chain reflects the rotation (a new `cert(function=controller)` attestation supersedes the old). The group's quorum dereferences member-identity → current controller's key; signatures verify transparently. No `quorum-update` attestation needed for the group.

**Trade-offs:**

| Property | Concrete-peer-ID | Identity-resolved |
|---|---|---|
| Verification cost | Direct lookup + Ed25519 verify | Identity resolution + lookup + verify |
| Member rotation | Each rotation requires group quorum-update | Transparent through member's identity |
| Trust model | Group trusts specific keypairs | Group trusts members' identity layers |
| Failure mode if member's identity is compromised | Group unaffected (different keypair) | Group is affected — compromised identity affects group operations |
| Coordination cost | High over time (many quorum-updates) | Low (members rotate independently) |
| Audit trail | Every member rotation visible in group's history | Member rotations not in group's history |

**Choosing:**

- Use **concrete-peer-ID** when:
  - Member rotation is rare AND coordination is cheap.
  - Audit requires group-side visibility into each member's key changes.
  - Trust is anchored to specific keypairs (e.g., regulated environments).
- Use **identity-resolved** when:
  - Member rotation is frequent.
  - Members manage their own identity hygiene independently.
  - The group wants to delegate identity-rotation responsibility to members.

**Operational requirement for identity-resolved mode (controller-cert reachability).** Identity-resolved validation requires the verifier to walk member-identity → current controller's key via the constituent's live `cert(function=controller)` attestation. Certification attestations live under `system/identity/quorum/{q}/cert/{hash}` in the constituent's tree (per `EXTENSION-IDENTITY.md` v3.0 §4.2 / §5.1) — internal-only by default. Reachability from external verifying peers is NOT automatic. **Implementations support `signer_resolution: "identity-resolved"` (per §11.1 conformance), but operational deployment is bounded by the reachability mechanism the deployment chooses.** Three mechanisms are recognized:

1. **Shared infrastructure (formalized; safe default).** The group's verifying peer runs on the constituent's hardware, with native access to the constituent's `cert(function=controller)` attestation tree. This is common when the group is hosted on a member's machine (per §7.5). No additional plumbing required.

2. **Constituent-published certification (convention; not formalized in v1.1).** The constituent publishes their current `cert(function=controller)` attestation at a path the group's verifying peers can pull (e.g., a per-relationship subtree). EXTENSION-IDENTITY v3.0 supports per-relationship publication only for `cert(function=agent)` attestations (§4.4); a per-relationship publication mode for `cert(function=controller)` would need a future amendment. Deployments using this approach SHOULD document the convention.

3. **Embedded resolution proof (convention; not formalized in v1.1).** The constituent attaches a current-`cert(function=controller)`-attestation snapshot to each signature (or to a periodic refresh entity), so the verifier doesn't need to walk the constituent's tree. Shifts the bandwidth cost to per-operation but eliminates the publication-mode requirement.

If reachability isn't satisfied, identity-resolved verification fails closed (the verifier can't resolve the constituent's current controller's key). Implementations SHOULD surface this as a structured `certification_unreachable` error rather than silent rejection.

**Recommended scope for v1.1 deployments.** Treat shared-infrastructure (mechanism 1) as the supported topology for identity-resolved mode. Mechanisms (2) and (3) are conventions deployments may adopt with documentation, but cross-impl interoperability is not guaranteed until a future spec amendment standardizes one of them. Deployments that need cross-fleet identity-resolved validation (where the verifier and constituent do not share infrastructure) SHOULD prefer dedicated per-group keys in concrete mode (per `GUIDE-GROUP.md §3.7` Option B) until the gap is closed.

**Composition with multi-sig primitive (no core dependency).** Multi-sig core is keypair-based and stays that way. `signer_resolution` is purely an extension-level interpretation; it does NOT change the V7 multi-sig core verifier. Identity-resolved mode resolves to concrete keypair hashes at cap issuance time; the V7 multi-sig verifier sees concrete keypairs regardless of mode and applies M6's local-peer-in-signers rule unchanged. See `EXTENSION-IDENTITY.md` v3.0 §3.1 "signer_resolution" for the full architectural framing.

**Multi-controller constituents (per IA19).** In identity-resolved mode with a multi-controller constituent (an identity with multiple concurrent controllers per `EXTENSION-IDENTITY.md` v3.0 §11.6), **any live controller's key** of the constituent's identity satisfies that constituent's signature in K-of-N validation. The constituent contributes one signature; which of their live controller's keys produces it is the constituent's choice.

---

## 6. Group Handler

The handler at `system/group` is the proper create/update path for group entities, member entries, subgroup references, and acting-on-behalf-of-attestations. Direct `tree:put` to these subtrees is reserved for system-extension and administrative use.

### 6.1 Handler Manifest

```
system/handler := {
  pattern:    "system/group"
  name:       "group"
  operations: {
    form:                       { input_type:  "system/group/form-request",
                                  output_type: "system/group/form-result" }
    dissolve:                   { input_type:  "system/group/dissolve-request",
                                  output_type: "system/protocol/status" }
    merge:                      { input_type:  "system/group/merge-request",
                                  output_type: "system/protocol/status" }
    split:                      { input_type:  "system/group/split-request",
                                  output_type: "system/protocol/status" }
    add_member:                 { input_type:  "system/group/add-member-request",
                                  output_type: "system/protocol/status" }
    remove_member:              { input_type:  "system/group/remove-member-request",
                                  output_type: "system/protocol/status" }
    change_role:                { input_type:  "system/group/change-role-request",
                                  output_type: "system/protocol/status" }
    add_subgroup:               { input_type:  "system/group/add-subgroup-request",
                                  output_type: "system/protocol/status" }
    remove_subgroup:            { input_type:  "system/group/remove-subgroup-request",
                                  output_type: "system/protocol/status" }
    attest_acting_on_behalf:    { input_type:  "system/group/attest-acting-on-behalf-request",
                                  output_type: "system/protocol/status" }
    revoke_acting_on_behalf:    { input_type:  "system/group/revoke-acting-on-behalf-request",
                                  output_type: "system/protocol/status" }
  }
}
```

Total: 11 handler operations. All require dispatched EXECUTEs except `form`, which has a bootstrap exemption (§6.2 form).

### 6.2 form

Creates a new group.

```
system/group/form-request := {
  fields: {
    name:               {type_ref: "primitive/string"}
    initial_quorum:     {type_ref: "system/identity/quorum"}    ; the initial signer set
                                                                 ; and threshold; per
                                                                 ; signer_resolution mode
    governance:         {type_ref: "system/group/governance-spec"}
    initial_members:    {array_of: {type_ref: "system/group/member"}}
                                                                 ; member entries for
                                                                 ; founding members
    metadata:           {type_ref: "primitive/any", optional: true}
  }
}

system/group/form-result := {
  fields: {
    group_id:           {type_ref: "system/hash"}
    quorum_id:          {type_ref: "system/hash"}
    published_handle:   {type_ref: "system/hash"}        ; group's contact-side handle
                                                          ; per §3.1
  }
}
```

The peer initiating `form` sets up:
1. The group entity at `system/group/{group_id}`.
2. The group's quorum entity at `system/group/{group_id}/identity/quorum/{quorum_id}`.
3. The initial `cert(function=controller)` attestation (K-of-N signed by the initial quorum, naming the group's controller's key).
4. The group's published handle (controller's key in three-key default; identifier key in multi-key advanced) keypair generation (handled by the initiator).
5. The initial `quorum-publish` attestation (K-of-N signed by the quorum, per `EXTENSION-IDENTITY.md` v3.0 §4.6).
6. Member entries for each initial member.

**Bootstrap exemption.** Like the identity extension's `configure` / `create_quorum` / initial `create_attestation` calls, the first `form` call on a fresh group goes via L0 access (peer-owner authority on the agent hosting the group). After bootstrap, all group operations go through dispatched EXECUTEs. Conformance is observed via post-state (the group entity, quorum, and initial certification attestation exist), not via wire trace — same convention as identity bootstrap (per `EXTENSION-IDENTITY.md` v3.0 §12.5).

### 6.3 Member Management Operations

```
system/group/add-member-request := {
  fields: {
    group_id:           {type_ref: "system/hash"}
    member_peer_id:     {type_ref: "system/hash"}     ; the new member's published handle
                                                       ; (their controller's key in three-key
                                                       ;  default; their identifier key in
                                                       ;  multi-key advanced)
    role:               {type_ref: "primitive/string"}
    expires_at:         {type_ref: "primitive/uint", optional: true}
    metadata:           {type_ref: "primitive/any", optional: true}
  }
}

system/group/remove-member-request := {
  fields: {
    group_id:           {type_ref: "system/hash"}
    member_peer_id:     {type_ref: "system/hash"}
  }
}

system/group/change-role-request := {
  fields: {
    group_id:           {type_ref: "system/hash"}
    member_peer_id:     {type_ref: "system/hash"}
    new_role:           {type_ref: "primitive/string"}
  }
}
```

Authority: per governance.
- Founder K-of-N: K founders sign.
- Admin set: K admins sign (per the operation's threshold).
- All-members: K members sign (per threshold).
- On-chain: oracle attests the on-chain proposal outcome.

Implementations MAY relax the threshold for routine operations (e.g., adding a low-privilege member only requires 1 admin signature; promoting to admin requires K). The threshold-per-op convention is captured in the governance spec.

For `change_role` operations that affect quorum membership (admin_set or all_members governance), the handler chains into the unified-attestation rotation flow (per `EXTENSION-IDENTITY` v3.0): produces a `quorum-update` attestation (kind=quorum-update; K-of-N by current quorum) AND a fresh `quorum-publish` attestation (kind=quorum-publish; K-of-N by previous quorum per the supersedes-chain rule in v2.0 §4.6) reflecting the updated signer set. Both attestations are produced together via two `create_attestation` calls.

**`change_role` chained quorum-update ordered writes.** When the chained pair (member-entry update + `kind="quorum-update"` attestation) is required, the handler's behavior is pinned by three MUSTs:

1. **Validation atomicity (MUST).** The `change_role` handler MUST validate both the member-entry update AND the chained quorum-update end-to-end **before persisting either entity**. This includes RL2 attenuation on the member-entry change, K-of-N signature presence on the quorum-update, and membership consistency between the two. If validation of any leg fails, the handler MUST return `409 chain_failed` and persist nothing.
2. **Write order (MUST).** When validation succeeds, the handler MUST persist **the member-entry update first, then the quorum-update**. The order is load-bearing:
   - Member-entry persists, quorum-update fails → the member is recognized as admin at the role layer; not yet in K-of-N. They can call admin-scoped role ops; cannot sign quorum-level decisions. **Safe.** Caller retries; the quorum-update is idempotent on retry (content-addressed).
   - Quorum-update persists, member-entry fails (the rejected order) → the member is in K-of-N without admin role recognition. Privilege escalation surface. **Unsafe.**
3. **Recovery (MUST).** The chained `quorum-update` MUST be signed by the current quorum (the K-of-N that validates the change). If the second-leg write fails (storage error, etc.), the handler returns `409 chain_failed` to the caller. Caller retries the full chain; per-entity content-addressing makes the first-leg re-write idempotent.

Implementations MUST NOT require a tree-level transactional-bundle primitive. Per-entity atomicity (already provided by the entity store) plus validation atomicity plus ordered writes is sufficient to satisfy the safety invariant ("no in-quorum-without-admin-role state is ever observable").

**Quorum-update supersedes chain (MUST).** When `change_role` (or `remove_member` below) chains into the unified rotation flow, the resulting `quorum-update` attestation MUST set its `supersedes` field to the content_hash of the immediately-previous `quorum-update` attestation (or to the original `quorum` entity's hash if this is the first update). This mirrors the individual-identity supersedes-chain rule in `EXTENSION-IDENTITY.md` v3.0 §4.3. The chain is what lets contacts validating a compromise-recovery rotation walk back to a known-cached `quorum-publish` attestation. Implementations MUST NOT skip the `supersedes` field; cross-peer attestation validation depends on a continuous chain.

For `remove_member`, the handler:
1. Deletes the member entry.
2. Deletes any role assignments scoped to the group's context (`system/role/{group_id}/assignment/{member_peer_id}/...`).
3. **Federation cascade walk (per IA21).** Walks all `system/group/*/members/*` entries whose `derived_from_parent` field points at this group AND whose `member` matches the removed member, and removes them in cascade. Independent memberships (no `derived_from_parent` set) are unaffected (default; safer semantics). See §7.4 for the federation patterns.
4. If the member was a quorum constituent (admin_set / all_members), chains into the unified rotation flow (a `kind="quorum-update"` attestation plus a fresh `kind="quorum-publish"` attestation) to remove them — same three-MUST ordered-writes rule as `change_role` above (validation-first, member-entry-before-quorum-update, `409 chain_failed` on failure), and same supersedes-chain MUST.

### 6.4 Subgroup Management Operations

```
system/group/add-subgroup-request := {
  fields: {
    parent_group_id:    {type_ref: "system/hash"}
    subgroup_id:        {type_ref: "system/hash"}
    relationship:       {type_ref: "primitive/string", optional: true}
  }
}

system/group/remove-subgroup-request := {
  fields: {
    parent_group_id:    {type_ref: "system/hash"}
    subgroup_id:        {type_ref: "system/hash"}
  }
}
```

Authority: parent group's quorum.

Subgroup creation is two-step:
1. The subgroup itself is formed (its own group entity, identity stack, initial members) — typically via a separate `form` call.
2. The parent attests the relationship via `add_subgroup`.

Implementations MAY combine these into a higher-level "form_subgroup" convenience, but the underlying operations are these two.

**Graph hygiene (per IA16).** `add_subgroup` MUST enforce:

- **Cycle prevention (MUST).** `add_subgroup(parent, candidate)` MUST reject if a transitive walk from `candidate` upward through the subgroup graph reaches `parent` (a cycle would result).
- **Self-reference rejection (MUST).** `add_subgroup(A, A)` MUST be rejected.
- **Depth bound (SHOULD).** Implementations SHOULD bound subgroup-graph traversal depth at 32. Authority resolution that exceeds the depth bound MUST fail closed (per §5.5).

Mutual subgroup-side consent (the candidate subgroup's own quorum acknowledging the parent-subgroup relationship) is informative-only and deferred from v1.0. The parent unilaterally declaring is the current behavior; making consent symmetric would require a new entity type (subgroup-acknowledgment) and a new dispatch flow. Deployments using `add_subgroup` document the asymmetry to subgroup operators out-of-band.

### 6.5 Lifecycle Operations

```
system/group/dissolve-request := {
  fields: {
    group_id:           {type_ref: "system/hash"}
    reason:             {type_ref: "primitive/string", optional: true}
  }
}

system/group/merge-request := {
  fields: {
    source_group_id:    {type_ref: "system/hash"}
    target_group_id:    {type_ref: "system/hash"}
    role_translation:   {type_ref: "primitive/any", optional: true}
                                                       ; map from source roles to target roles
  }
}

system/group/split-request := {
  fields: {
    source_group_id:        {type_ref: "system/hash"}
    member_distributions:   {array_of: {type_ref: "primitive/any"}}
                                                       ; per-output-group member distribution
    resource_distributions: {array_of: {type_ref: "primitive/any"}}
                                                       ; per-output-group resource distribution
  }
}
```

**Dissolve.** Quorum-signed (the group's quorum). Process (composed from identity-extension primitives):
1. Revoke all member entries (they're no longer authorized within the group).
2. **For each of the group's agents**, enumerate the live `cert(function=agent)` attestations for that agent across all publication modes (internal, public, per-relationship, embedded-where-locally-stored) and invoke `system/identity:revoke_attestation` (per `EXTENSION-IDENTITY` v3.0 §6.6) on each, passing the attestation's stored path as the resource target. Per `EXTENSION-IDENTITY.md` v3.0 §9.5 (long-lived cap survival across rotation), revocation of the agent certs is what breaks downstream V7 cap chains: outstanding cross-peer caps signed by those agents lose their authorizing identity-graph edge, and contacts processing the revocations through standard sync no longer recognize the agents as the group's. The retirement-of-the-operational-key path (`kind="cert-retirement"` per `EXTENSION-IDENTITY` v3.0 §4.9) is also valid for groups whose dissolve coincides with retiring the controller's key; routine dissolve uses agent-cert revocation alone.
3. Archive (don't delete) the group entity, marking dissolved-at timestamp. Useful for historical record and audit.
4. Optionally hand off resources to a successor group or to individual members.

Dissolution is structural; it does not automatically destroy the group's data. Application policy decides what happens with shared resources.

**Sync-latency-bounded propagation.** Dissolve's effect on outstanding caps held by third parties propagates through standard V7 cap revocation: once a contact's sync pulls the device-attestation removals, V7 cap chains rooted at the dissolved group's agents fail to validate (the agent is no longer attested as the group's). The window between dissolve and contact-side recognition is sync-latency-bounded; deployments wanting tight dissolve windows configure faster sync. This matches the property already documented for runtime-peer rotation in `EXTENSION-IDENTITY.md` v3.0 §9.5; the group extension does not introduce a separate cache-policy framework.

**Merge.** Quorum-signed by BOTH source and target. Process (composed from identity-extension primitives):
1. Members from source are added to target (their roles may need translation if the role names differ; the optional `role_translation` field maps).
2. **Compose with `kind="cert-rotation-handoff"`** per `EXTENSION-IDENTITY` v3.0 §4.7: produce a rotation-handoff attestation under `system/group/{source_id}/identity/public/attestation/...` from the source group's published handle (its controller's key in three-key default; its identifier key in multi-key advanced) to the target group's published handle, dual-signed by both. Contacts of the source process the handoff via the standard identity-attestation pipeline (per `EXTENSION-IDENTITY` v3.0 §6.8 `process_attestation`) and update their cached handle for the source identity to the target's. Additionally, for each of source's agents, enumerate live `cert(function=agent)` attestations and invoke `system/identity:revoke_attestation` on each (as in dissolve step 2) to invalidate caps issued by those agents.
3. Source group is archived (dissolved with `reason: "merged_into_{target}"`).
4. Subgroup memberships of source flow up to target.

The handoff + revocation composition is what makes merge mechanical: contacts of the source group learn of the handle change via the rotation-handoff attestation through their normal sync pipeline; outstanding caps signed by source's agents are invalidated via the agent-cert revocation flow.

**Split.** Quorum-signed by source. v1.0 scope is bounded to: **form children + dissolve parent + manual cap re-issuance under children**.

1. Form one or more new groups (per `member_distributions`) — each child is a separate `system/group:form` call with its own quorum, identity stack, and initial members.
2. Distribute members per the spec.
3. Distribute resources per `resource_distributions` (application territory).
4. Dissolve the parent group (per the dissolve flow above, including agent-cert revocation for each parent agent).
5. Caps that contacts held against the parent group are invalidated by the dissolve revocations. Children must manually re-issue caps to contacts under their own agents.

A 1→N rotation primitive that automatically redirects contacts from the parent to the appropriate child is **deferred** to a future amendment. v1.0 deployments accept the manual-reissuance limitation. Real deployments needing automatic redirection should surface the use case for a future split-rotation extension.

### 6.6 attest_acting_on_behalf and revoke_acting_on_behalf

```
system/group/attest-acting-on-behalf-request := {
  fields: {
    group_id:           {type_ref: "system/hash"}
    delegate:           {type_ref: "system/hash"}      ; the member acting on the group's behalf
    scope:              {array_of: {type_ref: "system/capability/grant"}}
                                                        ; what the delegate is authorized to do;
                                                        ; reuses V7's grant shape (per IA20)
    not_before:         {type_ref: "primitive/uint", optional: true}
    expires_at:         {type_ref: "primitive/uint", optional: true}
  }
}

system/group/revoke-acting-on-behalf-request := {
  fields: {
    group_id:           {type_ref: "system/hash"}
    delegate:           {type_ref: "system/hash"}
    scope:              {array_of: {type_ref: "system/capability/grant"}}
                                                        ; identifies the attestation(s) to revoke
                                                        ; by matching scope
  }
}
```

Authority: per governance (the group's quorum signs the resulting attestation or revocation; signature gathering follows the normal K-of-N flow for the governance pattern).

`attest_acting_on_behalf` creates an `acting-on-behalf-of-attestation` entity (§3.5), stored at `system/group/{group_id}/acting-on-behalf-of/{delegate_id}/{hash}`. The delegate uses this attestation to sign messages "from the group" (Pattern 2 of §8).

`revoke_acting_on_behalf` removes matching attestations from the group's tree. Recipients processing subsequent messages from the delegate fail validation if the revoked attestation is referenced. This gives operators recourse against abuse without waiting for `expires_at` to roll over.

**Concurrency (informative).** Multiple delegates MAY hold acting-on-behalf-of attestations with overlapping scopes; both attestations are valid; conflict resolution between contradictory statements is application-level, not protocol-level. The protocol does not legislate "whose voice wins" when two delegates publish contradictory statements within the same scope.

### 6.7 Determinism

Handler operations MUST be deterministic across implementations. Given the same input (request) and the same tree state, all conformant impls MUST produce the same output (entity hashes, status codes). This is required for cross-impl validation and for sync convergence — divergent impls would produce different content_hashes for the same logical operation.

---

## 7. Composition with Other Extensions

### 7.1 Composition with EXTENSION-IDENTITY

The group's identity-stack reuses `EXTENSION-IDENTITY.md` directly under `system/group/{group_id}/identity/...`. All entity types, all publication modes, all rotation flows carry over.

The `signer_resolution` field on the quorum entity (concrete vs. identity-resolved) is what controls the dual-mode signature dispatch described in §5.6. It lives on `EXTENSION-IDENTITY.md §3.1`'s quorum schema and is used by both individual-identity quorums and group quorums.

**Cap issuance.** When the group's agent issues a V7 cap to an external party (e.g., partnership_server issues a cap to an external contractor), the cap is signed by the group's runtime-peer keypair (ENTITY-CORE-PROTOCOL.md §5.5 line 1968 unchanged — root caps are signed by the local peer). The group's quorum and controller's key never sign V7 caps; they sign attestation entities (`cert(function=controller)`, member entries, `acting-on-behalf-of-attestation`, etc.) at the entity level via `verify_k_of_n_signatures` and `verify_attestation`. This preserves the two-mechanisms invariant (`EXTENSION-IDENTITY.md` v3.0 §3.6 / §12.3) for the group's namespace.

**Multi-sig caps as group authority.** If the group needs to issue a cap that requires K-of-N quorum approval (e.g., a joint-authority cap to a contractor where 2-of-3 founders must sign), the cap is a multi-sig V7 cap (per `PROPOSAL-MULTISIG-CORE-PRIMITIVE.md`) with `signers` populated from the group's quorum constituents. The cap's `signers` field always contains concrete keypair hashes regardless of `signer_resolution` mode (per §5.6 architectural invariant); the V7 multi-sig verifier validates against keypairs.

### 7.2 Composition with EXTENSION-ROLE

A group is a natural role-context. Roles within a group use `group_id` as the role context:

```
system/role/{group_id}/{role_name}                                 ← role definition
system/role/{group_id}/assignment/{member_peer_id}/{role_name}     ← role assignment
system/role/{group_id}/excluded/{peer_id}                          ← group-scoped exclusion
```

Per `EXTENSION-ROLE` v1.5 R6 (multi-role per peer per context), a member can have multiple roles within a group (admin + finance, manager + IC, etc.).

**Member entry vs role assignment.** Two ways to record "Alice is an admin in Acme":
- **Member entry** (this extension): `system/group/{acme_id}/members/{alice_id}/...` with `role: "admin"`.
- **Role assignment** (role extension): `system/role/{acme_id}/assignment/{alice_id}/admin`.

These overlap. The convention this spec pins:
- Member entry records the member's PRIMARY role and acts as the membership record. Revoking the member entry = removing from group entirely.
- Role assignments record ADDITIONAL roles within the group (orthogonal grants beyond the primary role).
- Removing a member removes their member entry AND all role assignments scoped to that group.

For everyday "Alice is a member with role X," the member entry is sufficient. For complex multi-role situations, role assignments add finer granularity.

**Group-scoped role definitions.** Role definitions (templates) at `system/role/{group_id}/{role_name}` are scoped to the group. A "manager" role in Acme is different from a "manager" role in Globex; assigning Alice to Acme's manager role does not grant her anything in Globex. This composes with role v1.5's existing context-isolation model.

**Governance and role assignments.** Per §5, the governance pattern determines who can sign role-context operations within the group — same authority as for member operations. Role assignments are entity-creations like any other group operation; the governance signing rules apply.

### 7.3 Composition with the Registry

A group has a name (per §3.1's `name` field). Names appear in the registry layer (per `PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE.md`):

- `acme.example` resolves to Acme's published handle (its controller's key in three-key default; its identifier key in multi-key advanced) + bootstrap_endpoints.
- `acme.example/engineering` resolves to Engineering's published handle (subgroup).

The registry resolver returns the group's identity attestations (notably the `kind="quorum-publish"` attestation contacts use as the trust anchor for compromise-recovery rotations, per `EXTENSION-IDENTITY.md` v3.0 §4.6 / §9.4) and bootstrap_endpoints, just like for an individual user.

**Member-of-group addressing.** `alice@acme.example` resolves to Alice's identity + Acme membership context. The resolver returns:
- Alice's published handle and bootstrap endpoints.
- An attestation that Alice is a member of Acme (the `system/group/{acme_id}/members/{alice_member_id}` entry, signed by Acme's quorum).
- Optionally, Alice's role within Acme.

A receiving peer caches this. Future messages from Alice can be displayed as "Alice from Acme" (Pattern 1 of §8).

**Federation.** Cross-org addressing (`alice@globex.example` reaching `bob@acme.example`) works via the registry layer's federation hops. No new mechanism in the group extension.

### 7.4 Cross-Group Identity Sharing for Federations

When two groups partner (e.g., Acme and Globex form a joint partnership group), the partnership group's agents need to verify caps used by members of either parent group. That verification requires the verifying peer to have the member's `kind="cert", function="agent"` attestations (per `EXTENSION-IDENTITY.md` v3.0 §4.4) for the relevant member identity, so it can recognize the member's runtime-peer keypair as currently attested.

**Two patterns the architecture supports:**

**Per-member individual sharing (default; MUST per §11.1).** Each member of the partnership group individually publishes their `cert(function=agent)` attestations to the partnership's identity via per-relationship publication mode (per `EXTENSION-IDENTITY.md` v3.0 §4.4 publication modes). Concretely: the member calls `system/identity:publish_attestation` with `new_mode: "per-relationship"` and `contact_id: {partnership_id}`, which writes to `system/identity/relationships/{partnership_id}/cert/{hash}` in the member's tree. The partnership's agents (including partnership_server) sync this per-relationship subtree and can verify the relevant members' caps.

- Pros: privacy-preserving — each member chooses what to expose. Members not in the partnership don't have their attestations leaked.
- Cons: scales linearly with members. For a partnership of 100 employees, 100 individual configurations needed. Coordination overhead.

**Bulk attestation sync (convenience; SHOULD per §11.2 for federations of meaningful size).** A parent group (Acme) provides its members' attestations in bulk to the partnership group. Mechanically: Acme's group infrastructure runs a sync flow that publishes member attestations (for members who have opted in to be in the partnership) into a partnership-visible namespace.

Implementation: an Acme-side mechanism that, when a member is added to the partnership group, automatically copies that member's `cert(function=agent)` attestations into the partnership's tree (via a per-relationship namespace under Acme's tree, OR via direct write to the partnership's tree if Acme has caps for it).

- Pros: scales with group size, not membership churn. Members who opt in to a partnership don't have to configure individually.
- Cons: requires the parent group to have a sync convention; trusts the parent's bulk-attestation flow; less granular per-member control.

**Recommended default for federation deployments:**

1. Each member who joins the partnership opts in via a configuration choice ("share my agents with this partnership").
2. Their parent group's infrastructure (Acme's) propagates their attestations to the partnership.
3. New members of the partnership are auto-handled by the same mechanism.
4. When a member leaves, their attestations are revoked (deleted from the partnership's tree).

This gives members opt-in control plus operational scaling for the federation.

**Cascading vs. independent membership (per IA21).** When a member is removed from a parent group (e.g., Carol leaves Acme), what happens to that member's membership in groups the parent participates in (e.g., the Acme+Globex partnership)? Two patterns are supported, selected per partnership-membership entry via the optional `derived_from_parent` field on `system/group/member` (§3.3):

- **Independent (default; absent `derived_from_parent`).** Partnership membership is independent of parent membership — Carol stays in the partnership unless explicitly removed. This is the safer, failing-safe default.
- **Cascade (opt-in; `derived_from_parent` set to parent group hash).** Partnership membership is gated on parent membership — Carol leaving Acme automatically removes her from any group whose `derived_from_parent` points at Acme.

The cascade walk runs in `remove_member` (§6.3): the handler walks `members` entries across all groups whose `derived_from_parent` points at this group AND whose `member` matches the removed member, and removes them in cascade. Real federations use both patterns; the field is optional with independence as the default.

### 7.5 Group Infrastructure on a Member's Hardware

A common pattern in informal groups (community garden, neighborhood mailing list, small open-source project, family) is that the group's infrastructure runs on a member's individual hardware — Carol hosts garden_calendar_server on her home machine; Alice hosts family_server on her desktop. The member doesn't transfer ownership of their hardware to the group; they just *operate* the group's agent on it.

**Key separation.** The group's agent has its own keypair, attested by the group's identity via a `kind="cert", function="agent"` attestation (signed by the group's controller's key, or by the group's identifier key in multi-key advanced). That keypair is not the member's individual keypair. Even though the member's machine holds the keypair material, the keypair acts on behalf of the group, not on behalf of the member.

**Per-runtime-peer peer-config.** A host machine operating as multiple agents (e.g., `carol_desktop` for Carol's personal identity AND `garden_calendar_server` for the garden group's identity) MUST keep peer-configs strictly separate per the per-runtime-peer-identity rule from `EXTENSION-IDENTITY.md` v3.0 §3.2. Each agent has its own peer-config in its own namespace; trusts its own quorum; has its own `controller_grants`; participates in its own bindings. Peer-configs do not share state across identities.

**Trust model.** Members of the group trust the host to:
- Run the keypair on the group's behalf (sign things the group expects, not unauthorized ones).
- Keep the machine secure (so the keypair isn't extracted by an attacker).
- Cooperate with migration if they leave the group or hand off hosting duties.

This is a real trust assumption, but it's no different from any "someone has to host it" scenario in distributed systems. The group can mitigate by:
- Replicating the agent across multiple members' hardware (multiple `cert(function=agent)` attestations in the group's identity graph; the group survives any single host's compromise).
- Limiting what the host can do via cap scoping (the group's agent holds caps with specific scopes, not blanket admin).
- Auditing: the group's tree records all operations the agent signs.

**Hosting transition (Carol → David).**

1. The group's quorum (or whoever has add-runtime-peer authority — typically the group's controller's key acting under its certification) calls `system/identity:create_attestation` with `kind="cert", function="agent"` and `attested = david_host_keypair_hash` (per `EXTENSION-IDENTITY.md` v3.0 §4.4 / §6.4).
2. The new keypair takes over operations (members redirect their EXECUTE targets, services migrate state).
3. Optionally, the old `cert(function=agent)` attestation for `carol_host_keypair` is revoked via `system/identity:revoke_attestation` (per `EXTENSION-IDENTITY` v3.0 §6.6) with the attestation's stored path as resource.

Standard runtime-peer rotation under the v2 identity primitives. No new mechanism.

**Compromise scenario.** Carol's machine is compromised; the group's runtime-peer keypair is exfiltrated. The group revokes the compromised `cert(function=agent)` attestation via `system/identity:revoke_attestation`; members' verifiers reject signatures from the compromised keypair on next sync (the agent is no longer attested as the group's; downstream cap chains rooted at it fail per `EXTENSION-IDENTITY.md` v3.0 §9.5); the group provisions a new agent on different hardware via a fresh `cert(function=agent)` attestation. Compromise window is sync-latency-bounded; the group survives (its identity is the quorum, not any one agent).

**Hostile-host scenario.** Carol stops cooperating but won't relinquish hosting. The group's quorum signs revocation of the agent cert for Carol-as-host's runtime-peer keypair (quorum must reach threshold without Carol if Carol is the one being removed; if the agent cert was signed by the group's controller's key alone, revocation is single-sig from that key). The group provisions a new host via a fresh `cert(function=agent)` attestation; members redirect. Carol's old machine still has the old keypair, but signatures from it are rejected by the group's verifiers (the attesting edge in the peer graph is gone). The architecture cannot prevent Carol from running her old keypair; it can only prevent the rest of the group from trusting it. That's sufficient — the group continues without her.

**Operational guidance.** For sensitive groups, consider redundant hosts (multiple agents running on different members' hardware). For low-stakes groups, single-volunteer-host is acceptable. The architecture supports both; the choice is deployment policy.

---

## 8. Acting-on-Behalf-of Patterns

When a member of a group sends a message that involves the group, two patterns apply.

### 8.1 Pattern 1 — Member Acting Independently with Group Context

The member's agent signs the message (V7 standard). The recipient's peer recognizes the member (the agent holds a live `kind="cert", function="agent"` attestation under the member's identity, per `EXTENSION-IDENTITY.md` v3.0 §4.4). The recipient's peer also sees a separate attestation that the member is a member of the group.

The recipient's address book entry: "Alice (member of Acme)" — both Alice's individual identity and her group membership are visible.

**Implementation.** Alice's outgoing message envelope includes (or references) the `system/group/{acme_id}/members/{alice_member_id}` entry. The recipient's peer fetches Acme's group entity for context. Application UI shows "Alice from Acme."

**Use case:** casual; Alice mentions she's at Acme but she's writing as herself. Most everyday messaging.

### 8.2 Pattern 2 — Group Acting via Member as Messenger

The member's agent signs the message, but the message is "from the group" semantically. Alice is the technical signer; Acme is the semantic author.

This requires Acme's published handle (its controller's key in three-key default; its identifier key in multi-key advanced — per `EXTENSION-IDENTITY.md` v3.0 §1.1 / §11.4) to attest Alice as authorized to act on Acme's behalf. The cap chain root is Alice's agent (ENTITY-CORE-PROTOCOL.md §5.5 unchanged); the message envelope includes `acting_on_behalf_of: {acme_published_handle_hash}` plus the `acting-on-behalf-of-attestation` entity (§3.5) signed by Acme's published handle authorizing Alice for this action.

The recipient's address book entry: "Acme (via Alice)" — Acme is the semantic author; Alice is just the messenger.

**Implementation.** Needs an `acting_on_behalf_of` field in the message envelope plus the attestation entity. The receiving peer validates: the attestation is signed by a currently-authoritative published handle of Acme (verified via the contact's cached identity state for Acme), the delegate is a recognized member of Acme, the scope covers the action, the attestation isn't expired, and the attestation has not been revoked (per §6.6).

**Lifecycle (open-ended-with-revocation).** v1.1+ pins acting-on-behalf-of attestations to an **open-ended-with-revocation** lifecycle: an attestation is issued (optionally with `expires_at`), is valid until either expiry or explicit revocation via `revoke_acting_on_behalf` (§6.6), and covers any number of messages within its `scope`. **Per-act mode (a fresh attestation per message, with `target_message_hash`) is rejected from v1.0.** If a deployment needs per-act semantics — i.e., the group authorizing exactly one specific message — it can use a separate V7 cap from the group's agent covering that one message. This produces a cleaner spec, less surface, and aligns with the role v1.5 lifecycle model.

**Concurrency.** Multiple delegates MAY hold acting-on-behalf-of attestations with overlapping scopes; both attestations are independently valid. Conflict resolution between contradictory statements (two delegates publishing within the same scope) is application-level, not protocol-level — see §6.6.

### 8.3 When to Use Which

- **Pattern 1** — casual. Most everyday messaging. No new attestation needed; the membership entity is sufficient.
- **Pattern 2** — formal. Press releases, contracts, official announcements. Needs the `acting-on-behalf-of-attestation` entity per `attest_acting_on_behalf` (§6.6).

Both patterns are useful. The architecture supports both; the choice is application policy.

---

## 9. Open Questions and Future Work

### 9.1 Dynamic Quorum (Admin Set)

When the admin set changes (someone becomes/stops being an admin), the group's quorum changes. The identity extension's `kind="quorum-update"` attestation flow handles this (per `EXTENSION-IDENTITY.md` v3.0 §4.3) — but the trigger is "the admin set changed," which is a group-extension concern.

The group extension's `change_role` op chains into the unified rotation flow when promoting/demoting an admin (per §6.3): two `system/identity:create_attestation` calls produce a `kind="quorum-update"` attestation (K-of-N by current quorum) plus a fresh `kind="quorum-publish"` attestation (K-of-N by previous quorum per the supersedes-chain rule in `EXTENSION-IDENTITY.md` v3.0 §4.6) reflecting the updated signer set. The chaining is normatively pinned to validation-first ordered writes — see §6.3 for the three MUSTs (validation atomicity, member-entry-before-quorum-update write order, `409 chain_failed` on failure). Conformance is observed via post-state (the quorum entity reflects the updated admin set; the live `quorum-publish` attestation reflects the new signer set) plus the safety invariant that no in-quorum-without-admin-role state is ever observable.

### 9.2 Member Identity Compromise

If a member's individual controller's key is compromised, can the attacker act as that member within the group?

Yes. The attacker controlling the member's keys can:
- Send messages as "[member] from [group]."
- Use any caps the member holds within the group.
- (If the member is an admin) sign group operations as part of the admin K-of-N.

Mitigation: revoke the member's membership (delete the member entry); revoke any caps issued to them. The member's individual identity recovery (per identity ext's compromise flow) handles the individual-level compromise; their group membership is restored separately by re-adding them after recovery.

This is an architectural consequence: group authority depends on members' individual identity security. If members have weak keys, the group is weak. Groups should set governance thresholds proportional to the strength of members' individual identities.

### 9.3 Group Migration

Migrating a group from one governance structure to another (e.g., founder K-of-N → admin set as the group grows) requires:
- Defining the new governance.
- Updating the quorum entity (via `kind="quorum-update"` attestations chained with a fresh `kind="quorum-publish"` per the rotation flow in `EXTENSION-IDENTITY.md` v3.0 §4.3 / §4.6).
- Possibly re-issuing the controller's key's `cert(function=controller)` attestation (when the new quorum needs to re-certify the controller's key).

Whether the group extension provides a `migrate_governance` operation or whether this is handled by composition with the unified rotation flow + `change_role` is left to deployments. v1.0 does not normatively define `migrate_governance`; this is a candidate for a future amendment.

### 9.4 Cross-Group Resource Sharing

A group has a server (or other resource). A subgroup wants exclusive access to it for the subgroup's matters. How is that expressed?

- Option A: parent issues caps to subgroup (subgroup's published handle as grantee). Subgroup distributes among members.
- Option B: the server is a agent of the subgroup (transferred from parent to subgroup at subgroup-formation).

Option B is cleaner if the resource is conceptually subgroup-only. Option A is for shared resources where the parent retains primary authority. Both compose with existing primitives. Application territory.

### 9.5 Catastrophic Governance Loss

If a group's founders all become unavailable (death, departure, etc.), the group's quorum drops below threshold. The group can't be operated.

Mitigation: groups SHOULD have a backup recovery mechanism. Could be:
- A "recovery quorum" outside the controller quorum (board members, lawyers, trustees).
- Hierarchical: parent group can rescue subgroup if subgroup's quorum fails.
- On-chain: governance-mandated successor.

This is the "business continuity" problem. Architecturally tractable; specifics depend on the deployment.

**Threshold tuning for voluntary-participation groups.** Groups whose membership is voluntary (open-source projects, neighborhood associations, community organizations) face a special case of this problem: maintainers / coordinators drift away over time without explicit handoff. A 2-of-3 quorum becomes unworkable if even one constituent goes inactive without notice. Deployments SHOULD set thresholds proportional to expected attrition rates:

- For tightly-coupled organizations (employer / employee, family): tight threshold ratios are acceptable (e.g., 2-of-3, 3-of-5).
- For loosely-coupled voluntary groups (open-source projects, community groups): looser ratios (e.g., 2-of-5, 3-of-7) tolerate drift before the quorum drops below threshold.
- For groups with churn-prone constituents: consider the "stand-in" pattern — additional pre-promoted constituents whose authority activates only on attestation that primary constituents are unreachable. Composes via `EXTENSION-ROLE` v1.5 + caveats; doesn't need new primitives.

The architecture doesn't enforce any particular tuning; this is deployment policy. But the protocol surfaces the choice via the `threshold` field on the quorum and via the governance pattern selection.

### 9.6 Privacy of Group Membership

Should it be public who's in a group? Some yes (a public organization's employees); some no (a private support network).

Per the identity extension's `kind="cert", function="agent"` attestation publication modes (`EXTENSION-IDENTITY.md` v3.0 §4.4), member entries can be in different publication modes:
- Public: "Alice is publicly known to be a member of Acme." Lives in `system/group/{acme_id}/identity/public/members/...` (or just `members/`).
- Internal-to-group: "Alice's membership is visible within Acme but not externally." Lives in `system/group/{acme_id}/identity/internal/members/...`.
- Per-relationship: "Bob knows Alice is in Acme; Carol doesn't." Lives in `system/group/{acme_id}/identity/relationships/{contact_id}/members/...`.

Same publication-mode discipline as for `cert(function=agent)` attestations. Deployments choose per-group what's public, internal, or per-relationship.

---

## 10. Security Considerations

### 10.1 Quorum Threshold and Member Identity Strength

Group authority depends on the security of constituent identities. A founder K-of-N quorum with K=2 is only as strong as the weakest 2 founders' individual identities. Deployments should set thresholds proportional to expected attacker capability and to members' individual key custody practices.

### 10.2 Compromise-Window Latency

Revocation of group `cert(function=agent)` attestations (e.g., on host compromise) propagates via standard sync mechanisms. The compromise window is sync-latency-bounded — same property as any runtime-peer compromise per `EXTENSION-IDENTITY.md` v3.0 §9.5 (long-lived cap survival across rotation). The group survives (its identity is the quorum, not any one agent); the compromised agent is replaceable.

### 10.3 Hostile-Host Risk

When group infrastructure runs on a member's hardware, the member can in principle extract the group's runtime-peer keypair. Mitigations are deployment policy (redundant hosts, scope-limited caps, audit trails) — see §7.5. The architecture cannot prevent a hostile host from using extracted keys; it can only prevent the rest of the group from trusting them (by revoking the `cert(function=agent)` attestation that bound the keypair to the group's identity).

### 10.4 Single-Oracle DAOs

For on-chain governance (§5.4), single-oracle deployments inherit oracle compromise as DAO compromise. K-of-N oracle quorums (the oracle group is itself a sub-group with K-of-N constituents) mitigate this. Deployments using single-oracle SHOULD acknowledge the single-point-of-trust risk.

### 10.5 Federation Trust

In federation deployments using bulk attestation sync (§7.4), the partnership group trusts the parent groups' bulk-sync flows. If a parent group's sync flow is compromised, attacker-controlled attestations could be propagated to the partnership. Mitigations: per-member individual sharing as the foundation (always supported, MUST per §11.1); bulk sync as opt-in convenience; partnership-side validation that bulk-synced attestations are signed by the expected published handles of the source identities.

### 10.6 Quorum Drift in Identity-Resolved Mode

In `identity-resolved` mode (§5.6), if a member's individual identity is compromised, the attacker — by rotating the member's controller's key (via a compromise-recovery flow rooted at the member's quorum, or by directly forging signatures if the member's quorum itself is compromised) — gains the member's quorum slot in the group automatically. This is transparent to the group (no group-side `quorum-update` required) but is also a vector for attacker-controlled access. Mitigations: groups using identity-resolved mode SHOULD have stronger governance thresholds, faster compromise-detection mechanisms, and out-of-band confirmation channels for sensitive operations.

In `concrete` mode, this attack requires the attacker to also forge a `quorum-update` attestation to swap the new (compromised) keypair into the group's signers list — which itself requires K-of-N approval. The trade-off is rotation rigidity vs. attack-surface tightness. See §5.6 for the choosing criteria.

### 10.7 Frozen-Cap Horizon under Constituent Compromise (Identity-Resolved + Multi-Sig)

In identity-resolved mode (§5.6), outstanding multi-sig caps signed before a constituent's individual compromise inherit the constituent's compromise — the frozen keypairs they were signed against are now forgeable by the attacker. The group has no automatic notification of constituent compromise; recovery is the constituent's identity-extension flow, not the group's.

This is a real security trade-off that surfaces only for deployments combining identity-resolved mode with multi-sig caps. It is silent in §10.6 (which focuses on entity-level signature drift). Spelling it out here:

- **Mitigation (SHOULD).** Deployments using identity-resolved + multi-sig SHOULD short-cap-TTL to bound the window during which compromised constituents can forge frozen-keypair signatures.
- **Mitigation (MAY).** Deployments MAY subscribe to constituents' `cert(function=agent)` attestation revocations as an out-of-band signal to trigger group-side cap re-issuance. This is informational only; the protocol does not automate the re-issuance.

Cross-ref: `EXTENSION-IDENTITY.md` v3.0 §3.1 for the `signer_resolution` framing; §5.6 above for the concrete-vs-identity-resolved trade-offs.

---

## 11. Conformance

### 11.1 MUST Implement

- All entity types in §3 (group, member, subgroup, acting-on-behalf-of-attestation) and the reused identity-extension types (the three primary types `quorum`, `peer-config`, `attestation`, plus the `identity-binding` helper) under the group's namespace per §3.6.
- Path conventions (§4.1) for the four-subtree identity stack (`quorum/`, `internal/`, `public/`, `relationships/`, `peer-config`) and the group-specific member / subgroup / acting-on-behalf-of subtrees.
- Audience separation (§4.2) — tier separation describes external sync exposure, not which peers hold what; reused from `EXTENSION-IDENTITY.md` v3.0 §5.2.
- Reserved path segments (§4.3) — `identity`, `members`, `subgroups`, `acting-on-behalf-of` are reserved within `system/group/{group_id}/`.
- Handler operations (§6) at `system/group` with the manifest in §6.1. All 11 ops MUST be implemented (`form`, `dissolve`, `add_member`, `remove_member`, `change_role`, `add_subgroup`, `remove_subgroup`, `attest_acting_on_behalf`, `revoke_acting_on_behalf`, plus `merge` and `split` at minimum as no-ops returning a "not yet supported" status, OR fully implemented per §6.5).
- **`change_role` chained quorum-update ordered writes (§6.3).** Validation atomicity, member-entry-before-quorum-update write order, and `409 chain_failed` on failure are MUST. Implementations MUST NOT require a tree-level transactional-bundle primitive.
- **Subgroup graph hygiene (§3.4 / §6.4 / §5.5).** Cycle prevention, self-reference rejection, and hierarchical authority-walk fail-closed on cycle are MUST. Depth bound (32) is SHOULD.
- **Lifecycle composition (§6.5).** `dissolve` MUST invoke `system/identity:revoke_attestation` for each live `cert(function=agent)` attestation across all publication modes for each of the group's agents; `merge` MUST compose with a `kind="cert-rotation-handoff"` attestation (per `EXTENSION-IDENTITY` v3.0 §4.7) plus the same per-runtime-peer agent-cert revocation flow. `split` is bounded to "form children + dissolve parent + manual cap re-issuance"; 1→N rotation is deferred. Dissolve revocation propagation is sync-latency-bounded per `EXTENSION-IDENTITY.md` v3.0 §9.5 (no separate cache-policy framework in v2).
- **Acting-on-behalf-of structured scope and revocation (§3.5 / §6.6 / §8.2).** Scope MUST be `array of system/capability/grant`. `revoke_acting_on_behalf` MUST be supported. Lifecycle is open-ended-with-revocation; per-act mode is rejected from v1.0.
- **Federation cascading membership (§3.3 / §6.3 / §7.4).** Both independence (default; absent `derived_from_parent`) and cascade (opt-in via `derived_from_parent`) MUST be supported; `remove_member` MUST walk for cascade entries and remove them.
- Governance patterns: founder_k_of_n, admin_set, all_members. The on_chain and hierarchical patterns MAY be deferred to SHOULD (§11.2).
- **Dual-mode signer resolution** via the `signer_resolution` field on the reused identity-extension quorum schema (per `EXTENSION-IDENTITY.md` v3.0 §3.1): both `concrete` (default) and `identity-resolved` modes MUST be supported. `signer_resolution` is an extension-level interpretation; multi-sig core stays keypair-based and unchanged regardless of mode (per §5.6 / §7.1). Implementations MUST recognize the `signer_resolution` mode flag and reject quorums with unrecognized values.
- **Cross-group identity sharing** (§7.4): per-member individual sharing MUST be supported (standard per-relationship publication mode for `cert(function=agent)` attestations; no new mechanism). Bulk attestation sync is SHOULD (§11.2).
- **Compose correctly** with `EXTENSION-IDENTITY.md` v3.0 and `EXTENSION-ROLE.md` v1.5. The group extension reuses identity entities and uses role v1.5's R6 multi-role per peer per context.
- **Two parallel mechanisms enforced** per `EXTENSION-IDENTITY.md` v3.0 §3.6 / §12.3 invariant: group-extension attestation entities are validated only by `verify_k_of_n_signatures` / `verify_attestation` at the entity level; never by V7 cap-chain machinery.

### 11.2 SHOULD Implement

- Subgroup entity and subgroup management ops (§3.4, §6.4).
- Hierarchical and on_chain governance patterns where deployments need them.
- `merge` and `split` lifecycle operations (complex; can be deferred to a later phase).
- Acting-on-behalf-of patterns, particularly Pattern 2 with the `acting-on-behalf-of-attestation` entity (§3.5, §8.2).
- Bulk attestation sync for federations of meaningful size (§7.4).
- `migrate_governance` convenience op (composition with the unified rotation flow + `change_role`) for groups whose governance evolves over time (§9.3).

### 11.3 MAY Implement

- Federation conventions beyond what falls out of group + registry composition (§7.3, §7.4).
- Custom governance patterns beyond the five recognized ones (§5).
- Application-level UI/UX for membership management, subgroup hierarchies, voting flows, etc.

### 11.4 Compatibility

- **No core protocol changes.** This extension is purely additive on top of identity / role / multi-sig.
- **Composes cleanly with the in-flight identity-stack specs.** Does not require changes to `EXTENSION-IDENTITY.md` v3.0, `EXTENSION-ROLE.md` v1.5, or `PROPOSAL-MULTISIG-CORE-PRIMITIVE.md`.
- **No multi-sig core dependency for identity-resolved mode.** Identity-resolved mode resolves to concrete keypair hashes at cap issuance time; the V7 multi-sig verifier sees concrete keypairs in cap signers regardless of mode; M6 (local-peer-in-signers) applies unchanged. Multi-sig core stays keypair-based and standalone (per §5.6 / §7.1).
- **Independent of registry backend choice.** Works with any registry layer (self-hosted, DID:web, DHT, blockchain).

---

## 12. Examples

> **Notation in worked examples.** These examples use narrative names (Alice, Bob, Carol, Dave, Acme, Globex) for readability; the spec body proper uses cryptographic-structural notation (per `EXTENSION-IDENTITY` v3.0). When referring to a specific key, examples use `K_alice_op` (Alice's controller's key), `K_acme_op` (Acme's controller's key as a group), etc. Examples assume the three-key default (per `EXTENSION-IDENTITY.md` v3.0 §11.3) — published handle equals controller's key — unless otherwise noted.

### 12.1 Forming a Group (Founder K-of-N)

Acme founders Alice, Bob, and Carol form a group.

1. Alice initiates `system/group:form` (via L0 bootstrap on her agent):
   ```
   form-request: {
     name: "Acme",
     initial_quorum: {
       signers:    [K_alice_op, K_bob_op, K_carol_op],     ; founders' controller's keys;
                                                            ; in three-key default these
                                                            ; are also their published handles
       threshold:  2,
       signer_resolution: "concrete"
     },
     governance: {
       type:      "founder_k_of_n",
       founders:  [K_alice_op, K_bob_op, K_carol_op],
       threshold: 2
     },
     initial_members: [
       {member: K_alice_op, role: "founder"},
       {member: K_bob_op,   role: "founder"},
       {member: K_carol_op, role: "founder"}
     ]
   }
   ```
2. The handler:
   - Creates the group entity at `system/group/{acme_id}` (with `published_handle = K_acme_op_hash` once the group's controller's keypair is generated).
   - Creates the group's quorum entity at `system/group/{acme_id}/identity/quorum/{quorum_id}` via `system/identity:create_quorum`.
   - Generates the group's controller's keypair `K_acme_op`.
   - Creates a `kind="cert", function="controller"` attestation under `system/group/{acme_id}/identity/quorum/{quorum_id}/attestation/{cert_hash}` via `system/identity:create_attestation`, K-of-N signed by the founder quorum (K=2 of [K_alice_op, K_bob_op, K_carol_op]), `attesting = quorum_id`, `attested = K_acme_op_hash`. This authorizes K_acme_op as Acme's controller's key.
   - Creates a `kind="quorum-publish"` attestation under `system/group/{acme_id}/identity/public/attestation/{cqa_hash}`, K-of-N signed by the founder quorum, `attesting = quorum_id`, `attested = quorum_id` (self), `properties = {published_handle: K_acme_op_hash, signers: [K_alice_op, K_bob_op, K_carol_op], threshold: 2}`. This is the contact-side trust anchor for compromise-recovery rotations (per `EXTENSION-IDENTITY.md` v3.0 §4.6 / §9.4).
   - Creates a `kind="cert", function="agent"` attestation for the bootstrapping agent hosting Acme's infrastructure (single-sig from K_acme_op, `attested = bootstrap_runtime_peer_hash`, `properties.mode = "internal"`).
   - Persists the three member entries at `system/group/{acme_id}/members/{K_alice_op}/{hash}`, etc.
   - Persists Acme's peer-config at `system/group/{acme_id}/identity/peer-config` for the bootstrap agent.
3. The form-result returns `{group_id: acme_id, quorum_id, published_handle: K_acme_op_hash}`.

After bootstrap, all subsequent group operations go through dispatched EXECUTEs.

### 12.2 Adding a Member

Alice (as one of Acme's founders) adds Dave:

1. Alice's agent dispatches `system/group:add_member`:
   ```
   add-member-request: {
     group_id:        acme_id,
     member_peer_id:  K_dave_op,                    ; Dave's published handle
                                                     ; (his controller's key in three-key default)
     role:            "engineer"
   }
   ```
2. The handler verifies governance: founder_k_of_n requires K=2 signatures. Alice's signature alone is insufficient; Bob co-signs (async signature gathering per `EXTENSION-IDENTITY.md` v3.0 §8 if Bob is offline at request time).
3. After K signatures present, the member entry is persisted at `system/group/{acme_id}/members/{K_dave_op}/{hash}`.
4. Caps for Dave are derived from his role: `system/role/{acme_id}/{engineer}` is the role definition; `system/role:assign` derives capability tokens for Dave per `EXTENSION-ROLE` v1.5 mechanics.

### 12.3 Member Acts on Behalf of the Group (Pattern 2)

Acme issues a press release. Alice is the one drafting and sending it, but the message is officially "from Acme."

1. Acme's published handle (K_acme_op in three-key default) signs an `acting-on-behalf-of-attestation` for Alice with scope "press releases" — single-sig per §3.5. (If governance requires K-of-N approval before the published handle signs, that's an application-layer flow producing the K-of-N approval that authorizes K_acme_op to issue this attestation; the on-the-wire entity is single-sig from the published handle.)
2. Alice's agent drafts the message and signs it (V7 standard).
3. The message envelope includes `acting_on_behalf_of: K_acme_op_hash` plus a reference to (or embedded copy of) the acting-on-behalf-of-attestation.
4. The recipient's peer:
   - Verifies the message signature (Alice's agent; recognized via the live `kind="cert", function="agent"` attestation under Alice's identity that attests her agent).
   - Validates the acting-on-behalf-of-attestation: signed by K_acme_op, which the recipient recognizes as Acme's currently-authoritative published handle (cached at TOFU and updated through the standard rotation-handoff / rotation-recovery pipeline per `EXTENSION-IDENTITY` v3.0 §6.8); scope covers the action; not expired; not superseded or revoked.
   - Recognizes the message as "from Acme (via Alice)."

### 12.4 Subgroup Formation (Engineering as a Subgroup of Acme)

1. Alice (as Acme founder) initiates `system/group:form` for Engineering. Engineering's initial quorum is the engineering admins; governance is `admin_set`. Engineering gets its own published handle (`K_engineering_op` in three-key default), its own quorum, its own `cert(function=controller)` and `quorum-publish` attestations, its own agents.
2. Engineering is created as its own group with its own full identity stack at `system/group/{engineering_id}/identity/...`.
3. Acme's quorum signs a `system/group:add_subgroup` linking Engineering to Acme:
   ```
   add-subgroup-request: {
     parent_group_id: acme_id,
     subgroup_id:     engineering_id,
     relationship:    "department"
   }
   ```
4. The subgroup entity is persisted at `system/group/{acme_id}/subgroups/{engineering_id}/{hash}`.
5. Members can now be cross-group: Alice is a member of Acme AND Engineering — two member entries, one in each group's `members/` subtree, both keyed by `K_alice_op` (her published handle).

### 12.5 Federation (Acme + Globex Partnership)

Acme and Globex form a joint partnership group:

1. The partnership group is formed with constituents drawn from both Acme's and Globex's quorums (e.g., 1-of-Acme-quorum + 1-of-Globex-quorum, threshold 2, requiring sign-off from both sides). The partnership has its own published handle `K_partnership_op`.
2. Members from both parent groups join the partnership.
3. Each member runs an opt-in configuration to publish their `kind="cert", function="agent"` attestations to the partnership identity via per-relationship publication mode: `system/identity:publish_attestation` with `new_mode: "per-relationship", contact_id: partnership_id`. The attestations land at `system/identity/relationships/{partnership_id}/cert/{hash}` in the member's tree.
4. (Optionally, per §7.4 SHOULD) Acme's and Globex's group infrastructure run bulk-sync flows that propagate opted-in members' agent certs to the partnership's tree.
5. When a Globex employee Dave accesses partnership_server, partnership_server validates the V7 cap chain (root = partnership_server's own agent keypair, locally rooted per ENTITY-CORE-PROTOCOL.md §5.5), then checks that the EXECUTE signer is one of Dave's agents per the cached `kind="cert", function="agent"` attestations under Dave's identity (synced via the per-relationship or bulk-sync flow above).

When a member leaves the partnership, the bulk-sync flow propagates the agent-cert revocations; partnership_server updates its caches and subsequent caps signed by the departed member's agents fail validation.

---

## 13. Document History

- **v1.4 — full alignment with EXTENSION-IDENTITY v3.0:** Mechanical sweep aligning the group spec to v3.0's cert/function reorganization. Vocabulary updates throughout: `kind="certification"` → `kind="cert", function="controller"`; `kind="device"` (in v1.x prose) and `kind="runtime"` → `kind="cert", function="agent"`; `kind="contact-face"` → `kind="cert", function="identifier"`; `kind="contact-quorum"` → `kind="quorum-publish"`; `kind="rotation-handoff"` / `kind="rotation-recovery"` → `kind="cert-rotation-handoff"` / `kind="cert-rotation-recovery"`; `kind="retirement"` → `kind="cert-retirement"`. Prose updates: `operational peer` / `operational key` → `controller` / `controller's key`; `runtime peer` → `agent`; `contact-face peer` → `identifier peer`; `contact-quorum` → `quorum-publish`; `multi-operational` → `multi-controller`. Path segments: `attestation/` → `cert/`. Field renames: `operational_grants` → `controller_grants`. Spec references throughout: `EXTENSION-IDENTITY` v2.0/v2.1 → v3.0. §1 / §2.1 / §2.2 stale enumerations of "all eight kinds" updated to "all six kinds" with v3.0 enumeration. New §1.2 paragraph noting group-specific functions (moderator, treasurer, federation-bridge, etc.) compose via v3.0's app-defined-function mechanism without needing new attestation kinds — group inherits this extensibility for free. Status advanced "Draft" → "Stable". No mechanism change; the spec gains v3.0's structural capabilities (app-defined functions; sub-controller chains structurally available for federation hierarchies) at the cost of one mechanical rename pass. Bumps GROUP v1.3 → v1.4.

- **v1.3 — full inline alignment with EXTENSION-IDENTITY v2.1:** Completed the inline rewrite that v1.2 deferred. Sections updated: §3.1 group entity (`public_identity` field renamed to `published_handle` with v2 vocabulary commentary); §3.3 member entity (member-field comment updated); §3.5 acting-on-behalf-of-attestation (signer clarified as the group's published handle — controller's key in three-key default, identifier key in multi-key advanced; new normative paragraph explaining why this stays as a group-namespaced entity rather than folding into unified `system/identity/attestation` — it carries scope semantics that no structural attestation kind has, so it sits between identity attestations and V7 caps); §6.2 form-result (`public_group_id` renamed to `published_handle`); §6.3 add-member-request (member-field comment updated); §6.5 dissolve / merge / split (substantive: `revoke_peer` op replaced with `revoke_attestation` per-mode loop; `public-identity-rotation` replaced with `cert-rotation-handoff`; the `EXTENSION-IDENTITY §10.1` verifier-cache-miss-policy framework simplified to "sync-latency-bounded propagation" matching v2's actual model); §7.1 composition with identity (Op / operator-delegation vocabulary retired); §7.2 composition with role (cross-refs to `EXTENSION-ROLE` v1.5); §7.3 registry composition (`Public_acme` / `Public_alice` retired in favor of "published handle"); §7.4 cross-group identity sharing (per-relationship publication mode for `cert(function=agent)` attestations replaces the v1.x runtime-peer-attestation vocabulary; `Public_partnership` retired); §7.5 host-on-member infrastructure (substantive: hosting-transition, compromise, and hostile-host scenarios re-expressed under v2 agent-cert revocation flow; section refs to identity remapped — §3.5 → §4.4, §3.8 → §3.2, §5.9 → §6.6, §9.3 → §9.5/§9.6); §8.1, §8.2 acting-on-behalf-of patterns (Pattern 2 implementation walks the v2 process_attestation pipeline; lifecycle paragraph cross-refs `EXTENSION-ROLE` v1.5); §9 open questions (§9.1 dynamic quorum cross-refs the unified rotation flow; §9.2 member compromise updated to "controller's key"; §9.3 group migration mentions the unified rotation flow; §9.6 privacy of group membership references `kind="cert", function="agent"` attestation publication modes per v3.0 §4.4); §10 security considerations (§10.2 cross-ref updated to `EXTENSION-IDENTITY` v3.0 §9.5; §10.5 federation trust references published handles; §10.6, §10.7 vocabulary updated; cross-ref to v3.0 §3.1 for `signer_resolution`); §11 conformance (lifecycle composition MUST reframed in v2 ops; identity / role version pins updated to v3.0 / v1.5; two-mechanisms invariant cross-ref updated to v3.0 §3.6 / §12.3); §12 worked examples 12.1–12.5 (substantive rewrite under v2 framing — form expressed as composition of `create_quorum` + multiple `create_attestation` calls per kind; member entries keyed by published handle; acting-on-behalf-of attestation single-sig from the group's published handle; federation expressed via per-relationship publication of `cert(function=agent)` attestations). The v1.2 vocabulary mapping table at the top of the document is removed (no longer needed). Mechanism unchanged across the rewrite. Bumps GROUP v1.2 → v1.3.

- **v1.2 — vocabulary alignment with EXTENSION-IDENTITY v2.0:** Cross-extension vocabulary refresh paired with the identity v2.0 fresh write. Mechanism unchanged; terminology and cross-references updated where load-bearing. Specifically: top-level **identity-vocabulary mapping table** added (right after the Terminology note) so readers can interpret v1.x terms still appearing in §7–§9 detailed prose and §13 worked examples. **§1 Overview / §1.1 / §1.2 / §1.4 / §1.5** rewritten to use v2.0 vocabulary (key-graph; three primary entity types; six attestation kinds; controller's key; certification; identifier; etc.). **§2 layer table** rewritten with Operational key + optional Contact-face key rows; mechanics list rewritten with kind names. **§3 storage path table (former §3.6)** rewritten to reference `system/identity/attestation` with `kind` discriminator; old per-entity-type rows replaced with kind-keyed rows. **§3.7 K-of-N helper** updated with v3.0 §3.5 / §3.6 cross-refs. **§4.1 path layout diagram** updated to show kind-discriminated `attestation/{hash}` paths under quorum/, internal/, public/, relationships/. **§5.6 signer reference style** rewritten with controller's key + certification language; "operator-delegation reachability" recast as "certification reachability" with v2.0 §4.2 cross-ref; "Multi-Op constituents" recast as "Multi-operational constituents" with v2.0 §11.6 cross-ref. **§6.2 form** rewritten to use `cert(function=controller)` and `quorum-publish` attestation language; bootstrap exemption updated to reference v2.0 `configure` / `create_quorum` / initial `create_attestation` calls. **§6.3 quorum-update supersedes chain (MUST)** updated with v2.0 §4.3 and §4.6 cross-refs; "rotate_quorum" recast as "the unified rotation flow producing `quorum-update` + `quorum-publish` attestations via two `create_attestation` calls". Depends header updated: V7 v7.35+, EXTENSION-IDENTITY v3.0, EXTENSION-ROLE v1.5+. Deeper sections (§7–§9 detailed discussions, §13 worked examples) retain v1.x naming with the top-level mapping table guiding interpretation. A subsequent refinement pass will update those sections inline.

- **v1.1 Draft, hygiene amendment:** §5.6 gains an "Operational requirement for identity-resolved mode (operator-delegation reachability)" subsection plus a "Recommended scope for v1.1 deployments" paragraph. Surfaced during GUIDE-GROUP authoring: identity-resolved validation requires the verifier to walk `Public_X` → current Op via the constituent's `operator-delegation`, but operator-delegations are internal-mode in `EXTENSION-IDENTITY.md §3.3` (default audience: the constituent's fleet only), so reachability from external verifying peers is not automatic. The amendment names three reachability mechanisms — shared infrastructure (formalized in v1.1; only mechanism guaranteed for cross-impl interop), per-relationship publication of operator-delegation (convention only), and embedded resolution proof (convention only). Recommended scope: v1.1 deployments treat shared-infrastructure as the supported topology for identity-resolved mode; cross-fleet identity-resolved validation should use dedicated per-group keys in concrete mode (per `GUIDE-GROUP.md §3.7` Option B) until a future amendment formalizes one of the convention mechanisms. Conformance unchanged — both `concrete` and `identity-resolved` modes remain MUST per §11.1; the limitation is operational, not on implementation support. Cross-references added to `EXTENSION-IDENTITY.md §3.3` (constituent-side note) and `PROPOSAL-IDENTITY-INFRASTRUCTURE.md §13.4` (tracked future amendment).

- **v1.1 Draft:** Absorbed IA15–IA21 from `PROPOSAL-IDENTITY-ARC-FIXES.md` (Revision 9). **IA15** rewrites §6.5 dissolve/merge/split as compositions of identity-extension primitives — `dissolve` invokes `EXTENSION-IDENTITY:revoke_peer(scope: 'all')` per agent; `merge` composes with `public-identity-rotation` (handoff) plus `revoke_peer` for source's agents; `split` is bounded to "form children + dissolve parent + manual cap re-issuance" with 1→N rotation deferred; verifier cache-miss policy from `EXTENSION-IDENTITY §10.1` cross-referenced for dissolve. **IA16** adds subgroup-graph hygiene to §3.4 / §6.4 / §5.5 — cycle prevention MUST, self-reference rejection MUST, depth bound 32 SHOULD, hierarchical authority-walk fail-closed on cycle MUST (returns `403 governance_walk_cycle`); mutual subgroup-side consent informative-only and deferred. **IA17** replaces §6.3's hand-waved "chains into rotate_quorum" + §9.1's punted atomicity with three MUSTs — validation atomicity (validate end-to-end before persisting either leg), write order (member-entry first, then quorum-update), recovery (`409 chain_failed` on failure; caller retries; chained quorum-update signed by current quorum); explicit MUST that implementations not require a tree-level transactional-bundle primitive. **IA18** adds new §10.7 "Frozen-Cap Horizon under Constituent Compromise (Identity-Resolved + Multi-Sig)" — outstanding multi-sig caps signed before a constituent's individual compromise inherit the constituent's compromise; SHOULD short-cap-TTL; MAY subscribe to constituents' `runtime-peer-attestation` revocations as out-of-band signal. **IA19** adds one-line cross-ref in §5.6 — in identity-resolved mode with a multi-Op constituent, any live Op of the constituent's identity satisfies that constituent's signature in K-of-N validation. **IA20** restructures acting-on-behalf-of — §3.5 schema's `scope` becomes `array of system/capability/grant` (was `primitive/any`); §6.6 adds new `revoke_acting_on_behalf` op (manifest count goes 10 → 11); §8.2 pinned to open-ended-with-revocation lifecycle (per-act mode rejected from v1.0); concurrency note added (overlapping scopes valid; conflict resolution application-level). **IA21** adds optional `derived_from_parent: system/hash` field to §3.3 `system/group/member` schema; `remove_member` walks for cascade entries and removes them; default is independence (failing-safe); cascade is opt-in. §11.1 conformance updated to record all five MUSTs. No core protocol changes; no changes to identity, role, or multi-sig — the extension stays purely additive on top of the in-flight stack.

- **v1.0 Draft:** Initial extension spec, derived from `PROPOSAL-EXTENSION-GROUP.md` (Draft + Amendment 1 + multi-sig layering correction). Group as a type of identity that reuses `EXTENSION-IDENTITY.md` v1.1's four-layer model directly under the group's namespace (§3.6 reused types). Four group-specific entity types (group, member, subgroup, acting-on-behalf-of-attestation). 10 handler operations (form, dissolve, merge, split, add_member, remove_member, change_role, add_subgroup, remove_subgroup, attest_acting_on_behalf). Five governance patterns recognized: founder_k_of_n, admin_set, all_members, on_chain, hierarchical. Two signer-reference modes: concrete (default) and identity-resolved (advanced) — both MUST per §11.1. Two acting-on-behalf-of patterns (member-with-context vs. group-as-author). Cross-group identity sharing for federations (per-member individual MUST, bulk attestation sync SHOULD). Group infrastructure on member's hardware (per-runtime-peer-identity peer-config rule from `EXTENSION-IDENTITY §3.8` applies). No core protocol changes; no changes to identity, role, or multi-sig — the extension is purely additive on top of the in-flight stack. Companion to `EXPLORATION-DEPLOYMENT-SHAPES.md` (which validated the use cases this extension serves). Multi-sig layering invariant respected: `signer_resolution` is purely an extension-level interpretation; multi-sig core stays keypair-based and standalone (per §5.6 architectural note).
