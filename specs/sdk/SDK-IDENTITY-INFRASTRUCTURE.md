# SDK Identity Infrastructure — Specification

**Version**: 0.5
**Status**: Active
**Depends**: `SDK-OPERATIONS.md` (v1.6+); `EXTENSION-ATTESTATION.md` (v1.2+); `EXTENSION-QUORUM.md` (v1.2+); `EXTENSION-IDENTITY.md` (v3.5+); `EXTENSION-ROLE.md` (v2.0+).
**Companion**: `sdk-domain/guides/GUIDE-IDENTITY-SDK.md` — application-developer-facing tutorial; sister to `core-protocol-domain/guides/GUIDE-IDENTITY.md`.
**Audience**: SDK implementers (Go peer team, Rust impl, Python impl, future impls). Cross-references for application developers building on top.

---

## 1. Scope and audience

This document specifies the SDK surface for the identity stack — the set of operations, helpers, and on-disk conventions that SDK implementations expose so application code can build identity-aware peers.

**Scope:**
- Substrate operations surface (`system/attestation`, `system/quorum`).
- Identity convention-layer surface (`system/identity` — handler ops + bootstrap path).
- Identity-coherence aspects of `system/role` (the role↔identity composition seam).
- Bootstrap helper library contract (shared across impls).
- Rotation lifecycle hooks.
- On-disk identity bundle layout (cross-impl interop; SHOULD-tier).

**Out of scope:**
- The protocol behavior of identity ops — see `EXTENSION-IDENTITY.md`, `EXTENSION-ATTESTATION.md`, `EXTENSION-QUORUM.md` for normative protocol semantics.
- Application UX (pairing dialogs, recovery flow UIs, cold-key custody mechanisms — those are application territory).
- Group surface — placeholder §9 below; lands when EXTENSION-GROUP v1.5 stabilizes.

**Relationship to other SDK docs:**
- `SDK-OPERATIONS.md` covers L0/L1 base ops (tree, dispatch, query, watch, connect, peer lifecycle). This doc adds the identity-stack surface.
- `SDK-EXTENSION-OPERATIONS.md` documents per-extension SDK wrappers for non-identity-stack extensions (continuation, subscription, revision, history, query, inbox, transaction, compute, network, clock, content, type, role). The identity-stack extensions (`system/attestation`, `system/quorum`, `system/identity`) are documented HERE rather than in `SDK-EXTENSION-OPERATIONS.md` because the surface composes across all three and includes a non-handler-op helper library.
- `GUIDE-SDK-PATTERNS.md` covers L2 patterns. This doc references those patterns where applicable; adds identity-specific patterns in `GUIDE-IDENTITY-SDK.md`.

**Conformance philosophy.** Per `SDK-OPERATIONS.md` §scope: this spec defines **operation semantics — what goes in, what comes out, what errors can occur — not API shapes**. Language-idiomatic differences (method chaining vs functional options vs context managers vs builders) are expected and encouraged. SHOULD-tier convergence on signatures and helper-library contracts; MUST-tier on operation semantics that compose with protocol-layer guarantees.

---

## 2. Reading order

1. **This spec (§3-§9)** — for SDK implementers building the surface.
2. `sdk-domain/guides/GUIDE-IDENTITY-SDK.md` — application-developer-facing walkthroughs ("Do you need identity?"; provisioning; pairing; rotation; recovery).
3. `core-protocol-domain/specs/ARCHITECTURE-IDENTITY-INFRASTRUCTURE.md` — architecture overview; the layered picture and decision log.
4. `EXTENSION-IDENTITY.md` v3.5, `EXTENSION-ATTESTATION.md` v1.2, `EXTENSION-QUORUM.md` v1.2 — normative protocol behavior.
5. `core-protocol-domain/guides/GUIDE-IDENTITY.md` — architecture-spec-language deep guide.

---

## 3. The identity stack at a glance

Three extensions compose into the identity stack:

```
EXTENSION-ATTESTATION    — substrate edge type: signed claim "A asserts X about B"
       ↑
EXTENSION-QUORUM         — substrate node primitive: K-of-N signing roster
       ↑
EXTENSION-IDENTITY       — structured composition layer: function (controller / agent /
                           identifier / app-defined), recovery flows, publication modes
```

Identity v3.5 introduces essentially two of its own entity types — `peer-config` (per-agent local state) and `identity-binding` (helper inner type) — and orchestrates the substrate primitives (`system/attestation`, `system/quorum`) into user-visible flows like `:configure`, `:create_attestation`, `:supersede_attestation`. See `core-protocol-domain/specs/SYSTEM-IDENTITY-COMPOSITION.md` for the navigation overview.

**Three function values** discriminate identity-context attestations: `controller` (the persona managing the peer; published as the contact-side handle in 3-key default), `agent` (per-device daemon; attested by controller), `identifier` (4-key advanced — separates the published handle from the rotating controller key). App-defined functions are also permitted.

**Four properties.kind values** for identity-context attestations: `identity-cert`, `identity-rotation-handoff`, `identity-rotation-recovery`, `identity-retirement`.

