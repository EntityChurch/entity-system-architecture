# Role Extension — Normative Specification

**Version**: 2.0
**Status**: Draft
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.39+)
**Optional**: EXTENSION-SUBSCRIPTION.md (v3.10+) — reactive grant lifecycle
**Optional**: EXTENSION-CONTINUATION.md (v1.7+) — automated grant issuance and exclusion cascade
**Related**: EXTENSION-NETWORK-GROUP.md — consumes role types for group membership permissions
**Analysis**: REVIEW-GROUP-EXTENSION-ANALYSIS.md §§20, 22-27, 30, 34
**Proposal**: PROPOSAL-COHERENT-CAPABILITY-AUTHORITY.md (RL1, RL2, RL3)
**Encoding**: ENTITY-CBOR-ENCODING.md (ECF)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. Overview

The role extension provides named grant bundles and context-scoped peer authorization. A role is a named set of capability grant entries. Assigning a peer to a role means issuing those grants. Excluding a peer from a context means denying all access within that context regardless of held tokens.

Roles are a management abstraction over the capability system (ENTITY-CORE-PROTOCOL.md §5). They do not replace capabilities — grants remain the enforcement mechanism. Roles provide the organizational structure: named permission bundles, per-peer assignments, and context-scoped exclusion.

### 1.1 Scope

This extension defines:

- **Role definitions** — named sets of capability grant entries
- **Role assignments** — binding a peer identity to a role within a context
- **Role exclusions** — denying a peer all access within a context
- **Role handler** — `system/role` operations `assign`, `unassign`, `exclude` (the proper create path for assignments and exclusions; v1.1)
- **Tree path conventions** — where roles, assignments, and exclusions are stored
- **Context semantics** — how contexts scope roles independently
- **Grant derivation convention** — how assignments translate to capability tokens
- **Bootstrap grant convention** — using role definitions as templates for initial grants

This extension does **not** define:

- Group membership lifecycle (EXTENSION-NETWORK-GROUP)
- Automated grant issuance for non-role-handler actors (convention territory, uses EXTENSION-CONTINUATION)
- Cross-context queries (future EXTENSION-QUERY territory)

### 1.2 Design Principles

**Role definitions are tree data; assignments and exclusions go through the role handler (v1.1).** Role definitions are entities stored at conventional paths and may be read by any handler. Assignments and exclusions are load-bearing — they trigger capability token derivation or denials — and are created via the `system/role` handler's operations rather than direct `tree:put`. See §1.3 (Coherent Capability) for the rule.

**Context scopes everything.** A role exists within a context. An assignment binds a peer to a role within a context. An exclusion denies a peer within a context. No cross-context interaction — contexts are independent.

**Capabilities remain the enforcement mechanism.** The dispatch layer verifies capability tokens (core §5.2). Roles add a management layer on top: named bundles that any handler can use as grant templates. The verification algorithm does not change.

**Exclusion is additive.** An exclusion entity adds a denial. It does not modify or revoke existing capability tokens. Tokens remain cryptographically valid but are denied at the context level.

### 1.3 Coherent Capability

`system/role/{context}/{role_name}`, `system/role/{context}/assignment/*`, and `system/role/{context}/excluded/*` entities are load-bearing: creating a role definition determines what assignments mint; creating an assignment derives capability tokens (§5.1) for the assignee; creating an exclusion adds a denial. The role handler at `system/role` is the **canonical entry point** for role definitions, assignments, and exclusions. Going through `system/role:define`, `system/role:assign`, `system/role:unassign`, `system/role:exclude` (and `:re-derive`, `:delegate`) ensures RL2 (§4.3), the re-derive cascade (§5.5), and the layer-1 token sweep (§6.1) all run.

**Direct `system/tree:put` to role-extension paths bypasses these mechanisms.** This is permitted by the capability system if a deployment grants such authority — `system/tree` is an extension like any other; capabilities govern its operations. Deployments that want cascade through the role handler MUST NOT grant raw `system/tree:put` on `system/role/*`, `system/capability/grants/role-derived/*`, or related paths to entities that would write outside the handler. Per the Coherent Capability principle (PROPOSAL-COHERENT-CAPABILITY-AUTHORITY), application-level grants SHOULD target `system/role:*` operations.

The startup-time L0 path (§4.5, IA13) is the only path that legitimately writes role entities outside the handler, and it is L0-only.

**No spec-level rejection mechanism exists or is planned.** v1.5's "rejected at content validation" wording was an architectural error and is removed in v1.6. There is no kernel; capabilities are the only enforcement on raw tree writes. See §1.5.2 for the framing rule.

### 1.5 Framing clarifications

Three architectural rules are absolute across the system; v1.5 violated them in places, which is what produced most of the spec drift the v1.6 batch fixes. These rules are explicit in v1.6 so future spec changes can be checked against them.

#### 1.5.1 Encoding rule — Base58 PeerID has exactly one home

| Position | Encoding | Spec |
|---|---|---|
| Universal-root path segment `/{peer_id}/rest/...` | Base58 | ENTITY-CORE-PROTOCOL.md §1.4, §7.4 |
| `system/peer` entity's `peer_id` field (the human-shareable handle) | Base58 string | ENTITY-CORE-PROTOCOL.md §3.5 |
| Wire-level peer-id strings on connection messages | Base58 | ENTITY-CORE-PROTOCOL.md §4 |
| **Every other peer reference** — non-root path segments, body fields typed `system/hash`, `granter` / `grantee` / `signer` cap fields, content-addressed identity-entity references | **Lowercase hex of `system/hash`** (33-byte format-coded digest of the `system/peer` entity) | ENTITY-CORE-PROTOCOL.md §1.2, §3.6; identity §5; quorum §7 |

In role v1.6, all `{peer_id}` segments under `system/role/.../` and `system/capability/grants/role-derived/.../` use lowercase hex of the assignee's `system/hash` (the content_hash of their `system/peer` entity). Template-variable `{peer_id}` substitutes to the same form. Base58 PeerID does NOT appear in role-extension paths, body fields, or template substitutions. (v1.5 §3.2 told impls to use Base58 for non-root segments — that contradicted V7's universal-root-only rule; v1.6 corrects.)

#### 1.5.2 No-kernel-rejection rule — capabilities govern writes

There is no "kernel" in this system. `system/tree` is an extension. Writes to any path go through whatever handler is registered at that pattern; the only enforcement on raw `system/tree:put` is the capability system. If a peer holds a capability authorizing `system/tree:put` on `system/role/{context}/...`, that write happens — bypassing the role handler, bypassing RL2, bypassing the re-derive cascade. **This is a deployment choice**, not an architecture violation. v1.6 reframes §1.3 around capability discipline (don't grant what you don't trust); it does NOT mandate any kernel-level rejection mechanism, because no such mechanism exists.

#### 1.5.3 Scope rule — the role extension does not reimplement V7

The role extension is a **mapping layer**. It maps users (peer IDs in V7-only deployments; identity references when the identity extension is installed) to capability grants via templated role definitions and assignments. It does not implement, reimplement, or replace any V7 security primitive:

- **Cap chain verification** is V7's `verify_capability_chain` (ENTITY-CORE-PROTOCOL.md §5.5).
- **Attenuation checks** (parent ≤ child) are V7's `is_attenuated` / `grant_subset` / `scope_subset` (ENTITY-CORE-PROTOCOL.md §5.6).
- **Pattern matching** for handlers, resources, operations, peers, excludes — uniform across all dimensions including `*` wildcard support — is V7's `matches_pattern` / `matches_scope` (ENTITY-CORE-PROTOCOL.md §5.4 / §5.2).
- **Revocation checks** are V7's `is_revoked` against the local tree binding (ENTITY-CORE-PROTOCOL.md §5.5).

Role's contribution is upstream of all of this: produce a capability-token entity (with grants populated by template resolution), persist it at the role-derived storage path, and let V7 take over. RL2 is the **only** check role does — and even RL2 just calls V7's `is_attenuated` (ENTITY-CORE-PROTOCOL.md §5.6) on the proposed grants vs the caller's authority. If a future role-spec change introduces text that re-litigates V7 verification, attenuation, or pattern matching, that text should be removed and replaced with a reference to the V7 section.

### 1.4 Relationship to the Capability System

```
Capability system (core §5)         → enforcement: verify, check_permission
  ↑ issues tokens from
Role extension (this spec)           → management: define, assign, exclude
  ↑ consumed by
Group extension (EXTENSION-NETWORK-GROUP) → coordination: membership lifecycle
```

The capability system does not depend on roles. Peers can issue ad-hoc grants without roles. Roles provide organizational meaning — a named bundle with a context — that any handler can consume.

### 1.5 Relationship to the Identity Extension

(RI1–RI3 of v1.2 absorption; vocabulary updated for `EXTENSION-IDENTITY` v2.0.) The role extension and the identity extension compose at a well-defined seam. This section documents the seam so neither extension's mechanics need to mention the other beyond cross-references.

**RI1 — Assignments are typically operational-key-authorized.** When a deployment uses `EXTENSION-IDENTITY.md` v2.0, role assignments are typically caused by the operational key (a key with a live `certification` attestation under the trusted quorum) acting via the local peer→operational-key cap installed by `system/identity:configure`. The operational key dispatches an EXECUTE to `system/role:assign`; the caller capability is the local peer→operational-key cap; RL2 (§4.3) validates that the operational-key-authorized cap covers the role's grants. This is the canonical management flow: the operational key manages roles within the identity's authority. The role extension is unchanged whether the caller is the operational key, a group handler, a continuation, or any other actor with sufficient authority — the operational key is just the typical caller in identity-extension deployments.

For deployments that don't use the identity extension (V7-only peers), role assignments are caused by the peer's owner directly via startup-time L0 access (§4.5) or via dispatched EXECUTEs whose caller capability is whatever V7 cap the peer-owner has issued. The role-extension mechanics don't change.

