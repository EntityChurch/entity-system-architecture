# Architecture: Identity, Authority, and Deployment Infrastructure

**Version**: 1.0
**Status**: Active
**Authoritative scope:** Informative architecture reference. Normative content lives in the underlying specs (V7, EXTENSION-ATTESTATION, EXTENSION-QUORUM, EXTENSION-IDENTITY, EXTENSION-ROLE, EXTENSION-GROUP) and in `SYSTEM-IDENTITY-COMPOSITION.md` §6 (ownership boundaries).
**Audience:** Architecture reviewers; cross-impl reviewers; new readers needing the full identity-stack picture. Read this first when approaching identity/authority/deployment work.
**Scope:** Identity, role, group, multi-sig, registry, deployment patterns; how they compose into a flexible decentralized identity-and-authority system.

**How this doc relates to its neighbors:**
- `SYSTEM-ARCHITECTURE.md` — whole-system six-layer map (core protocol → composition → extensions → SDK → applications). This doc zooms in on the identity portion of that picture.
- `SYSTEM-IDENTITY-COMPOSITION.md` — short navigation doc for "how do I read the three identity-stack specs in order?" with normative §6 ownership boundaries. This doc gives the deeper architecture rationale, decision log, deployment progression, and invariants.
- `EXTENSION-ATTESTATION.md` / `EXTENSION-QUORUM.md` / `EXTENSION-IDENTITY.md` — the three normative spec files for the identity stack itself.
- `EXTENSION-ROLE.md` / `EXTENSION-GROUP.md` — extensions that compose on top of identity.

**Lineage:** This document graduated from a tracker proposal that captured the v2.0/v2.1/v2.2/v2.3/v3.0 unified-attestation rewrite cycle, the v3.2 substrate split, the v3.3 / v1.1 / v7.37 cross-impl-feedback batch, and the role v1.6 / v1.7 / v2.0 → v2.0 + Amendments 1-2 cycle. See §13 Document history for the full change log. The originating proposal artifact lives at `proposals/implemented/PROPOSAL-IDENTITY-INFRASTRUCTURE.md`.

---

## 1. What this document is

The architecture-side work on identity, authority, and deployment patterns has produced a substantial body of artifacts — multiple specs across the layered stack, several plans and reviews, deep explorations, and three user guides. This document is the comprehensive entry point. It captures:

- What the stack looks like end-to-end.
- Where each artifact sits and what state it's in.
- How the layers compose — the progression a deployment chooses through.
- What the architectural invariants are and what's preserved across changes.
- Where to start reading depending on what you're reviewing.

It is **not** a tracker of in-flight proposals; the landed v2.0/v2.1/v2.2 rewrite cycle and the landed v3.2 substrate split superseded the prior in-flight set; the architecture-side work for the substrate is now in a stable state on disk. Two queued activities remain: **cross-impl convergence** against the absorbed text (Go reference impl first; Python and Rust against v3.2 spec stack as conformance target — see §9), and the **EXTENSION-GROUP v1.5 sweep** (`proposals/PROPOSAL-EXTENSION-GROUP-V1.5.md` outline; full v1.5 lands AFTER substrate stabilizes in cross-impl). This document focuses on the stable picture, with brief lineage in §12 for readers who want to trace the path the design walked.

---

## 2. The architecture in one paragraph

The identity stack provides the structural primitives for users (and groups) to manage long-lived recoverable identities across multiple devices and contacts. Identity is modeled as **a peer graph rooted at a quorum, with certs (function-discriminated) conferring authority to peers** — a K-of-N quorum at the structural root; a controller certified by the quorum; agent peer keys (per-device daemons) attested by the controller; optionally a separate identifier peer for hygiene-rotation independence. Functions (controller, agent, identifier, app-defined) emerge from the cert attestations referring to a peer, not from a fixed layered hierarchy. Identity v3.2 is a **structured composition layer** over substrate primitives `EXTENSION-ATTESTATION` (signed-claim edge type) and `EXTENSION-QUORUM` (K-of-N node) — both reusable beyond identity. The capability layer (V7) handles per-request authorization. The role extension provides templated grant management. The multi-sig primitive lets K-of-N authorities sign capability roots at the cap layer. The group extension reuses the identity model for collective entities. The registry layer resolves human-shareable names to identities. Together these compose into a flexible, decentralized identity-and-authority system that scales from a single user with one device through families, businesses, and federated organizations to public-consensus structures.

---

## 3. The layered picture

```
┌───────────────────────────────────────────────────────────────────────────┐
│  Application layer                                                        │
│  (UIs, apps, business logic, application-specific extensions)             │
└───────────────────────────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────────────────────────┐
│  Discovery layer                                                          │
│  Registry resolution (name → identity + endpoints + attestations)         │
│  Eight-level continuum: out-of-band → petname → group-shared →            │
│  federated → DID:web → DHT → blockchain → hybrid                          │
└───────────────────────────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────────────────────────┐
│  Group layer (EXTENSION-GROUP v1.4; v1.5 sweep queued post-substrate-     │
│  stabilization to align path layout with the new top-level system/quorum) │
│  Collective identities; membership; lifecycle; subgroups; federation      │
│  Reuses identity's three primary entity types under group's namespace     │
└───────────────────────────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────────────────────────┐
│  Role / authority management layer (EXTENSION-ROLE v1.5)                  │
│  Templated grant derivation; multi-role per (peer, context);              │
│  three-layer exclusion model                                              │
│  Operational key as the typical caller; integrates with identity          │
└───────────────────────────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────────────────────────┐
│  Identity layer — three extensions (v3.2 substrate proposal stack):        │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ EXTENSION-IDENTITY v3.2 (composition layer)                         │  │
│  │ Function discrimination (controller / agent / identifier);          │  │
│  │ cert-chain walking; handle-bearing distinction; recognition;        │  │
│  │ contact-side caching; recovery flows; publication modes;            │  │
│  │ sub-controller chains; identity-resolved signer registration        │  │
│  │ Two entity types only: peer-config + identity-binding               │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                  ↓ depends on                  ↓ depends on               │
│  ┌──────────────────────────────────┐  ┌────────────────────────────────┐ │
│  │ EXTENSION-QUORUM v1.0            │  │ EXTENSION-ATTESTATION v1.0     │ │
│  │ K-of-N node type (system/quorum);│  │ Signed-claim primitive         │ │
│  │ verify_k_of_n_signatures;        │  │ (system/attestation);          │ │
│  │ current_signer_set;              │  │ verify_signature, liveness,    │ │
│  │ quorum-update / quorum-publish   │  │ walks, lookups, revocation     │ │
│  │ as attestation conventions;      │  │ search; revocation as          │ │
│  │ pluggable signer-resolution      │  │ properties.kind="revocation"   │ │
│  └──────────────────────────────────┘  └────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────────────────────────┐
│  Capability layer                                                         │
│  V7 §5 — chain verification, attenuation, revocation                      │
│  Multi-sig primitive (K-of-N at root cap layer; PROPOSAL-MULTISIG)        │
│  NOTE: cap-layer multi-sig and entity-layer quorum are STRUCTURALLY       │
│  DISTINCT mechanisms in different validation paths. Three parallel        │
│  mechanisms total: V7 caps (incl. multi-sig caps) + system/attestation    │
│  + system/quorum entities. See §6.3.                                      │
└───────────────────────────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────────────────────────┐
│  Core protocol (V7)                                                       │
│  Entities, signatures, peer roots, capability mechanics                   │
│  ENTITY-CORE-PROTOCOL.md (current: v7.36; §1.5a Identity & Access      │
│  Management is the access-management entry point from the core; §1.2 /    │
│  §1.5 / §7.3 / §7.4 reframed peer-ID + hash format codes as multicodec-   │
│  style varints — Multikey Change A, wire-byte-identical for current      │
│  values)                                                                  │
└───────────────────────────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────────────────────────┐
│  Network layer                                                            │
│  EXTENSION-NETWORK — transport, NAT traversal, relays                     │
└───────────────────────────────────────────────────────────────────────────┘
```

Each layer is independently usable. Users who don't want the higher tiers fall back to V7 + caps directly. Users who need the full stack opt into each piece. The progression through the stack is the central design property — see §5.

---

## 3a. The structural matrix — identity × capability progressions

The system has **two orthogonal progressions** that together form a design-space matrix. Each row × column intersection is a specific design point. v3.2 identity fills one column; group v1.4 (v1.5 sweep queued) fills another; future extensions fill more.