**Four publication modes** (`properties.mode`) determine canonical storage path: `internal` (identity's own agents only), `public` (registry/two-tier sync), `per-relationship` (named contact only), `embedded` (no tree path; cap-embedded).

---

## 4. SDK access levels for identity ops

Identity ops fall into two access-level classes per `SDK-OPERATIONS.md` §2.7. SDKs MUST surface this distinction so application code knows which path it's on.

### 4.1 L0 bootstrap exemption (pre-controller-cap)

The first `system/identity:configure` call on a new peer cannot go through dispatched EXECUTE because the local peer→controller cap doesn't exist yet — the cap that authorizes future identity ops is the OUTPUT of `:configure`, so it cannot be the INPUT. Per `EXTENSION-IDENTITY.md` §6.5 startup boundary:

> Startup is the period from peer instantiation through the first `system/identity:configure` call that succeeds and issues at least one local-peer→controller cap. During startup, operations that establish initial peer state (the first `:create_quorum`, the first `:create_attestation` calls producing top-level controller certs, the initial `:configure`) run via the SDK's L0 direct-store path — peer-owner authority, no dispatch authorization required (no authority chain has been provisioned yet). After startup completes, all dispatched EXECUTEs require capability validation per V7 §5.

**SDK conformance:**
- The bootstrap helper library (§7) is the canonical L0 surface for the bootstrap step.
- SDK MUST route the first `:configure` call through L0 direct-store (not dispatched EXECUTE).
- SDK MUST NOT accept further L0 direct-store calls into identity-handler-owned namespaces post-bootstrap (per `EXTENSION-IDENTITY.md` §6.3 phase-2a normative — local-handler-op writes vs sync-arrival writes are distinguished; L0 is reserved for the peer-owner's bootstrap code).
- SDK MUST surface the L0 boundary visibly per `SDK-OPERATIONS.md` §2.7.1 (separate accessor or distinct names).

`SDK-OPERATIONS.md` §2.7.1 L0 use-cases table SHOULD include a row for identity bootstrap (added in Step 4 matching pass).

### 4.2 L1 dispatched ops (post-bootstrap)

After `:configure` completes, all subsequent identity ops run as dispatched EXECUTE under the local peer→controller cap (or appropriate sub-cap). Standard `peer.execute(...)` path per `SDK-OPERATIONS.md` §2.3 dispatch flow.

The same mechanism applies to substrate ops (`system/attestation:*`, `system/quorum:*`) — those are always dispatched (substrate has no bootstrap exemption; they only operate on already-installed quorums and on attestations that flow through `:create_attestation` etc.).

**Capability:** the local peer→controller cap (issued by `:configure`) is the typical caller cap. Other identity ops (attestation creation, supersede, revoke, publish) take this cap as their authority. Sub-controller chains and agent-authority ops use the appropriate intermediate cap per `EXTENSION-IDENTITY.md` §6.

---

## 5. Substrate operations surface

The substrate primitives expose a small, generic op surface. SDKs SHOULD provide typed wrappers; underlying mechanism is `peer.execute("system/attestation", ...)` and `peer.execute("system/quorum", ...)`.

> **Request/result entity shapes (where the wire schema lives).** The CBOR structure of request and result entities exchanged via EXECUTE — field names, types, optionality, encoding — is normatively defined at the protocol-side specs, NOT at this SDK-side spec. SDK helpers deserialize against those schemas. Cross-impl wire compatibility derives from the protocol-side schemas.
>
> Sources of truth:
> - `EXTENSION-ATTESTATION.md` §6 — substrate op input/output types.
> - `EXTENSION-QUORUM.md` §6 — substrate op input/output types.
> - `EXTENSION-IDENTITY.md` §6 — convention-layer op input/output types.
> - `EXTENSION-ROLE.md` §4.2 — role op input/output types.
> - V7 §3.7 — handler-op type registration convention (`{handler-path}/{op-name}-request` / `{handler-path}/{op-name}-result`); types registered at `system/type/{type_name}` during handler installation.
>
> SDKs SHOULD share request/result type definitions across the impl's own handler code and SDK client (single source of truth per impl). When two impls disagree on a field name, the protocol-side spec resolves it; the SDK is the deserialization layer.

### 5.1 `system/attestation` operations

Per `EXTENSION-ATTESTATION.md` §6.

| Op | SDK wrapper sketch | Notes |
|---|---|---|
| `:create` | `peer.attestation.create(attesting, attested, properties, supersedes?)` | Generic signed-claim creation. Authorized via caller cap covering `system/attestation:create`. Path is required (`SDK-OPERATIONS.md` path-as-resource MUST per V7 §3.2). |
| `:supersede` | `peer.attestation.supersede(predecessor_hash, new_properties)` | Strict-by-design wrapper: copies `attesting`/`attested` from predecessor (correct for VC, reputation, audit, provenance — supersession on the same subject by the same attester). Identity controller rotation legitimately changes both — the IDENTITY-layer `:supersede_attestation` (§6) handles that case via REBIND_KINDS by calling substrate `:create` directly with explicit `supersedes`. |
| `:revoke` | `peer.attestation.revoke(attestation_hash)` | Generic revocation as `properties.kind = "revocation"` attestation. |
| `:verify` | `peer.attestation.verify(attestation_hash)` | Single-sig validation per §4.1; consumer-driven liveness check via §4.3. |

**Coherent capability subsection** (per `SDK-EXTENSION-OPERATIONS.md` §15.1 GR1): direct `tree:put` of `system/attestation` entities is permitted by V7 kernel but bypasses substrate validation. Application grants SHOULD cover the `system/attestation:*` operations rather than raw `tree:put`. The SDK MAY provide both surfaces but SHOULD make the boundary visible (separate accessors or distinct names) per `SDK-OPERATIONS.md` §2.7 access-level discipline.

### 5.2 `system/quorum` operations

Per `EXTENSION-QUORUM.md` §6.

| Op | SDK wrapper sketch | Notes |
|---|---|---|
| `:create` | `peer.quorum.create(signers, threshold, signer_resolution?, name?)` | K-of-N node primitive. `signer_resolution` defaults to `concrete`; `identity-resolved` mode is registered by EXTENSION-IDENTITY at install time (per §3.4). |
| `:update` | `peer.quorum.update(quorum_id, new_signers, new_threshold, supersedes)` | Self-event attestation; signed by the quorum's current K signers. |
| `:publish` | `peer.quorum.publish(quorum_id, published_handle?, properties?, supersedes?)` | Publication snapshot; `published_handle` is a generic consumer-extension hook (per `EXTENSION-QUORUM.md` §3.3 v1.2 abstraction — substrate stays neutral on field semantics; consumers define meaning within their own framework). |
| `:verify` | `peer.quorum.verify(quorum_id, target_hash, as_of?)` | K-of-N signature validation against the resolved signer set; `as_of` for historical-state lookup per §5.2 IDENTITY-2. |

**Closed-namespace ownership.** Per `EXTENSION-QUORUM.md` §3.4 v1.2 normative: the `system/quorum/...` subtree is owned by EXTENSION-QUORUM. SDKs MUST NOT write to or expose write surfaces inside `system/quorum/...` outside of these handler ops.

---

## 6. Identity convention-layer surface

Per `EXTENSION-IDENTITY.md` §6.

| Op | SDK wrapper sketch | Notes |
|---|---|---|
| `:configure` | `peer.identity.configure(trusts_quorum, bindings)` | **L0 bootstrap-exempt per §4.1.** SDK MUST route the first call through direct-store. Validates bindings, enumerates live controller certs, verifies K-of-N signatures, issues local-peer→controller caps at `system/capability/grants/identity/peer-to-controller/{controller_hex}` (per PI-9), binds cap signature siblings at `{cap_path}/signature` (per PI-10), persists peer-config. Errors per §6 binding error contract: `404 binding_cert_not_found`, `400 binding_missing_*_cert`, `400 binding_cert_wrong_kind`, `400 binding_controller_not_live`. |
| `:create_quorum` | `peer.identity.create_quorum(signers, threshold)` | Identity-specific quorum creation with `signer_resolution: "identity-resolved"` registered. Wraps substrate `system/quorum:create`. |
| `:create_attestation` | `peer.identity.create_attestation(kind, function, mode, attesting, attested, properties)` | Per-mode dispatch: `internal/public/per-relationship` bind to canonical path per §5.3; `embedded` returns the entity without binding. Phase 1 valid-modes-per-function enforcement (`400 invalid_mode_for_function` per PI-11). |
| `:supersede_attestation` | `peer.identity.supersede_attestation(predecessor_hash, new_properties, new_attesting?, new_attested?)` | REBIND_KINDS-aware: for `identity-cert` (and other kinds in REBIND_KINDS), calls substrate `:create` directly with explicit `supersedes` and caller-supplied new attesting/attested (controller rotation case — substrate `:supersede` would block). For non-rebind kinds, delegates to substrate `:supersede` and preserves attesting/attested. |
| `:revoke_attestation` | `peer.identity.revoke_attestation(attestation_hash)` | Cap cascade-by-default per PI-13: walks `system/capability/grants/identity/peer-to-controller/*` and unbinds caps whose `grantee` matches the revoked controller's `attested`; signature siblings unbind alongside. Convergence framing — peers without the revocation observed yet remain in the convergence window; deployments with stricter requirements layer optional cap-validation-time re-checks (MAY). |
| `:publish_attestation` | `peer.identity.publish_attestation(attestation_hash, target_mode)` | MOVE semantics with tombstone-style recovery per PI-3: bind new path, unbind old; on partial failure, emit a `recovery_signal` controller-event per PI-5. SDK SHOULD surface the event to the controller-equivalent operator interface so it can be cleared. |
| `:process_attestation` | (sync-hook; not directly invoked) | Validation/side-effect split per §6.3 + PI-5. SDKs typically do not expose this as an explicit op — it fires automatically on attestation arrival. Failure-subset events (`recovery_signal` + `failure_observation` subkinds) emit at `system/identity/events/...`; SDK MAY provide a stream to consume them. |

**Identity-cert kinds.** Identity-context attestations carry `properties.kind ∈ {identity-cert, identity-rotation-handoff, identity-rotation-recovery, identity-retirement}` and `properties.function ∈ {controller, agent, identifier, app-defined}`. The (kind, function, mode, [contact_id]) tuple deterministically computes the canonical storage path per `EXTENSION-IDENTITY.md` §5.3. **`properties.mode` is REQUIRED on ALL identity-cert kinds** per the v3.2 substrate split (eliminates the in-flight rotation race that v3.0's runtime `is_handle_bearing_in_current_shape` lookup had).

**Coherent capability subsection.** Identity ops are the canonical authorized path for creating identity-cert entities. Direct `tree:put` of `system/attestation` entities under `system/identity/...` paths is permitted by V7 kernel but bypasses identity validation (PI-11 valid-modes enforcement; supersede REBIND_KINDS discipline). Application grants SHOULD cover `system/identity:*` operations, not raw `tree:put` to `system/identity/...` paths.

**Layer 1 / Layer 2 separation (V7 §5.10 v7.52, normative for protocol; informational here).** Capability verification on identity-derived caps has two architecturally separate layers: Layer 1 (the cap-chain verdict — pure cap-layer state: signatures, structural linkage, attenuation, caveats, TTL, revocation entries — cross-peer deterministic) and Layer 2 (local policy applied to a valid chain — MAY use arbitrary local state, free to diverge across peers). The identity convention layer's `IdentityBindingChecker` (EXTENSION-IDENTITY §12.3 v3.10) is the canonical Layer 2 example. SDKs that compose identity-derived chains for application use **MUST NOT** introduce hooks that mutate the Layer 1 verdict; SDK wrappers that compose authorization decisions over a valid chain **MAY** apply Layer 2 policy. SDKs **SHOULD** expose Layer 1 as a distinct entry point (a `verify_chain`-style function that consults no extension state) from composite authorization paths (a `verify_request`-style function composing Layer 1 with Layer 2 post-gates); the Go reference impl's separation (`core/capability/delegation.go:VerifyChain` vs `core/protocol/auth.go:VerifyRequestWithBinding`) is the recommended shape.

---

## 7. Identity-coherence aspects of `system/role`

The role surface itself (`:define`, `:assign`, `:unassign`, `:exclude`, `:unexclude`, `:re-derive`, `:delegate`) lives in `SDK-EXTENSION-OPERATIONS.md` §13 (refreshed in Step 4 matching pass against role v2.0). This section covers only the role↔identity composition seam.

**Caller capability for role ops.** When the identity extension is installed, the typical caller cap for `system/role:*` ops is the local peer→controller cap issued by `:configure`. Role's RL2 (caller-cap-covers-derived-grants) check uses this cap. The cap resolves to the local peer's controller (via the cap's grantee chain), and the role-derived caps issued downstream are root caps (per role v2.0 PR-1 — `parent: null`, `granter: local_peer_identity.content_hash`).

**Multi-role per (peer, context).** Per role v2.0 §5.5 — `:assign` may be called multiple times for the same (peer, context) with different role names; tokens compose; verification per V7 picks the presented token. The SDK helper for this is straightforward; no special composition logic at the SDK layer.

**Multi-agent concurrent re-derive.** When multiple identity agents run concurrently and a role definition mutates, each agent's `:re-derive` cascade is independent; per-assignee re-derivation per role v2.0 §5.5 (issue T_new, then revoke T_old; SI-15 skipped_grantees on RL2 mid-cascade failures). The SDK doesn't need additional coordination at this layer — the protocol-side per-write CRDT contract (per EXTENSION-REVISION) and the role-extension's three-layer exclusion model handle convergence.

**Bootstrap composition with identity.** Role's startup-time L0 derivation path (per `EXTENSION-ROLE.md` §4.5) and identity's L0 bootstrap path both run pre-handler-registration. Per role v2.0 PR-4, L0 access remains available to peer-owner code throughout the peer's lifetime (intentional; out-of-scope of dispatch). The SDK's bootstrap helper library (§8) operates against the L0 surface for both layers in startup ordering: identity's `:configure` first (issues local peer→controller cap), then role definitions and assignments via dispatched EXECUTE under that cap.

**Member-to-member delegation.** Role v2.0 §5.6 + Amendment 1 PR-8.2: `system/role:delegate` is opt-in per role (delegate-ability MUST include the explicit `delegate` grant in the role's grants list). The delegate-cap is rooted at the delegator's runtime peer (member-to-member), not at the role handler. Identity-coherence-wise, this means delegation chains are 2 deep (delegation cap → role-derived root cap), and the delegator's local peer→controller cap is the runtime authority. SDK helpers for `:delegate` SHOULD surface the depth-2 chain expectation (per VALIDATION-MATRIX TV-RD-DELEGATE-CHAIN-DEPTH).

---

## 8. Bootstrap helper library contract

Per the architecture-team direction (Rev 6 banner): SDK implementations SHOULD provide a shared helper library that orchestrates the bootstrap, pairing, custody, rotation, and recovery flows. The contract is **normative on signatures and semantics, advisory on language idioms**. Cross-impl convergence enables identity-bundle interop (cross-impl tools operating on each other's bundles) and reduces application-developer cost.

### 8.0 Computation/I/O separation (normative for portability)

The bootstrap helper library MUST be structured as two layers, even when a particular implementation chooses to fuse them at the public API:

1. **Pure computation layer.** Generates keypairs, constructs entities (attestations, peer-config, quorum, identity-cert chain), assembles signatures, produces an in-memory `IdentityBundle`. No platform I/O. No filesystem, no network, no IndexedDB. Conformant on any platform with cryptographic primitives + entropy + the entity-core types.
2. **Persistence layer.** Platform-specific. Writes the bundle to a backing store (filesystem on native/desktop; IndexedDB or browser storage on WASM; potentially network-attached storage in cluster deployments). Conforms to the §8.4 on-disk layout when persisting to a filesystem; uses an equivalent structural mapping for non-filesystem backends.

**Why normative.** Browser WASM is a real deployment target for this system. A bootstrap helper that interleaves keypair generation with `os.WriteFile`-style I/O is unportable. Implementations that fuse the two layers (e.g., a single function that generates AND persists) MAY do so for the public API; they MUST keep the pure layer accessible (e.g., via a separate function returning the in-memory bundle, or an option flag suppressing persistence) so WASM and other I/O-constrained environments can use the helper without forking.

**Conformance:**
- The pure computation layer is MUST.
- The persistence layer is SHOULD on impls targeting platforms with persistent storage (native, Tauri, server). MAY for ephemeral environments.
- Public API surface is implementation-defined; the layered structure underneath is not.

**Reference deployment shapes:**
- Native/CLI/Tauri: pure layer + filesystem persistence per §8.4.
- Browser WASM: pure layer + IndexedDB or LocalStorage persistence (custodial-side); identity bundles MAY be ephemeral if the application's design uses session-lived peers.
- Test/headless: pure layer only; bundle held in memory; no persistence.

### 8.1 Helper signature catalog

Helpers split into four functional groups:

```
; --- Bootstrap and pairing ---
BootstrapNewIdentity(opts: BootstrapOpts) → IdentityBundle
  ; Generates: K-of-N quorum keypairs (transient — see §8.2 custody),
  ;            controller keypair (or identifier+controller in 4-key opt-in),
  ;            agent keypair for the runtime peer this is invoked on,
  ;            identity-cert attestations binding agent → controller → quorum,
  ;            quorum-publish attestation seeded with controller as published_handle.
  ; opts.separate_identifier: true → 4-key shape (identifier as published handle;
  ;                                  controller becomes internal management key).
  ; opts.initial_agent_count: N (default 1) → multi-agent at bootstrap.
  ; opts.controller_count: N (default 1) → multi-controller (sub-controller chains
  ;                                        attested by top-level controller).
  ; Output: IdentityBundle (on-disk layout per §8.4).

BootstrapFromExistingKeypair(opts, existing_keypair) → IdentityBundle
  ; V7-only → identity-aware migration. Reuses existing peer keypair as the agent
  ; for this runtime peer; generates fresh quorum + controller; attests new chain.

PairNewDevice(existing_bundle: IdentityBundle, new_device: NewDeviceParams,
              ceremony: PairingCeremony) → AgentCertAttestation
  ; Pairs a new device's agent keypair into an existing identity. Drives the
  ; PairingCeremony adapter (§8.5) for the ceremony UX. On approval, creates an
  ; identity-cert (function=agent) attestation signed by an existing agent under
  ; the controller cap, and persists the result for sync to the new device.

; --- Quorum custody (transient-bootstrap → steady-state) ---
ExportQuorumConstituent(quorum_ref: QuorumRef, member_id, destination: ExportDestination)
                                          → ExportRecord
  ; Exports a constituent's private key to the chosen destination
  ; (file path, QR display, NFC, hardware token, paper print, remote-wipe handoff).
  ; Removes the private key from the bundle's local storage; preserves the
  ; constituent's public key (still needed for K-of-N verification).

GetQuorumDistributionStatus(quorum_ref: QuorumRef) → DistributionStatus
  ; Returns per-constituent state and a `safe: bool` indicating whether at least
  ; (N - K + 1) constituents have been exported.

; --- Routine rotation ---
RotateAgent(quorum_ref, old_agent_ref, new_agent_keypair) → IdentityCertAttestation
  ; Supersedes the old agent's identity-cert with a new one (substrate :supersede
  ; via REBIND_KINDS path; the agent's attesting may stay the same or change).

RetireAgent(quorum_ref, agent_ref) → IdentityRetirementAttestation
  ; Creates an identity-retirement attestation; cap cleanup cascade per PI-13.

RotateController(quorum_ref, new_controller_keypair) → IdentityCertAttestation
  ; Supersedes the controller's identity-cert; K-of-N quorum signs the new cert.
  ; Multi-step ceremony if quorum constituents are in separate custody — see §8.6.

; --- Recovery (K-of-N ceremony; multi-step, possibly multi-session) ---
BeginRecovery(quorum_ref, kind, proposal) → RecoveryRequest
  ; kind ∈ {"rotate_controller", "rotate_identifier", "rotate_quorum"}.
  ; Constructs a recovery-request entity (proposed new keypair or new quorum
  ; membership) and persists to local store so the ceremony can resume across
  ; sessions.

AddRecoverySignature(req: RecoveryRequest, constituent_id, signature) → RecoveryRequest
  ; Idempotent. Signature obtained however the deployment retrieves cold keys.

IsRecoveryComplete(req: RecoveryRequest) → bool

FinalizeRecovery(req: RecoveryRequest)
              → IdentityCertAttestation | IdentityRotationRecoveryAttestation
                | QuorumUpdateAttestation
  ; Constructs the rotation entity per `kind` (using identity-rotation-recovery
  ; for controller/identifier rotation paths), writes to peer's tree, triggers
  ; propagation via standard sync.

AbandonRecovery(req: RecoveryRequest)

; --- Cross-extension lifecycle ---
RotationReissueOutstandingGrants(rotated_peer, new_authority, filter?)
              → [ReissuedCap]
  ; Iterates system/capability/grants/* under the rotated peer; filters per
  ; deployment policy; re-issues each cap under new_authority; emits each new
  ; cap for delivery via the consuming extension's normal flow.

RevokePeer(runtime_peer, scope) → revocations
  ; scope ∈ {"internal", "public", "per-relationship", "all"}.
  ; Per-mode propagation per EXTENSION-IDENTITY §5.2 sync surface.
```

### 8.2 Quorum custody (transient-bootstrap → steady-state)

Bootstrap necessarily generates K-of-N quorum constituent keypairs in one location — you can't K-of-N-sign without all K. Per `core-protocol-domain/guides/GUIDE-IDENTITY.md` §3.3 and §13.5, the design intent is that constituent private keys are immediately distributed across separate custody (paper, second device, trusted holder, hardware token) and removed from the original location. Leaving them colocated with the runtime peer is the catastrophic-loss surface.

Helpers make the two-phase pattern explicit:

```
ExportDestination := File(path) | QRDisplay | NFC | HardwareToken | PaperPrint | RemoteWipe
ExportRecord     := { member_id, exported_at, destination_kind, removed_local: bool }

DistributionStatus := {
  threshold:    uint                    ; K
  total:        uint                    ; N
  constituents: [{ member_id, status: "local" | "exported" | "transferred" }]
  safe:         bool                    ; true iff at least (N-K+1) exported
}
```

The application's bootstrap UI uses `GetQuorumDistributionStatus` to nag the operator until `safe == true`. The helper provides primitives and bookkeeping; the UX is application territory.

### 8.3 Recovery (K-of-N ceremony)

Recovery scenarios — controller compromise, identifier compromise, quorum membership change — share the same shape: construct a recovery-request entity, gather K constituent signatures from separately-stored keys, finalize. Signature gathering is **inherently asynchronous and ceremony-driven** (someone has to physically retrieve cold keys and sign), often spanning multiple sessions across days. A single-call helper would elide this complexity; the multi-step API surfaces it.

```
RecoveryRequest := {
  request_entity_hash:   hash       ; the entity constituents sign over
  target_quorum_ref:     QuorumRef
  kind:                  "rotate_controller" | "rotate_identifier" | "rotate_quorum"
  proposal:              <kind-specific proposal>
  threshold:             uint
  eligible_signers:      [PeerID]
  collected_signatures:  [Signature]
  created_at:            timestamp
  expires_at:            timestamp?
}
```

**In-session adapter (optional convenience).** For deployments where the recovery ceremony fits in one session (operator has all cold keys at hand), a `SignatureGatherer` adapter mirroring `PairingCeremony` MAY wrap the multi-step API:

```
SignatureGatherer := interface {
  RequestSignature(constituent_id, request_entity_hash) → Signature
}
```

The adapter calls `BeginRecovery`, drives `RequestSignature` until threshold is met (each call may prompt the user, scan a QR, etc.), and calls `FinalizeRecovery`. Implementations MAY provide both APIs — multi-step for asynchronous ceremonies, adapter for in-session convenience.

### 8.4 On-disk identity bundle layout (SHOULD-tier)

Per the layout principle in §15 of `SDK-OPERATIONS.md` (promoted in Step 4 matching pass): a freshly-bootstrapped identity has more on-disk material than a properly-set-up one. Quorum constituent private keys are present transiently during bootstrap and are expected to be exported to separate custody immediately afterward.

```
~/.entity/
├── identities/
│   ├── {name}{,.json,.pub}                    ; LEGACY (V7-only) — flat keypair files. KEEP indefinitely.
│   └── {name}/                                ; IDENTITY-AWARE — directory bundle.
│       ├── identity.toml                      ; metadata: name, identifier_id (4-key only),
│       │                                      ;   controller_id, quorum_id, schema_version
│       ├── quorum/
│       │   └── {member_id}/
│       │       ├── public_key                 ; ALWAYS PRESENT — for K-of-N verification
│       │       ├── keypair                    ; PRESENT during bootstrap; exported & removed afterward
│       │       └── export_status              ; "local" | "exported_to_<destination>" | "transferred_<date>"
│       ├── controllers/
│       │   └── {controller_id}/keypair        ; Controller keypair(s); top-level + sub-controllers
│       ├── agents/
│       │   └── {agent_id}/keypair             ; Agent keypair for this runtime peer
│       ├── identifier-keypair                 ; 4-key only — identifier (the published handle)
│       └── bootstrap-log                      ; Provisioning ceremony record (audit trail; NOT load-bearing)
├── peers/
│   └── {name}/
│       ├── keypair                            ; runtime peer keypair (V7-only and identity-aware both)
│       ├── config.toml                        ; startup: listen_addr, storage_backend, extensions
│       ├── grants.toml                        ; LEGACY — connection-time grants. KEEP for V7-only mode.
│       └── identity.toml                      ; OPTIONAL — present iff peer is identity-aware.
│                                              ;   { identity_name, trusts_quorum, agent_grants, identifiers }
```

**Three modes:**
1. **V7-only mode.** Flat `identities/{name}/{public_key,private_key}` and flat `peers/{name}/keypair` plus `config.toml` + `grants.toml`. No identity extension. Absence of `peers/{name}/identity.toml` signals V7-only.
2. **Identity-aware single-identifier mode.** Identity bundle directory + `peers/{name}/identity.toml` referencing the bundle. Default for identity-extension users.
3. **Multi-identity host mode.** Multiple `peers/{name}/` directories on the same host, each with its own keypair and `identity.toml` referencing a different identity bundle. Per `EXTENSION-IDENTITY.md` §3.8 — peer-configs MUST NOT share state across identities; structural separation enforces this.

**Conformance levels:**
- Layout structure: SHOULD for identity-aware mode (cross-impl interop benefits).
- Path naming: SHOULD (cross-impl tools should be able to operate on each other's bundles).
- **Manifest filename: `identity.toml` SHOULD-tier** (cross-impl bundle tooling expects this name). Implementations that pre-date this convention with a different filename (e.g., `bundle.json`) MAY keep their existing name during a transition window but SHOULD migrate to `identity.toml` (or accept both for read) so cross-impl tools have a single canonical lookup. The filename convention is at the SDK-side; underlying serialization format inside the file is implementation-defined per the next bullet.
- File formats inside the bundle: implementation-defined as long as cross-impl tools can read at least the public components (public keys, peer IDs, attestation hashes). The manifest itself SHOULD use TOML when named `identity.toml` (filename suggests format) but JSON-content-with-`.toml`-extension is non-conformant; implementations using JSON should use a different filename (e.g., `bundle.json`) and accept the cross-impl-tool-cost trade-off.
- Override via env var or CLI flag: MUST be supported (operational deployments need this).

**Migration path.** A V7-only peer becomes identity-aware via `BootstrapFromExistingKeypair` (§8.1): bundle is created in `identities/{name}/`, `peers/{name}/identity.toml` is added, the peer's keypair is unchanged (peer_id stable). Caller-side tools see the existing peer with new identity machinery. No retroactive migration of contacts.

**`identity.toml` minimal core schema (SHOULD-tier for cross-impl interop).** Identity-side: `name`, `identifier_id` (4-key only), `controller_id`, `quorum_id`, `schema_version`. Peer-side: `identity_name`, `trusts_quorum`, `agent_grants`, `identifiers`. Implementations MAY extend with their own fields. Same pattern as `Cargo.toml` — small standardized core, freely extensible.

### 8.4a Canonical wire shape for cross-impl bundle export/import (SHOULD-tier; MUST for impls claiming cross-impl bundle conformance)

The on-disk layout in §8.4 specifies how a single implementation stores identity material locally. Cross-impl bundle interop — where one implementation exports a bundle that a different implementation imports — requires a portable wire shape that is independent of any one impl's local-storage decisions. The canonical wire shape is **entity-shape**:

```
system/identity/bundle/v1 := {
  fields: {
    schema_version:        {type_ref: "primitive/uint"}   ; = 1 for this revision
    caller_keypair_public: {type_ref: "system/multikey"}  ; Ed25519 public anchor of the
                                                           ;   identity (the controller's identifying key)
    entities:              {type_ref: "system/envelope"}   ; Closure of identity-coherence entities
  }
}
```

The `entities` envelope carries (at minimum) the following materialized entities + their transitive closure:
- `system/identity/peer-config` (the current peer-config entity for the identity)
- `system/identity/identity-binding` (peer ↔ identity binding)
- The current `system/identity/identity-cert` chain (controller chain)
- The current `system/quorum/...` quorum entity referenced by the controller chain
- The attestation history covering the live controller chain (so verification can re-walk it)
- Public keys of quorum members (for K-of-N verification)

**Restore semantics (canonical).** Import writes each entity into the local content store, then binds at the canonical identity paths (`system/identity/...`, `system/quorum/{trusts_quorum}/...`). No bootstrap ceremony replay. The Ed25519 hash of `caller_keypair_public` MUST match the controller's identifying key in the included entities; mismatch → `bundle_keypair_mismatch/400`.

**Cross-impl conformance (SHOULD-tier).** Implementations claiming "cross-impl bundle interop" MUST support both `ExportEntityBundle` and `RestoreFromBundle` for this shape. Implementations MAY additionally support local-only shapes (e.g., keypair+ceremony for bootstrap-from-fresh-materials use cases) — those local shapes are not cross-impl portable and SHOULD use distinct operation names (e.g., `BootstrapFromMaterials` vs. `RestoreFromBundle`).

**Operation-naming convention (normative).** `BootstrapFromMaterials(keypairs, metadata)` constructs a fresh identity from raw inputs by minting all entities (single-signer; matches Go's existing `BootstrapIdentity` and Rust's Phase-1 `bootstrap`). `RestoreFromBundle(entity_bundle)` rehydrates a previously-exported identity by writing entities and binding paths; no minting. The two operations have distinct semantics; conflating them under one name is non-conformant.

**Rationale.** Multi-signer custody (EXTENSION-QUORUM §3.2/§3.3 K-of-N) cannot be represented in the keypair+ceremony shape — members own their own keypairs and the bootstrapper is not a custodian of them. WASM contexts cannot assume a filesystem; entity-shape is filesystem-agnostic since entities go through the content store API on every platform. The entity-shape contract is strictly more expressive than the keypair shape (single-signer still works; multi-signer additionally works). Per the identity-bundle canonical-wire-shape proposal.

**No private key material — security non-goal (normative).** The `system/identity/bundle/v1` wire shape MUST NOT carry private key material in any field, encoding, or transitive entity. Private keys move through the per-peer keystore via a separate channel (filesystem PEM file, OPFS, app config dir, hardware token, paper backup, etc.); bundles carry only the materialized identity-coherence entities (peer-config, identity-binding, identity-cert chain, quorum entity, attestation closure, public-key anchors). The receiver's pre-condition for `RestoreFromBundle` is a **public-key match** between the local keystore's keypair and the public_key carried in `bundle.identity_entity` — not a PEM round-trip, not a signature with bundle-supplied material. Implementations that emit private bytes into the bundle are non-conformant; any such bytes already shipped MUST be treated as compromised key material (rotate). This non-goal is load-bearing because bundles are portable artifacts (disk, cloud sync, transport) while private keys are device-local secrets. Conflating the two surfaces is the defect class this clarification pre-empts. Driven by the Rust identity-bundle private-key-defect review (Rust v1 carried a `keypair_pem` field; rescinded in v2 same-day with a public-key-only receiver precondition).

### 8.5 Adapter interfaces for application UX

```
PairingCeremony := interface {
  GenerateOffer(new_device_keypair) → Offer    ; e.g., QR code payload
  PresentOffer(offer)                          ; Show to user (QR, code, NFC etc.)
  AwaitApproval() → ApprovalDecision           ; User approves on existing device
}
```

Application teams implement the ceremony per their UX (mobile QR scan, desktop dialog, CLI prompt). The helper library doesn't dictate UX.

### 8.6 In-process L0 vs dispatched-EXECUTE boundary

The helper library presents a uniform surface, but underneath helpers split into two classes:

- **Bootstrap helpers (in-process L0):** `BootstrapNewIdentity`, `BootstrapFromExistingKeypair`, and the very first `PairNewDevice` call on a new device run via L0 direct-store because the local peer→controller cap doesn't exist yet.
- **Post-bootstrap helpers (compose dispatched EXECUTE):** `RotateAgent`, `RetireAgent`, `RotateController`, `RotationReissueOutstandingGrants`, `RevokePeer`, `ExportQuorumConstituent`, the recovery API. They run by composing dispatched `EXECUTE` calls under controller authority. Recovery's `FinalizeRecovery` ultimately writes a quorum-signed rotation entity, but the per-step helpers (adding signatures, persisting the in-progress request) run under controller authority since they touch the local tree's recovery-request state.

Implementations MAY surface the boundary explicitly (separate sub-packages, distinct method namespaces) or keep it transparent. The L0 path MUST be unavailable to runtime code paths post-bootstrap (per §4.1).

**Pure helper, not handler.** Each helper composes existing primitives (keypair generation, entity construction, signing, store writes or dispatched EXECUTEs). No new protocol mechanism. The output of each helper is one or more materialized entities ready to flow through normal sync.

---

## 9. Rotation lifecycle hooks

Per the rotation patterns in `EXTENSION-IDENTITY.md` §6 + `core-protocol-domain/guides/GUIDE-IDENTITY.md`, the SDK surfaces the following events to applications:

| Event | When it fires | What apps SHOULD do |
|---|---|---|
| **Agent rotation (supersede)** | `RotateAgent` finalizes | Runtime peers' caps unaffected (granter is runtime peer, not agent). No SDK action required at rotation time. Application caps issued downstream remain valid. |
| **Agent retirement** | `RetireAgent` finalizes | The local peer→controller cap for the retired agent is revoked. Other live agents' caps remain. Application code running under that agent fails with 401; SDK SHOULD surface a structured `agent_retired` signal so apps can re-acquire credentials. |
| **Runtime peer retirement (`revoke_peer(scope: 'all')`)** | `RevokePeer(scope: 'all')` finalizes | Caps that runtime peer issued downstream die on next `is_revoked` chain-walk. Per the `RotationReissueOutstandingGrants` helper, the rotating peer SHOULD invoke it before retiring. SDK exposes the helper; application code drives the deployment policy (short-TTL / SDK-tracked / accept-loss). |
| **Controller rotation (handoff)** | `RotateController` finalizes via dual-sig handoff | Contacts process the rotation via the attestation channel (per-mode propagation). Cap chains from the new controller are issued post-rotation. SDK SHOULD surface the rotation event so apps watching identity changes can update UI. |
| **Controller rotation (recovery)** | K-of-N recovery ceremony finalizes | Same as handoff from contacts' perspective. Handle-cache update post-§9.4-validate per `EXTENSION-IDENTITY.md` §6.3. |

**`RotationReissueOutstandingGrants` deployment patterns:**
- **Short-TTL (recommended default):** caps with short expiration; rotation is a non-event because caps will refresh anyway.
- **SDK-tracked:** SDK tracks outstanding caps in a registry; on rotation, helper iterates and re-issues.
- **Accept-loss:** deployment accepts that some caps may fail post-rotation and chooses re-acquisition over re-issuance.

Documented per ARC-FIXES IA26 §3 deployment-pattern guidance (originating direction).

---

## 10. Group surface (placeholder)

EXTENSION-GROUP v1.5 is queued for landing after substrate cross-impl stabilizes (per `proposals/PROPOSAL-EXTENSION-GROUP-V1.5.md`). The group SDK surface (`form`, `add_member`, `remove_member`, `change_role`, `add_subgroup`, `remove_subgroup`, `dissolve`, `merge`, `split`, `attest_acting_on_behalf`, `revoke_acting_on_behalf`) lives here when v1.5 lands. The surface reuses identity's primitives — a group is a type of identity reusing the substrate primitives + agent-style runtime peers under the group's namespace.

Until v1.5 lands, applications use `EXTENSION-GROUP` v1.3 directly via `peer.execute("system/group", ...)` without dedicated SDK wrappers. This section will fill in.

---

## 11. Open questions / deferred

1. **Helper library packaging.** Whether `BootstrapNewIdentity`, `PairNewDevice`, etc. live in a single language-idiomatic module per impl or split (bootstrap / pairing / custody / rotation / recovery as separate sub-packages). Implementation-defined; convergence on the contract (signatures + semantics) is the goal, not on packaging.
2. **`SignatureGatherer` adapter optionality.** Listed as MAY in §8.3. As impls land, validate whether it earns its own surface or whether the multi-step API is sufficient.
3. **Cross-impl bundle test vectors against §8.4a wire shape.** Land in the next cross-impl conformance cycle after Go's `ExportEntityBundle`/`RestoreFromBundle` adapter lands. Each impl produces a bundle on machine A, exchanges with machine B (different impl), imports successfully. Verification artifacts: public-key recovery, attestation walk, peer-config binding. Replaces the original §8.4 filesystem-layout test-vector framing — wire shape is now the cross-impl conformance surface; filesystem layout is per-impl storage.
4. **`identity.toml` schema validation.** Whether implementations validate the standardized minimal core (name, identifier_id, controller_id, quorum_id, schema_version) on read, or accept-anything-with-extension-fields. Recommendation: validate on read (catch corrupted bundles early).
5. **Group v1.5 surface.** Filled in when group v1.5 lands.
6. **Whether the SDK identity-infra spec eventually elevates to a system-module classification.** Per architecture-team direction (Rev 6 banner): for now this is library-development guidance for SDK implementers. Future re-classification deferred.

---

## 12. Document history

- **v0.2 — egui-Rust review absorption:** Three additions per the egui-Rust team's SDK-identity-reconciliation review (a four-source reconciliation: architecture spec ↔ Go ↔ Rust SDK ↔ base SDK contract). **§5/§6 Request/result entity shapes pointer** (their §2.3 / §4.1) — clarifying note that wire schema is normatively defined at the protocol-side specs (EXTENSION-ATTESTATION/QUORUM/IDENTITY §6; EXTENSION-ROLE §4.2; V7 §3.7 type-registration convention); SDKs deserialize against those schemas. Cross-impl wire compatibility derives from the protocol-side schemas, not the SDK surface. **§8.0 Computation/I/O separation (normative for portability)** (their §2.5) — bootstrap helper library MUST structure as two layers: pure computation (in-memory bundle, no I/O, WASM-compatible) + platform-specific persistence (filesystem/IndexedDB/etc.). Public API may fuse them; the layered structure underneath is mandatory. Browser WASM is a real deployment target; the previous helper signatures implied filesystem coupling that doesn't translate. **§8.4 Manifest filename `identity.toml` SHOULD-tier** (their §2.6 / §4.2) — pin the manifest filename for cross-impl bundle tools; Go's `bundle.json` is pre-spec and SHOULD migrate to `identity.toml` (or accept both for read) during a transition window. Closes the cross-impl interop ambiguity between Go's empirical name and the spec's sketch. No mechanism change for any of the three; clarifications + portability framing only.

- **v0.1 draft:** Initial Rev 6 draft per the SDK identity-infrastructure-alignment proposal (Rev 6 banner). Bottom-up from Go's `ext/role/sdk/` + `ext/identity/sdk/` empirical baseline; top-down from v3.5 substrate vocabulary (controller / agent / identifier; identity-cert kinds; properties.mode discipline; substrate ops). Sections: scope; reading order; identity stack at a glance; SDK access levels (L0 bootstrap exemption + L1 dispatched); substrate ops surface (attestation + quorum); identity convention-layer surface; identity-coherence aspects of role; bootstrap helper library contract (signature catalog + custody + recovery + bundle layout + adapters + L0/L1 boundary); rotation lifecycle hooks; group placeholder; open questions. Companion: `sdk-domain/guides/GUIDE-IDENTITY-SDK.md` (drafted Step 3); matching pass on `SDK-OPERATIONS.md` / `SDK-EXTENSION-OPERATIONS.md` / `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` (Step 4).
