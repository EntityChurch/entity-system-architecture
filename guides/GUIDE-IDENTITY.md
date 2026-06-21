# Identity: User Guide

**Status**: Active
**Audience:** Application developers and operators implementing or integrating Entity Core identity management. Assumes familiarity with capabilities (GUIDE-CAPABILITIES.md) and basic V7 concepts (peers, signatures, the tree).
**Spec reference:** `EXTENSION-IDENTITY.md` v3.3, `EXTENSION-ATTESTATION.md` v1.1, `EXTENSION-QUORUM.md` v1.1, `EXTENSION-ROLE.md` v1.5, `ENTITY-CORE-PROTOCOL.md` v7.37 §1.5a (Identity and Access Management) + §5 (capability system); orientation: `SYSTEM-IDENTITY-COMPOSITION.md`.
**Related guides:** GUIDE-CAPABILITIES.md, GUIDE-ATTESTATION.md, GUIDE-QUORUM.md, GUIDE-MULTISIG.md.

---

## 1. What identity is (and what it isn't)

V7 alone gives you per-peer authorization: each peer is its own root, signs caps for its own resources, and every cap chain bottoms out at a peer-identity entity. That's enough to grant Bob access to Alice's photos. It is *not* enough to answer "are these two peers the same Alice?" or "Alice's laptop was lost — is the new laptop still Alice?"

The identity extension fills that gap. It defines how a single user (or service) is recognized across multiple agents, how authority flows from a recovery anchor through an operational identity to those agents, and how external contacts (Bob) keep their address book pointing at the right Alice across rotations.

**Identity is a graph of peers.** The extension models identity as a graph of peers (Ed25519 keypairs, per V7 §7.3) connected by typed attestations: a K-of-N quorum at the structural root; an controller certified by the quorum; agents (one per device daemon) attested by the controller; optionally a separate identifier peer (in the multi-key advanced shape) attested by the controller for hygiene-rotation independence. The graph is the identity — there is no canonical "identity peer" sitting above the graph. Functions (operational, runtime, contact-face, quorum constituent) emerge from a peer's position in the attestation graph, not from a fixed type. See EXTENSION-IDENTITY v3.0 §2.0 for the substrate framing.

**It is a parallel attestation layer.** The extension's entities are non-cap `system/attestation` entities (the substrate type from `EXTENSION-ATTESTATION`) that identity validates at the entity level via `identity_verify_cert` — a topology-first orchestrator that composes attestation/quorum primitive helpers with identity's own predicates (`identity_topology_for`, `identity_is_quorum_link`, `identity_is_authorized_revoker`). They do not enter V7 chain verification. Cross-peer cap chains stay V7-clean; identity recognition happens through processing those attestations.

**Three-extension layering (substrate split).** Identity v3.3 is a convention layer over two substrate primitives (per `SYSTEM-IDENTITY-COMPOSITION.md`):

- `EXTENSION-ATTESTATION` v1.1 — defines `system/attestation` (the universal entity type), generic graph operations (`walk_attesting_chain`, `find_attestations_targeting`, `is_attestation_live`, supersession with predecessor-revival, `default_find_authorizing`, generic `revocation` kind).
- `EXTENSION-QUORUM` v1.1 — defines `system/quorum` (K-of-N node primitive), `verify_k_of_n_signatures`, `current_signer_set` with chain walking via `quorum-update` attestations, `quorum-publish` for state snapshots, `is_quorum_id`, pluggable signer-resolution (`concrete` default; identity registers `identity-resolved`).
- `EXTENSION-IDENTITY` v3.3 — convention layer. Owns only two types of its own: `system/identity/peer-config` (per-agent local state) and `system/identity/identity-binding` (helper). Everything else (certs, lifecycle events) is a `system/attestation` whose `properties.kind` identity has registered: `"identity-cert"` (with `properties.function`), `"identity-rotation-handoff"`, `"identity-rotation-recovery"`, `"identity-retirement"`. Quorum self-events (`quorum-update`, `quorum-publish`) belong to `EXTENSION-QUORUM`; identity is a consumer.

This guide focuses on the user-facing identity patterns. For the substrate primitives standalone, see `GUIDE-ATTESTATION.md` and `GUIDE-QUORUM.md`.

**Two parallel mechanisms.** V7 caps for permission chains. Identity attestations for identity trust. They never mix. When you look at any entity in this extension, you can ask: is it a V7 cap (used in chains, rooted at a peer) or an attestation (held as evidence, validated at the entity level)? Every entity in this guide has a clear answer.

What the identity extension does:
- Recognizes a recovery quorum (K-of-N constituent peers) for one user.
- Tracks which **controller** the quorum has certified (the user's hot key for internal management).
- Tracks which **agents** are authorized to act under the identity (per-device daemons).
- Optionally tracks a separate **identifier peer** that contacts trust as the identity's published handle (only in the four-key advanced shape).
- Issues *one* V7 cap per agent per live controller — the **local peer→controller cap** — that lets the controller drive that peer (per spec §3.2 `controller_grants` resolution rule).
- Provides primitive operations (`create_attestation`, `supersede_attestation`, `revoke_attestation`, `publish_attestation`, `process_attestation`, plus structural setup) for managing the graph.

What the identity extension does **not** do:
- Modify V7 chain verification — chains stay pure V7.
- Sign cross-peer caps — those are signed by agent keys (V7 §5.5 line 1968).
- Grant permissions to others — that's `system/role` (see GUIDE-CAPABILITIES.md §3 and EXTENSION-ROLE.md v1.5).
- Coordinate agents as a cluster service — that's `system/cluster` (planned, separate; despite the name overlap, "cluster" is reserved for that).
- Multi-user / shared identities — that's `system/group` (planned).

If your need is "Alice grants Bob access to her photos," you don't need this extension at all — V7 + role v1.5 cover it. You need this extension when you want **Alice to outlive any single key or any single device.**

---

## 2. The peer graph and common shapes

The extension models identity as a graph of peers connected by typed attestations. There are eight attestation kinds (covered in detail in `EXTENSION-IDENTITY` v3.0 §4); the user-facing story is which **shape** of graph you choose.

### 2.1 The progression

**V7-only (floor).** One peer, one keypair. No identity infrastructure. Single-device, acceptable-loss, no recovery. The identity extension is not installed. Conformant; this is the floor.

**V7 + role (minimum-viable usage of the system).** Add `system/role` for templated grant management. Connect two computers, set roles, modify from there. Still no identity infrastructure. Most users start here.

**V7 + role + identity (three-key default).** Opt into the identity extension. The shape:

- **A recovery quorum** — K of N constituents you keep in cold custody (paper, second device, hardware token, trusted holder).
- **An controller** — your hot key, certified by the quorum (the certification is a `kind="identity-cert", function="controller"` attestation). Signs internal management entities (peer-config, role assignments, agent certs). Contacts cache this as your handle.
- **One or more agents** — per-device daemon keypairs. Each agent is attested as one of your runtimes via a `kind="identity-cert", function="agent"` attestation signed by the controller.

This gives you: recovery (the quorum can certify a fresh controller if the existing one is compromised), multi-device (multiple agents all attested as the same identity), stable cross-peer recognition (contacts cache the controller's key; recognition survives any individual runtime-peer rotation).

**V7 + role + identity + multi-key advanced (four-key, opt-in).** Adds a separate **identifier peer** for hygiene-rotation independence — the controller rotates frequently (hygiene; per-incident; policy-driven) WITHOUT contacts re-validating; the identifier peer is what contacts cache and rotates rarely (only on compromise). The controller signs a `kind="identity-cert", function="identifier"` attestation establishing the binding; subsequent agent certs are signed by the identifier peer (not the controller), since contacts cache the identifier peer's key as the identity's handle.

Most users don't need this. It's an opt-in for users who specifically want operational-key rotation independent from contact recognition.

### 2.2 Three-key default vs four-key advanced — quick guide

| Property | Three-key default | Four-key advanced |
|---|---|---|
| Recovery from operational key loss | Yes (quorum certifies replacement; contacts process `rotation-recovery`) | Yes (quorum certifies replacement; contacts unaffected because the identifier's key is unchanged) |
| Routine operational-key rotation | Disturbs contacts (they process a `rotation-recovery` to update the cached handle) | Transparent to contacts (the identifier's key is the cached handle; only it rotates from contacts' perspective) |
| Multi-device recognition | Yes | Yes |
| Number of keys to manage (beyond agents + cold quorum) | 1 (operational) | 2 (operational + contact-face) |
| Spec sections | §3 default examples | §11.4 + §6.2 multi-key custody |

If you're not sure: **start with three-key default.** It covers recovery + multi-device + cross-peer recognition. Switch to four-key advanced only if you specifically want hygiene rotation that doesn't involve contacts.

### 2.3 Properties emerge from the graph

A peer's function in an identity is determined by the attestations referring to it, not by anything stored on the peer itself (per `EXTENSION-IDENTITY` v3.0 §2.3 — "function" is the identity-side noun; "role" is reserved for `EXTENSION-ROLE.md`):

- "P is the controller" ⇔ ∃ live `certification` attestation with `attested = P`.
- "P is a agent" ⇔ ∃ live `runtime` attestation with `attested = P`.
- "P is the identifier peer" ⇔ ∃ live `contact-face` attestation with `attested = P`.
- "Q is the recovery quorum" ⇔ Q is the structural root in `peer-config.trusts_quorum` AND ∃ live `quorum-publish` attestation with `attested = Q` (self).

This means: establishing a function is creating an attestation. Rotating a function-bearer is superseding an attestation. Retiring a peer is revoking an attestation. The mechanism is uniform across all four functions.

### 2.4 What stays cold and what's hot

| Layer | Heat | Used when |
|---|---|---|
| Quorum constituents (K-of-N) | Cold; K of N keys distributed across paper / hardware / secondary devices | Provisioning; quorum updates; certifying a new controller after compromise; recovery rotation |
| Operational peer | Hot, encrypted at rest | Daily internal management — peer-config writes, role assignments, attesting agents |
| Contact-face peer (4-key only) | Hot, encrypted at rest | Signing agent certs contacts care about; routine handoff rotation |
| Runtime peer keypairs | Hot, on each running device | Signing cross-peer V7 caps (every cap Bob receives) |

Most of the time, only agent keypairs are actively in use. The controller wakes up for internal management. The quorum emerges for provisioning, rotation, and recovery.

---

---

## 3. Provisioning a fresh identity

Alice opens the application for the first time. She's chosen to opt into identity (per §1's progression — V7-only would suffice for single-device acceptable-loss; V7 + role would cover templated grant management without identity infrastructure; identity is what she opts in for when she wants **recovery, multi-device recognition, or stable cross-peer recognition across rotations**). This section walks the **provisioning ceremony** — the one-time bootstrap where her identity's peer graph is constituted.

We lead with the **three-key default** (recommended for most users opting into identity, per `EXTENSION-IDENTITY` v3.0 §11.3). This is the recommended shape: it gives the load-bearing identity properties at the smallest reasonable surface. The four-key advanced variant — adding a separate identifier peer for hygiene-rotation independence — is covered at §3.5; consider it only if you specifically need that extra property.

**Be honest about what you're committing to.** Compared to V7 alone (one runtime keypair), three-key default adds: a quorum (typically 3 cold keys; default threshold K=2 of 3), one controller (hot, on Alice's primary devices), and the agent keypairs themselves (one per device). The user-visible cost is the quorum-key custody ceremony (paper backup, hardware token, trusted holder, etc.) — done once at provisioning, exercised only on rotation events. Most users find this manageable; users who don't should stay V7-only or V7 + role.

### 3.1 What the ceremony produces

After provisioning under the three-key default, Alice has:

1. **A quorum entity** — `system/quorum/{quorum_id_hex}` — listing the N constituent keys and threshold K (default: N=3 keys, K=2). Structural; not signed. Lives in the substrate-owned `system/quorum/` subtree (per `EXTENSION-QUORUM` §3.1, `EXTENSION-IDENTITY` v3.3 §3.1). The structural root of Alice's identity.
2. **A `kind="identity-cert", function="controller"` attestation** — at `system/identity/public/cert/{cert_hash_hex}` (3-key default; `properties.mode = "public"`) or `system/identity/internal/cert/{cert_hash_hex}` (4-key advanced; `properties.mode = "internal"`). K-of-N signed by quorum constituents; `attesting = quorum_id`, `attested = K_op_alice_hash`. This is the structural edge from the quorum to the controller, authorizing K_op_alice as Alice's controller's key (per v3.3 §4.2). **`properties.mode` is REQUIRED at create-time** — set per intended deployment shape; the cert's path is fixed for its lifetime.
3. **An operational keypair `K_op_alice`** — Alice-held, encrypted at rest. Hot key; signs internal management entities (peer-config writes, future agent certs). **In three-key default, K_op_alice IS the contact-face** — contacts cache it as Alice's handle directly; there is no separate `kind="identity-cert", function="identifier"` attestation in this configuration. (Four-key advanced introduces one; see §3.5.)
4. **A `kind="quorum-publish"` attestation** — at `system/quorum/{quorum_id_hex}/event/{cqa_hash_hex}` — K-of-N signed by the quorum; `attesting = quorum_id` (self), `attested = quorum_id` (self), `properties = {published_handle: K_op_alice_hash, signers: [...], threshold: 2}`. Owned by `EXTENSION-QUORUM` (§3.3); identity is a consumer. This is the **contact-visible** record of the quorum: the quorum subtree (`system/quorum/{trusts_quorum}/...`) is part of identity's published sync surface (per v3.3 §5.2 dual-subtree two-tier sync). Contacts cache the `quorum-publish` to validate compromise-recovery rotations against the cached signer set (per v3.3 §9.4).
5. **A `kind="identity-cert", function="agent"` attestation for this agent (the laptop)** — single-sig from K_op_alice; `attested = laptop_runtime_peer_hash`; `properties.mode = "internal"` by default. **Default mode is internal** (privacy by default per v3.0 §4.4 / §10.1); Alice promotes the mode later via `publish_attestation` when she shares the agent with a contact. The attestation is per-runtime-peer; one entity per agent per (mode, contact_id). Publication modes:
   - **internal** (default): `system/identity/internal/cert/{hash_hex}` — held only within Alice's fleet; not visible to any contact.
   - **public**: `system/identity/public/cert/{hash_hex}` — surfaced through Alice's registry layer (per `PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE.md`); discoverable by anyone who resolves Alice's published handle.
   - **per-relationship**: `system/identity/relationships/{contact_id_hex}/cert/{hash_hex}` — only the named contact sees it.
   - **embedded**: travels inside specific cap envelopes; never written to a tree path.
6. **Peer-config on this agent** at `system/identity/peer-config` — per-runtime-peer-identity local config recording: which quorum it trusts, what `operational_grants` K_op_alice holds on this peer, which identity-bindings this peer participates in (per v3.0 §3.2).
7. **A local V7 cap** — the *only* V7 cap the identity extension issues. Granter = this agent, grantee = K_op_alice, parent = null. K_op_alice uses this cap to make EXECUTEs on this agent for internal management.

That's it: three peer functions (quorum constituents / operational / agent) and three attestation kinds (`certification` / `quorum-publish` / `runtime`). Compare to four-key advanced (§3.5), which adds one more peer function (contact-face) and one more attestation kind (`contact-face`).

The **set** of Alice's agents is implicit in whichever `kind="identity-cert", function="agent"` attestations exist in whichever modes — there is no global "runtime-peer-set" entity. Different contacts may see different subsets depending on which attestations are in which publication modes.

### 3.2 The handler operation: `system/identity:configure`

The ceremony culminates in a sequence of identity-handler calls. The application generates keypairs, constructs entities, gathers signatures, and invokes the handler ops. Per `EXTENSION-IDENTITY` v3.0 §6, the relevant ops are:

1. **`create_quorum`** (§6.3) — persists the quorum entity.
2. **`create_attestation`** for `kind="identity-cert", function="controller"`, `kind="quorum-publish"`, and the initial `kind="identity-cert", function="agent"` attestation (§6.4) — each persists the entity at its canonical storage path after K-of-N or single-sig validation.
3. **`configure`** (§6.2) — finalizes peer setup, persists peer-config, issues the local V7 cap.

The `configure` call:

```
configure(
  trusts_quorum:        hash(quorum_entity),
  operational_grants:   [...],                          ; what K_op may do on this peer
  bindings:             [{                              ; one entry per identity this peer
                                                         ;   participates in
    handle_attestation:    hash(certification_att),    ; references the
                                                        ;   kind="identity-cert", function="controller" attestation
                                                        ;   in three-key default;
                                                        ;   the kind="identity-cert", function="identifier"
                                                        ;   attestation in four-key advanced
                                                        ;   (per v3.0 §3.4 polymorphic)
    runtime_attestation:    hash(device_att),
    label:                 "Alice (laptop)"            ; optional; for UI
  }],
  publish_contact_quorum: true                          ; default; set false for privacy opt-out
                                                         ;   (compromise-recovery falls back to OOB)
)
```

The handler validates everything and persists state. Concretely (per v3.3 §6.2):
1. Validates the quorum entity exists at `system/quorum/{trusts_quorum_hex}` (referenced via `EXTENSION-QUORUM`).
2. Enumerates live `kind="identity-cert", function="controller"` attestations referencing the quorum and validates each via `identity_verify_cert` — the topology-first orchestrator (v3.3 §3.6) that selects K-of-N (top-level certs), single-sig (sub-controller / agent / identifier certs), or dual-sig (rotation-handoff) per the kind+function tuple, then composes substrate primitives (`QUORUM.verify_k_of_n_signatures`, `ATTESTATION.verify_attestation_signature`, etc.) to do the work. **Note:** `identity_verify_cert` is the SOLE validator for identity attestations; it MUST NOT be confused with V7's `verify_capability_chain` (per v3.3 §2.2 three-parallel-mechanisms invariant — see §13.3).
3. (If `publish_contact_quorum: true`) Locates the live `kind="quorum-publish"` attestation; validates K-of-N from the quorum.
4. For each binding: validates `handle_attestation` is `kind="identity-cert", function="controller"` (three-key) or `kind="identity-cert", function="identifier"` (four-key); validates `runtime_attestation` is `kind="identity-cert", function="agent"` and references this agent.
5. Persists peer-config.
6. Issues one local V7 cap per live operational key (granter = this peer, grantee = K_op, parent = null).

The first `configure` call (and the initial `create_quorum` / `create_attestation` calls) run via the **bootstrap exemption** (v3.0 §6.9 / §12.5): they go through the SDK's L0 direct-store path with peer-owner authority — the local peer→K_op cap doesn't exist yet, so K_op has no way to authorize the call. After bootstrap, all identity-handler operations require K_op's authority via that local cap. Bootstrap conformance is observed via post-state (peer-config established, local cap issued, binding records present), not via wire trace.

This is the same bootstrap pattern as `EXTENSION-ROLE` v1.5 (R3): L0 access during initial provisioning, dispatched EXECUTE thereafter.

### 3.3 What "the application generates the keypairs" means in practice

For default provisioning on a desktop app:
- **K_op_alice** is generated and stored encrypted-at-rest with a passphrase.
- **The N quorum constituents** (default N=3) are generated. The application prompts the user to back them up — paper, second device, hardware token, trusted holder. The application can defer (and continue nagging) but the user *should* back them up before walking away. **Skipping this is the catastrophic-loss scenario** (see §13.5); UI should make the backup step prominent.
- **The laptop's agent keypair** is generated on the laptop and never leaves it.

For a CLI install, the application might dump cold keys to files and instruct the operator to move them. For a server install, the operator might generate cold keys offline and import the public halves only. The protocol doesn't care; key custody is application territory.

### 3.4 Subsequent agents

Provisioning creates the *first* agent's setup. Adding more agents later (Alice's phone, a server, a CLI session) goes through a different flow — see §4.