### 3a.1 Identity progression (rows — wider scope of self)

1. **V7 peer** — single keypair; self-sovereign; no recovery
2. **Identity** — multi-key with quorum-rooted recovery (this extension; v3.2 — composition layer over `system/attestation` + `system/quorum` substrate primitives)
3. **Group identity** — collective; members/admins as quorum constituents (`EXTENSION-GROUP`)
4. **Cluster** — coordinated runtime fleet for HA / replication / leader-election (planned)
5. **Federation** — groups of groups (planned)
6. **Public-consensus** — global namespace (long-term)

### 3a.2 Capability progression (columns — wider scope of authorization)

A. **V7 local cap** — peer self-sovereign authorization
B. **Role assignment** — templated grant management (`EXTENSION-ROLE`)
C. **Identity-anchored cap** — authorization respecting identity structure (caps issued by agents under controllers under quorums)
D. **Group cap** — authorization within a group
E. **Cluster cap** — authorization within a cluster (planned)
F. **Federated cap** — authorization across federations (planned)

### 3a.3 The matrix

|  | A. V7 local cap | B. Role assign | C. Identity cap | D. Group cap | E. Cluster cap | F. Federated cap |
|---|---|---|---|---|---|---|
| **1. V7 peer** | ✓ | n/a | n/a | n/a | n/a | n/a |
| **2. Identity** | ✓ | ✓ | ✓ | n/a | n/a | n/a |
| **3. Group** | ✓ | ✓ | ✓ | ✓ | n/a | n/a |
| **4. Cluster** | ✓ | ✓ | ✓ | ✓ | ✓ | n/a |
| **5. Federation** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **6. Public** | (TBD) | (TBD) | (TBD) | (TBD) | (TBD) | (TBD) |

Each cell is a specific design point. Higher rows enable more capability columns. Some intersections are filled (V7 peer + V7 local cap); some are partial (group + role assign — group has its own role surface); some are unfilled (cluster + cluster cap is planned).

### 3a.4 What the matrix tells us

- **Identity (row 2) fills columns A-C.** It's the substrate for B and C (role assignments and identity-anchored caps both depend on identity structure).
- **Group (row 3) extends to column D.** Group reuses identity primitives (quorum + certs) and adds membership/governance.
- **Naming should span the matrix.** Terms used in identity (controller, agent, identifier) should read correctly when reused by group, cluster, etc. Where names break, it's a signal worth investigating.
- **Cluster, federation, public are future rows** — each cell in those rows is genuinely future work.

This matrix is the single picture that positions identity within the larger stack. It's also the orientation tool for new readers asking "where does X fit?" Future architectural work (cluster spec, federation patterns) should fill cells; cross-row sweeps should preserve naming consistency where the same concept appears in multiple cells.

See `EXPLORATION-IDENTITY-LENSES-AND-CONVERGENCE.md` §7 for the deeper analysis that produced this matrix.

---

## 3b. Identity-layer scope and adjacent extensions (informative)

A small set of concerns are intentionally OUT OF SCOPE at the identity layer because they're properly handled at higher / parallel layers. Mapping is informative — it lets readers understand where each concern actually lives in the broader stack rather than expecting EXTENSION-IDENTITY to cover everything identity-shaped.

| Concern | Identity-layer position | Where it's actually addressed |
|---|---|---|
| 4-key cross-fleet reachability | Out of scope at identity layer; identity quorum stays single-fleet (per `EXTENSION-IDENTITY.md` §9.4) | EXTENSION-GROUP / EXTENSION-CLUSTER handle multi-fleet topologies; cross-fleet identity-resolved validation defers to those layers |
| Federated multi-quorum identity | Out of scope; `peer-config.trusts_quorum` is a single hash | EXTENSION-GROUP provides federation patterns (groups of groups; cross-group sharing per the group spec) |
| Skipping the controller layer | Not a feature; chain is structural (quorum → controller → agent) | The minimal alternative is V7-only (no identity extension installed) — `peer-config` doesn't exist; no controller, no agent, no cert chain |
| Peer recognized by multiple unrelated users without separate agents | Out of scope; each agent is in one peer-config and trusts one quorum | Multi-binding (per `EXTENSION-IDENTITY.md` §11.5) is a different shape: same host machine runs multiple agents (one per identity); each agent is a distinct peer-id |
| Cross-identity certs at the cert layer | Not a feature at identity-cert layer | EXTENSION-ROLE provides cross-identity authority via templated grants; EXTENSION-GROUP provides cross-identity collective authority |

Future extensions adding scope (cluster, transaction, governance, VC, reputation, etc.) inherit this framing — when a new concern surfaces and "identity should handle this" is suggested, this table is the prompt to ask "does it actually fit at the identity layer, or is it the right shape for a layer above?"

---

## 4. The stack — current state

The "real" specs and supporting artifacts that implementations target. Versions reflect current state on disk as of the v2 rewrite cycle landing.

### 4.1 Core protocol — `ENTITY-CORE-PROTOCOL.md` v7.36

Capability mechanics, entity model, signatures, peer-rooted authority. Stable; small amendments arrive through proposals. v7.35 added §1.5a "Identity and Access Management" — the access-management entry point referencing the identity + role extensions as the canonical surface above the V7 floor. v7.36 (paired with the substrate-extraction landing) refreshed §1.5a to reference the three-extension layering (attestation + quorum + identity convention layer) and applied **Multikey Change A**: hash format codes (§1.2), peer-ID `key_type` and `hash_type` (§1.5), peer-ID derivation algorithm (§7.4), and a normative paragraph at §7.3 — all reframed as multicodec-style LEB128 varints with our private code subset. Wire-byte-identical for current values (Ed25519 = 0x01, SHA-256 = 0x01); forward-compatible for future allocations beyond 0x7F.

Pending core amendment:
- **Multi-sig primitive** (`PROPOSAL-MULTISIG-CORE-PRIMITIVE.md`) — adds polymorphic granter (single-sig | multi-sig) with root-only K-of-N capability roots. Design adopted; cross-impl implementation underway; V7 spec-side absorption of the polymorphic granter schema is the remaining step before the proposal moves to `implemented/`.

### 4.2 Multi-sig — `PROPOSAL-MULTISIG-CORE-PRIMITIVE.md`

K-of-N capability primitive at the V7 layer. Items M1–M12 + Amendments 1, 2 covering schema (polymorphic granter; root-only constraint), verifier (four sites in ENTITY-CORE-PROTOCOL.md §5.5 + helper), wire encoding, N ceiling, threshold attenuation (issuance-only), revocation (keypair-based, single mechanism), storage path. Used for joint accounts, escrow, dual control, threshold approval, ad-hoc organizational sign-off, personal multi-device user authority. Independent of identity — multi-sig serves use cases that need K-of-N at root cap level *without* any extension installed.

The architectural relationship to identity (per the corrected layering): `signer_resolution` is purely an extension-level interpretation. Identity-resolved quorums resolve to concrete keypair hashes at cap issuance time; the V7 multi-sig verifier sees concrete keypairs regardless of mode and applies M6's local-peer-in-signers rule unchanged. No core dependency on identity.

### 4.3 Identity layer — three extensions (v3.2 substrate proposal stack)

The identity layer is composed of three extensions per the active proposal stack:

#### 4.3.1 EXTENSION-ATTESTATION v1.0 — substrate primitive

Defines `system/attestation` (the edge type in the signed graph): single-sig validation, supersedes-chain walks, liveness checks, generic graph operations (parameterized by consumer-supplied predicates), revocation as `properties.kind = "revocation"` attestations. ~400-700 LOC per impl. The substrate that quorum and identity both build on.

#### 4.3.2 EXTENSION-QUORUM v1.0 — K-of-N node primitive

Defines `system/quorum` (the K-of-N node type): `verify_k_of_n_signatures`, `current_signer_set`, pluggable signer-resolution (`concrete` built in; `identity-resolved` registered by EXTENSION-IDENTITY at install). Quorum-update and quorum-publish are `system/attestation` entities with `properties.kind` conventions; quorum extension provides the K-of-N topology rule. ~250-400 LOC per impl.

#### 4.3.3 EXTENSION-IDENTITY v3.2 — composition layer