**RI2 — Assignees can be any peer ID.** The `{peer_id_hex}` segment in `system/role/{context}/assignment/{peer_id_hex}/{role_name}` is any V7 peer-identity hash (lowercase hex of the assignee's `system/peer` content_hash per §3.1). There is no requirement that the assignee be an identity-extension-managed peer; a V7-only peer (one keypair = one identity) is a perfectly valid assignee. The role's derived caps target whatever peer ID was named at assignment time. If the assignee is identity-extension-managed, the cap will be exercised by one of that identity's runtime peers (per `EXTENSION-IDENTITY.md` §4.2 agent certs); if the assignee is V7-only, the cap will be exercised by that single keypair's owner. The role extension does not interpret the peer's identity-extension state.

**RI3 — Startup-time L0 composes with identity attestation processing.** When the identity extension is installed, the startup-time L0 path (§4.5) for role assignments runs in conjunction with the identity-extension's own startup-time setup (`system/identity:configure`, `system/identity:create_quorum`, initial `system/identity:create_attestation` calls per `EXTENSION-IDENTITY.md` §6). The startup-time paths share the L0 direct-store mechanism (peer-owner authority; no caller capability) and the same exclusion check (`is_excluded`) before issuing root caps. After startup-time setup completes, all subsequent role assignments dispatch through `system/role:assign` with the operational key as the typical caller (RI1).

(EXTENSION-IDENTITY currently uses "bootstrap" terminology for the same L0-time concept that this spec calls "startup-time"; a cross-extension terminology sweep is queued as a follow-up. Within v1.6, role uses "startup-time"; identity references are paraphrased to match where they appear in role's text.)

**Ordering and the shared `is_excluded` helper (MUST).** The `is_excluded(context, peer_id)` helper used by both identity startup-time setup and role startup-time L0 access to gate root-cap issuance MUST be implemented such that it returns a defined answer regardless of startup-time ordering — i.e., the helper does NOT depend on identity startup-time having run first. It is a pure tree:get on the exclusion subtree at `system/role/{context}/excluded/{peer_id_hex}`; if no exclusion entity exists at that path (which is true when the tree is fresh, before any startup-time setup has run), it returns "not excluded." This means: the helper is safe to call from either startup-time path in either order. In dual-extension deployments, identity startup-time conventionally runs first (so peer-config and the local peer→operational-key cap are available for subsequent runtime use), but role startup-time MAY run before identity startup-time, after, or interleaved — the layer-2 exclusion check fires correctly regardless. Implementations MUST NOT add ordering constraints that would let an excluded peer receive startup-derived caps depending on which extension's startup-time path ran first.

For V7-only deployments, role startup-time L0 runs on its own without identity startup-time; the peer-owner's L0 path is the authority. Functional equivalence: in both cases, startup-time issues root caps directly; runtime issues attenuated caps through `system/role:assign`.

---

## 2. Type Definitions

### 2.1 Role Definition

A named set of capability grant entries:

```
system/role := {
  fields: {
    name:     {type_ref: "primitive/string"}
              ; Human-readable role name. Unique within the context.
    grants:   {array_of: {type_ref: "system/capability/grant-entry"}}
              ; Grant entries to issue when this role is assigned.
              ; Uses the same five-dimensional grant structure as
              ; system/capability/token (core §3.6).
    metadata: {type_ref: "primitive/any", optional: true}
              ; Extension point for domain-specific role properties.
  }
}
```

Stored at: `system/role/{context}/{name}`

The `grants` field uses `system/capability/grant-entry` (core §3.6) — the same type used in capability tokens. Each grant entry has five dimensions: handlers, resources, operations, peers, constraints. The role bundles multiple grant entries under a name.

**Template variables.** Grant entries in role definitions MAY contain template variables in path values. The convention uses `{name}` where `name` refers to a well-known substitution:

| Variable | Substituted with | Example |
|---|---|---|
| `{context}` | The role's context path | `group/team-alpha` |
| `{peer_id}` | The assigned peer's identity | The peer_id of the assignee |

Template variables are resolved at grant issuance time, not at role definition time. The role entity stores the literal template string. The handler issuing grants performs the substitution.

> **Informative — Template Example**: A role definition at `system/role/group/team-alpha/member` with a grant entry `resources: {include: ["shared/{context}/*"]}` would resolve to `resources: {include: ["shared/group/team-alpha/*"]}` when a grant is issued.

### 2.2 Role Assignment

Binds a peer identity to a role within a context:

```
system/role/assignment := {
  fields: {
    role:        {type_ref: "primitive/string"}
                 ; Role name within this context.
    assigned_by: {type_ref: "system/hash"}
                 ; Identity hash of the peer that created the assignment.
    assigned_at: {type_ref: "primitive/uint"}
                 ; Milliseconds since Unix epoch.
    metadata:    {type_ref: "primitive/any", optional: true}
                 ; Extension point (e.g., expiry, conditions).
  }
}
```

Stored at: `system/role/{context}/assignment/{peer_id_hex}/{role_name}`

The `role` field references a role name within the same context. The role definition MUST exist at `system/role/{context}/{role}` for the assignment to be meaningful. An assignment referencing a non-existent role definition is valid as tree data but produces no grants.

The `{peer_id_hex}` in the path is the **lowercase hex of the assignee's `system/hash`** — the content_hash of the assignee's `system/peer` entity (ENTITY-CORE-PROTOCOL.md §3.6 grantee convention). For ECFv1-SHA-256, this is 66 hex characters starting with `00`. Same form as identity v3.3 cert paths, quorum v1.1 event paths, and V7 invariant pointer paths. **Base58 PeerID is reserved for the universal-root segment `/{peer_id}/...` only** (ENTITY-CORE-PROTOCOL.md §1.4); it does NOT appear in role-extension paths or template substitutions. See §1.5.1 for the encoding rule.

The `{role_name}` final segment supports **multi-role per (peer, context)**: a peer MAY hold multiple roles concurrently in the same context (e.g., `admin` + `auditor`, `manager` + `IC`). The (peer, role) pair is the assignment key; multiple assignments under the same peer accumulate. To remove one role while keeping others, delete only that assignment entry.

**Multi-role token presentation (per IA14).** When peer P holds multiple role-derived tokens for context X (one per assigned role), P presents whichever token covers the requested operation. V7's verifier validates the presented token; there is no protocol-level token-union step. If a token is later revoked (via re-derive or unassign), other live tokens covering the operation continue to authorize P.

### 2.3 Role Exclusion

Denies a peer all role-derived access within a context:

```
system/role/exclusion := {
  fields: {
    excluded_by: {type_ref: "system/hash"}
                 ; Identity hash of the peer that created the exclusion
                 ; (content_hash of the creator's system/peer entity).
    excluded_at: {type_ref: "primitive/uint"}
                 ; Milliseconds since Unix epoch.
    reason:      {type_ref: "primitive/string", optional: true}
                 ; Human-readable reason ("evicted", "compromised", "departed").
  }
}
```

Stored at: `system/role/{context}/excluded/{peer_id_hex}` where `{peer_id_hex}` is lowercase hex of the excluded peer's `system/hash`, per §1.5.1 / §3.1.

The excluded peer's identity is fully determined by the path segment; **no body `peer_id` field is needed**. (Earlier drafts had a `peer_id: system/hash` body field; it was redundant with the path and induced divergent encoding across implementations during the v1.5 drift period — Python, Go, and Rust each invented a different representation. Removing the field eliminates the redundancy.)

An exclusion entity indicates that the identified peer SHOULD be denied access to resources within this context, regardless of any capability tokens the peer holds. The exclusion does not revoke tokens directly — it adds a context-level denial; the layer-1 sweep (§6.1) revokes the role-derived caps as a cascade.

An exclusion SHOULD take precedence over an active assignment. If both `system/role/{context}/assignment/{peer_id_hex}/{role_name}` and `system/role/{context}/excluded/{peer_id_hex}` exist, the exclusion wins. The assignment MAY be deleted as cleanup, but the exclusion is the authoritative denial.

### 2.4 Derived-Token Link

Records the linkage between a (peer, role, context) assignment and the role-derived capability token it issued:

```
system/role/derived-token-link := {
  fields: {
    token_hash: {type_ref: "system/hash"}
                ; Content hash of the role-derived capability token.
    issued_at:  {type_ref: "primitive/uint"}
                ; Milliseconds since Unix epoch; tie-break for
                ; multi-cap-per-(peer, role) cases (re-derive overlap).
  }
}
```

Stored at: `system/role/{context}/derived-tokens/{peer_id_hex}/{role_name}` (per §3.1; sibling subtree to `assignment/` and `excluded/`).

One linkage entity per (peer, role, context) tuple. Re-derive may briefly leave multiple linkage entities (overlap window per §5.5); the handler iterates all when sweeping. `unassign` removes both the assignment entity and the linkage entity; `re-derive` writes a new linkage entity referencing T_new and removes the old one referencing T_old.

The linkage entity exists in role's namespace; the role-derived cap itself lives at `system/capability/grants/role-derived/{context}/{peer_id_hex}/{token_hash}` (per §3.1). The two paths are independent — V7's `is_revoked` operates on the cap path; role's lifecycle ops walk the linkage path. This separation keeps role's bookkeeping out of V7's capability namespace.

---

## 3. Tree Path Conventions

### 3.1 Path Structure

All role data lives under the `system/role/` prefix:

```
system/role/{context}/                                            ; Context root
system/role/{context}/{role_name}                                 ; Role definition
system/role/{context}/assignment/                                 ; Assignment root
system/role/{context}/assignment/{peer_id_hex}/{role_name}        ; Peer assignment
                                                                    ; (multi-role per peer; per R6)
system/role/{context}/excluded/                                   ; Exclusion root
system/role/{context}/excluded/{peer_id_hex}                      ; Peer exclusion
system/role/{context}/derived-tokens/                             ; Linkage root (sibling subtree)
system/role/{context}/derived-tokens/{peer_id_hex}/{role_name}    ; Per-(peer, role) linkage
                                                                    ; entity (system/role/derived-token-link
                                                                    ; per §2.4); references role-derived
                                                                    ; cap content_hash
```

Role-derived capability tokens are stored at a dedicated path under the capability subtree (per R4):

```
system/capability/grants/role-derived/{context}/{peer_id_hex}/{token_hash}
```

The role-derived path is pinned so revocation flows can locate tokens deterministically (`is_revoked` lookups follow this path; `unassign` and `exclude` delete from this path; layer-1 fleet-wide sweep operates on this subtree per §6.5). Caps that should NOT be subject to role's exclusion lifecycle MUST be stored elsewhere (per §6.5 storage-path semantics).

**Encoding of non-root path segments (normative).** All `{peer_id}` segments in role-extension paths — assignment, exclusion, derived-tokens linkage, role-derived cap storage, delegation cap storage — use **lowercase hex of `system/hash`** (the assignee's identity-entity content_hash). For ECFv1-SHA-256, this is 66 hex characters starting with `00`. All `{*_hash}` segments — `{token_hash}`, `{role_hash}`, etc. — use the same form: lowercase hex of the full `system/hash` byte sequence (format code byte + digest), no prefix. The display form `ecfv1-sha256:<hex>` (ENTITY-CORE-PROTOCOL.md §1.2 line 117) is UI-only and MUST NOT appear in path segments. Base58 PeerID is reserved for the universal-root segment `/{peer_id}/...` only (ENTITY-CORE-PROTOCOL.md §1.4) and MUST NOT appear in role-extension paths or template substitutions. This rule is consistent with identity v3.3 (`{contact_id_hex}`, `{cert_hash_hex}`), quorum v1.1 (`{quorum_id_hex}`, `{event_hash_hex}`), and ENTITY-CORE-PROTOCOL.md §3.5 invariant pointer paths (`{content_hash_hex}` for signatures). See §1.5.1 for the encoding rule.

### 3.2 Context Naming

The `{context}` segment is a free-form path that scopes the roles. Conventions for context naming:

| Usage | Context | Example full path |
|---|---|---|
| Group roles | `group/{group_name}` | `system/role/group/team-alpha/member` |
| Service access | `service/{service_name}` | `system/role/service/media-server/consumer` |
| Admin management | `admin` | `system/role/admin/operator` |
| Peer trust levels | `trust` | `system/role/trust/full` |
| Social/public | `social` | `system/role/social/friend` |

Context names SHOULD use lowercase alphanumeric characters and hyphens. Forward slashes within the context create sub-contexts (e.g., `group/team-alpha` is a context, not two nested contexts). The entire `{context}` segment is treated as a single opaque string for scoping purposes.

**Reserved role names (per R10).** The role names `assignment`, `excluded`, and `derived-tokens` are RESERVED and MUST be rejected by the `system/role:define` handler with **400 `reserved_role_name`**. These names collide with the namespace they share — `system/role/{context}/assignment/...`, `system/role/{context}/excluded/...`, and `system/role/{context}/derived-tokens/...` are dedicated subtrees for assignment, exclusion, and linkage entities respectively (per §2.3, §2.4, §3.1), so a role definition under any of those names would create a path collision. Any other role name is permitted. (This is handler-side parameter validation on `:define`, not a kernel-level write rejection — see §1.5.2 for the no-kernel-rejection framing.)

**Peer ID encoding in path templates (normative).** The `{peer_id}` substitution in template strings (e.g., for grant scopes inside role-definition `resources` / `handlers` patterns) substitutes to the **lowercase hex of the assignee's `system/hash`** — the same encoding used for non-root path segments per §3.1. The `{context}` substitution is the literal context-path segment as it appears in the role's location. Base58 PeerID is NOT used in role-extension templates or path segments. (v1.5 §3.2 incorrectly told impls to use Base58 for non-root segments; v1.6 corrects per V7's universal-root-only rule — see §1.5.1.)

**Closed-namespace ownership (normative).** The `system/role/...` subtree is owned by EXTENSION-ROLE. Role definitions, assignments, exclusions, and derived-token linkages bind under their respective documented paths within this subtree. Other extensions MUST NOT bind paths inside `system/role/...`. Extensions consuming role state (audit, notification, query indexes, etc.) MUST use their own top-level namespace, not nest under `system/role/{context}/`. Mirrors the closed-namespace invariant in `EXTENSION-ATTESTATION.md` §7 and `EXTENSION-QUORUM.md` §3.4.

### 3.3 Visibility and Sync

Role entities are regular tree entities. Their visibility is controlled by capability grants on the tree paths:

- A peer with `get` access to `system/role/{context}/*` can read role definitions, assignments, and exclusions for that context.
- A peer with `put` access to `system/role/{context}/assignment/*` can create or modify role assignments.
- A peer with `put` access to `system/role/{context}/excluded/*` can create exclusions.

When group members sync the group's role context (e.g., `system/role/group/team-alpha/*`), all members receive the same role definitions, assignments, and exclusions. Content addressing ensures convergence — same entity data produces the same hash on every peer.

---

## 4. Role Handler

The role handler at `system/role` is the runtime entry point for role definitions, assignments, exclusions, and delegations. Operations flow through dispatched EXECUTE so RL2, re-derive cascade, and layer-1 token revocation can be enforced. Bypass via raw `system/tree:put` is a deployment choice governed by capability discipline; see §1.3 / §1.5.2 for the framing.

Implementations MAY observe direct tree-mutation arrivals on role-extension paths and trigger the equivalent cascade as a defense-in-depth fallback for misconfigured deployments — but this is OPTIONAL, not a conformance MUST. The conformance baseline is "the role handler does the right thing when called; capability discipline is the deployer's job."

The startup-time L0 path (§4.5) is the only path that legitimately writes role entities outside the handler; it is L0-only (no dispatched EXECUTE) and runs before handler registration.

### 4.1 Handler Manifest

```
system/handler := {
  pattern:    "system/role"
  name:       "role"
  operations: {
    define:     { input_type: "system/role/define-request",     output_type: "system/role/define-result" }    ; per IA11
    assign:     { input_type: "system/role/assign-request",     output_type: "system/role/assign-result" }
    unassign:   { input_type: "primitive/any",                  output_type: "system/protocol/status" }      ; empty-params per V7 §3.2
    exclude:    { input_type: "primitive/any",                  output_type: "system/role/exclude-result" }  ; empty-params
    unexclude:  { input_type: "primitive/any",                  output_type: "system/protocol/status" }      ; empty-params
    re-derive:  { input_type: "system/role/re-derive-request",  output_type: "system/role/re-derive-result" }
    delegate:   { input_type: "system/role/delegate-request",   output_type: "system/role/delegate-result" }  ; per IA22
  }
}
```

### 4.2 Operation Inputs

Role operations carry their target tree path in `EXECUTE.resource.targets[0]` per the path-as-resource convention (ENTITY-CORE-PROTOCOL.md §3.2). The handler decomposes the path to extract context and assignee/peer-id.

```
; define (per IA11) — write a role definition through the handler. Resource is
; the role-definition path; the handler validates the proposed grant set
; against the caller's capability (RL2 at definition-write time) and triggers
; re-derive cascade on mutation (§5.x).
system/role/define-request := {
  fields: {
    grants:   {array_of: {type_ref: "system/capability/grant-entry"}}
              ; New grant set for the role definition.
    metadata: {type_ref: "primitive/any", optional: true}
  }
}

system/role/define-result := {
  fields: {
    role_path:        {type_ref: "system/tree/path"}
                      ; Echoes EXECUTE.resource.targets[0] as received per
                      ; V7's canonicalization-on-input model — caller chose
                      ; the form (peer-relative or peer-qualified); handler
                      ; preserves it. Do NOT canonicalize.
    re_derived_count: {type_ref: "primitive/uint", optional: true}
                      ; Number of (peer, role) assignments re-derived as a
                      ; cascade of this definition mutation.
  }
}

; assign — caller authorizes against the assignment path being created.
system/role/assign-request := {
  fields: {
    role: {type_ref: "primitive/string"}    ; role name within the context (selector,
                                            ; not a path the caller authorizes against)
  }
}

system/role/assign-result := {
  fields: {
    assignment_path: {type_ref: "system/tree/path"}
    derived_tokens:  {array_of: {type_ref: "system/hash"}, optional: true}
                     ; Hashes of capability tokens issued for the assignee
  }
}

; unassign / exclude / unexclude have no params content — the path is in
; EXECUTE.resource. Callers send the empty-params shape per V7 §3.2.

system/role/exclude-result := {
  fields: {
    exclusion_path:       {type_ref: "system/tree/path"}
    revoked_token_hashes: {array_of: {type_ref: "system/hash"}, optional: true}
                          ; Hashes of role-derived caps revoked by the
                          ; layer-1 sweep at exclusion time (§6.1).
  }
}

; unassign-result and unexclude-result use the existing system/protocol/status
; shape per the manifest in §4.1.

; re-derive (per R5) — re-issues role-derived tokens for all assignees of a
; context after a role-definition mutation (e.g., grant changes, attenuation
; changes). Caller authorizes against the role-definition path being changed.
; See §5.5 for re-derive lifecycle semantics (IA9): ordered writes, grace
; window, delivery, in-flight EXECUTE behavior.
system/role/re-derive-request := {
  fields: {
    role: {type_ref: "primitive/string"}    ; role name being re-derived (selector;
                                            ; the path is in EXECUTE.resource)
  }
}

system/role/re-derive-result := {
  fields: {
    re_derived_count:     {type_ref: "primitive/uint"}
                          ; Number of (peer, role) assignments that were
                          ; successfully re-derived. Empty-set re-derive
                          ; (zero assignments) returns 200 with count=0;
                          ; that's a valid no-op cascade.
    revoked_token_hashes: {array_of: {type_ref: "system/hash"}, optional: true}
    new_token_hashes:     {array_of: {type_ref: "system/hash"}, optional: true}
    skipped_grantees:     {array_of: {type_ref: "system/hash"}, optional: true}
                          ; Per §5.5 RL2-failure-mid-cascade: assignees
                          ; skipped due to RL2 failure. Each entry is the
                          ; system/hash of the skipped assignee's identity
                          ; entity — same form as the cap grantee field;
                          ; same form as the {peer_id_hex} path segment,
                          ; expressed as raw bytes. Caller can recover the
                          ; assignment by reading
                          ; system/role/{context}/assignment/{hex(grantee)}/{role}.
                          ; Skipped assignees retain T_old.
  }
}

; Type uniformity: revoked_token_hashes, new_token_hashes, and skipped_grantees
; are all array_of: system/hash. Callers consume them with the same
; hash-handling code path.

; delegate (per IA22) — member-to-member delegation. The delegator delegates
; a role they hold (or a subset of its grants) to another peer. The derived
; cap is rooted at the delegator's runtime peer; standard role machinery
; handles revocation and exclusion. See §5.6 for delegation lifecycle.
system/role/delegate-request := {
  fields: {
    delegate:   {type_ref: "system/hash"}
                ; Recipient of the delegation (content_hash of recipient's
                ; system/peer entity per V7 §3.6 grantee convention).
    context:    {type_ref: "primitive/string"}
                ; Context path within which the delegation applies.
    role:       {type_ref: "primitive/string"}
                ; Role being delegated; MUST be a role the delegator holds.
    scope:      {array_of: {type_ref: "system/capability/grant-entry"}}
                ; Subset of the role's grants (delegator MAY attenuate).
                ; MUST NOT contain template variables (per §5.6 step 3a).
    expires_at: {type_ref: "primitive/uint", optional: true}
  }
}

; The delegator field has been removed — the caller is implicit from
; ctx.execute.data.author. Multi-tenant on-behalf-of submission is out of
; scope for v1.6 (see §5.6 locality invariant).
;
; context and role are primitive/string (not system/hash) because they
; participate as path segments in §5.6 storage-path construction; treating
; them as hashes would require a normative hash → context-path lookup
; with no compelling use case.

system/role/delegate-result := {
  fields: {
    delegation_token_hash: {type_ref: "system/hash"}
                           ; Hash of the issued delegation cap.
  }
}
```

**Path decomposition.** The handler parses `ctx.resource.targets[0]` to extract the context and peer identity components:

- `define`: resource is `system/role/{context}/{role_name}` (the role-definition entity being written; per IA11).
- `assign`: resource is `system/role/{context}/assignment/{assignee_peer_id_hex}/{role_name}`.
- `unassign`: resource is `system/role/{context}/assignment/{assignee_peer_id_hex}/{role_name}` (specific role to remove) or `system/role/{context}/assignment/{assignee_peer_id_hex}` (all roles for that peer in this context, when the trailing role segment is omitted).
- `exclude` / `unexclude`: resource is `system/role/{context}/excluded/{peer_id_hex}`.
- `re-derive`: resource is `system/role/{context}/{role_name}` (the role definition entity whose mutation triggers re-derivation).
- `delegate`: resource is `system/capability/grants/role-derived/{context}/{delegate_peer_id_hex}/{token_hash}` (the path where the delegation cap will land; per IA22). Implementations MAY accept the assignment path of the delegator's role and synthesize the storage path internally.

Implementations SHOULD share a small helper (`parse_assignment_path`, `parse_exclusion_path`, `parse_role_definition_path`) across the ops rather than re-implementing string splits per op. Malformed paths are rejected with **400 `malformed_resource`**.

**Selector-vs-path consistency (normative).** When both the EXECUTE resource path and `params.role` (or any other selector field that overlaps a path segment) carry a role-name selector, they MUST agree. The handler rejects mismatch with **400 `invalid_request`** (subcode `role_path_mismatch`). This applies to `assign`, `unassign`, `re-derive`, and `delegate` — any op whose request schema declares a role/context selector that the resource path also carries.

### 4.3 Assign Handler Logic — RL1, RL2

```
handle_assign(ctx, params):
  ; Step 1: assignment path comes from EXECUTE.resource per V7 §3.2.
  if ctx.resource is null or len(ctx.resource.targets) != 1:
    return error(400, "ambiguous_resource",
      "assign requires exactly one resource target (the assignment path)")
  assignment_path = ctx.resource.targets[0]
  (context, assignee) = parse_assignment_path(assignment_path)
  if invalid:
    return error(400, "malformed_resource",
      "expected system/role/{context}/assignment/{assignee_peer_id}")

  ; Step 2: validate params.role (the role-name selector)
  if params.role is null:
    return error(400, "invalid_assign_request", "role is required")

  ; Step 3: resolve role definition (handler reads under its own grant)
  role_def_path = "system/role/" + context + "/" + params.role
  role_def = ctx.entity_tree.get(role_def_path)
  if role_def is null:
    return error(404, "role_not_found",
      "No role definition at " + role_def_path)

  ; Step 4: resolve role-derived grants (template resolution per §5.2)
  derived_grants = []
  for grant_entry in role_def.data.grants:
    resolved = resolve_templates(grant_entry, {
      "context": context,
      "peer_id": assignee
    })
    derived_grants.append(resolved)

  ; Step 4b: R7 layer-2 exclusion check — block new derivation for
  ; excluded peers BEFORE doing any other work. This is the secondary
  ; layer of the three-layer exclusion model (R7); see §6.
  if is_excluded(context, assignee, ctx):
    return error(403, "assignee_excluded",
      "Cannot assign role to a peer in the context's exclusion subtree")

  ; Step 5: RL2 — caller's capability MUST cover all derived grants.
  ; Reformulated per R1 to use V7's existing `is_attenuated` primitive
  ; (ENTITY-CORE-PROTOCOL §5.5). The check is: a hypothetical
  ; capability with the role's grants must be a (non-strict) attenuation
  ; of the caller's capability — i.e., the caller's authority covers
  ; everything the role would mint.
  ;
  ; This prevents an assigner from minting tokens broader than their own
  ; authority. Different from CT2/SB1 chain-root (those check chain
  ; termination of an embedded cap reference). Role-derived grants are
  ; synthesized from a template, so the check is "writer's authority
  ; covers what gets minted" rather than chain-root.
  ;
  ; Synthetic-cap construction (per SI-29 / §5.3 item 4). is_attenuated
  ; expects two capabilities (with .data.grants and .data.expires_at);
  ; derived_grants is a grant-list. The synthetic cap MUST be constructed
  ; with expires_at = MIN(parent.expires_at, role.ttl, caller_cap.expires_at)
  ; over the defined values (omit absent values from the minimum) so V7
  ; §5.6's nil-vs-finite rule does not spuriously fail when the caller has
  ; finite expiry and the synthetic cap would otherwise be null-expires.
  ; See §5.3 item 4 for the rationale.
  hypothetical_cap = synthetic_cap_from(
    derived_grants,
    expires_at = MIN_DEFINED(
      handler_capability.data.expires_at,
      role_def.data.ttl,
      ctx.caller_capability.data.expires_at))
  if not is_attenuated(hypothetical_cap, ctx.caller_capability):
    return error(403, "assigner_authority_insufficient",
      "Caller capability does not cover role-derived grant for " + params.role +
      " (RL2 fails: derived grants are not an attenuation of the caller's authority)")

  ; Step 6: persist the assignment under the handler's own grant
  assignment = {
    type: "system/role/assignment"
    data: {
      role:        params.role,
      assigned_by: ctx.execute.data.author,
      assigned_at: now_ms()
    }
  }
  ctx.entity_tree.put(assignment_path, assignment)

  ; Step 7: derive and persist capability tokens (per §5.1)
  derived_token_hashes = derive_and_persist_tokens(
    context, assignee, role_def, ctx)

  return {
    status: 200,
    result: {
      assignment_path: assignment_path,
      derived_tokens:  derived_token_hashes
    }
  }
```

### 4.4 Unassign / Exclude / Unexclude

All three operations read their target path from `ctx.resource.targets[0]` (ENTITY-CORE-PROTOCOL.md §3.2 path-as-resource) and carry no params content — callers send the empty-params shape per ENTITY-CORE-PROTOCOL.md §3.2.

`unassign` removes the assignment entity and revokes derived tokens. The handler decomposes context and assignee from the resource path. It SHOULD verify the caller's capability covers the role's grants — same RL2 check as `assign` — to prevent narrow-cap callers from removing assignments they couldn't have made.

**Unassign token-revocation mechanism (per IA12).** On `unassign(peer, role, context)`, the handler MUST:

1. Remove the assignment entity at `assignment/{peer_id_hex}/{role_name}`.
2. Read the linkage entity at `system/role/{context}/derived-tokens/{peer_id_hex}/{role_name}` (per §2.4) to recover the content hash of the role-derived cap. If multiple linkage entities exist (re-derive overlap; rare with grace=0 default), iterate all.
3. Mark each linked cap revoked (ENTITY-CORE-PROTOCOL.md §5.1 standard mechanism — remove the tree binding at `system/capability/grants/role-derived/{context}/{peer_id_hex}/{token_hash}`).
4. Remove the linkage entity at `derived-tokens/{peer_id_hex}/{role_name}`.

This makes `unassign` and `exclude` parallel revocation flows with different triggers; see §6.4.1 for the explicit unassign flow alongside the exclusion flow.

`exclude` writes an exclusion entity at the resource path (`system/role/{context}/excluded/{peer_id_hex}`). It requires the caller's capability to cover (or be authoritative over) the context — the dispatch capability check on `resource` enforces this. Exclusion immediately denies the peer access regardless of held tokens (§5).

`unexclude` removes the exclusion entity at the resource path. Same authority requirement as `exclude`.

### 4.4a Concurrent-op semantics for `:assign` / `:unassign` (normative)

Each handler invocation snapshots the role-definition state at handler entry and acts against that snapshot. Concurrent `system/role:assign` ops on the same (peer, context, role) tuple by different operational keys each run their own RL2 check, mint their own derived token, and write their own assignment + linkage entities. Tree CAS resolves the assignment-entity at last-writer-wins on the metadata path. Multiple derived tokens may briefly exist; the assignee picks up whichever delivers first (per §5.5 delivery default).

Concurrent `system/role:assign` and `system/role:unassign` interleavings resolve at the per-entity tree CAS level — last write to the assignment path wins; any tokens issued by the loser remain bound until explicit cleanup (next `:unassign`, `:re-derive`, or `:exclude`). Implementations MUST NOT serialize concurrent ops across operational keys — same rule as §5.5 multi-operational concurrent re-derive. Doing so would block multi-operational concurrency from `EXTENSION-IDENTITY.md` §11.6.

### 4.5 Startup-time L0 access (peer-owner-only path)

Startup-time role assignments occur before the role handler is registered. They are written via direct tree:put under the startup administrative context. **Startup-derived tokens have `parent: null` and `granter: local_peer_id`** (ENTITY-CORE-PROTOCOL.md §5.5 line 1968 root-cap convention). RL2 does not apply at startup time (no caller capability exists yet — the peer-owner is acting via L0 direct-store).

Startup-time derivation is distinct from runtime derivation:

- **Runtime derivation** (post-startup): dispatches to `system/role:assign` with a caller capability. RL2 enforces. Token's `parent` is the role handler's grant.
- **Startup-time derivation** (initial peer setup): pre-handler-registration; the peer-owner's L0 direct-store API provisions root caps directly. No RL2 (no caller cap). Token's `parent: null`. Granter is the local peer.

Implementations MUST distinguish startup-time from runtime derivation. **Startup-time derivation MUST be callable only via the SDK's L0 direct-store API (peer-owner authority); runtime code paths MUST NOT invoke L0 derivation (per IA13).** The L0 API surface is for SDK-managed startup-time setup only; post-startup, all derivation is dispatched through `system/role:assign`. Leaving an L0 callable code path open at runtime is a soft privilege escalation surface (RL2 is bypassed by construction on the L0 path).

**When startup-time L0 ends (per IA13).** After the role handler is registered in the dispatch table, the L0 derivation path is closed for that handler. The SDK MAY re-enter L0 via a deliberate recovery flow (e.g., a peer-owner ceremony to re-issue root caps after key rotation), but doing so is an explicit ceremony, not a runtime fallback. Implementations MUST ensure the L0 derivation entry point is not reachable from any dispatched EXECUTE path or from any code that runs after handler registration in the normal startup sequence.

**R7 layer-2 (block new derivation) applies at startup-time, too.** The exclusion check (`is_excluded(context, peer_id, ctx)`) MUST fire on the L0 startup-time path before issuing root caps, not only on the dispatched `assign` path. Implementations expose a single helper (e.g., `is_excluded(context, peer_id)`) that both the handler and the L0 startup-time path call. This prevents excluded peers from receiving startup-derived caps (the only path that doesn't run through the role handler).

This applies symmetrically to identity startup-time setup from `EXTENSION-IDENTITY.md §5.10`: the identity handler's `configure` operation is also called via L0 during startup. Same mechanism, different operation. (EXTENSION-IDENTITY currently uses "bootstrap" terminology for this same concept; a cross-extension terminology sweep is queued as a follow-up — not in v1.6 scope.)

**Conformance test vector (linguistic-only enforcement, per SI-12).** Implementations MUST include a test vector demonstrating that calling the L0 derivation entry point AFTER `peer.handlers.register(...)` raises an implementation-specific exception (Python: `RuntimeError`; Go: panic or wrapped error; Rust: `Result::Err`). The exact exception type is implementation-defined; the observable behavior is "post-registration L0 invocation fails fast and does not produce a cap." Static-analysis enforcement (lints, lifetime annotations) is permitted as additional defense-in-depth but does not replace the runtime check.

**L0 boundary semantics for general tree access (per PR-4; v2.0).** L0 access remains available to peer-owner code throughout the peer's lifetime, not just at startup. Peer-owner code holding the peer object can read or write the entity tree directly without role enforcement, exclusion enforcement, or capability checking. **This is intentional** — the peer process IS the trust boundary; role and exclusion are L1+ dispatched concerns; L0 is explicitly out of scope for both. The startup-time L0 derivation hardening described above (closing the L0 derivation entry point post-handler-registration) is specifically about role-derivation entry points, not about L0 tree access in general. Any code holding a peer object can bypass role enforcement by going through L0 directly; this is the deployment's trust assumption about its own process boundary.

### 4.6 Determinism of `resolve_templates` and `derive_grants`

`resolve_templates` (§5.2) and `derive_grants` MUST be deterministic across implementations. Role-derived grants MUST resolve identically on every conformant impl for RL2 to be cross-impl consistent — a non-deterministic derivation could pass on one impl and fail on another for the same inputs. Implementations are encouraged to share test vectors for derivation determinism.

### 4.7 Initial grant policy (first-contact cap derivation)

When a peer initiates contact and the local peer has no prior recognition, the **initial grant policy** decides what initial cap (if any) the new peer receives. This is a deployment-policy concern, not an architectural mechanism — the role-extension primitives don't constrain the choice. The choice space is documented here so deployments converge on consistent behavior.

**Three policy modes:**

- **anonymous-allow.** Unknown peer with no prior attestation receives a configured default role (e.g., `anonymous-reader` for a public blog, `proposer` for an open-source project). Standard startup-time derivation per §4.5 applies. Useful for genuinely public services.

- **anonymous-deny.** Unknown peers are rejected unconditionally. No initial cap; the peer must have an out-of-band-established trust relationship (e.g., an explicit role assignment) before any cap is issued. Useful for private services that should never expose anonymous endpoints.

- **recognize-on-attestation.** Unknown keypair, known identity. Useful when the deployment expects identity-extension-running peers: the peer presents a device attestation (`system/identity/attestation` with `kind="identity-cert", function="agent"` per `EXTENSION-IDENTITY.md` §4.2) signed by an identity the local peer recognizes (or has TOFU-accepted), and the policy derives a role based on what's known about that identity. Falls back to anonymous-allow or anonymous-deny if no recognized attestation accompanies the contact.

**Composition with R7 (three-layer exclusion):**

Layer 2 of R7 (block-new-derivation) is checked BEFORE the initial grant policy executes. If the requesting peer's identity (or, where TOFU applies, the peer's keypair) is in the exclusion subtree, no initial cap is derived regardless of policy mode. Excluded peers cannot bypass via anonymous-allow or recognize-on-attestation.

**Where the policy is configured (normative path).**

The policy is stored at `system/role/initial-grant-policy` (normative path, reserved within the role-extension namespace). The path is reserved so peers can inspect each other's policy via standard tree access (subject to capability scope). Implementations MUST use this path. Suggested configuration shape:

```
system/role/initial-grant-policy: {
  unknown_peer:       "anonymous-allow" | "anonymous-deny" | "recognize-on-attestation"
  default_role:       "<role_name>"  ; applied when unknown_peer = "anonymous-allow"
  identity_required:  bool           ; applied when "recognize-on-attestation"
}
```

Defaults: `unknown_peer: "anonymous-deny"`, `default_role: null`. Privacy-preserving and conservative; deployments opt-in to anonymous-allow when they have a public-facing role to grant.

**When this matters:**

- Public services (blogs, open data, registry resolvers): typically `anonymous-allow` with a tightly scoped default role.
- Internal/team services (org infrastructure, family servers): typically `anonymous-deny`; only roster members get caps.
- Mixed services (a public-read / authenticated-write API): `anonymous-allow` with a low-privilege default role; explicit role assignments grant elevated access.
- Identity-extension deployments: `recognize-on-attestation` — accept new keypairs that present a recognized identity attestation, then map to a role per the recognized identity.

This is policy, not protocol. No spec text required to enforce a particular choice. The role extension's primitives (startup-time derivation per §4.5; exclusion per §6) compose to cover all three modes; this section makes the policy surface explicit so deployments don't have to invent it ad hoc.

(Renamed in v1.6 from "Anonymous fallback policy" / "Bootstrap policy". The umbrella term "initial grant policy" covers all three modes — the third mode is non-anonymous. Path renamed from `system/role/bootstrap-policy` to `system/role/initial-grant-policy`.)

---

## 5. Grant Derivation

### 5.1 From Assignment to Capability Token

When a handler creates a role assignment, it SHOULD also issue capability tokens derived from the role definition. The derivation process:

```
derive_grants(context, peer_id_hex, role_name):
  ; peer_id_hex is the lowercase hex of the assignee's system/peer content_hash
  ; (per §1.5.1 / §3.1).

  ; 1. Read the assignment
  assignment = tree.get("system/role/{context}/assignment/{peer_id_hex}/{role_name}")
  if assignment is null: return []

  ; 2. Check for exclusion
  exclusion = tree.get("system/role/{context}/excluded/{peer_id_hex}")
  if exclusion is not null: return []

  ; 3. Read the role definition
  role = tree.get("system/role/{context}/{role_name}")
  if role is null: return []

  ; 4. Resolve template variables in grant entries
  ;    {peer_id} substitutes to peer_id_hex (NOT Base58 PeerID); see §1.5.1 / §5.2.
  resolved_grants = []
  for grant_entry in role.data.grants:
    resolved = resolve_templates(grant_entry, {
      "context": context,
      "peer_id": peer_id_hex
    })
    resolved_grants.append(resolved)

  ; 5. Create capability token (v2.0: root cap, structurally identical to startup-time L0 path §4.5)
  ;    grantee is system/hash (raw bytes) of the assignee's identity entity —
  ;    same value as peer_id_hex expressed as raw bytes rather than hex
  ;    (see "grantee field encoding" note below).
  ;    granter is the content_hash of the local peer's system/peer entity
  ;    (NOT the Base58 PeerID string); same value as the startup-time L0 path uses.
  ;    parent is null — RL2 enforces caller-cap coverage at issue time;
  ;    chain-walk at use time terminates at this cap.
  token = {
    type: "system/capability/token",
    data: {
      grants:     resolved_grants,
      granter:    local_peer_identity.content_hash,    ; v2.0 PR-1: explicit content hash
      grantee:    bytes_from_hex(peer_id_hex),
      parent:     null,                                 ; v2.0 PR-1: root cap (was: handler_capability_hash)
      created_at: now_ms(),
      expires_at: MIN_DEFINED(handler_grant.expires_at, role_def.ttl, caller_cap.expires_at)
    }
  }
  ; Sign token with local peer's signing key.
  ; Persist at system/capability/grants/role-derived/{context}/{peer_id_hex}/{token_hash}.

  ; 6. Write linkage entity (per §2.4) so unassign / re-derive / exclusion
  ;    sweeps can locate this token via (peer, role, context).
  linkage = {
    type: "system/role/derived-token-link",
    data: {
      token_hash: hash(token),
      issued_at: token.data.created_at
    }
  }
  ; Persist at system/role/{context}/derived-tokens/{peer_id_hex}/{role_name}.

  return [token]
```

**v2.0 root-cap shape (per PR-1).** Runtime role-derived caps are root caps: `parent: null`, `granter = local_peer_identity.content_hash`. Structurally identical to the startup-time L0 derivation path per §4.5. The runtime authority check is RL2 at issue time (§4.3 step 5 — caller's capability covers the proposed grants); the use-time chain-walk (ENTITY-CORE-PROTOCOL.md §5.5) terminates at the role-derived cap because there is no `parent` to walk further. The previous "handler grant as cap-chain parent" model (v1.x) conflated issue-time authority (caller cap) with use-time chain validation (parent cap), which broke on production-shaped peers where the role handler's own grant is too narrow to cover what the role would mint. v2.0 separates the two cleanly: issue-time authority lives in RL2; use-time validity lives in the cap signature + tree binding.

**Side benefit — §10.2 op-key rotation invariance is structural.** Under v2.0, a role-derived cap has no chain above itself **when wielded as a root by the assignee** (the assignment use case); for that use the §10.2 invariance becomes structural rather than a cap-shape constraint — there is no link above it that could reference the operator key. **Scope of the claim (normative):** this "no chain above" property is *use-relative*, not absolute. Under `system/role:delegate` (§5.6) the role-derived cap becomes the **parent** of the delegate's cap — i.e. an interior link in a chain transported to and re-verified at the delegate's (and any third) peer. As a chain-participating capability it MUST therefore have its signature discoverable at the V7 invariant pointer path per `ENTITY-CORE-PROTOCOL.md` §3.5 (v7.44); it is a registered cross-peer chain-construction site per ENTITY-CORE-PROTOCOL.md §5.8. The structural rotation-invariance argument above applies to the assignee-root use; it does not exempt the `:delegate` interior-link use from the §3.5 / §5.8 obligations.

**Role-derived caps survive controller revocation (normative).** Role-derived capability tokens are root caps (`parent: null`, `granter: local_peer.content_hash`). They have no cap-chain dependency on the controller cert that authorized the local peer to act for an identity. Revoking a controller via `system/identity:revoke_attestation` (which cascades to peer-to-controller caps per `EXTENSION-IDENTITY.md` §6.4) does NOT cascade to role-derived caps. The local peer continues to honor caps it issued until something explicitly revokes them at the role layer. Deployments wanting role-derived caps revoked when a controller is revoked MUST run `system/role:re-derive` (per role-definition mutation) or `system/role:exclude` (per excluded peer) explicitly; the role handler does NOT observe identity-layer revocation events.

**Local-peer-keypair rotation invalidates outstanding role-derived caps (deployment-policy).** Role-derived caps are rooted at the local peer's `system/peer` content_hash. Rotating the local peer's keypair (in Stage-2 deployments, via `system/identity:supersede_attestation` for the agent cert under REBIND_KINDS per `EXTENSION-IDENTITY.md` §6.0b) changes the `system/peer` entity's `public_key` field, producing a new content_hash. Outstanding role-derived caps reference the prior `granter` hash and become unverifiable on V7 chain validation (the granter no longer resolves to a present `system/peer` entity). Deployments rotating agent-peer keypairs MUST run `system/role:re-derive` for each role context post-rotation to re-issue role-derived caps under the new granter. Stage-1 deployments (no identity layer) don't face this — there's no rotation primitive without identity; a lost keypair is a lost peer.

**`grantee` field encoding (normative).** The `grantee` field on a role-derived capability token is `system/hash` per ENTITY-CORE-PROTOCOL.md §3.6 — specifically, the content_hash of the assignee's `system/peer` entity. Same value as the `{peer_id_hex}` segment in path conventions per §3.1, expressed as raw bytes rather than hex. **Not** the peer-id-derived digest `0x00 || SHA256(public_key)` — that's `0x00 || SHA256(public_key)`, but the peer-entity content_hash is `0x00 || SHA256(ECF({type: "system/peer", data: {peer_id, public_key, key_type}}))`. Different preimages, different hashes.

The handler does not need to "look up" the assignee's identity entity to construct the cap. The path segment `{peer_id_hex}` IS lowercase hex of the identity-entity hash; the handler reads the hash directly from the path (`grantee = bytes_from_hex(path_segment_peer_id_hex)`). No PeerID-to-hash resolution; no type-index walk; no resolver hook. Identity-entity availability for downstream cap-chain verification follows V7's standard envelope-packaging pattern: when the cap is delivered to the assignee or used cross-peer, the envelope `included` carries whatever entities ENTITY-CORE-PROTOCOL.md §5.5 chain verification needs (granter's identity, signatures, etc.). That's V7's responsibility, not role's.

Implementations that previously used `0x00 || SHA256(public_key)` as `grantee` (re-tagging the peer-id-derived digest with the system/hash format byte) MUST migrate to the identity-entity content_hash. Caps issued under the old scheme cannot be verified cross-peer because the `grantee` does not resolve to any real identity entity in `included` lookups (ENTITY-CORE-PROTOCOL.md §5.5 line 1965).

**Linkage entity (normative, per §2.4 + SI-5).** After issuing the cap, the handler MUST write a linkage entity at `system/role/{context}/derived-tokens/{peer_id_hex}/{role_name}` referencing the cap's content hash and the issuance timestamp. This is the bookkeeping that makes `unassign`, `re-derive`, and the exclusion sweep able to locate role-derived caps by (peer, role, context) tuple.

**RL2 fail-closed (per IA10).** If the handler's capability does not cover any grant entry in the role definition, the assign MUST fail with **403 `assigner_authority_insufficient`** (RL2 — see §4.3 Step 5). Partial tokens are not minted. This reconciles §5.1 with §4.3: the v1.3 prose said "the entry MUST be omitted from the derived token," which contradicted §4.3's fail-closed framing and silently produced partial tokens that a low-authority caller didn't request. Fail-closed surfaces the authority gap to the caller rather than minting a token of unexpectedly narrow scope.

### 5.2 Template Resolution

Template variables in grant entry path values are resolved by string substitution:

```
resolve_templates(grant_entry, variables):
  for dimension in [handlers, resources]:
    for scope in [include, exclude]:
      for i, path in grant_entry[dimension][scope]:
        for var_name, var_value in variables:
          path = path.replace("{" + var_name + "}", var_value)
        grant_entry[dimension][scope][i] = path
  return grant_entry
```

Template resolution is purely textual — no path normalization or validation during resolution. The resolved paths are validated as part of normal grant verification.

**Substitution values (normative).** When `derive_grants` (§5.1) calls `resolve_templates`, the supplied variables are:
- `{context}` — the literal context-path segment as it appears in the role's location.
- `{peer_id}` — the lowercase hex of the assignee's `system/hash` (the identity-entity content_hash; same form as the `{peer_id_hex}` path segment per §3.1). **Not Base58 PeerID** — see §1.5.1 for the encoding rule.

Templates substitute into `handlers` and `resources` dimensions only (not `peers`, `operations`, or constraints). After substitution, the resulting paths are matched against tree paths during V7's `check_permission` per ENTITY-CORE-PROTOCOL.md §5.2 — and tree paths use the hex form for non-root segments throughout the substrate stack, so substitution and matching are consistent.

### 5.3 Token Lifetime

The derived capability token inherits lifetime constraints from:

1. **The parent capability's expiry.** The derived token's `expires_at` MUST NOT exceed the parent's `expires_at` (core §5.6). The cap-chain `parent` of a role-derived token is the issuing handler's own grant (per §5.1), not the caller's capability — items 1 and 4 are distinct sources.
2. **The handler's policy.** The issuing handler MAY set a shorter `expires_at` based on its own policy (e.g., group policy timeout, refresh cycle).
3. **The role definition's metadata.** If the role definition includes a `ttl` in its metadata, the handler SHOULD use it as the default lifetime.
4. **The caller capability's expiry (implicit upper bound, per SI-29).** RL2's `is_attenuated` check (§4.3 step 5) treats the caller capability as the structural parent of the synthetic cap built from `derived_grants`. Per ENTITY-CORE-PROTOCOL.md §5.6's nil-vs-finite rule (line 2047), if the caller cap has finite `expires_at` and the synthetic cap has `expires_at: null`, `is_attenuated` returns `false` and the assign fails 403 `assigner_authority_insufficient`. Implementations MUST construct the synthetic cap such that this check passes when the caller's authority covers the role's grants — the conventional construction is `expires_at = MIN(parent.expires_at, role.ttl, caller_capability.expires_at)` taking only the defined values into the minimum (omit absent values from the minimum; result is `null` only if all three are absent).

If no explicit lifetime is set, the token inherits the parent capability's lifetime. Handlers SHOULD set explicit lifetimes to enable grant refresh cycles.

**Vacuous parent constraint (normative, per SI-23).** When the parent capability has no `expires_at` (infinite), the "MUST NOT exceed parent's `expires_at`" constraint is vacuous; the child MAY have any `expires_at` (finite or none). This is the natural consequence of "infinity dominates" — no finite child can exceed an infinite parent. Deployments that want to bound infinite-parent children to a default lifetime do so via deployment policy, not via the spec.

**Why item 4 is the typical-case bound (informative).** In identity-aware deployments, the caller of `system/role:assign` is typically the operational key holding a finite-expires `local-peer→operational-key` cap (refresh-cycle hygiene per `EXTENSION-IDENTITY.md`). That finite caller cap propagates as an upper bound on every role-derived cap minted under it. Item 1 (cap-chain parent) is the issuing handler's grant, which may be infinite or finite depending on the handler's own provisioning; item 4 is the caller's authority, which is finite in operational-key-driven flows. Both fire; the MIN is what governs. v1.6 missed this distinction; v1.7 makes it explicit so impls converge on the same expiry source set rather than each picking a subset.

### 5.4 When Derivation Happens

Grant derivation is not automatic. It is triggered by the handler that creates the role assignment. Common triggers:

| Trigger | Handler | Action |
|---|---|---|
| Peer joins a group | Group handler | Creates assignment + derives grants |
| Peer connects with known assignment | Bootstrap logic | Reads existing assignment + derives grants |
| Role definition changes | Continuation (convention) | Re-derives grants for all assignees |
| Peer promoted/demoted | Handler changing role | Deletes old assignment, creates new one + derives grants |

The role extension does not define when derivation happens — only how. The timing is determined by the handler that manages the context.

### 5.5 Re-derive Lifecycle (per IA9)

The `re-derive` op (§4.1, §4.2) re-issues role-derived tokens for **all assignees holding the role being re-derived** in the context. (v1.5 said "all assignees of a context" in places and "all assignees of the role" in others; v1.6 pins per-role per the request schema. Whole-context re-derive is a caller-side loop over the role definitions in the context.)

v1.3 left the lifecycle underspecified — no atomicity story, no grace window, no delivery mechanism, no in-flight EXECUTE semantics. v1.4 pinned the first four; v1.6 adds RL2-failure-mid-cascade and concurrent-ops semantics:

- **Ordered writes (MUST, default pinned to issue-first).** Per-assignee, the handler MUST issue T_new BEFORE revoking T_old. The ordering is load-bearing for safety:
  - Issue persists, revoke fails → assignee holds both tokens briefly. Over-authorization is bounded by the grace window. **Safe.**
  - Issue fails (no revoke yet) → assignee keeps T_old. **Safe.**
  - Revoke-first (the rejected default): revoke persists, issue fails → assignee stranded with no token. **Unsafe.**

  The handler MUST persist T_new before T_old's revocation. Per-entity tree atomicity (provided by the entity store) makes each leg recoverable; **no rollback or multi-entity transactional bundle primitive is required.** Implementations MAY offer revoke-first as a deployment-configurable mode for security-critical contexts where stranding is preferable to overlap; default MUST be issue-first.

- **RL2 failure mid-cascade (MUST, skip-and-continue).** If RL2 fails for an assignee mid-cascade (the caller's capability no longer covers the resolved grants for that specific assignee — e.g., a peer-specific template made the grant exceed the caller's authority), that assignee retains T_old; the cascade continues for remaining assignees. Skip-and-continue is preferred over abort because abort would leave earlier-issued T_new in place without their corresponding T_old revocations. The `re-derive-result.re_derived_count` reports only the count of successful per-assignee re-issuances; failed assignees are reported via `skipped_grantees` (per the §4.2 schema).

- **Grace window (MAY).** Implementations MAY configure an overlap window during which T_old and T_new are both valid (default: 0). The mechanism for deferring T_old's revocation across the grace window is implementation-defined; behavior across implementation restarts mid-grace is implementation-defined. Deployments under tight security keep grace=0; deployments prioritizing availability extend. (Downgraded from v1.5's SHOULD per SI-11 — spec'ing a state model is premature when no impl needs non-zero grace; revisit when a deployment requires it.)

- **Delivery (MUST).** T_new MUST be delivered to the assignee via a named mechanism; "the assignee will figure it out" is not a conformant delivery story. Two mechanisms are spec-recognized:

  - **Pull-via-subscription on `system/role/{context}/assignment/{peer_id_hex}/...`** (DEFAULT). The assignee subscribes to its assignment subtree at peer-config time; on `re-derive`, the new assignment entity (containing T_new metadata or a reference) appears in the subtree via standard tree-sync, and the assignee's runtime peer picks it up on the subscription's next tick. Works in any deployment that has sync extension installed; no additional dependency. Use this unless you have a reason not to.
  - **Inbox push to the assignee's inbox at `system/inbox/{peer_id_hex}/...`** (OPT-IN). For deployments that have `EXTENSION-INBOX` installed and prefer push semantics over pull. T_new is enqueued onto the assignee's inbox; the assignee processes the inbox entry to obtain T_new. Adds a runtime dependency on the inbox extension; offers lower latency at the cost of that dependency.

  Implementations MUST document which mechanism they use in their conformance report. Implementations MAY support both (deployment-configurable per role context) provided each named mechanism satisfies the lifecycle invariants below; selection is operational, not protocol-level.

- **In-flight EXECUTE semantics.** EXECUTEs in flight using T_old during re-derive return **401 `capability_revoked`** (fail-fast); the client retries with T_new. This is the expected client retry pattern.

**Multi-operational concurrent re-derive (per IA9).** With concurrent multi-operational deployments (per `EXTENSION-IDENTITY.md` §11.6 — multiple operational keys live concurrently under the same quorum), two operational keys may both trigger `re-derive` for the same role context concurrently. Each re-derive runs its own per-assignee ordered-write flow independently. Tree CAS resolves at the assignment-entity level (last-write-wins on the assignment metadata if both operational keys update the same path). Resulting tokens are independent — both may be issued; the assignee receives whichever delivers first; both pass RL2 if either operational key's authority covers the role definition. Implementations MUST NOT serialize concurrent re-derives across operational keys; doing so would block multi-operational concurrency.

**Concurrent ops and read-time semantics (normative, per SI-13).** Re-derive snapshots the role-definition state at handler entry and cascades against that snapshot. An `:assign` or `:unassign` that lands during the cascade is independent: the new op reads role-def at its own handler entry time and acts against whatever state is current there. Concurrent re-derives MAY overlap; each runs its own per-assignee loop independently against its own snapshot. Last-writer-wins applies at the per-assignment-entity tree CAS level; multiple T_new tokens may be issued concurrently, with the latest-arrived prevailing at the assignee per delivery order.

**Conformance.** MUST for write-order (issue-first default; revoke-first available as deployment-configurable mode); MAY for grace window (downgraded from SHOULD per SI-11); MUST for delivery mechanism naming; MUST for non-serialization across concurrent operational keys; MUST for skip-and-continue on RL2-failure-mid-cascade. Implementations MUST NOT require a tree-level transactional-bundle primitive; per-entity atomicity plus pinned write-order is sufficient.

### 5.6 Member-to-Member Delegation Lifecycle (per IA22)

The `delegate` op (§4.1, §4.2) lets a member B delegate his role-authority within a context to another peer C while B retains the role assignment. Use cases include vacation coverage, role-handoff during transitions, and ad-hoc temporary delegation. Composing delegation through role keeps it within the role lifecycle model — exclusion, re-derive, and layer-1 sweep all work on delegation caps the same way they work on role-assigned caps. Doing it via raw V7 caps would bypass role's lifecycle.

**Mechanics.**

0. **Locality invariant (normative, per SI-19).** `:delegate` MUST be invoked on the delegator's own runtime peer (`ctx.local_peer_id == ctx.execute.data.author`). Remote-caller delegation (caller is not the local peer) is rejected with **400 `delegator_must_be_local_peer`** — this is a precondition error (the request is malformed for the deployment's locality model), NOT a permission error (403). The error matches the shape of `400 malformed_resource` and `400 invalid_request` used elsewhere in this spec. The invariant ensures the handler can sign the issued delegation cap with the local keypair, which is the delegator's keypair by construction. Multi-tenant deployments compose differently — a multi-tenant server hosts multiple identities, each on its own runtime peer; each identity's `:delegate` runs on its own peer. Remote-caller delegation (a cap-submission flow where the delegator pre-signs and submits to a host peer for storage) is out of scope for v1.6; if a future use case requires it, it MUST be specified separately.

1. Caller B invokes `system/role:delegate` with `delegate: C_identity_hash`, `context`, `role`, `scope` (subset of role's grants), and optional `expires_at`. The delegator is read from `ctx.execute.data.author`; the `delegator` field has been removed from the request schema in v1.6 (per SI-21 — caller is implicit).
2. The handler verifies B (the author) holds the named `role` in `context` (B's assignment exists at `system/role/{context}/assignment/{B_peer_id_hex}/{role}`).
3. **Scope literal-only (normative, per SI-20).** The `scope` parameter MUST be literal — no template variables. The handler rejects `scope` containing `{context}` or `{peer_id}` substrings (or any other template variable) with **400 `scope_must_be_literal`**. Only the role-definition's grants undergo template resolution: the role def's grants are re-resolved with `{peer_id: B_peer_id_hex}` for the RL2 coverage check (verifying the delegator B's authority covers the scope), and again with `{peer_id: C_peer_id_hex}` for the issued delegation cap (binding the cap to recipient C). The `scope` parameter is the literal subset B is delegating to C.
4. **RL2 attenuation check applies to the delegator's authority, not the operational key's**: the `scope` MUST be an attenuation of the role's grants as held by B. A member can only delegate authorities they hold; they MAY attenuate to a subset.
5. **Parent selection (normative, per SI-22).** The handler resolves the delegation cap's `parent` field by reading B's linkage entity at `system/role/{context}/derived-tokens/{B_peer_id_hex}/{role}` (per §2.4 + §3.1). The parent is the role-derived cap referenced by the linkage entity for B's role assignment. If multiple linkage entities exist for the same (B, role, context) tuple (re-derive overlap; rare with grace=0 default), tie-break by the linked cap's `issued_at` field descending — the most recently-issued parent.
6. The handler resolves templates (`{context}`, `{peer_id: C_peer_id_hex}`) per §5.2, applies `scope` as the attenuating filter, and issues a delegation cap with `granter: B_runtime_peer`, `grantee: C_identity_hash`, `parent: <selected role-derived cap hash>`. The cap is rooted at B's runtime peer (not at the role handler's grant — this is what makes it "member-to-member"), and signed by B's runtime peer.
7. The delegation cap is persisted at `system/capability/grants/role-derived/{context}/{C_peer_id_hex}/{token_hash}` so layer-1 exclusion sweep (§6.1) and `unassign` revocation (§4.4) reach it like any other role-derived cap.

**Lifecycle composition.**

- **Exclusion of C in `context`.** Layer-1 sweep deletes the delegation cap at `role-derived/{context}/{C_peer_id_hex}/...`; layer-2 prevents re-delegation via `delegate` (the op consults `is_excluded(context, delegate)` before issuing).
- **Exclusion of B in `context`.** B's role assignment is removed; subsequent `delegate` ops by B fail (no role held). Existing delegation caps from B remain valid until they expire or are revoked through B's own role-derived cap chain (V7 `is_revoked` cascading from B's now-removed assignment-derived cap).
- **`unassign(B, role, context)`.** B's role-derived cap is revoked per §4.4; downstream delegation caps that chain from it become unverifiable on next use (ENTITY-CORE-PROTOCOL.md §5.5 chain validation fails — granter B's parent cap is no longer present).
- **`re-derive` of the role definition.** Re-derivation re-issues B's role cap; standard cascade applies — B's old cap is revoked, his outstanding delegation caps to C chain from the now-revoked parent and become unverifiable. B MUST re-issue delegations after re-derive if continued coverage for C is required.
- **`expires_at`.** When the delegation cap expires, V7 `is_revoked` rejects it on next use. No handler-side cleanup is required.

**Conformance.** MUST. RL2 attenuation against the delegator's authority (not the operational key's). The delegation cap MUST be rooted at the delegator's runtime peer; storage path MUST be the role-derived path so role-extension machinery reaches it.

**Delegate-ability is opt-in per role (normative, per v2.0 PR-8).** For a role to be delegatable via `:delegate`, the role definition's `grants` MUST include an entry covering `system/role:delegate` on `system/role/{context}/assignment/*`:

```
{
  handlers:   {include: ["system/role"]},
  resources:  {include: ["system/role/{context}/assignment/*"]},
  operations: {include: ["delegate"]}
}
```

(May be combined with the standard `assign` / `unassign` operations on the same grant entry — see GUIDE-ROLE.md §14.1's `manager` role example.)

The protocol-layer dispatch check (ENTITY-CORE-PROTOCOL.md §5.2 `check_permission`) verifies that the caller cap covers the EXECUTE target before the role handler runs. A role-derived cap whose grants do not include `system/role:delegate` cannot satisfy this check; the EXECUTE is rejected with **403 `capability_denied`** at the dispatch layer, before the handler logic (RL2 attenuation; locality SI-19; etc.) runs. This makes delegate-ability a deliberate role-level policy choice rather than an implicit property of every role.

In-process delegation tests that construct the handler context directly bypass the dispatch check and won't surface this requirement; wire-level fixtures will. Surfaced by Go's `acme_14_1_delegate_under_controller_cap` Tier 3 fixture; cross-impl test vector is TV-RD-DELEGATE-GRANT-REQUIRED in `VALIDATION-MATRIX-IDENTITY-FOUNDATIONS.md` §9.5.

---

## 6. Exclusion — Three-Layer Model

(R7 + R8 of v1.2.) Exclusion in v1.1 was framed as "denies access universally" but the v1.1 mechanism only worked where handlers explicitly opt in to checking. v1.2 makes the scoping honest: exclusion enforces at three distinct layers, each with different reach. Reading the model linearly: layer 1 is the strongest (token revocation; foundational); layer 2 catches new derivation paths; layer 3 is opt-in for handler-level enforcement of in-flight requests.

### 6.1 The three layers

**Layer 1 (primary): Token revocation.** When `exclude` is invoked, the handler revokes the excluded peer's role-derived tokens by deleting them from `system/capability/grants/role-derived/{context}/{peer_id_hex}/...` (per R4 storage path). V7's standard `is_revoked` check (§5.5) sees the missing tokens and rejects the cap on next use. This is the foundational mechanism — it works through V7's existing primitives without any new role-extension machinery.

**Layer 2 (secondary): Block new derivation.** Before any derivation path mints a new token for a peer, the exclusion subtree at `system/role/{context}/excluded/{peer_id_hex}` is consulted; if an exclusion entity is present, derivation is rejected. This applies to:

- The runtime `system/role:assign` op (per §4.3 Step 4b — `is_excluded(context, assignee)` check before RL2).
- The startup-time L0 derivation path (per §4.5): the SDK L0 path MUST consult the exclusion subtree before issuing root caps. Startup-time L0 is the only path that doesn't run through the role handler, so the check has to live on the SDK side. Implementations expose a single helper (e.g., `is_excluded(context, peer_id)`) that both the handler and the L0 path call.
- The `re-derive` op (per §4.2 R5): re-derivation MUST consult the exclusion subtree per assignment; excluded peers don't get re-derived tokens even if their role-definition changes.

The invariant: **no derivation path can mint a token for a peer with an active exclusion in that context, regardless of which entry point is used.**

**Layer 3 (tertiary, opt-in): Handler-level context-aware checks.** Handlers operating in a context-scoped namespace MAY check for exclusion before processing requests from a peer. This is convention; the role spec does not normatively require it. Group handlers SHOULD check; service-role handlers MAY; the generic `system/tree` handler does NOT. The check is application-layer policy:

```
; In a context-aware handler:
handle_op(ctx, params):
  if is_excluded(context, ctx.execute.data.author, ctx):
    return error(403, "peer_excluded_in_context")
  ; ... normal op handling ...
```

Layer 3 provides the strongest semantics ("excluded peer cannot do anything in this context, even with valid tokens") but only where handlers opt in. Without layer 3, a peer with already-issued (and not-yet-revoked) tokens can still use them at handlers that don't check exclusion. Layer 1 (token revocation) is what makes layers 2 and 3 a complete mechanism in practice.

See **§6.6** for the normative atomicity contract that closes the (`is_excluded`, `tree:put`) TOCTOU race in `:assign`, `:re-derive` per-assignee leg, and `:delegate` (target peer). Layer 2's ordering of the exclusion check before token derivation is a check-then-act pattern; §6.6 specifies the conformant ways to make it observably atomic.

### 6.2 Checking Convention

```
is_excluded(context, peer_id, ctx):
  exclusion = ctx.entity_tree.get("system/role/" + context + "/excluded/" + peer_id)
  return exclusion is not null
```

A single tree read per check. The cost is one hash lookup in the location index. Implementations SHOULD share this helper across the role handler, the startup-time L0 path, and any opt-in layer-3 callers.

### 6.3 Where Layer 3 Checks

Two options, both valid:

**Handler-level (RECOMMENDED).** Each handler that uses roles checks exclusion as part of its operation processing. The group handler checks `system/role/group/{name}/excluded/{author}` before processing group operations. A service handler checks `system/role/service/{name}/excluded/{author}` before processing service operations. The check is explicit in the handler code.

**Dispatch-level (future consideration).** The dispatch layer (core §6.5) checks exclusion before routing to the handler. This requires the dispatch layer to derive a context from the resource path — a mapping that is not yet defined. When context extraction is standardized (see §8.1), dispatch-level checking becomes viable. Until then, handler-level checking is sufficient.

### 6.4 Exclusion and Capability Tokens (post-R7)

The previous v1.1 prose said "exclusion does NOT invalidate capability tokens." v1.2 corrects this: **layer 1 of R7 deletes the excluded peer's role-derived tokens via the pinned storage path (R4)**, so V7's `is_revoked` check rejects them. Tokens issued OUTSIDE the role-derived subtree (e.g., direct-issued caps from a runtime peer to an excluded peer for some other purpose) are not affected; those are managed under V7's standard cap-revocation mechanics, not by `exclude`.

This means:
- A peer with role-derived tokens AND an exclusion entry: layer 1 has deleted the role-derived tokens; subsequent V7 `is_revoked` checks reject them. Effectively access denied.
- A peer with non-role-derived tokens AND an exclusion entry: layer 1 doesn't touch those tokens (they're outside the role's subtree). Layer 3 (if the handler opts in) catches them at the handler level. Without layer 3, the tokens remain usable at handlers that don't check exclusion.
- Removing the exclusion entity does NOT auto-restore role-derived tokens (they were deleted). Re-assignment via `system/role:assign` (or a `re-derive`, if appropriate) is required.

The honest scoping: exclusion is a strong context-level signal, but the strength of enforcement depends on which layers are in play. For most deployments, layer 1 + layer 2 are sufficient (token revocation + block-new-derivation). Layer 3 is the additional stiffening for context-aware handlers that want to refuse in-flight requests from excluded peers without waiting for token revocation to propagate.

### 6.4.1 Unassign Token-Revocation Flow (per IA12)

`unassign` and `exclude` are parallel revocation flows with different triggers. The unassign flow:

1. **Trigger.** `system/role:unassign` is dispatched against `system/role/{context}/assignment/{peer_id_hex}/{role_name}` (or the role-omitted form for all roles in the context).
2. **Remove assignment.** The handler removes the assignment entity at the resource path (§4.4).
3. **Locate role-derived tokens via the linkage entity (per SI-5 + §2.4).** The handler reads `system/role/{context}/derived-tokens/{peer_id_hex}/{role_name}` to recover the content hash of the role-derived cap issued for this (peer, role, context) tuple. Multiple linkage entities may exist if re-derive overlap leaves more than one live cap; iterate all and revoke each.
4. **Mark revoked.** Each identified token is revoked via ENTITY-CORE-PROTOCOL.md §5.1's standard mechanism — the tree binding at `system/capability/grants/role-derived/{context}/{peer_id_hex}/{token_hash}` is removed. V7's `is_revoked` check sees the missing token on next use and rejects.
5. **Remove the linkage entity.** The handler removes the linkage entity at `derived-tokens/{peer_id_hex}/{role_name}` after the tokens it referenced are revoked.

The exclusion flow (§6.1 layer 1) does the same step 3-5 sweep, but it is triggered by an exclusion entity rather than an assignment removal. Both flows converge on V7's standard `is_revoked` mechanism for downstream rejection. Specifying both explicitly avoids the v1.3 ambiguity where `unassign` was described as "revokes derived tokens (mechanism per §6.4)" while §6.4 only documented the exclusion flow.

### 6.5 Exclusion Propagation

Exclusion entities are regular tree entities. They propagate through the standard sync mechanism:

1. Authority writes exclusion entity to `system/role/{context}/excluded/{peer_id_hex}`
2. Emit fires locally
3. Sync propagates the entity to peers that sync the context prefix
4. Each peer that receives the exclusion entity stores it in their tree
5. Subsequent requests from the excluded peer are denied on any peer that has the exclusion

Propagation latency depends on sync topology. During the propagation window, some peers may not yet have the exclusion. This is the same consistency model as all tree data — eventually consistent via sync.

**Fleet-wide reactive sweep (per IA8 — MUST).** A runtime peer that receives an exclusion entity for `system/role/{context}/excluded/{peer_id_hex}` via tree-sync MUST sweep its local `system/capability/grants/role-derived/{context}/{peer_id_hex}/...` subtree and delete **any capability tokens bound at that subtree on this peer (regardless of which peer in the fleet originally issued them).** This makes layer-1 enforcement (§6.1) actually fleet-wide, matching the design intent: when Alice excludes Bob from context X **on `alice_desktop`**, the exclusion entity syncs to `alice_phone` and `alice_phone` MUST then sweep its own role-derived subtree for Bob — not just `alice_desktop`'s.

Without this reactive bridge, `alice_phone` would keep issuing/holding role-derived tokens for Bob even after `alice_desktop` excluded him, defeating fleet-wide exclusion. The sweep is local to each runtime peer's own role-derived subtree (peers do not sweep one another's subtrees); the synced exclusion entity is the trigger.

**Storage-path semantics (normative, per SI-7).** The role-derived path `system/capability/grants/role-derived/{context}/{peer_id_hex}/...` is the canonical location for role-derived caps under role's lifecycle management. The broad sweep operates on this path. Caps that should NOT be swept on context exclusion (peer-private grants outside role's lifecycle, application-specific cap subtrees, multi-extension shared grants) MUST be stored elsewhere — `system/capability/grants/<other-namespace>/...` for example. Storing non-role-derived caps under the role-derived path opts them into role's exclusion lifecycle; storing them elsewhere keeps them outside it. This is a deployment / extension-design choice, not a protocol-level distinction.

**Write-origin discrimination (normative, per SI-17).** Implementations MUST be able to distinguish role-handler-originated writes (the handler op landed and produced the entity) from out-of-handler writes (tree-sync arrival, direct `tree:put`) so the IA8 fleet-wide sweep can short-circuit on handler-originated writes (avoiding double-processing). Mechanism is implementation-defined. Two recognized patterns:
- **Write-context tag.** The local emit context carries a `handler_pattern` field; the sync hook checks `event.context.handler_pattern == ROLE_HANDLER_PATTERN` and short-circuits on match.
- **Idempotent sync hooks.** The sync hook re-runs the cascade on every relevant write but the cascade is idempotent (sweeping an already-empty subtree is a no-op); double-processing is harmless.

Implementations using the idempotent-hook pattern MUST document idempotency on the cascade operations.

**Conformance.** MUST. Cross-impl test vector: an exclusion entity arrives on a non-issuing runtime peer; that peer sweeps its own `role-derived/{context}/{peer_id_hex}/...` subtree and the issuing-side tokens that have already synced to it.

### 6.6 Atomicity of Layer-2 Exclusion Check (per PR-2 SEC-2; v2.0)

Implementations MUST ensure that, observable to any caller, an exclusion entity bound at `system/role/{ctx}/excluded/{peer_id_hex}` cannot coexist with an unswept role-derived capability at `system/capability/grants/role-derived/{ctx}/{peer_id_hex}/...`, regardless of `:assign` / `:re-derive` (per-assignee leg) / `:delegate` (target peer) request interleaving with `:exclude`.

The check-then-act pattern between the layer-2 `is_excluded` pre-check (§4.3 step 4b) and the role-derived cap `tree:put` (§4.3 step 7) is a TOCTOU race. A concurrent `:exclude` can complete its sweep AFTER the assign's pre-check passes but BEFORE the cap is bound, leaving the cap unswept. This is a privilege-escalation surface: an attacker timing `:assign` to coincide with `:exclude` retains a valid cap that the spec says they shouldn't have.

Implementations MUST close this race by any of:

- **(a)** Post-issue re-check + rollback on the three affected handler paths — `:assign`, `:re-derive` per-assignee leg, `:delegate` target. After the cap (and signature, linkage entity) `tree:put` completes, re-check `is_excluded`; if true, roll back the cap, signature, linkage entity, and (for `:assign`) the assignment entity, and return 403 to the caller.
- **(b)** Fine-grained locking around the (`is_excluded`, `tree:put`) compound operation.
- **(c)** Optimistic concurrency with retry on conflict.
- **(d)** Any other strategy producing the same observable outcome.

Implementations with additional derivation entry points (e.g., a hypothetical `:re-derive-bulk` variant) MUST extend the same atomicity guarantee.

**Reference benchmark (informative; Go).** `TestSEC2_AssignExcludeRace` in `entity-core-go/ext/role/security_test.go` runs 100 concurrent (`:assign`, `:exclude`) iterations × 5 outer runs = 500 race opportunities per execution. Pre-fix: 2/100 iterations leaked the race state. Post-fix (option a): 1000 race opportunities under `go test -race`, 0 leaks.

**Race-detector guidance per impl (informative).** Cross-thread interleaving without a race detector is insufficient for this conformance category. Go uses `go test -race`. Rust impls SHOULD exercise this property under `miri` and/or `loom` (concurrency-sequence search). Python impls SHOULD exercise under their concurrency runtime's thread-safety harness (asyncio, threading). The benchmark "0 leaks across N iterations under the impl's race detector" is the conformance shape; N is impl-defined but SHOULD be ≥ the iteration counts that previously surfaced the race in any peer-impl pre-fix.

**Conformance.** MUST. Cross-impl test vector: TV-RD-RACE-AE in `VALIDATION-MATRIX-IDENTITY-FOUNDATIONS.md` §9.

---

## 7. Initial Grant Convention

(Renamed from "Bootstrap Grant Convention" in v1.6 per SI-28. The convention is the same: use role definitions as templates for initial capability grants issued to connecting peers. The renamed "initial grant policy" of §4.7 is the deployment-policy surface for selecting WHICH initial role to use; this section is the MECHANICS for issuing the initial cap once selected.)

### 7.1 Role Definitions as Initial Grant Templates

A peer MAY use a role definition as the template for initial capability grants issued to connecting peers. This supports anonymous and public access patterns without per-connection role assignments.

```
On new connection:
  1. Determine applicable role:
     a. Check for existing assignment: tree.get("system/role/{context}/assignment/{peer_id_hex}/{role_name}")
     b. If found: use the assigned role
     c. If not found: consult the initial grant policy at system/role/initial-grant-policy
        (per §4.7) for the deployment's chosen handling of unknown peers.
  2. Read role definition: tree.get("system/role/{context}/{role_name}")
  3. Check for exclusion: tree.get("system/role/{context}/excluded/{peer_id_hex}")
  4. If excluded: issue minimal initial grant only (system/type/*, system/handler/*)
  5. If not excluded: derive grants from role definition (§5.1)
  6. Include derived grants in AUTHENTICATE response (core §4.4)
```

### 7.2 Default Initial Role

A peer's initial grant policy at `system/role/initial-grant-policy` (per §4.7) configures the default role for connecting peers that have no existing assignment. The `default_role` field on the policy entity names the role; the policy's mode (`anonymous-allow` / `anonymous-deny` / `recognize-on-attestation`) governs whether the default role is issued.

When a peer connects and no assignment exists for that peer in the configured context, the initial-grant logic uses the default role to derive initial grants per the policy. This enables public access without per-connection state.

### 7.3 No Per-Connection State for Anonymous Access

When using the initial grant convention for anonymous/public access:

- No `system/role/assignment` entity is created for the connecting peer
- The capability token is issued with the connecting peer's identity as grantee
- The token has a TTL (session-scoped or short-lived)
- When the connection drops and the TTL expires, the grant is gone
- No tree state accumulates for anonymous connections

This pattern supports public servers, social peers, and any scenario where read access should be available without explicit role assignment.

### 7.4 Resolver mechanism is SDK / deployment territory (per PR-5; v2.0)

The mechanism for connecting `system/role/initial-grant-policy` (per §4.7) and the §7.2 default-role lookup to the connection handler's grant resolution at AUTHENTICATE time is **implementation-defined**. SDK frameworks SHOULD provide a built-in role-aware resolver that reads the policy entity and consults the assignment / exclusion subtrees per the policy mode (`anonymous-allow` / `anonymous-deny` / `recognize-on-attestation`). Deployments without an SDK-provided resolver MUST configure their own resolver to realize the policy; otherwise the policy entity has no effect.

The role spec defines the policy entity, the policy modes (§4.7), the layer-2 exclusion check ordering (per §4.7 + §6.1), and the default-role lookup (§7.2). It does NOT define where the resolver code lives or how the SDK exposes the registration hook. The SDK-side surface is specified separately (Rev 6, in flight) and applies when ratified.

This means: a deployment's `system/role/initial-grant-policy` entity has effect only if the SDK or deployment provides the resolver wiring. A peer with the policy entity present but no resolver registered will fall back to whatever default the connection handler ships (typically static configured grants or open-access in dev mode), regardless of the policy's `unknown_peer` mode. Implementations SHOULD log a warning at peer startup if the policy entity is present but no role-aware resolver is registered.

---

## 8. Open Questions and Future Work

### 8.1 Dispatch-Level Exclusion Checking

Currently exclusion checking is handler-level (§5.3). If the dispatch layer (core §6.5) were to check exclusions, it would need to derive a context from the request:

- From resource path: `shared/team-alpha/doc.md` → context `group/team-alpha` (by convention or registration)
- From handler pattern: `system/group` → context derived from resource field

The context extraction rule is not yet defined. Options include:
- Convention-based: documented mapping from path prefixes to contexts
- Handler-declared: handlers register their context at handler registration time
- Explicit: the EXECUTE entity includes a context field (protocol change)

This is deferred to a future version. Handler-level checking (§6.3) is sufficient for all identified use cases.

### 8.2 Role Definition Mutability

A role definition can change after assignments exist. When the definition changes (via `system/role:define` per §4.1, IA11 — the default write path), the handler triggers a `re-derive` cascade for all current assignees of the role per §5.5's lifecycle (issue T_new before revoking T_old; grace window; named delivery; in-flight `401 capability_revoked`). Existing tokens are NOT silently left valid until TTL — they are revoked as part of the cascade per §5.5's ordered-write rule. The v1.3 framing ("existing tokens remain valid until TTL expiry" + "convention territory") is superseded by IA9 lifecycle pinning and IA11 handler-gated definition writes.

Implementations that prefer the looser write surface MAY observe role-definition tree mutations via tree-sync subscription on `system/role/{context}/{role_name}` paths and trigger `re-derive` on observation (per IA11 option (b)); the cascade itself follows the same §5.5 lifecycle. Either route — `system/role:define` (DEFAULT) or tree-mutation observer — MUST trigger re-derive; the difference is only in how role-definition writes are gated. See §9.6 for the security note.

### 8.3 Cross-Context Audit

Answering "what can peer X access across all contexts?" requires querying every context's assignment path. No single entity summarizes cross-context access. A future query extension could index role assignments by peer identity across contexts.

### 8.4 Bearer Capabilities

For public content distribution, bearer capabilities (grantee-less tokens) would simplify access — one published token instead of per-connection grants. This is an open question in the capability system (core spec), not specific to the role extension. The initial grant convention (§7) provides an alternative that works without protocol changes.

### 8.5 Cascading Exclusion

When a peer is excluded from a parent context (e.g., an organization), sub-contexts (departments, teams) may need exclusion as well. This extension does not define cascading behavior. A convention using continuations can chain exclusions across contexts (see REVIEW-GROUP-EXTENSION-ANALYSIS.md §30.8).

### 8.6 Multi-Role Token Selection (per SI-18)

When peer P holds multiple role-derived tokens for context X (one per assigned role), P presents whichever token covers the requested operation. **Token selection is a client-side concern** and is not addressed by the role extension. V7's verifier validates whatever token the client sends. Clients MAY implement smart-cap-selection logic (matching operation against `grants[i].operations.include`, preferring most-attenuated covering token, etc.), but that logic is local to the client and not part of any cross-impl conformance surface.

---

## 9. Security Considerations

### 9.1 Attenuation Invariant

Derived grants MUST satisfy the capability attenuation invariant (core §5.6). The issuing handler's capability is the parent. Each derived grant entry MUST be a subset of the parent's grants in all five dimensions. If a role definition contains grants that exceed the handler's own capability, the assign MUST fail with **403 `assigner_authority_insufficient`** (RL2 fail-closed per §4.3 / §5.1, IA10) rather than silently minting a partial token.

### 9.2 Exclusion Is Defense-in-Depth

Exclusion does not replace capability verification. It adds a context-level denial on top of the standard verification pipeline. A peer must pass both: capability chain verification AND exclusion check. Removing one layer does not bypass the other.

### 9.3 Exclusion Propagation Window

Between exclusion creation and sync propagation to all peers, some peers may not yet have the exclusion entity. During this window, the excluded peer may still access resources through peers that haven't received the exclusion. Short capability TTLs reduce this window — the excluded peer's tokens expire even without the exclusion.

### 9.4 Assignment Path Write Access

Write access to `system/role/{context}/assignment/*` is equivalent to granting authority within that context. A peer that can write assignments can assign itself (or others) any role, including roles with broad grants. The handler issuing grants still enforces attenuation (the derived grants cannot exceed the handler's own capability), but the assignment itself should be write-restricted to authorized peers.

### 9.5 Exclusion Path Write Access

Write access to `system/role/{context}/excluded/*` enables denial of access for any peer within the context. This is a powerful operation — equivalent to eviction. It should be restricted to context administrators (e.g., group leaders, service admins).

### 9.6 Role Definition Write Access

Role-definition write access is high-authority: a peer that can mutate `system/role/{context}/{role_name}` can silently change downstream grants for every assignee of that role. v1.6 routes role-definition writes through the `system/role:define` op (§4.1, §4.2):

- `system/role:define` performs RL2 at definition-write time — the caller's capability MUST cover the proposed grant set — and triggers `re-derive` cascade per §5.5.
- The clean security boundary: definition writes are gated by the role handler's `define` dispatch capability, restricted to the deployment's admin set.

Direct `system/tree:put` to `system/role/{context}/{role_name}` bypasses the handler, RL2, and the cascade. This is permitted by the capability system if a deployment grants such authority — but doing so opts out of role's lifecycle management. Per §1.3 / §1.5.2, deployments that want cascade through the role handler MUST NOT grant raw `system/tree:put` on `system/role/...` to entities that would write outside the handler. Implementations MAY observe direct tree-mutation arrivals on role-definition paths and trigger the equivalent `re-derive` cascade as a defense-in-depth fallback for misconfigured deployments — but this is OPTIONAL, not a conformance MUST. v1.5's "rejected at content validation" framing has been removed; there is no kernel-level rejection mechanism.

Role-definition write access SHOULD be gated by capability grants restricted to the deployment's admin set. The MUST is that any role-definition mutation through `system/role:define` triggers `re-derive` per §5.5; the rest is deployment policy.

### 9.7 Capability Discipline Replaces "Content Validation Rejection" (v1.6 reframe)

v1.5 used "rejected at content validation" framing for direct `tree:put` to role paths. v1.6 removes this framing entirely (per SI-10). The architectural reality:

- There is no kernel. `system/tree` is an extension. Capabilities are the only enforcement on raw tree writes.
- If a peer holds `system/tree:put` authorization on `system/role/...`, the write happens — bypassing the role handler, RL2, and the cascade. That's a deployment choice, governed by capability discipline.
- **Don't grant `system/tree:put` on role-extension paths to entities you don't trust.** Application-level grants SHOULD target `system/role:*` operations.

No spec-level rejection mechanism exists. No V7 amendment is planned for this purpose. See §1.5.2 for the framing rule.

---

## 10. Examples

### 10.1 Admin Management

```
; Role definitions
system/role/admin/operator → {
  type: "system/role",
  data: {
    name: "operator",
    grants: [
      { handlers:   {include: ["*"]},
        resources:  {include: ["*"]},
        operations: {include: ["*"]} }
    ]
  }
}

system/role/admin/auditor → {
  type: "system/role",
  data: {
    name: "auditor",
    grants: [
      { handlers:   {include: ["system/tree"]},
        resources:  {include: ["system/*"]},
        operations: {include: ["get", "list"]} }
    ]
  }
}

; Assignments
system/role/admin/assignment/{alice_id} → {
  type: "system/role/assignment",
  data: { role: "operator", assigned_by: <local_peer_id>, assigned_at: 1741700000000 }
}

system/role/admin/assignment/{bob_id} → {
  type: "system/role/assignment",
  data: { role: "auditor", assigned_by: <local_peer_id>, assigned_at: 1741700000000 }
}
```

Alice gets wildcard access. Bob gets read-only access to system paths. Both are in the `admin` context. No group involved.

### 10.2 Group Membership Roles

```
; Role definitions for team-alpha group context
system/role/group/team-alpha/member → {
  type: "system/role",
  data: {
    name: "member",
    grants: [
      { handlers:   {include: ["system/tree"]},
        resources:  {include: ["shared/{context}/*"]},
        operations: {include: ["get", "put", "list"]} },
      { handlers:   {include: ["system/tree"]},
        resources:  {include: ["system/group/{context}/*"]},
        operations: {include: ["get", "list"]} },
      { handlers:   {include: ["system/group"]},
        resources:  {include: ["system/group/{context}"]},
        operations: {include: ["status", "leave"]} }
    ]
  }
}

system/role/group/team-alpha/leader → {
  type: "system/role",
  data: {
    name: "leader",
    grants: [
      { handlers:   {include: ["system/tree"]},
        resources:  {include: ["shared/{context}/*"]},
        operations: {include: ["get", "put", "list", "delete"]} },
      { handlers:   {include: ["system/group"]},
        resources:  {include: ["system/group/{context}"]},
        operations: {include: ["*"]} }
    ]
  }
}

; Template {context} resolves to "group/team-alpha" at grant issuance time.
; Member gets: shared/group/team-alpha/* (read/write), system/group/group/team-alpha/* (read)
; Leader gets: shared/group/team-alpha/* (full), all group operations
```

### 10.3 Public Access

```
; Role definition for public context
system/role/public/reader → {
  type: "system/role",
  data: {
    name: "reader",
    grants: [
      { handlers:   {include: ["system/tree"]},
        resources:  {include: ["public/*"]},
        operations: {include: ["get", "list"]} }
    ]
  }
}

; Initial grant policy: use public/reader as default for connecting peers
; (per §4.7 — the policy is stored at the reserved path system/role/initial-grant-policy)
system/role/initial-grant-policy → {
  type: "system/role/initial-grant-policy",
  data: {
    unknown_peer: "anonymous-allow",
    default_role: "public/reader"
  }
}

; No assignment entities created — initial grant convention (§7) applies.
; Each connecting peer receives grants derived from the public/reader role.
```

### 10.4 Social Network

```
; Three tiers of access
system/role/social/public → {
  type: "system/role",
  data: {
    name: "public",
    grants: [
      { handlers: {include: ["system/tree"]},
        resources: {include: ["public/posts/*", "public/profile"]},
        operations: {include: ["get", "list"]} }
    ]
  }
}

system/role/social/follower → {
  type: "system/role",
  data: {
    name: "follower",
    grants: [
      { handlers: {include: ["system/tree"]},
        resources: {include: ["public/*", "followers/*"]},
        operations: {include: ["get", "list"]} }
    ]
  }
}

system/role/social/friend → {
  type: "system/role",
  data: {
    name: "friend",
    grants: [
      { handlers: {include: ["system/tree"]},
        resources: {include: ["public/*", "followers/*", "friends/*"]},
        operations: {include: ["get", "list"]} },
      { handlers: {include: ["system/tree"]},
        resources: {include: ["friends/wall/*"]},
        operations: {include: ["put"]} }
    ]
  }
}

; Friend list IS the assignment list
system/role/social/assignment/{bob_id}   → { role: "friend", ... }
system/role/social/assignment/{carol_id} → { role: "follower", ... }

; Anonymous readers: no assignment, initial grant from public role (§7)
; Bob: assignment exists → friend grants on connect
; Carol: assignment exists → follower grants on connect
; Unknown peer: no assignment → initial grant policy (§4.7) applies
```

### 10.5 Exclusion

```
; Exclude a peer from the team-alpha group context
system/role/group/team-alpha/excluded/{dave_id} → {
  type: "system/role/exclusion",
  data: {
    peer_id: <dave_id>,
    excluded_by: <leader_id>,
    excluded_at: 1741700000000,
    reason: "evicted"
  }
}

; Dave's existing capability tokens for team-alpha still pass chain verification.
; But any handler checking exclusion (§6.2) will deny Dave's requests.
; The exclusion syncs to all peers syncing system/role/group/team-alpha/*.
; Dave's assignment MAY be deleted as cleanup:
;   tree.put("system/role/group/team-alpha/assignment/{dave_id}", null)
```