### 3.5 Four-key advanced — when and why to add a identifier's key

Three-key default gives Alice three properties: recovery (the quorum can certify a fresh operational key if K_op is lost or compromised), multi-device recognition (multiple agent keys all attested as the same identity), stable cross-peer recognition (contacts cache K_op_alice; recognition survives runtime-peer rotations within the identity).

It does not give: **operational-key rotation invisible to contacts**. In three-key default, K_op_alice IS the contact-face. Rotating K_op_alice (for hygiene; per-incident; policy) requires contacts to process a `kind="identity-rotation-handoff"` (routine, dual-signed by old + new) or `kind="identity-rotation-recovery"` (compromise, K-of-N from quorum) attestation to update their cached handle. That's fine for occasional rotation; it becomes friction if the deployment wants frequent operational-key rotation without disturbing contacts at all.

**Four-key advanced** adds one more keypair, **K_cf_alice (the identifier's key)**, and one more attestation kind, `kind="identity-cert", function="identifier"`:

- K_cf_alice is what contacts cache as Alice's handle. It rotates rarely (only on compromise of K_cf_alice itself).
- K_op_alice rotates as often as the deployment wants — the contact-face is unchanged; contacts don't see operational-key rotations at all.
- The connection between K_op_alice and K_cf_alice is a single `kind="identity-cert", function="identifier"` attestation under `internal/` (op-signed; not visible to contacts). Per v3.0 §4.5.
- `kind="identity-cert", function="agent"` attestations are signed by K_cf_alice (not K_op_alice) — contacts cache K_cf_alice as the handle, so devices need to be attested by what contacts cache.

**Tradeoffs:**
- **+** Operational key rotation is invisible to contacts; hygiene-rotation cadence becomes deployment policy without contact cost.
- **+** Defense-in-depth: K_op compromise no longer immediately means contact-face compromise. The compromise of K_op rotates internally without forcing a contact-side handle update.
- **−** One more keypair to manage and back up.
- **−** Slightly more complex pairing flow (the agent cert chain now involves K_cf as the signer rather than K_op directly).

**When to choose four-key advanced:**
- Deployments with high-cadence operational-key rotation as policy (e.g., quarterly hygiene rotation).
- Compliance / regulatory environments where the operational key is treated as more sensitive than the identifier's key.
- High-value identities where defense-in-depth on the published handle is worth the extra key.

**When to stay three-key default:**
- Most users. The friction of contacts processing a rotation-recovery on K_op compromise is acceptable for the simpler key custody. Hygiene-rotation can be infrequent without a separate contact-face.

**Mechanically**, four-key provisioning produces all of §3.1 plus a `kind="identity-cert", function="identifier"` attestation (op→cf) and substitutes K_cf_alice as the signer of `device` attestations. The configure call's `bindings[].handle_attestation` references the `kind="identity-cert", function="identifier"` attestation rather than the `kind="identity-cert", function="controller"` one (per v3.0 §3.4 polymorphic field).

---

## 4. Adding a agent

Alice has provisioned her identity on her laptop. Now she wants to add her phone.

### 4.1 The pairing flow

1. Phone generates its own runtime keypair (`peer_id_phone`).
2. Phone displays a QR code with `peer_id_phone` and its desired scope.
3. Laptop scans the QR. The user approves.
4. Laptop's identity handler — running as K_op_alice via the local peer→K_op cap — invokes `system/identity:create_attestation` to produce a new `kind="identity-cert", function="agent"` attestation: single-sig from K_op_alice (in three-key default; from K_cf_alice in four-key advanced — per `EXTENSION-IDENTITY` v3.0 §4.4); `attested = peer_id_phone`; `properties.mode = "internal"` by default. The existing `kind="identity-cert", function="controller"` and `kind="quorum-publish"` attestations carry forward unchanged — they describe the structural identity, which the phone is joining; nothing requires re-signing them on each new pairing.
5. Laptop transmits to phone: the quorum entity, the live `kind="identity-cert", function="controller"` attestation(s), the live `kind="quorum-publish"` attestation, and the new `kind="identity-cert", function="agent"` attestation. (In four-key advanced, also the live `kind="identity-cert", function="identifier"` attestation.) Optionally K_op_alice's encrypted private key (if Alice wants K_op present on phone) — or not (if she wants K_op to remain laptop-only).
6. Phone calls `system/identity:configure` (its own bootstrap-exempt L0 call) with `trusts_quorum`, `operational_grants`, and the binding (`handle_attestation` references the live `kind="identity-cert", function="controller"` attestation in three-key, or the `kind="identity-cert", function="identifier"` attestation in four-key; `runtime_attestation` references the new agent cert for `peer_id_phone`).
7. Phone's configure handler validates everything against the quorum, persists peer-config, and issues the local peer→K_op cap. *If* K_op's private key was transmitted, phone can act as K_op directly; *if* not, phone can sign cross-peer caps as itself but has no K_op-authority for internal management on phone (typical for web apps where you don't want K_op's key in the browser).
8. Phone subscribes to the quorum's attestation paths and the public-attestation paths so it receives future certification updates, quorum updates, and rotation entities via sync.

After step 4, the phone's `device` attestation propagates via Alice's fleet sync (it lives in `internal/` so contacts don't see it). When Alice later wants Bob to recognize the phone, she calls `system/identity:publish_attestation` to promote the attestation's publication mode to public, per-relationship for Bob, or embedded (§4.3). Bob's view of Alice's agents updates per the chosen mode's propagation mechanism.

### 4.2 Tiered sync (audience, not distribution)

**The path tiers describe external sync exposure, not which Alice peers hold what.** All of Alice's agents (laptop, phone, alice_server, etc.) hold the *full* `system/identity/` subtree AND the substrate quorum subtree at `system/quorum/{trusts_quorum}/...`. The `internal/` / `public/` / `relationships/` split (under `system/identity/`) plus `system/quorum/{trusts_quorum}/...` describes what Alice's outward-facing infrastructure (typically the registry handler running on her externally-reachable peer) exposes to which contacts. This is a common point of confusion; see `EXTENSION-IDENTITY` v3.3 §5.2.

| Path subtree | Held by | Visible externally to | Mechanism |
|---|---|---|---|
| `system/quorum/{trusts_quorum}/...` (substrate-owned) | All of Alice's agents | All contacts | Two-tier sync — part of identity's published surface (dual-subtree); contacts subscribe to find quorum entity + `quorum-update` / `quorum-publish` events |
| `system/identity/internal/...` | All of Alice's agents | None | Internal sync only — never crosses the fleet boundary |
| `system/identity/public/...` | All of Alice's agents | All contacts (via two-tier sync and/or registry resolution) | Standard sync-scope cap covering `public/`, optionally surfaced via the registry |
| `system/identity/relationships/{contact_id}/...` | All of Alice's agents | Only the named contact | Three-tier sync — contact-specific sync-scope cap on `relationships/{contact_id}/` |
| `system/identity/contacts/{contact_handle_hex}/quorum-publish` | Per-runtime-peer cache | None (it's a cache of contacts' state) | Populated by `process_attestation` on arrival of a contact's `kind="quorum-publish"` attestation; consulted by §9.4 fail-closed validation when processing incoming `identity-rotation-recovery` attestations |
| `system/identity/peer-config` | Per-runtime-peer local | None | Not propagated; per-runtime-peer-identity (see below) |

**Peer-config is per-runtime-peer-identity, not per-machine.** A single host machine MAY operate as multiple agents — e.g., Carol's home machine may run as `carol_desktop` (agent of Carol's identity) AND `garden_calendar_server` (agent of the garden group's identity). Each agent has its own peer-config in its own namespace and trusts its own quorum. Peer-configs do not share state across identities. (See §11.6 multi-identity host machine.)

Tiered sync isn't a new mechanism. Alice grants each contact the appropriate sync-scope caps:
- Bob gets `system/identity/public/` AND `system/identity/relationships/{bob_id}/`.
- Carol gets `system/identity/public/` AND `system/identity/relationships/{carol_id}/`.

The path layout makes audience separation impossible to get wrong by accident.

### 4.3 Publication modes for `kind="identity-cert", function="agent"` attestations

Each agent's `kind="identity-cert", function="agent"` attestation exists in one of four publication modes (per `EXTENSION-IDENTITY` v3.0 §4.4). Alice picks per peer per audience. The default for newly-created attestations is **internal** — privacy by default; Alice opts-in to expose.

| Mode | Path | Audience | Use cases |
|---|---|---|---|
| **internal** (default) | `system/identity/internal/cert/{hash_hex}` | Alice's fleet only | Private peers (a daily-driver laptop Alice doesn't want any contact to know about); newly-paired peers before Alice has decided where to share them |
| **public** | `system/identity/public/cert/{hash_hex}` | All contacts via the registry | Public-facing infrastructure (alice_server, alice_blog_server); anyone who resolves Alice's published handle can see it |
| **per-relationship** | `system/identity/relationships/{contact_id_hex}/cert/{hash_hex}` | One named contact only | Privacy-preserving sharing — Bob knows about alice_phone; Carol doesn't |
| **embedded** | inside cap envelopes (no tree path) | Holders of the cap | Time-bounded grants; hand-offs; situations where the cap is the only relationship |

A agent can have attestations in multiple modes simultaneously (e.g., public + per-relationship for selected contacts). The handler operation `system/identity:publish_attestation(attestation_hash, new_mode, [contact_id])` writes the entity to the appropriate subtree; multiple invocations with different modes coexist (per v3.0 §6.7). Mode promotion / demotion uses the same op; per-mode revocation is via `revoke_attestation` (§10).

### 4.4 What the new agent can do

After pairing, the phone has:
- A local cap from itself to K_op_alice. K_op can now drive the phone via dispatched EXECUTEs (assuming K_op's key is present somewhere — on phone or on laptop driving via cross-peer dispatch).
- A live `kind="identity-cert", function="agent"` attestation under Alice's identity. Bob's peer recognizes phone's signed caps as "from Alice" once the attestation reaches him via whichever publication mode Alice chooses.
- Awareness of the quorum. Future certification updates, quorum-updates, and rotation entities propagate to phone via sync.

What it doesn't have unless explicitly given:
- The identifier's key (in four-key advanced; in three-key default the contact-face IS K_op).
- K_op's keypair (only if Alice transferred it during pairing).
- Authority to issue caps the operational key hasn't authorized.

---

## 5. Connecting to a contact

Alice and Bob each have their own provisioned identity. They want to start sharing.

### 5.1 First contact — Trust On First Use

The first time Alice's peer encounters Bob's published handle, there is no chain of attestation Alice's peer can validate against. Alice's peer accepts the binding by **TOFU** (Trust On First Use): "this is what Bob says is his identity; I'll remember it."

How Alice obtains Bob's published handle is application territory:
- QR code exchange in person.
- A signed introduction from a mutually trusted third party.
- A directory service.
- Email / text / paste-in-UI.

Once Alice's peer has Bob's published handle, it resolves Bob through whatever registry layer Bob uses (self-hosted, DID:web, DHT, blockchain — see `PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE.md`). The resolution returns Bob's live `kind="quorum-publish"` attestation and any `public/`-mode `kind="identity-cert", function="agent"` attestations Bob has chosen to publish. Alice's peer TOFU-accepts these on first contact and caches Bob's `kind="quorum-publish"` attestation under `system/identity/contacts/{bob_handle_hex}/quorum-publish` (per `EXTENSION-IDENTITY` v3.0 §5.1).

The cached `quorum-publish` attestation is **the trust anchor for compromise-recovery validation** (§8.2 / v3.0 §9.4): if Bob's published handle ever needs to be rotated via `kind="identity-rotation-recovery"`, Alice's peer validates the K-of-N signatures on the recovery against the constituent set + threshold recorded in the cached `quorum-publish`. Without a cached `quorum-publish`, recovery rotations fail closed.

If Bob has shared per-relationship attestations with Alice (e.g., Bob attests `bob_phone` only to Alice via `system/identity/relationships/{alice_id_hex}/cert/{hash_hex}`), those flow via Alice's three-tier sync of her relationship namespace with Bob, not via the registry.

After TOFU, future updates from Bob (new agents, rotations) are validated against the cached state, and handle rotations chain via `kind="identity-rotation-handoff"` (routine, dual-signed by old + new) or `kind="identity-rotation-recovery"` (compromise-recovery, K-of-N from Bob's quorum) attestations (§8). The standard `process_attestation` pipeline (v3.0 §6.8) handles arrivals.

If Alice's peer loses its record of Bob (reinstall, lost device, etc.), the next encounter is back to TOFU. This is consistent with all identity systems; it's not a bug.

**Registry reflection latency varies per backend.** Self-hosted registries reflect the underlying tree instantly; DID:web reflects when Alice publishes a fresh document and HTTP caches expire; DHT reflects per republish cadence; blockchain anchors reflect at block-time + finality. Alice's verifier and Bob's verifier should honor the resolver's TTL hints and not assume a single cache model.

### 5.2 Recognizing a agent as "Bob's"

Bob has multiple agents — `bob_laptop`, `bob_phone`, `bob_server`. Alice's peer needs to know which agent IDs map to "Bob," and the answer depends on which publication modes Bob has chosen for each peer.

When `bob_laptop` sends Alice's peer a cap signed by `bob_laptop`'s keypair, Alice's peer:
1. Verifies the cap chain (V7 standard — root is the local peer that issued, V7 §5.5 line 1968).
2. Checks: does Alice's peer have a live `kind="identity-cert", function="agent"` attestation under Bob's identity covering `bob_laptop`? Sources the verifier checks in order:
   - Embedded in the cap envelope itself (mode `embedded`).
   - Cached locally from prior interactions (any mode).
   - Per-relationship namespace (`system/identity/relationships/{alice_id_hex}/cert/{hash_hex}` from Bob's tree, synced via Alice's three-tier sync).
   - Public-mode attestations resolved through Bob's registry.
3. If found and valid (validates as `kind="identity-cert", function="agent"`, attesting key is currently authoritative for Bob's identity per v3.0 §4.4 verifier-acceptance rules, signature verifies), the cap is "from Bob"; Alice's peer caches the mapping in its address book and proceeds.

**Cache-miss is fail-closed.** If no `kind="identity-cert", function="agent"` attestation is cached and none is found in any accessible source, the verifier rejects the EXECUTE — TOFU acceptance only fires at first contact, never as ongoing compensation for missing attestations. Implementations MAY add deployment-level recovery flows (fetch-on-demand from the registry, surface a handler signal for retry-after-sync, etc.); these are application-layer compositions on top of the fail-closed default. The architecture explicitly does NOT define a cache-miss policy framework — the failure mode is uniform.

The address book entry, after recognition, looks like:

```
"Bob" →  {
  published_handle:    K_bob_handle_hash,           ; what Alice cached at TOFU
                                                     ;   (Bob's K_op in three-key default;
                                                     ;    Bob's K_cf in four-key advanced)
  contact_quorum:      { signers: [...], threshold: ..., attestation_hash: ... },
                                                     ; cached at TOFU; trust anchor for
                                                     ;   compromise-recovery rotations
  cached_devices:      {
    bob_laptop_id:  { mode: "per-relationship", attestation_hash: ... },
    bob_phone_id:   { mode: "embedded",         attestation_hash: ... },
    bob_server_id:  { mode: "public",           attestation_hash: ... }
  }
}
```

Alice's view of Bob's agents may be a strict subset of Bob's actual runtime-peer set — that's fine; Alice only needs the peers Bob shares with her.

### 5.3 What changes when Bob adds a device

Bob pairs a new phone. K_op_bob (or K_cf_bob in four-key advanced) signs a new `kind="identity-cert", function="agent"` attestation. Bob then chooses a publication mode for it via `system/identity:publish_attestation`:

- If Bob promotes the attestation to **public mode**, the registry reflects on its next refresh (latency per backend); Alice's address book updates on next pull.
- If Bob promotes to **per-relationship mode** for Alice specifically, the attestation appears at `system/identity/relationships/{alice_id_hex}/cert/{hash_hex}` in Bob's tree; Alice's three-tier sync with Bob picks it up.
- If Bob keeps it **internal**, Alice doesn't see it. Bob can hand the phone a specific cap-envelope-embedded attestation for time-bounded use without ever publicizing the phone.

What changes when Bob removes a device: see §10.

---

## 6. Composing with roles — granting access

Identity tells you *who* a peer is. Roles tell you *what* a peer is allowed to do. They compose at the seam where the controller makes role assignments via the local peer→K_op cap.

This section is identity-focused — just enough to show how the local cap connects to role mechanics. For the full role / capability story (RL2 attenuation, role lifecycle, exclusion semantics, etc.), see `GUIDE-CAPABILITIES.md` and `EXTENSION-ROLE.md` v1.5.

### 6.1 The typical flow

Alice wants Bob to see her shared work folder. She uses `system/role:assign`:

```
alice_runtime_peer.execute(
  "system/role", "assign",
  {context: "work", role: "team-member", assignee: K_bob_handle},
  capability: <local peer→K_op cap on alice_runtime_peer>
)
```

What happens:
1. Alice's agent dispatches the EXECUTE. The caller capability is the local peer→K_op cap installed by `system/identity:configure` (§3.2). The grantee of that cap is K_op_alice; Alice's request signs as K_op.
2. The role handler validates RL2 (per `EXTENSION-ROLE` v1.5 §4.3): the caller's authority must cover the role's grants, via V7's `is_attenuated` primitive.
3. The role handler persists the assignment entity at `system/role/work/assignment/{K_bob_handle}/team-member`.
4. The role handler mints derived V7 capability tokens — granter = `alice_runtime_peer` (V7 §5.5 line 1968), grantee = `K_bob_handle`, parent = null (root cap). The token is signed by `alice_runtime_peer`'s keypair.
5. The token is persisted at `system/capability/grants/role-derived/work/{K_bob_handle}/{token_hash}`.
6. The token is delivered to Bob (out-of-band, via a notification mechanism, etc.).

### 6.2 What's signed by what

This is where most confusion happens. Let's be explicit:

- The **role-assignment entity** (`system/role/work/assignment/{K_bob_handle}/team-member`) is authored by K_op_alice — K_op authorized the EXECUTE; K_op's hash appears in `assigned_by`.
- The **derived V7 cap** is signed by `alice_runtime_peer`. The granter is `alice_runtime_peer`. The chain root is `alice_runtime_peer`. V7 §5.5 line 1968 is satisfied.
- K_op never appears in the cap chain. K_op authorizes the role assignment; the agent signs the cap. This is the operational-key-confinement invariant per `EXTENSION-IDENTITY` v3.0 §9.2 / §12.2.

When Bob's peer uses the cap to access Alice's resources:
1. Bob constructs an EXECUTE; carries the cap.
2. Alice's agent verifies the cap chain (it roots at itself — trivially valid).
3. Alice's agent verifies Bob's signature on the EXECUTE matches the cap's grantee (`K_bob_handle`). Bob's agent must be *acting as* Bob's published handle for this — typically Bob's agent holds the appropriate keypair, or has been delegated from Bob's published handle via Bob's own role-derived caps in his namespace.

The whole flow is V7-clean. Identity contributes the authorization (K_op as caller via the local cap). Role contributes the policy (which grants, to whom). V7 contributes the cap chain (signed by the agent, validated against itself).

### 6.3 When K_op rotates, role assignments survive

Operational-key rotation (§7) replaces K_op with K_op_v2. The local peer→K_op cap is revoked; a new local peer→K_op_v2 cap is issued. But the existing role assignments — the entities at `system/role/work/assignment/...` — are still valid. The derived V7 caps are still valid (signed by `alice_runtime_peer`, which is unchanged).

What changes after rotation: future role assignments are made via K_op_v2 through the new local cap. K_op_v1 can no longer make role assignments at this agent (the old local cap is revoked). Existing assignments and derived caps are unaffected.

This is the structural reason the controller is upstream of cap chains rather than inside them: rotating K_op is local management, not a global cap reissuance.

---

## 7. Rotating the controller

K_op_alice is the hot key — used for every internal management EXECUTE. Compromise is possible. Routine rotation (yearly hygiene) is also recommended. This section covers both.

In v3's unified attestation model, "rotating the controller" composes from two `system/identity:create_attestation` calls — there is no dedicated `rotate_operator` op (per `EXTENSION-IDENTITY` v3.3 §6.5: composed flows are caller-side).

### 7.1 The flow

1. Alice generates `K_op_alice_v2` (fresh keypair).
2. Alice constructs a `kind="identity-retirement"` attestation for K_op_alice_v1: `attesting = quorum_id`, `attested = K_op_alice_v1_hash`, `properties = {final_certification: hash(certification_v1)}`. K-of-N quorum constituents sign. For routine rotation, K_op_alice_v1 might be one of the signers if the deployment includes the operational key in the quorum (rare); for compromise recovery, K_op_v1 cannot sign (its key is compromised) so the cold quorum constituents alone reach threshold.
3. Alice constructs a `kind="identity-cert", function="controller"` attestation for K_op_alice_v2: `attesting = quorum_id`, `attested = K_op_alice_v2_hash`, `properties.mode = "public"` (3-key default; same audience tier as the retiring v1 cert), `supersedes: null` (introducing a new operational key — the retirement attestation closes K_op_v1's chain endpoint, so the new certification doesn't supersede it). K-of-N quorum constituents sign (typically the same signing session as the retirement).
4. Once signatures accumulate (async per `EXTENSION-IDENTITY` v3.3 §8 if signers are offline), Alice invokes `system/identity:create_attestation` for both attestations. The retirement and the new controller cert each land in the same audience tier as the v1 cert they replace — `system/identity/public/cert/{ret_hash_hex}` and `system/identity/public/cert/{cert2_hash_hex}` in the 3-key default (or `internal/cert/...` in 4-key advanced). Path resolution per v3.3 §5.3 is purely a function of `properties.mode` on the attestation; lifecycle events inherit the target cert's tier.
5. Sync propagates to Alice's agents.
6. Each agent's `process_attestation` (per v3.3 §6.8) runs on each arrival:
   - On the retirement: revokes the local peer→K_op_v1 cap (deletes its tree binding).
   - On the certification: issues a new local peer→K_op_v2 cap.

After rotation:
- K_op_v1's signatures stop verifying at Alice's agents (the local cap to K_op_v1 is revoked).
- K_op_v2 takes over internal management.
- The agents are unaffected at the V7 layer (their cross-peer caps are still signed by themselves; the operational-key change is internal management).

### 7.2 The contact-side surface — three-key vs four-key

This is the key behavioral difference between the two configurations:

- **Three-key default.** K_op IS the contact-face. K_op rotation is also a published-handle rotation; Bob's address book needs to update. The compromise-recovery case requires a `kind="identity-rotation-recovery"` attestation chained with the new certification, so Bob's peer can update Bob's cached handle (§8.2). Routine rotation can use `kind="identity-rotation-handoff"` for the dual-signed handoff; compromise recovery uses `kind="identity-rotation-recovery"`.
- **Four-key advanced.** K_op rotates internally; the identifier's key K_cf is unchanged. Bob's address book is unaffected. The flow above (steps 1–6) is the entire ceremony — no rotation-handoff, no rotation-recovery, no contact-side changes. **This is the property four-key advanced exists for.**

§8 covers the published-handle rotation flows (rotation-handoff for routine; rotation-recovery for compromise). In three-key default, those flows fire whenever K_op rotates. In four-key advanced, those flows fire only when K_cf rotates.

### 7.3 What's orthogonal

- Quorum membership (§9) — rotates independently.
- Runtime peer additions/removals (§4, §10) — rotate independently.
- Contact-face key (four-key advanced; §8) — orthogonal to operational-key rotation.

### 7.4 Cleaning up downstream effects

After rotation, role assignments K_op_v1 made are still valid (the assignments are persisted entities, not signed by K_op directly; their `assigned_by` field records K_op_v1's hash but the derived caps are signed by agents). K_op_v2 can review existing assignments and use `system/role:unassign` for any that should be revoked due to whatever caused the rotation (e.g., operational-key compromise that may have led to bad assignments).

---

## 8. Rotating the published handle

Alice's published handle is what contacts cache as "Alice." Rotating it is a cross-peer event — Bob's address book updates as a result.

The published handle is **K_op in three-key default; K_cf in four-key advanced**. The two configurations have very different rotation cadences:

- **Three-key default.** Published-handle rotation = operational-key rotation. Frequency depends on hygiene policy; compromise of K_op forces recovery rotation.
- **Four-key advanced.** Published-handle rotation = identifier's key rotation. Frequency is rare (only on K_cf compromise — operational keys rotate without touching contacts; see §3.5).

The two cases — handoff and recovery — use different signing patterns on the rotation attestation, with very different security stories.

### 8.1 Handoff (routine rotation)

Both keys are still available. Alice generates the new handle key (`K_handle_v2` — either K_op_v2 in three-key, or K_cf_v2 in four-key advanced). Alice constructs a `kind="identity-rotation-handoff"` attestation: `attesting = K_handle_v1_hash`, `attested = K_handle_v2_hash`, `properties.target_cert = hash(target_cert_v1)`. **Dual-signed** — both K_handle_v1 and K_handle_v2 sign (per `EXTENSION-IDENTITY` v3.3 §4.3).

```
{
  type: "system/attestation",
  data: {
    properties: {
      kind:        "identity-rotation-handoff",
      target_cert: hash(target_cert_v1)   ; the cert being rotated
    },
    attesting:  K_handle_v1_hash,         ; the old handle key
    attested:   K_handle_v2_hash,         ; the new handle key
    supersedes: hash(prior_rotation)      ; or null if first rotation
  }
}
```

Alice invokes `system/identity:create_attestation` to persist. The rotation entity inherits the target cert's audience tier (per v3.3 §5.3 `same_tier_path`) — for a public-mode handle-bearing target it lands at `system/identity/public/cert/{rot_hash_hex}`. Sync propagates to contacts.

Bob's peer processes the rotation via `process_attestation` (v3.3 §6.8): validates both signatures (dual-sig topology per v3.3 §3.6 `identity_topology_for`); updates the cached handle for "Alice" from K_handle_v1 to K_handle_v2.

Neither the quorum nor the operational key (in four-key) signs this. The contact-face surface stays purely between the keys contacts know about.

### 8.2 Compromise recovery

The handle key is gone. Alice can't dual-sign — only one key (the new one) is available. Recovery uses the quorum.

1. Alice generates K_handle_v2 (fresh keypair).
2. Alice constructs a `kind="identity-rotation-recovery"` attestation: `attesting = quorum_id`, `attested = K_handle_v2_hash`, `properties = {old_handle: K_handle_v1_hash}`. K-of-N quorum constituents sign.
3. Alice invokes `system/identity:create_attestation` to persist. The rotation-recovery inherits the target cert's audience tier — for a public-mode handle-bearing target it lands at `system/identity/public/cert/{rec_hash_hex}`. Sync propagates to contacts.
4. Bob's peer processes the rotation via `process_attestation`: notices `kind == "identity-rotation-recovery"`, looks up the cached `kind="quorum-publish"` attestation under `system/identity/contacts/{K_handle_v1_hex}/quorum-publish`, and validates the K-of-N signatures on the recovery against the cached signer set + threshold (per `EXTENSION-IDENTITY` v3.3 §9.4).

If no `kind="quorum-publish"` is cached for the handle being recovered (e.g., Alice opted out of `quorum-publish` publication for privacy reasons; per `EXTENSION-QUORUM` §3.3), the rotation is **rejected fail-closed** (v3.3 §9.4). Recovery in that case falls back to out-of-band re-establishment: Alice sends Bob a fresh signed introduction; Bob TOFU-accepts the new identity.

The quorum entity itself (at `system/quorum/{quorum_id_hex}`) is **part of identity's published surface** in v3.3 (the dual-subtree two-tier sync per v3.3 §5.2). Bob's peer DOES sync the quorum subtree — both the quorum entity and its `quorum-update` / `quorum-publish` events. This lets Bob validate `current_signer_set` against the same chain Alice's agents see, rather than only through the cached `quorum-publish` summary. Operational keys stay internal across all flows including recovery (per §13.2 invariant).

**Privacy opt-out:** Deployments can suppress the `kind="quorum-publish"` attestation for privacy-sensitive contexts (`configure(publish_contact_quorum: false)`). The architecture supports both paths; the default is "publish" (recoverable without ceremony), with opt-out for users who prioritize privacy over recovery-ergonomics.

### 8.3 Operational keys never sign rotation entities (four-key advanced)

In four-key advanced, both forms of `identity-rotation-handoff` / `identity-rotation-recovery` exclude the operational key as a signer. The rotation-handoff is dual-signed by old + new identifier's keys; the rotation-recovery is K-of-N quorum-signed. Bob's peer, processing any rotation attestation under Alice's identity, never sees the operational key's signature. Operational-key confinement to internal/ paths (v3.3 §9.2) is preserved.

In three-key default, the published handle IS the operational key, so the "operational key never signs" framing doesn't apply at the level of contact-face attestations — but cap-chain confinement still does (operational keys never sign V7 caps that enter cross-peer chains; per v3.3 §12.2).

### 8.4 Window between compromise and recovery

If the published handle is compromised, the attacker can sign new `kind="identity-cert", function="agent"` attestations claiming arbitrary peers as "Alice's." Bob's peer processes these and updates its address book — potentially trusting attacker-controlled agents as Alice — until the recovery rotation propagates.

Mitigation:
- Short propagation windows for `public/` sync.
- Cap-level revocation of any caps issued by attacker-controlled agents (V7 standard, per v3.3 §9.5).
- Bob's peer can apply a "monotonic supersedes" check: an attestation chain whose `supersedes` doesn't form a continuous chain from the previously-known state is suspicious.

The architecture **bounds** this window rather than eliminating it. There is no identity system that eliminates the compromise-window; this one keeps it small and recoverable.

---

## 9. Quorum updates

The quorum's membership is rare-mutation. It changes when a constituent key is compromised (replace it), when a custodian becomes unavailable (replace), or when Alice deliberately restructures (raise threshold from 2 to 3, etc.).

### 9.1 The flow

1. Alice constructs a `kind="quorum-update"` attestation (owned by `EXTENSION-QUORUM` §3.2): `attesting = quorum_id` (current), `attested = quorum_id` (self), `properties = {new_signers: [...], new_threshold: K}`, `supersedes: hash(prior_quorum_update)` (or `null` if this is the first update).
2. K-of-N **current** quorum constituents sign the attestation (per `EXTENSION-QUORUM` §3.2; verified via `verify_k_of_n_signatures` against `current_signer_set` resolved by walking the quorum's `quorum-update` chain).
3. Alice constructs a fresh `kind="quorum-publish"` attestation (owned by `EXTENSION-QUORUM` §3.3) reflecting the new signer set + threshold. **Per `EXTENSION-QUORUM` §3.3 "previous pinning" rule, the supersede on `quorum-publish` is signed by the *previous* quorum** (the cached prior signer set is what contacts trust; continuity-of-chain is the property that lets compromise-recovery validation work).
4. Alice invokes operations on `EXTENSION-QUORUM` (`system/quorum:update` and `system/quorum:publish`) — identity is a consumer of these substrate operations. The quorum-update and quorum-publish both land under `system/quorum/{quorum_id_hex}/event/{hash_hex}` per `EXTENSION-QUORUM` §3.2/§3.3 path conventions.
5. Each agent (via sync) updates its view. Future K-of-N validation calls `QUORUM.current_signer_set(quorum_id)`, which walks the live `quorum-update` chain to determine the effective signer set. Future certifications and `identity-rotation-recovery` attestations validate against the resolved current quorum.
6. Contacts (via two-tier sync of `system/quorum/{trusts_quorum}/...`) pull the new quorum-publish and update their cached state at `system/identity/contacts/{handle_hex}/quorum-publish`.

The two attestations together — quorum-update plus chained quorum-publish — are the unified rotation flow. They are produced together via two `create_attestation` calls; there is no dedicated `rotate_quorum` op in v2.

### 9.2 What `quorum-update` cannot do

It cannot recover from below-threshold loss. If Alice has a 2-of-3 quorum and loses 2 of the 3 keys, K-of-current is 1, which is below the threshold. No valid quorum-update can be constructed. Alice's identity is unrecoverable — she has to start over with a new identity, which will have a new `quorum_id` and require Bob to TOFU-accept a fresh published handle.

This is the **catastrophic loss** case (`EXTENSION-IDENTITY` v3.0 §9.7). The architecture mitigates by encouraging geographic distribution and K<N defaults (2-of-3 tolerates one loss). It cannot eliminate the case.

---

## 10. Runtime peer revocation

Alice's laptop is stolen. The laptop's `peer_id_laptop` keypair is now in attacker hands.

### 10.1 The flow

In v2, runtime-peer revocation is **per-mode revoke_attestation calls** — one per live `kind="identity-cert", function="agent"` attestation for the agent, across whichever publication modes the attestation was in. There is no dedicated `revoke_peer(scope: "all")` op; the caller (or an SDK helper) enumerates the live agent certs and invokes `system/identity:revoke_attestation` per attestation (per `EXTENSION-IDENTITY` v3.0 §6.6).

1. Alice (acting as K_op via the local cap from any agent she still controls) enumerates live `kind="identity-cert", function="agent"` attestations whose `attested = peer_id_laptop`. Each attestation has its own storage path per its publication mode.
2. For each live agent cert, Alice invokes:
   ```
   system/identity:revoke_attestation
   resource: <the attestation's stored path>
   ```
   The handler removes the binding (the `device` attestation entity stays in the content store; its location-index removal is the authoritative revocation per V7's revocation model).
3. Internal propagation: at each remaining agent (via sync), `peer_id_laptop`'s `kind="identity-cert", function="agent"` attestation under `internal/` is gone. The agent is no longer "Alice's" within the fleet.
4. External propagation, per the modes the attestations were in:
   - **public mode:** the public-mode `device` attestation is deleted from `system/identity/public/`. The registry layer reflects the deletion on its next refresh; contacts pull on their schedule and update their caches.
   - **per-relationship mode:** the per-relationship attestation is deleted from each `system/identity/relationships/{contact_id}/` subtree where it was published. Each contact's three-tier sync picks up the deletion.
   - **embedded mode:** the cap envelopes carrying the embedded attestation are no longer trustable; Alice may also revoke the caps themselves via standard V7 cap revocation if needed.
5. Bob's peer (and other contacts) process the deletions via whichever mode applied: address book updates; caps signed by `peer_id_laptop` are no longer recognized as "from Alice."

**Per-mode revocation lets Alice revoke from a single contact without affecting others.** For a peer Alice wants to stop sharing with Bob (but keep using elsewhere), she revokes only the `system/identity/relationships/{bob_id_hex}/cert/{hash_hex}` for the device — leaving public-mode and other per-relationship attestations untouched.

### 10.2 What about caps Bob holds from the revoked peer?

Caps Bob holds from `peer_id_laptop` remain V7-cryptographically valid until standard V7 revocation removes them. The agent's tree binding for the cap has to be deleted, or the chain's root has to be invalidated.

But Bob's *trust judgment* — "this is from Alice" — flips immediately on attestation update. Even if the cap is V7-valid, Bob's peer no longer trusts `peer_id_laptop` as Alice's (no live `kind="identity-cert", function="agent"` attestation). Application policy decides whether to outright reject or to fall back to "anonymous reader" treatment.

For Alice's own resources where the now-stolen laptop is the chain root, V7 standard revocation applies. Alice (as K_op) can remove the tree binding; the chain breaks at the next `is_revoked` check. This is the property documented in `EXTENSION-IDENTITY` v3.0 §9.5.

### 10.3 Replacing the lost device

After revocation, Alice provisions a new agent — same flow as §4 — and adds it to her identity via a fresh `kind="identity-cert", function="agent"` attestation. Life continues. The stolen laptop's keypair is permanently out of Alice's identity.

---

## 11. Custody variants

The recommended default is **three-key** (quorum / operational / runtime); the **four-key advanced** variant adds a separate identifier's key. Beyond those two shapes, several custody variants compose on top.

### 11.1 Three-key default — recap

The recommended shape for users opting into identity. K_op IS the contact-face; contacts cache it directly. See §3.1 for what gets produced; §3.5 for the comparison to four-key.

This is what most users want. The variants below are reasons to deviate; if none of them apply to your deployment, stay three-key.

### 11.2 Four-key advanced (separate contact-face)

Adds K_cf as a separately-attested identifier's key (one more keypair, one more attestation kind: `kind="identity-cert", function="identifier"`). K_op rotates without contacts noticing; K_cf rotates rarely (only on K_cf compromise).

Choose four-key when: high-cadence operational-key rotation is policy; compliance / regulatory environments where K_op is treated as more sensitive than K_cf; defense-in-depth on the contact-face is worth one more key.

See §3.5 for the full discussion + tradeoffs.

### 11.3 Hardware-backed quorum constituent

A quorum constituent's keypair lives on a hardware token (YubiKey, etc.). Signing requires the token; the key never enters general-purpose memory.

Tradeoffs:
- **+** That constituent's key is materially harder to compromise.
- **−** Hardware-token UX for K-of-N signing ceremonies (rotations, certifications) — every event involving that constituent requires inserting the token.
- **−** Loss of the token reduces the effective quorum below threshold; recovery requires the remaining constituents alone reaching threshold (which typical 2-of-3 defaults handle, but tight thresholds don't).

Mechanically: same flow as the default; signing operations route through the hardware. SDK helpers should expose this as a configuration option, not a separate code path. Multiple quorum constituents may be hardware-backed independently.

### 11.4 Hardware-backed controller

K_op's keypair lives on a hardware token. Same tradeoff applied to the hot key.

- **+** K_op compromise becomes materially harder.
- **−** Hardware-token UX for every internal management EXECUTE — every agent cert Alice signs, every role assignment K_op authorizes.
- **−** Hardware loss forces operational-key rotation (retirement + new certification per §7).

In four-key advanced, the same option applies to K_cf — typically more useful there since K_cf is what contacts cache and its rotation is more disruptive.

### 11.5 1-of-1 quorum

K=1, N=1. The quorum is one keypair. Useful for users who actively choose simpler key custody over recoverability.

Tradeoffs:
- **+** No K-of-N coordination; one signature is enough.
- **−** Loss of the one quorum key = catastrophic loss (no recovery path).
- **−** Any rotation (operational, published-handle, quorum-update) requires that one quorum key being available; no tolerance for partial loss.

Mechanically: identical to default flow; `signers: [K_quorum], threshold: 1`. K-of-N validation works trivially. This is structurally complete (certifications, quorum-publish, rotations all work mechanically) but provides no recovery.

Per `EXTENSION-IDENTITY` v3.0 §11.2.

### 11.6 Concurrent multi-operational

Multiple controllers live concurrently under the same quorum (per `EXTENSION-IDENTITY` v3.0 §11.6). Each agent holds one local cap per live controller.

Useful for desktop + phone deployments where each device holds its own controller — the desktop's K_op_desktop and phone's K_op_phone each get their own `kind="identity-cert", function="controller"` attestation under the same quorum (with `supersedes: null` to introduce new concurrent controllers rather than supersede). Each device drives internal management with its own K_op.

Mechanically: the agent's peer-config has multiple `bindings` entries (or a single binding with a `handle_attestation` referencing the certification for the local controller).

### 11.7 Multi-binding (one agent in multiple identities)

A single agent participates in multiple distinct identities — e.g., Alice's laptop participates in both her personal identity and a service-account identity.

Mechanically: peer-config has multiple `bindings` entries (one per identity); each binding references its own handle-attestation and runtime-attestation. Each identity has its own quorum and own attestation graph. The agent trusts each quorum independently.

Per `EXTENSION-IDENTITY` v3.0 §11.5.

### 11.8 Parent-managed identity

A child Lee has an identity, a quorum, and agents like any other — but the keys live on the parents' hardware. Mechanically the architecture treats Lee as a regular identity; the parents act on Lee's behalf.

Concretely (under three-key default):
- Lee's quorum: parents' operational keys, e.g., `{K_op_alice, K_op_bob}`, threshold 2-of-2. No Lee-controlled keys at age 10.
- Lee's K_op (operational + contact-face in three-key): held on parents' hardware, encrypted at rest.
- `lee_tablet`: a agent of Lee's identity. Its keypair is on the device Lee uses; the `kind="identity-cert", function="agent"` attestation authorizing it is signed by Lee's K_op using parent-held keys.
- Operations Lee performs at `lee_tablet` (opening shared photos, joining the family calendar) are signed by `lee_tablet`'s runtime keypair and authorized by caps issued to `lee_tablet` by family infrastructure. **The user who acts is Lee (via lee_tablet); the identity authority who provisioned that user is the parents (via Lee's K_op held by parents).**
- Parents act as Lee for any operation requiring Lee's K_op signature (creating a new agent cert, revoking a agent cert, signing rotation entities for Lee's handle).

Custody transition (Lee grows up): see §11.10.

This pattern is mechanically supported by the existing primitives — no new mechanism. The architecture doesn't need a "parent-managed mode"; it just needs operational-key custody to be a deployment-policy choice. The same pattern covers any "delegate identity custody" use case (an estate trustee managing a deceased person's estate identity, an organization managing a service-account identity, etc.).

**Operational consideration:** parents must hold Lee's keys somewhere secure within their fleet. If parents' fleet is compromised, Lee's identity is compromised too. Mitigations are deployment policy (cold-store, rotation, etc.) — same considerations as any cold-key custody.

### 11.9 Multi-identity host machine

A single host machine MAY operate as N agents, one per identity it serves. Examples:
- Carol's home machine runs as `carol_desktop` (agent of Carol's identity) AND `garden_calendar_server` (agent of the garden group's identity).
- Alice's family-server hardware runs as `family_server` (agent of the family group) AND, optionally, also serves as `alice_server` (agent of Alice's identity).
- A small open-source project's CI host serves both the project group's identity and the maintainer's individual identity.

**Per-runtime-peer-identity peer-config.** Each agent has its own peer-config in its own namespace and trusts its own quorum independently. Peer-configs MUST NOT share state across identities even when co-located on the same hardware. Implementations MUST treat each agent as a separately-configured identity participant, with no cross-identity inheritance of `trusts_quorum`, `operational_grants`, or `bindings`. (Per `EXTENSION-IDENTITY` v3.0 §3.2.)

Trust model: members of each identity trust the host machine to operate the keypair faithfully. If the host is compromised, the keypair is at risk — for any of the identities running on it. Mitigations: cold backups for all involved identities; redundant agents across multiple hosts; cap scoping so each agent holds only the caps it needs.

This pattern is core to "shared infrastructure on someone's hardware" deployments — community gardens, family servers, open-source projects. See `EXTENSION-GROUP.md` §7.5 for the group-extension perspective.

### 11.10 Identity transition flows

Two recurring patterns for transitioning custody over time. Both use existing primitives; neither needs a new operation.

**Pattern A — Parent-to-self (a kid grows up).**

A subject's identity starts parent-managed (§11.8). Over years, custody transitions to the subject:

1. **Initial state (subject is 10):** quorum = {parents' operational keys}, threshold 2-of-2; subject's keys held by parents.
2. **Transition begins (subject is 16):** subject generates own operational key + cold backup. A `kind="quorum-update"` attestation (chained with a fresh `kind="quorum-publish"` per §9) adds them. New quorum: {K_op_alice, K_op_bob, K_op_subject, K_cold_subject}, threshold 3-of-4. K-of-N (parents) sign.
3. **Subject takes full control (subject is 18):** subject generates additional cold backups; another `kind="quorum-update"` removes parental constituents. New quorum: {K_op_subject, K_cold1_subject, K_cold2_subject, K_cold3_subject}, threshold 2-of-4. K-of-N of the prior quorum (typically subject + at least one parent, indicating consent on both sides) sign.
4. After step 3, the subject may rotate the operational key (privacy hygiene; §7) and update peer-config to use subject-owned hardware as the K_op-host.

**Pattern B — Operational authority transition within an existing identity.**

A user's individual identity is unchanged but the operational authority within their fleet changes — for example, transferring effective day-to-day operational role to a new device, or splitting K_op across multiple devices.

1. **Initial state:** quorum = {K_cold1, K_cold2, K_cold3}, 2-of-3; one live `kind="identity-cert", function="controller"` attestation authorizes K_op_v1.
2. **Add a second controller:** a fresh `kind="identity-cert", function="controller"` attestation authorizes K_op_v2 (with `supersedes: null` — multi-operational; both controllers hold authority concurrently per `EXTENSION-IDENTITY` v3.0 §4.2).
3. **Optionally retire the original:** a `kind="identity-retirement"` attestation closes K_op_v1's chain. K_op_v2 alone holds operational authority.

This composes with the rotation flow (§7). The "two controllers concurrently" interim state is normal; the spec doesn't restrict to exactly one controller per identity.

**Pattern C — Cross-organizational migration.**

Out of scope for the identity extension. A user moving between orgs is a relationship-change, not an identity-change; the user's individual identity is unchanged, what changes is their group membership. See `EXTENSION-ROLE` (role assignments) and `EXTENSION-GROUP` (membership) for those flows.

---

## 12. What Bob sees vs what Alice manages

A common mental-model error is thinking "Alice's identity" means the same thing to Bob and to Alice. It doesn't.

| What | Alice's view | Bob's view |
|---|---|---|
| Quorum entity | Lives at `system/quorum/{quorum_id_hex}` (substrate-owned per `EXTENSION-QUORUM`); she manages constituent keys directly | Visible — part of identity's published surface (dual-subtree two-tier sync per v3.3 §5.2); Bob's peer subscribes to `system/quorum/{trusts_quorum}/...` and sees the quorum entity plus its `quorum-update` / `quorum-publish` event chain |
| `kind="quorum-publish"` attestation | Lives under `system/quorum/{q}/event/{hash_hex}`; K-of-N signed (initial publish by current quorum; supersedes signed by previous quorum per `EXTENSION-QUORUM` §3.3 previous-pinning rule); records constituents and threshold | Visible — Bob's peer caches it at TOFU under `system/identity/contacts/{handle_hex}/quorum-publish`; the validation anchor for compromise-recovery rotations |
| `kind="identity-cert", function="controller"` attestation (top-level controller authorization) | **3-key default:** lives under `public/cert/{hash_hex}` (`properties.mode = "public"`; controller IS the handle). **4-key advanced:** lives under `internal/cert/{hash_hex}` (`properties.mode = "internal"`; controller is internal management). K-of-N quorum-signed | **3-key default:** Visible (public-mode). **4-key advanced:** Not visible (internal-mode) |
| K_op's keypair / public key | She has the private key; manages internal authority | **Three-key default:** visible — K_op IS the contact-face; this is "Alice" in Bob's address book. **Four-key advanced:** not visible at all; never appears in Bob's address book or any cap chain |
| `kind="identity-cert", function="identifier"` attestation (four-key only) | Lives under `internal/cert/{hash_hex}`; op-signed evidence K_cf is the contact-face. The cert itself is internal; identifier peer's KEY is what contacts cache | Not visible (cert is internal-mode); but the identifier's public key is visible via the agent certs it signs |
| K_cf's keypair (four-key only) | She has the private key; this is the contact-face | Visible — K_cf IS what Bob's address book records as "Alice" in four-key advanced |
| `kind="identity-cert", function="agent"` attestation (internal mode) | Lives under `internal/cert/{hash_hex}`; signed by K_op (three-key) or K_cf (four-key) | Not visible |
| `kind="identity-cert", function="agent"` attestation (public mode) | Lives under `public/cert/{hash_hex}` | Visible via Alice's registry — anyone resolving Alice's published handle can see |
| `kind="identity-cert", function="agent"` attestation (per-relationship mode) | Lives under `relationships/{bob_id_hex}/cert/{hash_hex}` | Visible to Bob only — three-tier sync of Bob's relationship namespace |
| `kind="identity-cert", function="agent"` attestation (embedded mode) | Embedded inside specific cap envelopes Alice issues | Visible only when Bob holds the carrying cap; not at any tree path |
| `kind="identity-rotation-handoff"` attestation | Lives in same audience tier as target cert (per v3.3 §5.3); dual-signed (old + new handle) | Visible if target is handle-bearing — Bob processes via `process_attestation` to update his cached handle |
| `kind="identity-rotation-recovery"` attestation | Lives in same audience tier as target cert; K-of-N quorum-signed | Visible if target is handle-bearing — Bob's peer validates against cached `quorum-publish`; on success updates the cached handle |
| `kind="quorum-update"` attestation | Lives under `system/quorum/{q}/event/{hash_hex}` (substrate-owned); K-of-N current quorum | Visible — part of the published quorum subtree; Bob's peer walks the chain via `QUORUM.current_signer_set` to track quorum membership over time |
| `kind="identity-retirement"` attestation | Lives in same audience tier as target cert; K-of-N quorum-signed | Visible only if the retired cert was in a contact-visible tier (public or per-relationship for the contact); otherwise not visible |
| Alice's published handle | The face she presents externally — K_op in three-key, K_cf in four-key | Visible — this is "Alice" in Bob's address book |
| Runtime peer IDs | Her devices/servers | Visible *per the publication mode of each peer's `kind="identity-cert", function="agent"` attestation* — Bob may see a strict subset of Alice's agents, by Alice's choice |
| Caps issued to Bob | Issued by her agents; she sees them in her tree | Visible — they were sent to him; he uses them |

**The asymmetry is structural.** Alice manages everything; Bob trusts only what's signed by Alice's currently-authoritative published handle (or, at recovery time, by the quorum). The operational key in four-key advanced is invisible to Bob across all flows.

This separation is what makes operational-key rotation cheap on the cross-peer side **in four-key advanced**: Bob doesn't need to do anything when K_op rotates — the agent certs contacts cache are signed by K_cf, which is unchanged. In **three-key default**, where K_op IS the contact-face, K_op rotation is also a published-handle rotation that Bob processes (rotation-handoff or rotation-recovery; §8). This is the central trade-off of three-key vs four-key: simpler key custody, or contact-side invisibility of operational rotation. Pick the property that matters more for your deployment.

---

## 13. Pitfalls and antipatterns

### 13.1 Treating the controller as the "user identity"

The controller is *operational*. Alice's published handle is *the user identity* externally. **In four-key advanced**, putting K_op in Bob's address book, signing cross-peer caps as K_op, or otherwise exposing K_op to contacts breaks the internal/external separation and undermines the property four-key exists for. Don't do this.

**In three-key default**, K_op IS the published handle by structural design — there is no separate contact-face to confuse it with. The pitfall here shifts: don't treat K_op as **only** an internal key when it's also serving as the contact-face. Rotation of K_op is a contact-side event in three-key (handoff or recovery; §8). If you want operational rotation to be invisible to contacts, you want four-key advanced (§3.5 / §11.2).

### 13.2 Forgetting the bootstrap exemption

The first `configure` / `create_quorum` / initial `create_attestation` calls have to use the SDK's L0 path; they can't go through dispatched EXECUTE because the local peer→K_op cap doesn't exist yet. Implementations that try to use the dispatched path during initial provisioning will fail with permission errors, then either bypass dispatch authorization (security hole) or fail to provision (UX disaster).

The bootstrap-distinguishing mechanism MUST be documented per implementation (per `EXTENSION-IDENTITY` v3.3 §6.9). Recommended: only the peer-owner's L0-direct-store can call those bootstrap operations. After the first call, the local cap exists and runtime invocations go through normal dispatch. Conformance is observed via post-state, not wire trace (per v3.3 §12.5).

### 13.3 Mixing identity attestations with V7 cap chains

A `system/attestation` entity in identity context (any of the four identity kinds) is **not** a capability token. Don't pass attestations as `capability` field on EXECUTEs; don't try to verify them with `verify_capability_chain`. They are non-cap entities validated at the entity level by `identity_verify_cert` — a topology-first orchestrator that composes substrate primitives (`ATTESTATION.verify_attestation_signature`, `QUORUM.verify_k_of_n_signatures`, etc.) per the kind+function tuple.

The architectural separation is the **three-parallel-mechanisms invariant** (v3.3 §2.2 / `SYSTEM-IDENTITY-COMPOSITION.md` §5): V7 capability tokens (validated by `verify_capability_chain`); `system/attestation` entities (validated by ATTESTATION primitives + identity's predicates); `system/quorum` entities (validated by QUORUM's `verify_k_of_n_signatures`). All three are structurally distinct entity types; no shared validation path. Mixing them undermines all three — the natural implementation mistake (reusing cap-chain walkers for "anything multi-signed") is explicitly forbidden.

### 13.4 Granting raw `tree:put` to identity paths

If application code has `tree:put` on `system/identity/...`, `system/quorum/...`, or `system/attestation/...`, it can bypass the identity handler's validation. The handler validates K-of-N signatures, supersedes chains, kind-specific structural constraints. Raw `tree:put` doesn't.

Per `GUIDE-CAPABILITIES.md` (Coherent Capability principle from PROPOSAL-COHERENT-CAPABILITY-AUTHORITY): grant `system/identity:configure`, `system/identity:create_attestation`, `system/identity:revoke_attestation`, etc. — not raw tree access to identity paths. Application-level grants should target the handler's operations (per `EXTENSION-IDENTITY` v3.3 §1.3).

### 13.5 Skipping cold-key backup

The application defaults to "we'll generate N=3 quorum keys, you should back at least N−K of them up." If the user defers the backup forever and then loses the runtime device that holds the cold keys, the quorum drops below threshold — catastrophic loss (per `EXTENSION-IDENTITY` v3.3 §9.7).

UI / setup ceremony should make the backup step prominent and continue nagging until it's done. Don't let users skip it permanently. Hardware token backup, paper backup, and delegation to a trusted holder are all valid — the application's job is to walk through the choice.

### 13.6 Conflating identity quorum with `system/cluster`

The word "cluster" is reserved for `system/cluster` — a planned extension for runtime-peer coordination (HA, replication, leader election). The identity quorum is NOT a cluster. It doesn't run, isn't online, doesn't process requests. It's a structural root that occasionally re-emerges to sign a new attestation.

Older docs and pre-synthesis explorations use "cluster" for what is now "identity quorum." Trust the current spec terminology.

### 13.7 Reading the path tiers as fleet-distribution boundaries

`quorum/` / `internal/` / `public/` / `relationships/` describe **external sync exposure**, not which Alice peers hold what. All of Alice's agents (laptop, phone, server, etc.) hold the *full* `system/identity/` subtree. The registry handler running on Alice's externally-reachable peer chooses what to expose to which contact. Implementations that try to enforce different fleet members holding different subsets of the tree are misreading the spec. Per `EXTENSION-IDENTITY` v3.0 §5.2.

### 13.8 Sharing peer-config across agents on the same machine

A host machine running multiple agents (e.g., Carol's machine = `carol_desktop` + `garden_calendar_server`, per §11.9) MUST keep peer-configs strictly separate. Each agent has its own `peer-config` in its own namespace; trusts its own quorum; has its own `operational_grants`; participates in its own bindings. Sharing state (e.g., reusing a single `trusts_quorum` field across both agents because they're "on the same machine") collapses the identity boundary and cross-contaminates the two identities. Per `EXTENSION-IDENTITY` v3.0 §3.2.

### 13.9 Silently TOFU-ing every cap that arrives without an attestation

TOFU applies only to **first contact** between two peers. Subsequent EXECUTEs from agents must validate against cached `kind="identity-cert", function="agent"` attestations under the sender's identity. If the attestation is missing, the verifier rejects fail-closed — per the unified model in v2 (§5.2). Implementations that silently TOFU every unattested cap effectively turn every agent into a first-contact case, defeating the rotation-tracking that attestations are supposed to provide.

Implementations MAY add deployment-level recovery flows on top of fail-closed (fetch-on-demand from the registry, surface a handler signal for retry-after-sync, etc.), but those are application-layer compositions — not policy frameworks the architecture defines. The default behavior is uniform: no live agent cert, no recognition.

---

## 14. Worked example: longest cross-extension cap chain through rotations

This example walks the full cross-extension chain (group → identity → role → compute) and shows where rotation does and doesn't bite. It exists for pedagogical reasons — to surface the architectural separation (V7 chain ends at agent; identity attestations layer above) in concrete terms.

Under v3.3's substrate model, the four rotation surfaces below are uniformly expressed as kind-discriminated attestation operations on `system/attestation` (the substrate entity type from `EXTENSION-ATTESTATION`) — one mechanism, four surfaces showing where it bites.

### 14.1 The setup

- Acme is a group (per `EXTENSION-GROUP`). Acme's quorum signs a `kind="identity-cert", function="controller"` attestation naming K_acme_op as Acme's current operational key.
- K_acme_op issues a local peer→K_acme_op cap on `partnership_server` — one of Acme's agents (attested via a `kind="identity-cert", function="agent"` attestation).
- K_acme_op calls `system/role:assign`, granting Bob admin authority on a resource Acme controls.
- The role handler issues Bob's agent a role-derived cap rooted at `partnership_server`.
- Bob installs a compute subgraph using the role-derived cap as `installation_grant`. Compute reactively dispatches to other handlers using the stored grant.

The result is a chain that crosses four extensions: group (Acme's quorum) → identity (Acme's certification + local peer→K_op cap) → role (Bob's role-derived cap) → compute (install grant).

### 14.2 Rotation surface 1: K_acme_op retires

Acme's operational key rotates — caller submits a `kind="identity-retirement"` attestation for K_acme_op_v1 plus a fresh `kind="identity-cert", function="controller"` for K_acme_op_v2 (per §7). On three-key default this is also a published-handle rotation (chained `kind="identity-rotation-recovery"` or `kind="identity-rotation-handoff"` at `public/`); on four-key advanced it stays internal.

**Operational-key rotation is invisible at the V7 cap layer.** Acme's agents' local peer→K_op caps update via `process_attestation` — the old local cap is revoked, a new one is issued — but the cross-peer caps Acme's agents signed are still anchored at the agents, not at K_op. So:

- The compute install grant survives (rooted at `partnership_server`'s keypair, unchanged).
- Bob's role-derived cap survives (rooted at `partnership_server`'s keypair, unchanged).

What does change is K_acme_op as the *future* issuer of new caps — subsequent role assignments and fresh local-peer→K_op cap issuance flow from K_acme_op_v2. Existing caps anchored at agents are unaffected. This is the property `EXTENSION-IDENTITY` v3.0 §9.5 documents.

### 14.3 Rotation surface 2: `partnership_server`'s agent retired

The agent retirement is a `system/identity:revoke_attestation` call against the live `kind="identity-cert", function="agent"` attestation(s) for `partnership_server` across all publication modes. Per V7 §5.5 chain root, when `partnership_server` is no longer attested as Acme's agent, every cap chain rooted at `partnership_server`'s keypair loses its authorizing identity-graph edge on next sync to verifying peers. Concretely:

- The compute install grant dies (its chain root is no longer recognized as Acme's).
- Bob's role-derived caps issued by `partnership_server` die.
- Subscriptions Bob held that were rooted at `partnership_server` die.

This is by design — the rotating peer is responsible for re-issuing outstanding grants from a new authority and pushing the new caps to consuming peers (the short-TTL / SDK-tracked-re-issuance / accept-loss patterns per `EXTENSION-IDENTITY` v3.0 §9.5).

### 14.4 Rotation surface 3: Bob retires his agent

Bob's agent rotation is symmetric to surface 2 — applied at Bob's side. Bob calls `system/identity:revoke_attestation` against the live `kind="identity-cert", function="agent"` attestation for his old agent; new agent certs are created for his replacement agent.

Cross-extension consequences:
- Subscription deliver tokens Bob issued are re-issued from Bob's new agent.
- Role-derived tokens whose grantee was Bob's old agent are swept and re-derived for Bob's new agent.
- NETWORK sessions Bob held are re-established under Bob's new agent ID.

Each consuming extension uses its standard re-issuance pattern; identity is just the place where the trigger (the old agent is no longer attested) becomes visible.

### 14.5 Rotation surface 4: Acme dissolves

When Acme dissolves (per `EXTENSION-GROUP` §6.5), all of Acme's `kind="identity-cert", function="agent"` attestations are revoked across all publication modes — one `system/identity:revoke_attestation` call per live agent cert per agent. Chain validation rejects everything Acme issued: `partnership_server`'s caps no longer have a live runtime-attestation edge under Acme's identity; downstream caps (Bob's role-derived cap, Bob's compute install grant) fail at the attestation check even if their V7 chains are still structurally intact.

This is the layered-validation property: V7 chain validity is necessary but not sufficient. Identity attestations layer above; when the agent that signed the cap is no longer attested as a member of the issuing identity, downstream cap validation fails.

The dissolution is sync-latency-bounded — caps stay validatable until the runtime-attestation revocations propagate to verifying peers' trees. See `EXTENSION-GROUP` §6.5 for the full dissolve flow.

---

## 15. Reference

### 15.1 Entity types

In v3.3, identity owns only **two** entity types — the rest of the substrate (the universal attestation entity, the K-of-N quorum primitive) is provided by `EXTENSION-ATTESTATION` and `EXTENSION-QUORUM`. Identity is a convention layer over both.

| Type | Owned by | Path | Signed by | Audience | Purpose |
|---|---|---|---|---|---|
| `system/quorum` | `EXTENSION-QUORUM` | `system/quorum/{quorum_id_hex}` | Not signed (structural) | All who sync the quorum subtree (identity exposes its `trusts_quorum` to contacts) | K-of-N node primitive — defines constituent set + threshold; signer-resolution mode (`concrete` default; `identity-resolved` registered by identity) |
| `system/attestation` | `EXTENSION-ATTESTATION` | per `properties.kind`, per identity convention (see §15.2 below) | per kind+function topology (K-of-N, single-sig, or dual-sig) | per identity's path conventions (internal / public / per-relationship / embedded) | Universal substrate edge type; carries identity certs, lifecycle events, revocations, and quorum self-events |
| `system/identity/peer-config` | `EXTENSION-IDENTITY` | `system/identity/peer-config` (per agent per identity served) | Not signed (local, not propagated) | This agent | Per-runtime-peer-identity local config (`trusts_quorum`, `controller_grants`, `bindings`) |
| `system/identity/identity-binding` (helper inner type) | `EXTENSION-IDENTITY` | inside `peer-config.bindings`; not stored independently | n/a | n/a | Helper inner type recording one runtime-peer-to-identity binding (`handle_attestation`, `runtime_attestation`) |

### 15.2 Attestation kinds (identity-context `system/attestation` entities)

Identity registers four `properties.kind` values on the substrate `system/attestation` entity. Quorum self-events (`quorum-update`, `quorum-publish`) are listed for completeness but are owned by `EXTENSION-QUORUM`. **`properties.mode` is REQUIRED on ALL identity-cert kinds** at create-time (per v3.3 §4.2 — eliminates the v3.0 in-flight rotation race); lifecycle events inherit the target cert's tier per v3.3 §5.3.

| Kind | function | Attesting | Attested | Sig topology | Storage path |
|---|---|---|---|---|---|
| `identity-cert` | `controller` (top-level) | quorum_id | controller's key | K-of-N from quorum | per `properties.mode`: `system/identity/public/cert/{h}` (3-key default) or `system/identity/internal/cert/{h}` (4-key advanced) |
| `identity-cert` | `controller` (sub-controller) | issuing controller's key | sub-controller's key | single-sig from `attesting` | `system/identity/internal/cert/{h}` (always internal) |
| `identity-cert` | `agent` | controller (3-key) or identifier (4-key) | runtime peer's key | single-sig from `attesting` | per `properties.mode`: `system/identity/{internal\|public}/cert/{h}` or `system/identity/relationships/{c}/cert/{h}` or embedded in cap envelope |
| `identity-cert` | `identifier` (4-key only) | controller | identifier's key | single-sig from controller | `system/identity/internal/cert/{h}` (cert is internal; identifier peer's KEY is contact-facing handle) |
| `identity-cert` | `<app-defined>` | per app | per app | per app (default: single-sig from `attesting`) | per app convention |
| `identity-rotation-handoff` | n/a | old handle key | new handle key | dual-sig (old + new) | same audience tier as target cert (per v3.3 §5.3) |
| `identity-rotation-recovery` | n/a | quorum_id | new handle key | K-of-N from quorum | same audience tier as target cert |
| `identity-retirement` | n/a | quorum_id | retired peer key | K-of-N from quorum | same audience tier as target cert |
| `revocation` (substrate-owned, used by identity) | n/a | revoker's key | target attestation hash | per consumer authority rule — identity uses `identity_is_authorized_revoker` (chain walks back to quorum) | same audience tier as target cert |

**Quorum self-event kinds** (defined by `EXTENSION-QUORUM`, not identity; identity is a consumer):

| Kind | Spec | Storage path | When identity uses it |
|---|---|---|---|
| `quorum-update` | `EXTENSION-QUORUM` §3.2 | `system/quorum/{q}/event/{h}` | Identity's quorum membership / threshold changes |
| `quorum-publish` | `EXTENSION-QUORUM` §3.3 | `system/quorum/{q}/event/{h}` | Identity publishes quorum state for contact-side discovery and compromise-recovery validation |

**Lifecycle event lookup.** `identity-rotation-handoff`, `identity-rotation-recovery`, and `identity-retirement` carry `properties.target_cert` referencing the cert they operate on. They do NOT carry their own `properties.function`; chain-walk predicates resolve the target cert to determine the conferred function (per v3.3 §3.6 `identity_confers_function`).

### 15.3 Handler operations (`system/identity`)

Seven primitive operations centered on graph primitives, per `EXTENSION-IDENTITY` v3.3 §6. Common patterns ("rotate the operational key", "revoke a agent entirely") are caller-side compositions. Identity-context create/supersede/revoke operations on `system/attestation` entities flow through identity's handler — which validates topology + identity-specific predicates — rather than directly through the substrate.

| Operation | Authorized by | Async? | Purpose |
|---|---|---|---|
| `configure` | Bootstrap: L0; subsequent: K_op via local cap | No | Initial peer setup; persist peer-config; issue local peer→K_op cap |
| `create_quorum` | Bootstrap: L0; subsequent: K_op via local cap | No | Initialize peer-config + delegate to `EXTENSION-QUORUM`'s `system/quorum:create` to persist a `system/quorum` entity at `system/quorum/{q_hex}` |
| `create_attestation` | Bootstrap: L0; subsequent: per kind+function (K_op, K_cf, or K-of-N quorum signatures) | Yes for K-of-N kinds | Create a new identity-context `system/attestation`; validates via `identity_verify_cert` per the kind+function topology |
| `supersede_attestation` | per kind | Yes for K-of-N kinds | Replace a live attestation with a successor (transitive supersession + predecessor-revival semantics from `EXTENSION-ATTESTATION`) |
| `revoke_attestation` | per kind authority — self-revocation always; authority-revocation per `identity_is_authorized_revoker` (chain walks back to quorum) | No | Issue a `kind="revocation"` substrate attestation targeting the cert; per-mode revocation when applied to `kind="identity-cert", function="agent"` |
| `publish_attestation` | K_op via local cap | No | Change publication mode of a `kind="identity-cert", function="agent"` attestation (`internal` / `public` / `per-relationship` / `embedded`) |
| `process_attestation` | Sync-hook (after tree:put on watched paths) | No | Validate an arriving attestation via `identity_verify_cert`; trigger kind-specific state updates (issue local cap on cert, update cached handle on rotation, cache quorum-publish on arrival, etc.) |

Quorum-membership operations (`system/quorum:update`, `system/quorum:publish`) are owned by `EXTENSION-QUORUM` — identity is a consumer. See `GUIDE-QUORUM.md`.

### 15.4 The seam with the role extension

Identity issues exactly one V7 cap per agent per live operational key: the local peer→K_op cap. K_op uses this cap as the caller capability when invoking role-handler operations. `EXTENSION-ROLE` v1.5's RL2 check (`is_attenuated`) validates that this caller capability covers the role's grants.

No identity mechanics change for role integration. No role mechanics change for identity integration. K_op is just a particular caller with a particular cap; the role handler doesn't care whether the caller is K_op, a group handler, or a continuation.

For the full identity-role integration story (RL2, role lifecycle, exclusion, role v2 considerations), see `GUIDE-CAPABILITIES.md` and `EXTENSION-ROLE.md` v1.5.

### 15.5 Where this fits in the broader extension stack

```
V7 capability system (chain verification)        ← unchanged by this extension
   ↑ used by
EXTENSION-IDENTITY v3.3 (this guide)              ← convention layer — owns peer-config + identity-binding
   ↑ uses
EXTENSION-ATTESTATION v1.1                        ← system/attestation entity type;
                                                    walk_attesting_chain, find_authorizing,
                                                    is_attestation_live, supersession,
                                                    generic revocation kind
   ↑ uses
EXTENSION-QUORUM v1.1                             ← system/quorum K-of-N node primitive;
                                                    verify_k_of_n_signatures, current_signer_set;
                                                    quorum-update / quorum-publish kinds

(see SYSTEM-IDENTITY-COMPOSITION.md for the orientation map across the three-extension stack)

EXTENSION-ROLE v1.5 (GUIDE-CAPABILITIES.md)       ← K_op's grants to others
   ↑ uses
local peer→K_op cap from identity                 ← bridges identity authorization to role mechanics

EXTENSION-GROUP v1.3                              ← multi-user / shared identities (reuses identity types
                                                    via signer_resolution: "identity-resolved" mode)
EXTENSION-CLUSTER (planned)                       ← coordinated agents (HA, replication)
```

The three-parallel-mechanisms invariant (v3.3 §2.2): V7 capability tokens, `system/attestation` entities, and `system/quorum` entities are structurally distinct entity types with no shared validation path. `verify_capability_chain` validates V7 caps; `identity_verify_cert` (composing `ATTESTATION` and `QUORUM` primitives) validates identity attestations; `verify_k_of_n_signatures` validates quorum entities. Don't mix them.

### 15.6 Spec section pointers

**EXTENSION-IDENTITY.md v3.3 (current):**
- §1 — overview, design principles, configuration progression
- §2 — the peer graph; substrate framing; cert-chain framework; three-parallel-mechanisms invariant; function correspondence
- §3 — type definitions (peer-config, identity-binding); §3.6 identity-specific validators (`identity_topology_for`, `identity_is_quorum_link`, `identity_is_authorized_revoker`, `identity_verify_cert` topology-first orchestrator)
- §4 — identity attestation kinds (4 owned + universal `revocation`); §4.2 `identity-cert` with required `properties.mode`; §4.2a publication modes; §4.2b sub-controller chains; §4.3 `identity-rotation-handoff`; §4.4 `identity-rotation-recovery`; §4.5 `identity-retirement`; §4.6 `revocation` consumer rule
- §5 — path conventions; §5.1 path layout; §5.2 audience and sync (dual-subtree two-tier sync); §5.3 canonical storage path (mode-driven, no runtime shape lookup)
- §6 — seven handler operations; §6.8 `process_attestation`; §6.9 bootstrap exemption
- §7 — async signature gathering
- §8 — security considerations (§9.4 compromise-recovery fail-closed; §9.5 long-lived cap survival)
- §10 — conformance
- §11 — configurations continuum (V7-only / 1-of-1 / three-key / four-key / multi-binding / multi-controller / sub-controller / app-defined functions / parent-managed)
- §12 — cross-extension invariants
- §13 — examples

**EXTENSION-ATTESTATION.md v1.1 (substrate):**
- §3 — `system/attestation` entity type; `properties.kind` open vocabulary; kind-ownership table
- §4 — abstract operations contract (`:create`, `:supersede`, `:revoke`, `:verify`); §4.0 ops contract
- §5 — graph operations (`walk_attesting_chain`, `find_attestations_targeting`, `is_attestation_live`, `find_live_head`, `default_find_authorizing`, `find_attestations_with_supersedes`, `find_attestations_with_kind`); §5.1 with `max_depth: 32` default; transitive supersession + predecessor-revival
- §9 — index invariants (`attesting`, `attested`, `properties.kind`, `supersedes`)

**EXTENSION-QUORUM.md v1.1 (substrate):**
- §3 — `system/quorum` entity type; §3.2 `quorum-update`; §3.3 `quorum-publish` with previous-pinning rule
- §4 — `verify_k_of_n_signatures`; §4.2 `current_signer_set` chain walk + `as_of` time-travel; §4.2.1 cache-invalidation contract; §4.3 `is_quorum_id`
- §5 — pluggable signer-resolution (`concrete` default; `identity-resolved` registered by `EXTENSION-IDENTITY`); §5.2 `max_resolver_depth = 8` + cycle detection; fail-closed unknown-mode

**ENTITY-CORE-PROTOCOL.md v7.37:**
- §1.5a — Identity and Access Management (informative reference into the three-extension layering)
- §3.7 — handler-op type registration convention
- §5 — capability system (signatures, chain verification)
- §5.5 — root capability rule (line 1968)
- §6.5 — dispatcher-level signature ingestion (uniform across kernel/substrate/identity/extension ops)

**SYSTEM-IDENTITY-COMPOSITION.md (Stable):** orientation map for the three-extension stack — read this first when navigating the substrate.

**Companion guides:**
- GUIDE-CAPABILITIES.md — RL2 / R0 / R1; admin-coherent vs full-admin; identity-role integration depth
- GUIDE-ATTESTATION.md — substrate attestation primitive (lifecycle, properties.kind, revocation, find_authorizing, supersession)
- GUIDE-QUORUM.md — substrate quorum primitive (K-of-N, signer-resolution, quorum-update / publish, is_quorum_id)
- GUIDE-MULTISIG.md — cap-layer multi-sig primitive (orthogonal to entity-layer QUORUM; see §6.2 arbiter rule for when to use which)

**Other extensions:**
- EXTENSION-ROLE.md v1.5 — role definitions, assignments, derivation
- EXTENSION-GROUP.md v1.3 — group extension; uses identity-resolved signer-resolution mode
- PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE.md — registry layer; backend continuum (self-hosted, DID:web, DHT, blockchain)
- EXPLORATION-IDENTITY-LENSES-AND-CONVERGENCE.md — design path that drove the substrate split
- EXPLORATION-IDENTITY-CONSOLIDATED-REVIEW.md — v2.0 unified-attestation key-graph design that v3.2 carries forward

---

## 16. Document history

- **v1.3 — substrate-stack alignment with EXTENSION-IDENTITY v3.3 + EXTENSION-ATTESTATION v1.1 + EXTENSION-QUORUM v1.1:** Mechanical and structural pass to align the guide with the substrate split (the attestation/quorum primitive extraction and EXTENSION-IDENTITY minimization) plus the v3.2/v3.3 cross-impl-feedback batch. Changes:
  - **Header / spec refs.** v1.2 → v1.3; spec references bumped to v3.3 / v1.1 / v1.1 / v7.37; new substrate explainer paragraph in the status block.
  - **Kind names normalized.** `kind="cert"` → `kind="identity-cert"`; `kind="cert-rotation-handoff"` → `kind="identity-rotation-handoff"`; `kind="cert-rotation-recovery"` → `kind="identity-rotation-recovery"`; `kind="cert-retirement"` → `kind="identity-retirement"`. Per v3.3 §3.3 namespace policy — all identity-context kinds are namespace-prefixed `"identity-"`.
  - **Path conventions.** Identity certs now live at `system/identity/{internal|public|relationships/{c}}/cert/{hash_hex}` (NOT `attestation/{hash}`). Quorum entity moved from `system/identity/quorum/{q}` to **`system/quorum/{q_hex}`** (substrate-owned). Quorum self-events (`quorum-update`, `quorum-publish`) at `system/quorum/{q}/event/{h}`. Lifecycle events (`identity-rotation-handoff`, `identity-rotation-recovery`, `identity-retirement`) inherit target cert's audience tier per v3.3 §5.3.
  - **`properties.mode` REQUIRED on ALL identity-cert kinds** (v3.3 §4.2) — set at create-time per intended deployment shape; cert's path is fixed for its lifetime. Eliminates the v3.0 in-flight rotation race.
  - **Validator name.** `verify_attestation` (v3.0 wrapper) → `identity_verify_cert` (v3.3 topology-first orchestrator over substrate primitives, per v3.3 §3.6).
  - **Substrate awareness.** §1 gains a "Three-extension layering" callout. §4.2 tier-sync table adds `system/quorum/{trusts_quorum}/...` as part of identity's published surface (dual-subtree two-tier sync per v3.3 §5.2). §12 quorum-entity row updated — Bob's peer DOES sync the quorum subtree in v3.3.
  - **§13.3 / §13.4 pitfalls.** Reframed around the three-parallel-mechanisms invariant. Raw `tree:put` pitfall now includes `system/quorum/...` and `system/attestation/...` paths.
  - **§15 Reference fully overhauled.** §15.1 entity types table reduced to identity-owned types + reference rows for substrate-owned types. §15.2 attestation kinds table replaced with v3.3-aligned names + per-function rows + quorum-self-event note. §15.3 handler ops table edits to reflect substrate orchestration. §15.5 stack diagram updated for the three-extension layering. §15.6 spec section pointers fully replaced — bumps to v3.3/v1.1/v1.1/v7.37 plus new entries for ATTESTATION, QUORUM, SYSTEM-IDENTITY-COMPOSITION; companion guides list adds GUIDE-ATTESTATION + GUIDE-QUORUM.
  - **No structural changes to §1–§14 narratives.** User-facing topical organization preserved; mechanism unchanged. The guide's mental model (three-key default vs four-key advanced; recovery via quorum; cross-peer recognition via cached `quorum-publish`; etc.) is identical. This is a vocabulary + path + cross-reference alignment pass.

- **v1.1 — vocabulary alignment with EXTENSION-IDENTITY v3.0 + Amendment 1:** Mechanical pass aligning the guide to the v3.0 spec. `kind="device"` → `kind="identity-cert", function="agent"` (kind name + `device_attestation` field → `runtime_attestation`); §2 retitled "The peer graph and common shapes" (was "The key graph"); §1 reframed around the peer-substrate insight ("identity is a graph of peers"); "operational key" → "controller" where the role-bearer is meant (kept as "operational key" in cap-name contexts — `peer→operational-peer cap` — and where the literal cryptographic key is meant — `K_op_alice`); spec references throughout bumped v2.1 → v3.0. "Role" reserved for `EXTENSION-ROLE.md` per v3.0 vocabulary policy; identity-side concept is "function" (operational function, runtime function, etc.). No structural changes to the guide; mechanism unchanged. See `proposals/implemented/PROPOSAL-IDENTITY-V2-VOCABULARY-AND-FIXES.md` for the v2.1 → v3.0 spec changes this guide tracks.

- **v1.0 — full alignment with EXTENSION-IDENTITY v2.1:** Completed the inline rewrite that v0.4 deferred. The v0.4 vocabulary mapping table is removed. Section structure preserved (the user-facing topical organization is fine for a guide); section *content* rewritten under v2 framing. Specifically:
  - **§3 Provisioning** — substantively rewritten. Leads with the **three-key default** per v2.1 §11.3 (op = contact-face; no separate Public_alice). Opening paragraphs frame *when* a user opts into identity, *why* (recovery, multi-device, stable cross-peer recognition), and *what they're committing to* (a quorum custody ceremony plus three key types vs. V7-alone's one). §3.1 entity-output list reduced from 8 items to 7 (no separate `public-identity-attestation` in three-key default). §3.2 walks the v2.1 configure handler signature with polymorphic `handle_attestation` field; cross-refs the seven-primitive op surface. New §3.5 covers four-key advanced as an opt-in variant with explicit *when to choose* / *when to stay three-key* tradeoff.
  - **§4 Adding a agent** — pairing flow re-expressed as `create_attestation(kind="device")` plus existing `kind="identity-cert", function="controller"` and `kind="quorum-publish"` attestations carrying forward. Tiered-sync table gains a `system/identity/contacts/{handle_hex}/quorum-publish` row (per v2.1 §5.1). Publication-modes table updated for `kind="device"` storage paths. (v1.1 renamed `kind="device"` → `kind="identity-cert", function="agent"` per v3.0 vocabulary.)
  - **§5 Connecting to a contact** — TOFU caches `kind="quorum-publish"` (the v2 trust anchor for compromise-recovery validation). Recognition validates `kind="device"` attestations via verifier-acceptance rules per v2.1 §4.4. The v1.x three-policy cache-miss framework simplified to the v2 fail-closed default; deployment-level recovery flows are application compositions, not architecture-defined policy.
  - **§6 Composing with roles** — explicit framing that this section is identity-focused; deeper identity-role integration lives in GUIDE-CAPABILITIES.md and EXTENSION-ROLE.md v1.5. K_op vocabulary; operational-key-confinement invariant cross-referenced (v2.1 §9.2 / §12.2).
  - **§7 Rotating the operational key** (was "Operator rotation") — rewritten as the two-attestation flow (`kind="identity-retirement"` + new `kind="identity-cert", function="controller"`); explicitly contrasts three-key default (operational rotation IS published-handle rotation; contact-side surface fires) vs. four-key advanced (operational rotation stays internal; contacts unaffected — *this is the property four-key advanced exists for*).
  - **§8 Rotating the published handle** (was "Public identity rotation") — `kind="identity-rotation-handoff"` (dual-sig) for routine rotation; `kind="identity-rotation-recovery"` (K-of-N from quorum) for compromise. Lead paragraph clarifies what the published handle IS in three-key vs. four-key. §8.2 fail-closed validation against cached `kind="quorum-publish"` per v2.1 §9.4.
  - **§9 Quorum updates** — rewritten as the unified rotation flow (`kind="quorum-update"` + chained `kind="quorum-publish"` per v2.1 §4.6 supersedes-by-previous-quorum rule). No dedicated `rotate_quorum` op in v2.
  - **§10 Runtime peer revocation** — re-expressed as per-mode `revoke_attestation` calls (one per live `kind="device"` attestation per mode); v1.x `revoke_peer(scope: "all")` op no longer exists.
  - **§11 Custody variants** — restructured. §11.1 recaps three-key default; §11.2 cross-refs four-key advanced; the rest covers genuine variants (hardware-backed, 1-of-1, concurrent multi-operational, multi-binding, parent-managed, multi-identity host, identity transition flows). The v1.x "Op = Public_alice collapse" variant retired (it's the v2 default, not a variant). Pattern A (parent-to-self transition) re-expressed in v2 vocabulary.
  - **§12 What Bob sees vs Alice manages** — table fully replaced with v2 entity types and kinds. New rows for K_op visibility (visible in three-key, not visible in four-key); for `kind="quorum-publish"` (the trust anchor) and `kind="identity-cert", function="identifier"` (four-key only); for kind-discriminated rotation entities. Concluding paragraph spells out the central three-key vs. four-key tradeoff at the "what does Bob need to process" level.
  - **§13 Pitfalls and antipatterns** — terminology updates throughout. §13.1 reframed for both three-key and four-key cases; §13.3 updated to two-mechanisms invariant per v2.1 §3.6; §13.9 cache-miss policy framing aligned with v2's fail-closed default.
  - **§14 Worked example** — re-walked under v2 framing. The four rotation surfaces (K_acme_op retires; partnership_server retires; Bob retires; Acme dissolves) expressed as kind-discriminated attestation operations on the unified `system/identity/attestation` entity. Cross-references updated to v2.1 §9.5 (long-lived cap survival) and EXTENSION-GROUP §6.5 (dissolve flow).
  - **§15 Reference** — full table replacement. §15.1 lists three primary entity types + identity-binding helper. New §15.2 attestation kinds table (eight kinds with sig topology and storage paths). §15.3 handler ops list reduced to seven primitives. §15.5 stack diagram updated (EXTENSION-GROUP v1.3 cross-ref). §15.6 spec section pointers fully updated to v2.1 (replaces the v1.x amendment-proposal pointers).
  - Header status bumped v0.4 → **v1.0**; the v0.4 vocabulary mapping table is removed (no longer needed). Mechanism unchanged across the rewrite.

- **v0.4 — partial vocabulary alignment with EXTENSION-IDENTITY v2.0:** Header updated (spec reference now v2.0). Top-level vocabulary mapping table added so readers can interpret v1.x terms still appearing in §3+. **§1 (What identity is)** rewritten with the key-graph framing: identity IS the graph; mechanism is the unified `attestation` entity validated by kind-discriminated `verify_attestation`; spec reference updated to V7 §1.5a + EXTENSION-IDENTITY v2.0. **§2** rewritten from "The four-layer model" to "The key graph and common shapes" — covers the full progression (V7-only floor → V7+role minimum-viable → three-key default → four-key advanced); explicit comparison table for three-key vs four-key; "properties emerge from the graph" section linking key roles to attestation kinds; "what stays cold and what's hot" table. §3–§14 retain v0.3 narrative pending a subsequent refinement pass; the vocabulary mapping table at the top guides interpretation.

- **v0.1:** Initial draft alongside the v1.0 spec. Walks the four-layer model, provisioning ceremony, device pairing, TOFU contact recognition, role v1.2 composition for access grants, rotation flows, custody variants, audience separation, pitfalls. Banner flagged "pending revision" per D1–D6 of the v1.1 amendment proposal — the runtime-peer-set discussion was out of date; the four-layer model and rotation flows still applied.

- **v0.2:** Refresh integrating the v1.1 amendment proposal (V1–V8 + C1–C3 + D1–D6) plus the Phase 5 architectural-review refinements. Major changes:
  - **§2 four-layer model table** updated: Public_alice now signs per-peer `runtime-peer-attestation`s rather than a global `runtime-peer-set`; quorum signs `public-quorum-attestation` for compromise-recovery validation.
  - **§3.1 provisioning output** restructured: `runtime-peer-set` removed; `public-quorum-attestation` (V1) added; per-peer `runtime-peer-attestation` with publication-mode discussion (default: internal).
  - **§3.2 configure handler** now produces `public-quorum-attestation` (V2); operational invariant called out (§3.10's two-parallel-mechanisms MUST).
  - **§4.1 pairing flow** updated to per-peer attestations; phone gets internal-mode attestation by default, promoted later via `publish_runtime_peer_attestation`.
  - **§4.2 tiered sync** rewritten: tier separation = external sync exposure, not which Alice peers hold what; per-runtime-peer-identity peer-config rule called out; relationships subtree added.
  - **§4.3 publication modes** new section walking the four modes (internal / public / per-relationship / embedded).
  - **§5.1 first contact** updated for registry resolution (per backend); `public-quorum-attestation` cached at TOFU; per-backend reflection latency note.
  - **§5.2 recognizing agents** rewritten for per-peer attestation lookup; verifier cache-miss policy (fetch-on-demand / reject-and-escalate / embedded-only) introduced; address book example updated.
  - **§8.2 compromise recovery** revised: validation against cached `public-quorum-attestation` (V4); fail-closed if not cached; privacy opt-out per V5 with out-of-band re-establishment fallback (V6).
  - **§10.1 revoke_peer flow** updated: scope param (`internal` / `public` / `per-relationship` / `all`) per D5; per-mode propagation paths.
  - **§11 custody variants** extended: §11.5 parent-managed identity (per `PROPOSAL-IDENTITY-RECOVERY-VALIDATION §12.6`); §11.6 multi-identity host machine (per EXTENSION-IDENTITY §3.8 + PROPOSAL-EXTENSION-GROUP §13.7); §11.7 identity transition flows (per EXTENSION-IDENTITY §6.3 — parent-to-self, operator transition, cross-org pointer).
  - **§12 What Bob sees** table updated to per-peer attestation rows by mode; `public-quorum-attestation` row added; runtime-peer-set row removed.
  - **§13 pitfalls** gains §13.7 (path tiers as fleet-distribution boundaries — wrong reading), §13.8 (sharing peer-config across agents on the same machine), §13.9 (silently TOFU-ing every cap without an attestation).
  - **§14.1/§14.2 reference tables** updated: `runtime-peer-set` removed; `public-quorum-attestation` added; per-peer attestation modes called out; handler ops updated (`publish_runtime_peer_attestation` replaces `publish_runtime_peer_set`; `revoke_peer` gains scope; `configure` and `rotate_quorum` produce `public-quorum-attestation`).
  - **§14.5 spec section pointers** restructured by document; v1.1 amendment proposal sections explicitly enumerated.
  - **Pending-revision banner removed**; status updated to v0.2. Guide tracks both the published EXTENSION-IDENTITY spec and the in-flight v1.1 amendment proposal; further refinement expected as the proposal lands and downstream implementations test it.

- **v0.3:** Added §14 "Worked example: longest cross-extension cap chain through rotations" per PROPOSAL-IDENTITY-ARC-FIXES IA29. Walks the four-extension chain (Acme group's quorum → Op_acme delegation + local peer→Op cap on `partnership_server` → role:assign granting Bob admin → Bob's role-derived cap → Bob's compute install grant) and four rotation surfaces (Op_acme retires; `partnership_server`'s agent retires; Bob retires his agent; Acme dissolves), showing where rotation does and doesn't bite per IA1–IA5, IA8, IA15, IA26, IA27. Renumbered prior §14 (Reference) to §15 and §15 (Document history) to §16.