The identity-specific composition over the two substrate primitives. **No new entity types beyond `peer-config` and `identity-binding`** — identity attestations are `system/attestation` entities with identity-specific `properties.kind` values (`"identity-cert"`, `"identity-rotation-handoff"`, `"identity-rotation-recovery"`, `"identity-retirement"`).

#### 4.3.4 What v3.2 contains

Multi-peer identity management as a graph of peers connected by typed attestations. v3.2 is a **structured composition layer** over the substrate primitives — the substrate split (formerly v3.0's single bundled extension) is justified by reuse for other consumers (group, cluster, future VC / reputation / governance), and identity actively composes the substrates rather than orthogonally consuming them: it registers an `identity-resolved` resolver against EXTENSION-QUORUM at install time, defines four `identity-*` properties.kind values owned in the kind-ownership table, and orchestrates substrate ops + V7 cap-layer ops + sync-hook side effects in single user-visible flows.

**Identity-owned types (only two):**
- `system/identity/peer-config` — per-agent local state (`trusts_quorum`, `controller_grants`, `bindings`).
- `system/identity/identity-binding` — helper inner type (lives inside `peer-config.bindings`); records this agent's role in one identity (which handle cert, which agent cert).

The K-of-N quorum entity (`system/quorum`) is owned by EXTENSION-QUORUM. The attestation entity (`system/attestation`) is owned by EXTENSION-ATTESTATION. Identity references both; identity does not define new entity types for them.

**Identity-context attestation kinds (four; namespace-prefixed with `"identity-"`):**

| Kind | Sig topology | Purpose |
|---|---|---|
| `identity-cert` (with `properties.function`) | K-of-N from quorum (top-level controller); single-sig from issuing controller (sub-controller, agent, identifier, app-defined) | Confers a function (controller / agent / identifier / app-defined) to a peer |
| `identity-rotation-handoff` | dual-sig (old + new key) | Routine cert rotation when old key is still available |
| `identity-rotation-recovery` | K-of-N from quorum | Compromise-recovery cert rotation when old key is unavailable |
| `identity-retirement` | K-of-N from quorum | Retires a cert without rotation |

Identity also USES the universal `"revocation"` kind (per `EXTENSION-ATTESTATION.md` §3.3) by applying its authority-revocation rule (per spec §3.6 `identity_is_authorized_revoker`).

Quorum self-events (`quorum-update`, `quorum-publish`) live in `EXTENSION-QUORUM.md` §3.2 / §3.3 and are NOT identity-owned kinds; an identity's quorum membership changes by calling `QUORUM.system/quorum:update` directly.

**Required `properties` on identity-cert kinds:** `function` (controller / agent / identifier / app-defined); `mode` (`internal` / `public` / `per-relationship` / `embedded`) — REQUIRED on ALL identity-cert kinds (eliminates v3.0's runtime shape lookup; pinned at create-time per per-function valid-modes table).

**Seven handler operations:** `configure`, `create_quorum`, `create_attestation`, `supersede_attestation`, `revoke_attestation`, `publish_attestation`, `process_attestation`. External surface preserved from v3.0; internals delegate to substrate primitives. `configure` additionally registers `identity-resolved` signer-resolution mode against EXTENSION-QUORUM's hook.

**Identity-specific validators:** `identity_topology_for`, `identity_is_quorum_link`, `identity_is_authorized_revoker`, `identity_verify_cert` (orchestration entry point composing primitive helpers + identity rules).

**Storage paths:** `system/identity/internal/cert/{h}` / `system/identity/public/cert/{h}` / `system/identity/relationships/{c}/cert/{h}` per audience tier (mode-driven); `system/identity/contacts/{published_handle}/quorum-publish` for cached contact-quorum-publishes; `system/identity/peer-config` for per-agent local state.

**Sync surface:** dual-subtree (`system/identity/public/...` AND `system/quorum/{trusts_quorum}/...`) — both subtrees are exposed to contacts at TOFU.

**Key configurations** (per v3.2 §11):
- V7-only (floor) — no identity extension installed.
- 1-of-1 quorum — structurally complete; no recovery.
- **Three-key default** (recommended) — quorum + controller + agents. Controller IS the contact-face. Recovery + multi-device + stable cross-peer recognition.
- **Four-key advanced** — adds a separate identifier peer for hygiene-rotation independence (controller rotates without disturbing contacts).
- Multi-binding — one agent in multiple identities (one peer-config per identity, per §3.2).
- Concurrent multi-controller — multiple top-level controllers under one quorum.
- Sub-controller chains — recursive controller delegation; chain terminates at the quorum.
- App-defined functions — apps issue certs with custom `function` values for app-specific authority.
- Parent-managed — quorum constituents held by a delegate (parent / trustee / org); subject acquires self-custody over time via `quorum-update`.

### 4.4 Role — `EXTENSION-ROLE.md` v1.5

Templated grant derivation at the role layer. Role definitions live at `system/role/{context}/{role_name}`; assignments at `system/role/{context}/assignment/{peer_id}/{role_name}`; exclusions at `system/role/{context}/excluded/{peer_id}`.

**Key mechanics:**
- RL2 attenuation check via V7's existing `is_attenuated` primitive.
- Multi-role per (peer, context).
- Three-layer exclusion model: token revocation (primary) + block-new-derivation (secondary) + opt-in handler-level (tertiary).
- Bootstrap exemption with `parent: null` root caps via the SDK's L0 direct-store path.

**Identity composition:** the controller is the typical caller for role operations via the local peer→controller cap. Role assignments are persisted entities not signed by the controller directly; derived V7 caps are signed by the agent peer (ENTITY-CORE-PROTOCOL.md §5.5 line 1968 unchanged). Controller rotation does not invalidate existing role assignments or derived caps.

### 4.5 Group — `EXTENSION-GROUP.md` v1.4 (v1.5 sweep queued post-substrate-stabilization)

Collective identities. A group **is** an identity (mechanically) — it has the same peer-graph structure as an individual identity, with the group's quorum constituents drawn from members or admins rather than personal backup keys. v1.4 is the current canonical surface (full inline alignment with identity v3.0). v1.5 sweep is queued to align with the substrate split (`PROPOSAL-EXTENSION-GROUP-V1.5.md` outline at `proposals/`); v1.5 lands AFTER substrate stabilizes in cross-impl. v1.0–v1.3 history is preserved in §13 of the spec.

**Reuses substrate primitives directly:**
- The group's quorum is a `system/quorum` entity (per `EXTENSION-QUORUM.md`). In v1.4 it lives nested under the group namespace; in v1.5 it moves to the top-level `system/quorum/{q}` and the group entity holds a `trusts_quorum` reference (path layout change is the main v1.5 delta).
- The group's identity-context attestations are `system/attestation` entities (per `EXTENSION-ATTESTATION.md`) with identity's `properties.kind` conventions (`identity-cert`, `identity-rotation-handoff`, `identity-rotation-recovery`, `identity-retirement`) — same as personal identities, just scoped under the group.
- One peer-config per group runtime peer (per identity §3.2's per-runtime-peer-identity rule; see EXTENSION-IDENTITY.md §11.5).

**Group-specific entities:**
- `system/group` — the structural definition (name, governance, published_handle, created_at).
- `system/group/member` — one entry per (group, peer); records primary role + optional federation cascade.
- `system/group/subgroup` — parent group's attestation that a subgroup exists in its hierarchy.
- `system/group/acting-on-behalf-of-attestation` — scoped delegation entity (member acts as the group's voice within scope). Carries scope semantics distinct from generic identity-cert attestations; sits between identity attestations and V7 caps.

**Five governance patterns** (per spec §5): founder K-of-N, admin set, all-members, on-chain, hierarchical.

**Two signer reference modes** (per `EXTENSION-QUORUM.md` §5): `concrete` (default; rigid; member rotation requires explicit quorum-update) and `identity-resolved` (advanced; transparent member rotation through current controllers; resolver registered by EXTENSION-IDENTITY at install).

**Two acting-on-behalf-of patterns** (per spec §8): member acting independently with group context (Pattern 1 — casual) and group acting via member as messenger (Pattern 2 — formal).

### 4.6 Registry / Discovery — `PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE.md`

Eight-level registry continuum from out-of-band exchange to public consensus blockchain. Pluggable resolver pattern: `system/registry:resolve(name) → {public_identity, bootstrap_endpoints, attestations, trust_anchor}`. Lookup vs discovery distinguished. Group-shared registries depend on the group extension.

**Key amendments:** bootstrap_endpoints are mandatory; registry is the broadcast point for public state — the user updates once, contacts pull on their schedule.

### 4.7 Network — `EXTENSION-NETWORK.md`

Transport layer. NAT traversal, relays, connection management. Composes orthogonally with the identity stack (rotation flows compose with sender-rotation; cap-chain validity is independent of network sessions).

### 4.8 Guides — user-facing

- **`GUIDE-IDENTITY.md` v1.2** — alignment with identity v3.0 (cert/function reorganization). Walks the progression (V7-only → V7 + role → V7 + role + identity → multi-key advanced); provisioning ceremony; pairing; TOFU contact recognition; role composition; rotation flows; custody variants; pitfalls; cross-extension worked example. Sweep against v3.2 (substrate split) is queued post-cross-impl-pass; mechanically the guide's narrative is unchanged because the user-facing UX is identical, only the internal entity-type vocabulary differs.
- **`GUIDE-CAPABILITIES.md`** — capability foundation. Kernel-vs-handler principle, coherent-capability principle (R0/R1), three-capability subscribe model, per-extension coherent-cap checklist, admin-with-coherent-constraints vs full-admin distinction, grant progression (L0–L3), pitfalls.
- **`GUIDE-MULTISIG.md`** — consumer-facing multi-sig guide. Local-peer-in-signers rule, issuance-only threshold attenuation, four use-case walkthroughs, async signature gathering, parallel entity-level K-of-N comparison, six pitfalls.

---

## 5. How it composes — the progression

The system's access-management surface is layered. Users self-select the layer they need. Each tier opts in to additional properties at additional surface cost.

### 5.1 V7-only (floor)

One peer keypair = one identity. Caps signed by that peer; chain roots at that peer; TOFU at first contact. Single-device, acceptable-loss, no recovery, no rotation.

**Properties:** Per-peer authority works. Bob can grant Alice access to a resource using V7 caps directly.

**What's missing:** No notion of "the same Alice across devices." Lose the device, lose the identity. No hygiene rotation. No multi-device.

**Conformant.** The identity extension is not installed; this is a complete usable configuration for deployments that don't need the higher tiers.

### 5.2 V7 + role (minimum-viable usage)

Add the role extension for templated grant management. Role definitions, assignments, exclusions; multi-role per (peer, context); three-layer exclusion model.

**Properties:** Templated permission management. Connect two computers, set roles, modify from there.

**What's still missing:** No identity infrastructure. Same single-device, no-recovery floor as V7-only on the identity side; just better authority management.

**This is where most users who don't need recovery start.**

### 5.3 V7 + role + identity (default — three-key)

Opt into the identity extension to gain three properties:
- **Recovery** — the quorum can certify a fresh controller if the existing one is lost or compromised (`identity-rotation-recovery` attestation).
- **Multi-device recognition** — multiple agent peer keys all attested as the same identity.
- **Stable cross-peer recognition** — contacts cache the controller's key (3-key) or identifier's key (4-key); recognition survives agent-peer rotations within the identity.

**Three-key default:** quorum (N constituents, K-of-N threshold) + one controller + M agent peer keys. The controller IS the contact-face — contacts cache it directly (`mode="public"` on the controller cert). Recommended for most users opting into identity.

**The cost:** quorum custody ceremony (paper backup, hardware token, trusted holder, etc.) at provisioning time; rotation events involve K-of-N coordination among quorum constituents.

### 5.4 + Four-key advanced (separate identifier peer)

Add an identifier peer as a separately-certified handle. Controller rotates independently of contact recognition.

**Properties gained:** Controller rotation invisible to contacts. Hygiene-rotation cadence becomes deployment policy without contact cost. Defense-in-depth: controller compromise no longer immediately means contact-face compromise (the identifier peer is the cached handle).

**The cost:** one more keypair to manage; an `identity-cert` with `function="identifier"` under `internal/`; agent certs (`function="agent"`) signed by the identifier peer rather than the controller.

**When to choose:** high-cadence controller rotation policy; compliance / regulatory environments treating the controller as more sensitive; defense-in-depth on the contact-face is worth one more key.

### 5.5 + Group (collective identities)

A group is an identity that represents multiple individuals collectively. It has the same peer-graph structure as an individual identity (in v1.4 scoped under `system/group/{group_id}/identity/...`; in v1.5 the quorum moves to top-level `system/quorum/{q}` with the group entity holding a `trusts_quorum` reference), plus membership management and lifecycle on top.

**Properties:** Multiple individuals can act for the group. Governance patterns determine who can sign group operations. Subgroups support nested organizational structures. Acting-on-behalf-of attestations let members speak as the group's voice within scope.

**Composition:** group reuses identity primitives directly (no parallel entity types); group composes with role for in-group authority management; group composes with registry for group-name resolution and federation hops.

### 5.6 + Federation (cross-group sharing)

Two or more groups (e.g., Acme + Globex) form a partnership group. Members of either parent group can interact with the partnership infrastructure.

**Properties:** Cross-group identity sharing — per-member individual sharing (privacy-preserving; default) and bulk attestation sync (convenience; SHOULD for federations of meaningful size). Partnership members can verify caps from members of either parent.

**Federation conventions** fall out of group composition + registry; the wire-level federation hops are the registry layer's concern. No new identity-stack mechanism needed.

### 5.7 + Public consensus (long-horizon)

Community-governed identities anchored on a blockchain or other consensus mechanism. Designed at concept level (per `EXPLORATION-DEPLOYMENT-SHAPES.md §14.6`), not validated end-to-end; pilot scope. Bridge layer (oracle service, on-chain contract) is application territory.

---

## 6. Cross-extension invariants

These five hold across every layer and every change in the stack. Every future extension touching this surface MUST preserve them. They are codified as `EXTENSION-IDENTITY.md` v3.2 §12.

### 6.1 Root capabilities are locally authorized

ENTITY-CORE-PROTOCOL.md §5.5 line 1968 generalized: single-sig caps satisfy this when granter = local peer. Multi-sig caps generalize it to "local peer in signers AND signed." Identity preserves it: caps to contacts root at agent peer keys, not at controller keys, identifier keys, or quorums. Role preserves it: derived caps root at the issuing agent peer.

### 6.2 Controllers never appear in cap chains

Controllers sign attestations (controller certs they receive from the quorum or another controller authorize them; agent certs, identifier certs, and sub-controller certs they themselves produce). Controllers never sign V7 capabilities that enter cross-peer chains. Bob's peer never sees a controller's key as a granter in any cap chain it processes.

V7 caps are signed by agent keys; controllers authorize the agent to issue them via the local peer→controller cap. Future extensions implementing similar internal-authorization patterns MUST preserve this separation.

### 6.3 Three parallel mechanisms with no shared validator (UPDATED for v3.2 split)

After the proposal stack lands, three structurally distinct entity classes exist with three structurally distinct validators:

1. **V7 capability tokens** (validated by `verify_capability_chain` in ENTITY-CORE-PROTOCOL.md §5.5). Includes single-sig caps and multi-sig caps (per multi-sig core primitive). Lives in cap-chain machinery.
2. **`system/attestation` entities** (validated by EXTENSION-ATTESTATION's helpers in §4 + consumer-specific topology rules). Used by identity, group, future VC, reputation, provenance, audit, etc.
3. **`system/quorum` entities** (validated by EXTENSION-QUORUM's `verify_k_of_n_signatures`). Used by identity, group, future cluster, transaction, governance, etc.

**No shared validator.** Cap-chain machinery NEVER opens an attestation or quorum entity. Identity validators NEVER call into V7's `verify_capability_chain`. Quorum's K-of-N validator NEVER processes a cap or identity-context attestation as anything other than the entity hash + signers being validated. Validation paths are structurally distinct.

The mechanisms share signature-counting at a low level (any K-of-N validator reduces to "do K of N hashes have valid signatures over this target?"). This is an implementation refactor opportunity, NOT a spec-level unification opportunity. The type-level distinction must be preserved.

**Future extensions needing K-of-N MUST pick one of the three existing mechanisms** — V7 multi-sig at the cap-issuance layer, EXTENSION-QUORUM at the entity-attestation layer, or compose attestation + custom topology — and not introduce a fourth. The three-parallel-mechanisms invariant is preserved.

**This is the v3.2 evolution of v3.0's two-mechanisms invariant.** v3.0 had: V7 caps (`verify_capability_chain`) + identity attestations (`verify_attestation`). v3.2 adds quorum as a third structurally-distinct entity type with its own validator, making the invariant three-parallel-mechanisms. The principle (separate entity types, separate validators, no shared path) is unchanged; the count goes 2 → 3.

### 6.4 TOFU + supersedes for cross-peer attestations

Cross-peer trust is established by Trust On First Use at first contact; subsequent updates chain via `supersedes` references signed by the previous-authoritative party. Used for the contact-face handle, `device` attestations, and `contact-quorum` attestations.

Future extensions introducing cross-peer attestations MUST follow the same pattern: TOFU at first contact; supersedes chains for updates; signatures over content_hash by the previous-authoritative key (or quorum, for K-of-N kinds).

### 6.5 Bootstrap exemption is L0; runtime is dispatched

Operations that establish initial peer state (`configure`, the initial `create_quorum`, the initial `create_attestation` calls; group `form`) run via the SDK's L0 direct-store path — peer-owner authority, no dispatch authorization required, since no authority chain has been provisioned yet. After bootstrap, all operations require dispatched EXECUTEs with proper capabilities.

**Conformance is observed via post-state, not wire trace.** Bootstrap is internal to the peer; cross-impl tests verify the post-bootstrap tree state, not on-the-wire control bytes.

---

## 7. Architectural commitments (decision log)

Major decisions captured for traceability. Items here are stable architectural commitments; anything that contradicts them is a real architectural change requiring proposal.

> **Note on `EXTENSION-IDENTITY v2.2` pointers below.** Entries 1–21 cite the v2.2 spec sections where the commitment was originally formalized. v2.2 (and v3.0) is archived at `core-protocol-domain/specs/extensions/archived/EXTENSION-IDENTITY-v3.0.md`; the substance of all five cross-extension invariants is preserved in v3.2 (per `EXTENSION-IDENTITY.md` v3.2 §12). Pointers are not rewritten cell-by-cell because the historical traceability is the value of this log. Entries 22–28 use v3.2-era pointers directly.

| # | Decision | Where decided | Why |
|---|---|---|---|
| 1 | Identity extension is opt-in atop V7; V7 alone is a valid configuration | `EXTENSION-IDENTITY.md` v2.2 §1.1 / §11.1 | Simplest deployments shouldn't pay the higher-tier complexity cost |
| 2 | Identity is a peer graph — peer functions emerge from edges (attestations), not from a fixed layered hierarchy | `EXTENSION-IDENTITY.md` v2.2 §2.0 / §2.3; `EXPLORATION-IDENTITY-CONSOLIDATED-REVIEW.md` | The continuum runs from no-extension through 1-of-1 quorum / three-key default / four-key advanced / multi-binding / multi-operational. The unified abstraction supports all shapes via attestation kind discrimination |
| 3 | Operational keys never in cross-peer cap chains | `EXTENSION-IDENTITY.md` v2.2 §1.2 / §12.2 | Preserves ENTITY-CORE-PROTOCOL.md §5.5 line 1968 and keeps internal management invisible to contacts |
| 4 | Two parallel mechanisms — V7 caps for permission; identity attestations as non-cap entities | `EXTENSION-IDENTITY.md` v2.2 §3.6 / §12.3 | Different validation needs (chain vs entity-level kind-dispatched); never mix |
| 5 | Three-key default is the recommended shape for users opting into identity | `EXTENSION-IDENTITY.md` v2.2 §11.3; `GUIDE-IDENTITY` v1.0 §3 | Smallest reasonable surface giving the load-bearing identity properties (recovery, multi-device, stable cross-peer recognition); four-key advanced is opt-in |
| 6 | Multi-sig primitive is root-only (parent: null required) | `PROPOSAL-MULTISIG-CORE-PRIMITIVE` M3 | Polymorphic granter without polymorphic grantee can't satisfy chain linkage; mid-chain multi-sig deferred indefinitely |
| 7 | Local peer must be in multi-sig signers AND must have signed | `PROPOSAL-MULTISIG-CORE-PRIMITIVE` M6 | Generalizes ENTITY-CORE-PROTOCOL.md §5.5 line 1968; means joint accounts have per-participant caps not shared-cap-from-anywhere |
| 8 | `signer_resolution` is purely an extension-level interpretation; multi-sig core stays keypair-based | `EXTENSION-IDENTITY.md` v2.2 §3.1; `EXTENSION-GROUP.md` v1.3 §5.6 / §7.1 | Multi-sig is core-protocol-independent (joint accounts, escrow, etc. need K-of-N at root cap level without any extension); identity-resolved quorums resolve to concrete keypair hashes at cap issuance time |
| 9 | `kind="identity-cert"` with `function="agent"` (in v3.2; was `kind="runtime"` in v2.2 / v3.0) supports four publication modes: internal / public / per-relationship / embedded | `EXTENSION-IDENTITY.md` v2.2 §4.4 → v3.2 §4.2a | User chooses what to publish to whom; default is internal (privacy by default). v3.2 promoted `properties.mode` to all identity-cert kinds (not just agent) to eliminate runtime-shape lookup |
| 10 | Registry layer is the broadcast point for public-mode state | `PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE` Amendment 2 | User updates once, registry reflects, contacts pull. No per-contact ceremony for public changes |
| 11 | RL2 for role uses V7's existing `is_attenuated` primitive | `EXTENSION-ROLE.md` v1.5 §4.3 | No new core primitive needed; existing scope-vs-scope coverage suffices |
| 12 | Multi-role per (peer, context) supported in role v1.5 | `EXTENSION-ROLE` v1.5 R6 | Real use cases have multiple roles per peer; tokens compose; verification unchanged |
| 13 | Three-layer exclusion model in role | `EXTENSION-ROLE` v1.5 R7 | Honest about what each layer enforces (token revocation primary, block-derivation secondary, opt-in handler-level tertiary) |
| 14 | "cluster" reserved for `system/cluster` extension; identity uses "quorum" | `EXTENSION-IDENTITY` v2.2; `PLAN-EXTENSION-LANDSCAPE` | Disambiguates terminology between identity quorum (structural; cold; rare-mutation) and cluster (runtime-peer coordination; hot; live operations) |
| 15 | A group is a type of identity; reuses identity's three primary entity types under the group's namespace | `EXTENSION-GROUP.md` v1.3 §1.5 / §3.6 | Group authority composes via existing identity primitives plus membership/lifecycle layer; no parallel entity-type duplication |
| 16 | Tier separation (`quorum/`, `internal/`, `public/`, `relationships/`) describes external sync exposure, not which peers hold what | `EXTENSION-IDENTITY.md` v2.2 §5.2 | All of an identity's runtime peers hold the full subtree; the registry handler chooses what to expose externally |
| 17 | CBOR data-field tags forbidden across the protocol | `ENTITY-CBOR-ENCODING` §11; `PROPOSAL-MULTISIG` Amendment 1 M8 | Tagged/untagged values produce different ECF bytes, breaking content-addressing determinism |
| 18 | Group-quorum signers support both concrete-peer-ID and identity-resolved reference styles | `EXTENSION-GROUP.md` v1.3 §5.6 | Concrete is rigid (rotation requires explicit quorum-update); identity-resolved is transparent (members rotate independently). Different deployments need different trade-offs; both modes MUST per conformance |
| 19 | Cross-group identity sharing supports per-member sharing (MUST) and bulk attestation sync (SHOULD) | `EXTENSION-GROUP.md` v1.3 §7.4 | Per-member is privacy-preserving but doesn't scale; bulk sync scales but requires parent group infrastructure. Federation deployments typically combine both |
| 20 | The `quorum-publish` attestation (in v3.2; was `contact-quorum` in v2.2) supersede on quorum rotation is signed by the *previous* quorum | `EXTENSION-IDENTITY.md` v2.2 §4.6 → `EXTENSION-QUORUM.md` v1.0 §3.3 | The cached prior signer set is the contact's trust anchor; continuity-of-chain is the property compromise-recovery validation relies on. Applies to ALL supersede triggers (quorum-update, rotation-handoff, rotation-recovery), not only quorum-updates. Quorum-publish moved to EXTENSION-QUORUM in v3.2; the previous-quorum-signs-supersedes rule is now generic (any consumer caching prior quorum-publish state externally relies on it) |
| 21 | Compromise-recovery validation fails closed on missing cached `quorum-publish` | `EXTENSION-IDENTITY.md` v2.2 §9.4 → v3.2 §9.4 | Recovery rotations cannot be accepted on the strength of arbitrary signatures; the contact's cached prior signer set is the trust anchor. Privacy opt-out (no quorum-publish publication) means compromise-recovery falls back to out-of-band re-establishment |
| 22 | The system is fundamentally a signed-graph substrate; identity is one convention layer over it | `PROPOSAL-EXTRACT-ATTESTATION-PRIMITIVE.md` v1.0; `EXPLORATION-IDENTITY-LENSES-AND-CONVERGENCE.md` §4a.13–§4a.16 | Attestations are edges in a directed graph; nodes are hash-addressable entities (peers, quorums, content); quorum is a special node type with K-of-N signature semantics. Identity is a convention over the substrate; group, VC, reputation, cluster, transaction, governance, audit, provenance are other conventions over the same substrate |
| 23 | Three parallel mechanisms with no shared validator (extends commitment #4) | `PROPOSAL-EXTRACT-ATTESTATION-PRIMITIVE.md` §2.1; this doc §6.3 | V7 caps (incl. multi-sig) + `system/attestation` + `system/quorum` are three structurally distinct entity classes with three structurally distinct validators. No shared validation path. Future K-of-N-needing extensions pick one mechanism, not invent a fourth |
| 24 | Identity attestations are `system/attestation` with `properties.kind` discriminator (no identity-specific entity types) | `PROPOSAL-MINIMIZE-EXTENSION-IDENTITY.md` v3.2 §3.3 | v3.2 collapses 6-kind enumeration on `system/identity/attestation` into convention values on `system/attestation` (`"identity-cert"`, `"identity-rotation-handoff"`, `"identity-rotation-recovery"`, `"identity-retirement"`, `"revocation"`). Identity contributes 2 entity types only: peer-config + identity-binding |
| 25 | `properties.kind` is open vocabulary; correctness disambiguation is path-scoped | `PROPOSAL-EXTRACT-ATTESTATION-PRIMITIVE.md` §3.2 | Within an extension's storage paths, kinds MUST be unambiguous. Across extensions, kinds MAY collide; path-scoping disambiguates. Namespacing (e.g., `"identity-cert"`) is RECOMMENDED for global tooling clarity but not REQUIRED for correctness |
| 26 | Revocation is a generic attestation primitive (`properties.kind = "revocation"`), not a separate type | `PROPOSAL-EXTRACT-ATTESTATION-PRIMITIVE.md` §3.3 | Self-revocation is universal mechanism (any attester can revoke own attestation). Authority-revocation rules are consumer-specific (identity says quorum can revoke its own certs; reputation might say self-only). Cascade revocation is automatic via chain walking |
| 27 | Quorum self-events (update, publish) are `system/attestation` entities with quorum-extension conventions | `PROPOSAL-EXTRACT-QUORUM-PRIMITIVE.md` §3.2, §3.3 | Quorum extension owns the `"quorum-update"` and `"quorum-publish"` `properties.kind` values; identity (and other consumers) call quorum's operations directly rather than minting their own. Quorum-update K-of-N is signed by current quorum; quorum-publish supersedes signed by previous quorum (preserves cached-state validation chain) |
| 28 | Cert path resolution is purely a function of `properties.mode` (no runtime shape lookup) | `PROPOSAL-MINIMIZE-EXTENSION-IDENTITY.md` v3.2 §5.3 | v3.0's `is_handle_bearing_in_current_shape` runtime lookup is removed in v3.2. Mode is REQUIRED on all identity-cert kinds; cert author chooses at create-time per per-function valid-modes table. Eliminates the in-flight-rotation race that v3.0's runtime lookup created |

---

## 8. Validation surfaces

When the stack changes, regression-test against these concrete validation surfaces. They are the body of work that defined "what the architecture supports" — changes that break them are real regressions.

### 8.1 Concrete deployment validation

| Surface | What it validates | Where it lives |
|---|---|---|
| Single-user (Alice canonical) | Where every piece of identity data lives; sync flow within fleet; registry handler; bilateral peer cap network within fleet; rotation flows | `EXPLORATION-ALICE-INFRASTRUCTURE.md` |
| Couple (Alice + Bob with household_server) | Cross-individual cap exchanges; shared infrastructure ownership; per-relationship sync | `EXPLORATION-DEPLOYMENT-SHAPES.md §14.1` |
| Family (parents + kids) | Parent-managed identities; identity transitions as kids grow up; parental controls | `EXPLORATION-DEPLOYMENT-SHAPES.md §14.2` |
| Neighbors / community group | Group infrastructure on member's hardware; informal trust; member departure | `EXPLORATION-DEPLOYMENT-SHAPES.md §14.3` |
| Small business (Acme) | Group governance; member roles; compromise / departure; cross-individual member ↔ org dynamics; operational-key rotation interaction with rigid concrete-peer-ID quorums | `EXPLORATION-DEPLOYMENT-SHAPES.md §14.4` |
| Federation (Acme + Globex partnership) | Cross-group identity sharing; member attestation propagation; partnership lifecycle | `EXPLORATION-DEPLOYMENT-SHAPES.md §14.5` |
| Public consensus (CommunityDAO) — *designed, not validated; pilot scope* | On-chain governance; oracle layer; member attestation from on-chain state; treasury bridge | `EXPLORATION-DEPLOYMENT-SHAPES.md §14.6` |
| Public blog (Bob + anonymous Carol) | Anonymous fallback; bootstrap default role; opt-in to recognized contact | `EXPLORATION-DEPLOYMENT-SHAPES.md §14.7` |
| **V7-only peer ↔ full-stack peer interop** | The additivity proof: a peer running only V7 (single keypair, no identity extension) can exchange caps with a peer running the full identity stack. Identity attestations are non-cap entities (ENTITY-CORE-PROTOCOL.md §5.5 chain validation never references them); TOFU on first cap; rotations propagate via attestation channel that V7-only peers can ignore | `EXPLORATION-DEPLOYMENT-SHAPES.md §14.8` |
| Open-source collaborator network | Informal-trust group with graduated participation (proposer → contributor → maintainer); voluntary participation; quorum threshold fragility under maintainer drift; forking via group-creation | `EXPLORATION-DEPLOYMENT-SHAPES.md §14.9` |
| Adversarial — lateral trust escalation | Compromised runtime peer's blast radius; cap chain integrity under hostile control; re-delegation bounded by RL2; lateral movement to other contacts bounded by attestation cache freshness | `EXPLORATION-DEPLOYMENT-SHAPES.md §14.10` |
| Compromise-window latency / RTO bounds | How fast revocation propagates through the fleet's bilateral cap network and through cross-peer attestation caches. Tight household: seconds. Loose federation: hours+. The architecture bounds the window via TTL and supersedes-chained attestations; deployments tune to RTO/RPO needs | `EXPLORATION-ALICE-INFRASTRUCTURE.md §13A.7`; `EXTENSION-IDENTITY.md` v2.2 §9 |
| Disclosure topology (10 use cases) | Per-peer attestation publication modes; minimum-disclosure principle in action | `EXPLORATION-IDENTITY-TOPOLOGY-AND-DISCLOSURE.md §2` |
| Deployment variants (canonical / isolated / encrypted relay / open relay / V7 fallback) | Different infrastructure trust models | `EXPLORATION-ALICE-INFRASTRUCTURE.md §11` |

### 8.2 Cross-cutting properties

**Minimum-disclosure principle.** The user publishes only the attestations needed for the actual exchanges they choose to have. The architecture provides primitives; users compose them. The architecture does NOT require publishing anything to be usable.

**Fall-back-to-V7 principle.** Users who don't want the identity stack just don't install it. V7 + caps work; one peer = one keypair = one identity. No multi-device, no recovery, no rotation. Acceptable for use cases that don't need the higher tiers. This is structurally enforced (identity attestations are non-cap entities; TOFU on first cap is unconditional).

**Configuration continuum.** Deployment supports a range from "don't install the extension" → "1-of-1 quorum" → "three-key default" → "four-key advanced" → "multi-binding" → "multi-operational" → "parent-managed" → "group memberships." Users self-select.

**Privacy modes for infrastructure.** Per `EXPLORATION-ALICE-INFRASTRUCTURE.md §11`: deployment variants include canonical (full-fleet sync; trusted infrastructure), isolated server (only syncs `public/`; pure receiver), encrypted relay (server can't read content), open relay (community-run untrusted relay), and V7 fallback (no identity ext).

**Bilateral cap network within the fleet.** Per `EXPLORATION-ALICE-INFRASTRUCTURE.md §13A`: each pair of an identity's runtime peers has bilateral caps composing with the role + identity machinery. Two patterns: simple fleet-template (one role definition + bootstrap policy auto-derives caps lazily on first contact) and complex explicit-bilateral (per-pair role assignments). Demand-driven; sparse network grows as the user uses peers.

---

## 9. What's next

### 9.1 Near-term

1. **Cross-impl convergence against the absorbed v2 stack.** Identity v2.2 (+ Amendment 1), role v1.5, group v1.3 are stable spec text. Implementation teams (Go, Python, Rust) refactor against it: collapse identity entity registry from 9 types (v1.x) to 3 + helper (v2); map kind-specific producer ops (rotate_operator, retire_operator, register_public_identity, etc.) into `create_attestation` / `supersede_attestation` / `revoke_attestation` / `publish_attestation` calls; add `process_attestation` as the read-side hook; align role and group implementations with the v1.5 / v1.3 absorptions; absorb the v2.2 vocabulary (`device` → `runtime` rename + `validate_attestation_structure` from Amendment 1) into the same refactor pass. Per v2.2 §10.4: ~3-5 days per impl in parallel.

2. **Multi-sig cross-impl convergence.** Multi-sig design is adopted; cross-impl implementation is in progress. V7 spec-side absorption of the polymorphic granter schema is the remaining step before `PROPOSAL-MULTISIG-CORE-PRIMITIVE.md` moves to `implemented/`.

3. **SDK proposal Rev 6.** `PROPOSAL-SDK-IDENTITY-INFRASTRUCTURE-ALIGNMENT.md` Rev 5 is currently stale-pending-v2.0 (renames Public→External, Op→Agent don't match v2.0 which dropped both entirely; helper signatures don't map to the seven-primitive op surface). Rev 6 needs to be written against the v2.2 surface; deferred per user direction until SDK becomes priority.

### 9.2 Medium-term

1. **`system/cluster` extension drafting.** Planned per `PLAN-EXTENSION-LANDSCAPE.md`. Depends on stable identity (now v2.2), role, group. Cluster is for runtime-peer coordination (HA, replication, leader election) — distinct from identity quorum.

2. **Validation of operational concerns** from `REVIEW-IDENTITY-ARCHITECTURE.md` §4.4 (bootstrap usability, cold-key-backup behavior, compromise-window timing) once first implementations land.

3. **Role v2 reconsideration** — only if implementation surfaces friction with role's current 3-type / 7-op surface. The unification opportunity exists (assignment + exclusion → unified binding with kind) but the case is much weaker than identity's was. Don't push preemptively. Per `EXPLORATION-IDENTITY-CONSOLIDATED-REVIEW.md` §10.1.

### 9.3 Long-term

1. Federation conventions (falls out of group composition; on-the-wire hops are registry concerns).
2. Consensus extension (community-governed; blockchain-anchored).
3. Registry backend extensions (DID:web, DHT, blockchain-specific).
4. Discovery extension (mDNS, peer-discovery).
5. NAT-traversal protocols (network extension).

### 9.4 Known constraints (active follow-ups)

Real architectural limitations whose resolution is deferred to specific future-amendment triggers. Tracked here so they don't get lost.

**Cross-fleet identity-resolved reachability.** Identity-resolved group quorums (per `EXTENSION-GROUP` v1.4 / v1.5 — `signer_resolution: "identity-resolved"` per `EXTENSION-QUORUM.md` §5.2) require the verifier to walk `member-identity → current controller` via the constituent's live top-level controller cert (`kind="identity-cert", function="controller"`). In the 4-key advanced shape these are at `mode="internal"` (under `system/identity/internal/cert/...`); in the 3-key default they are `mode="public"` (under `system/identity/public/cert/...`) — public certs are reachable to contacts via two-tier sync, but in 4-key the controller cert is internal and not externally reachable by default. When the verifier and constituent share infrastructure (the group's verifying peer runs on the constituent's hardware — typical when a group is hosted on a member's machine per group §7.5), reachability is native. When they don't (e.g., `partnership_server` verifies signatures from a member on their own laptop) AND the constituent uses 4-key, there is no formalized publication path for the internal controller cert to reach the verifier.

**Current state:** treat shared-infrastructure as the supported topology for identity-resolved mode under 4-key constituents. 3-key constituents work natively because the controller cert is `mode="public"` (reachable through two-tier sync). Two un-formalized convention paths exist for 4-key cross-fleet reachability (per-relationship publication of controller certs; embedded resolution proof attached to signatures) but cross-impl interoperability is not guaranteed. Deployments needing cross-fleet identity-resolved validation under 4-key should prefer dedicated per-group keys in `concrete` mode.

**Future-amendment trigger:** if production deployments need cross-fleet identity-resolved validation under 4-key, formalize one of the two convention paths via a follow-on proposal — extending `EXTENSION-IDENTITY` v3.2's per-relationship mode (currently primarily for agent certs) to also cover internal controller certs is the cleanest path. Documented in: future `EXTENSION-GROUP` v1.5 (operational requirement subsection); `EXTENSION-IDENTITY.md` v3.2 §4.2 + §4.2a (per-mode framing); `GUIDE-GROUP.md`.

### 9.5 This document's own future

When cross-impl convergence lands and the v2 stack is in production:
- Move this document from `proposals/` to `core-protocol-domain/specs/ARCHITECTURE-IDENTITY-INFRASTRUCTURE.md` alongside `SYSTEM-ARCHITECTURE.md` and `SYSTEM-COMPOSITION.md`.
- The status section gets shorter (current versions are the canonical surface; pending amendments collapse into history).
- Becomes the stable architectural reference for the identity / authority stack.

The graduation is editorial — content is already in architecture-overview shape; the move is a directory change.

---

## 10. Reading order for new reviewers

Different entry points based on what you're reviewing:

### 10.1 If you're new to the whole stack

1. This document (you're here).
2. `PLAN-EXTENSION-LANDSCAPE.md` — extension landscape and dependency order.
3. `EXTENSION-IDENTITY.md` v2.2 (the spec).
4. `EXPLORATION-IDENTITY-CONSOLIDATED-REVIEW.md` — the design statement explaining the v2 architecture decisions.
5. `EXPLORATION-ALICE-INFRASTRUCTURE.md` — concrete deployment walkthrough.
6. `EXPLORATION-DEPLOYMENT-SHAPES.md` — scaling shapes.
7. `EXTENSION-GROUP.md` v1.3 — group extension.

That's the architectural picture end-to-end.

### 10.2 If you're reviewing the identity work specifically

1. `EXTENSION-IDENTITY.md` v2.2 — the canonical spec.
2. `EXPLORATION-IDENTITY-CONSOLIDATED-REVIEW.md` — the design statement.
3. `GUIDE-IDENTITY.md` v1.0 — the user-facing perspective.
4. `EXPLORATION-ALICE-INFRASTRUCTURE.md` — concrete validation.
5. `reviews/REVIEW-REQUEST-IDENTITY-V2.md` — what's pending team feedback.

### 10.3 If you're reviewing the multi-sig work

1. `EXPLORATION-MULTISIG-CORE-PRIMITIVE.md` — the design exploration.
2. `PROPOSAL-MULTISIG-CORE-PRIMITIVE.md` (M1–M12 + Amendments) — the proposal.
3. `GUIDE-MULTISIG.md` — consumer guide.

### 10.4 If you're reviewing the role work

1. `EXTENSION-ROLE.md` v1.5 — the canonical spec.
2. `GUIDE-CAPABILITIES.md` — context (touches role mechanics; the identity-role integration depth lives here).

### 10.5 If you're reviewing the group work

1. This document for context.
2. `EXPLORATION-DEPLOYMENT-SHAPES.md` — what scales the group serves.
3. `EXTENSION-GROUP.md` v1.3 — the canonical spec.

### 10.6 If you're reviewing the registry/discovery work

1. `PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE.md` — the landscape.
2. `EXPLORATION-ALICE-INFRASTRUCTURE.md` §11–§12 — registry's role in the deployment.

---

## 11. Key explorations (active conceptual content)

- **`EXPLORATION-IDENTITY-CONSOLIDATED-REVIEW.md`** — the design statement for the v2.0/v2.1 unified-attestation rewrite. Walks the architectural shift from the four-layer hierarchy to the key-graph framing; SF1–SF16 dispositions; naming policy; identity + role positioning in V7. Read first when working in this area.
- **`EXPLORATION-ALICE-INFRASTRUCTURE.md`** — concrete deployment walkthrough. Where every piece of identity data lives, fleet sync, registry handler's role, network flow diagrams. Source of the audience-vs-distribution clarification (tier separation is about external sync exposure, not which peers hold what).
- **`EXPLORATION-DEPLOYMENT-SHAPES.md`** — eight deployment shapes from solo (V7-only) to public consensus (blockchain-anchored). For each: what's specific, what composes, what's deferred, failure modes.
- **`EXPLORATION-IDENTITY-TOPOLOGY-AND-DISCLOSURE.md`** — the rationale for publication modes. Walks ten use cases; concludes per-peer attestations with publication modes are the primitive (now landed as `kind="runtime"` modes in v2.2 §4.4; was `kind="device"` in v2.1).
- **`EXPLORATION-PEER-IDENTITY-MANAGEMENT.md`** — the synthesis exploration that fed the v1.x identity model. Conceptually superseded by the consolidated review (which extended it to v2's unified-attestation framing); kept as historical reference.

---

## 12. Lineage

This document originated as a v1.x living tracker (`PROPOSAL-IDENTITY-INFRASTRUCTURE.md`) capturing the in-flight identity-and-authority work — multiple proposals in flight, accumulated amendments, dependency tracking. The v1.x cycle produced:
- `EXTENSION-IDENTITY.md` v1.0 → v1.2 (via `PROPOSAL-IDENTITY-RECOVERY-VALIDATION` and `PROPOSAL-IDENTITY-ARC-FIXES`).
- `EXTENSION-ROLE.md` v1.2 → v1.4 (via `PROPOSAL-ROLE-V1.2` and the arc-fixes).
- `EXTENSION-GROUP.md` v1.0 → v1.1 (via `PROPOSAL-EXTENSION-GROUP` and the arc-fixes).
- `PROPOSAL-MULTISIG-CORE-PRIMITIVE` adopted; cross-impl in progress.
- The five cross-extension invariants codified.

The v2.0 fresh write — driven by recognizing identity is more naturally framed as a key-graph with kind-dispatched attestations than as a four-layer hierarchy — superseded the v1.x model. The architectural commitments held verbatim; the presentation overhauled (9 entity types → 3 primary; 11 ops → 7 generic; role-prescriptive vocabulary → cryptographic-structural). v2.1 absorbed cross-impl-feedback hygiene items (R1–R7).

Companion sweeps following v2.1 brought the stack to a uniform v2-aligned state:
- `EXTENSION-ROLE.md` v1.5 (vocabulary alignment).
- `EXTENSION-GROUP.md` v1.3 (full inline alignment; v1.2 partial-mapping bridge retired).
- `GUIDE-IDENTITY.md` v1.0 (full alignment; v0.4 partial-mapping bridge retired).
- `ENTITY-CORE-PROTOCOL.md` v7.35 §1.5a (Identity and Access Management entry point from the core); v7.36 refreshed §1.5a for the three-extension layering and applied Multikey Change A.

The architecture-side work for the substrate is now in stable state on disk. This document was rewritten for v2 to reflect that — the per-amendment tracker artifacts of the v1.x cycle were stripped; the comprehensive v2 architecture overview replaced them. v4.0 updated for the substrate split (attestation + quorum + identity v3.2 split landed spec-side); same-day amendments to the substrate specs absorbed Go team round-2 landed-review feedback (`is_quorum_id` definition, cache-invalidation contract, multi-context tie-break note). Pre-v2 history is preserved in the historical record (`proposals/implemented/` for absorbed proposals; archived spec versions; the consolidated review document).

---

## 13. Document history

- **v4.0 staleness sweep + Go team round-2 feedback absorbed:** Following the substrate-stack landing and Go team's `FEEDBACK-IDENTITY-FOUNDATIONS-LANDED-REVIEW.md` plus an internal architecture-team review pass: §2 architecture-in-one-paragraph rewritten with v3.2 vocabulary (controller/agent/identifier; convention-layer framing). §3 layered picture updated for v7.36 (Multikey Change A) and group v1.4 / v1.5-queued. §3a structural matrix updated to v3.2. §4.1 V7 prose extended with the v7.36 Multikey Change A details. §4.5 group section rewritten — describes substrate-primitives reuse (system/quorum + system/attestation) instead of v3.0's `system/identity/quorum` / `system/identity/attestation` paths; clarifies v1.4 nested vs v1.5 top-level path layout. §4.8 GUIDE-IDENTITY pointer corrected to v1.2 with v3.2 alignment status. §5.3, §5.4 progression sections updated to v3.2 vocab (controller / identifier; `identity-rotation-recovery` kind). §6.1, §6.2 cross-extension invariants updated to v3.2 vocab (controller/agent; controllers-never-in-cap-chains framing). §7 decision-log: header note added explaining v2.2 pointers refer to historical commitments still valid in v3.2; entries 9, 20, 21 explicitly track the v2.2-to-v3.2 vocabulary translation. §9.4 known-constraint cross-fleet identity-resolved reachability section reframed for v3.2 identity-cert vocabulary. §11 explorations list preserved (some references kept as historical lineage). §12 lineage, §13 history extended.

- **v3.0 vocabulary alignment + structural matrix:** Active-state references updated v2.2/v2.3 → v3.0. Layered picture (§3) identity-layer line updated for v3.0 vocabulary (controller/agents/identifier; sub-controller chains; app-defined functions). New §3a "Structural matrix" added — identity progression × capability progression matrix per `EXPLORATION-IDENTITY-LENSES-AND-CONVERGENCE.md` §7; provides single picture positioning identity within the larger stack. GUIDE-IDENTITY pointer updated to v1.2. EXTENSION-GROUP v1.3 still queued for v3.0 vocabulary sweep (substantial; tracked separately).

- **v2.2 vocabulary alignment:** Active-state references updated v2.1 → v2.2. Layered picture (§3) identity-layer line updated to v2.2 with peer-graph framing. Decision log entry 2 reframed around the peer-substrate framing (`v2.2 §2.0` substrate paragraph; "peer functions emerge from edges"). `kind="device"` → `kind="runtime"` and "key-graph structure" → "peer-graph structure" pass through prose; spec-section cross-references retain explicit version annotation when describing change history. Companion-sweep status updated: GUIDE-IDENTITY v1.0 → v1.1; EXTENSION-GROUP v1.3 still uses v2.1 vocabulary (sweep against v2.2 queued in the broader-terminology-review tracker). v2.0/v2.1 historical lineage preserved in §12. Reading-order pointers updated.

- **v2 rewrite:** Full rewrite reflecting the v2.0/v2.1 unified-attestation landing and companion sweeps (role v1.5, group v1.3, guide v1.0). Stripped per-amendment tracker artifacts of the v1.x cycle. Comprehensive architecture overview replaces in-flight tracker. Architectural invariants and decision log re-expressed in v2 vocabulary; commitment list updated to 21 entries (added: identity-is-graph, three-key-default, signer_resolution-extension-level, contact-quorum-supersede-by-previous-quorum, fail-closed-on-missing-contact-quorum). Cross-extension invariants list (5) preserved verbatim from v1.x — these are mechanism-truths that survive presentation changes. Reading order updated. Lineage section added. Targeted for graduation to `core-protocol-domain/specs/ARCHITECTURE-IDENTITY-INFRASTRUCTURE.md`.

- **v1.x cycle:** Originally drafted as a living tracker for the identity-and-authority work. Accumulated 17 amendments across the v1.x cycle covering: stress-test refinements, validation-surfaces formalization, hygiene passes, gap closures, multi-sig layering corrections (both directions), Go-team feedback integration, arc-fixes absorption (29 IA items across identity / role / group), comprehensive review pass. Full per-amendment history preserved in version control prior to the v2 rewrite. Key milestones from that cycle — the five cross-extension invariants codification, the configuration continuum, the audience-vs-distribution clarification, the absorption of stress-test gaps in their natural-home documents — carry forward verbatim into v2.
