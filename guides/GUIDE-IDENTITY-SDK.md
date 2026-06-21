# Guide: Identity SDK

**Status**: Active
**Source:** `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` v0.1.

---

## What this guide covers

How to use the identity-stack SDK surface from application code. This is the practical companion to the SDK contract spec.

**Scope:**
- Decision tree: when do you need identity at all?
- Provisioning a new identity (3-key default; 4-key advanced opt-in).
- Adding a runtime peer (device pairing).
- Connecting to a contact.
- Operating as an agent post-bootstrap.
- Rotation (when, how).
- Recovery (the K-of-N ceremony).
- Pitfalls.

**Out of scope:**
- The protocol behavior of identity ops — see `EXTENSION-IDENTITY.md`, `EXTENSION-ATTESTATION.md`, `EXTENSION-QUORUM.md`.
- Application UX details (how a QR pairing dialog should look; how cold-key custody is presented). Those are application-team territory; this guide says only what the SDK surfaces.
- The full SDK contract details — see `SDK-IDENTITY-INFRASTRUCTURE.md` for the complete operation semantics, helper library signatures, and on-disk bundle layout.

---

## 1. Do you need identity?

```
Do you need identity?

├── Single user, single device, acceptable to lose if device dies?
│   → No. Use V7 + role. One peer keypair, role-based grants. Done.
│
├── Single user, multiple devices OR want recovery?
│   → Yes, three-key (default). Quorum + controller + agent peers.
│
└── Want controller rotation independent of contact-side handle? (advanced)
    → Yes, four-key. Add identifier (the published handle).
       Controller rotates often; identifier rarely.
```

**The default is no identity.** A V7-only peer is fully conformant; it has a self-rooted keypair and capability chains rooted at that peer. For a single-user single-device deployment with no recovery requirement, V7 alone is the answer. Don't add identity machinery you don't need.

**The default identity shape is three-key.** When you do need identity, the structure is:
- A **K-of-N quorum** — the recovery anchor. Constituents in separate custody.
- A **controller** — the persona managing the peer; published as the contact-side handle.
- One or more **agents** — per-device daemons attested by the controller.

**The four-key shape is opt-in.** When you want the controller key to rotate without disturbing contact-side handle caches (e.g., aggressive controller rotation cadence on hot devices, while the published handle stays stable), separate the identifier (published handle) from the controller (internal management key). Set `BootstrapOpts.separate_identifier: true` at provisioning.

**Role and identity are orthogonal.** A peer may have role assignments without identity infrastructure (V7 + role minimum) or with the full identity stack. Role = what a peer can do; identity = who they are.

---

## 2. Provisioning

### 2.1 Three-key default (recommended)

```
bundle = peer.identity.bootstrap(BootstrapOpts {
  name:                "alice",
  quorum: { signers: 3, threshold: 2 },     // 2-of-3 K-of-N
  initial_agent_count: 1,                   // one agent (this device)
})
// bundle is the on-disk identity bundle (see §8.4 of SDK-IDENTITY-INFRASTRUCTURE.md)
```

After this returns, the bundle contains:
- 3 quorum constituent keypairs (transient — see §2.4 below).
- 1 controller keypair.
- 1 agent keypair (this runtime peer).
- Identity-cert attestations binding agent → controller → quorum.
- A `quorum-publish` attestation seeded with controller as published handle.

The peer is bound to the new identity via `peers/{name}/identity.toml`.

### 2.2 Four-key advanced (opt-in)

```
bundle = peer.identity.bootstrap(BootstrapOpts {
  name:                  "alice",
  quorum:                { signers: 3, threshold: 2 },
  separate_identifier:   true,              // 4-key opt-in
  initial_agent_count:   1,
})
```

In addition to the 3-key contents, the bundle now has an `identifier-keypair` — the contact-side published handle, separate from the rotating controller key.

### 2.3 Migrating an existing V7-only peer

```
bundle = peer.identity.bootstrap_from_existing(opts, existing_keypair)
```

Reuses the existing peer keypair as the agent for this runtime peer; generates fresh quorum + controller; attests new chain. The peer's `peer_id` is unchanged (stable across migration).

### 2.4 Quorum custody — distribute, don't hoard

After bootstrap, the K-of-N constituent **private keys** are colocated with the runtime peer. This is the catastrophic-loss surface. Distribute them immediately:

```
record = peer.identity.export_quorum_constituent(
  bundle.quorum_ref,
  member_id: 0,
  destination: ExportDestination::QRDisplay,
)
// QR rendered on screen; user scans with second device or prints to paper.
// The constituent's private key is removed from the bundle's local storage.
// Public key remains (needed for K-of-N verification of remote signatures).
```

Repeat for at least `(N - K + 1)` constituents. Use `get_quorum_distribution_status(bundle.quorum_ref).safe` to check progress; UI nags until safe.

**Constituent destinations:** file path (encrypted), QR display (paper print or scan to second device), NFC handoff, hardware token (YubiKey, etc.), trusted-holder transfer (record the transfer; private key wiped from this device).

### 2.5 Result

After §2.1 + §2.4, you have a bootstrapped identity with safe distribution. The peer can now operate as the agent for this identity; its caps are attested via the controller chain; recovery is possible if the controller key (or any agent) is lost — provided K constituents remain accessible.

---

## 3. Adding a runtime peer (device pairing)

You have an identity on device A; you want to add device B as a second runtime peer of the same identity.

```
// On the new device (B):
new_keypair = generate_keypair()
offer       = ceremony.generate_offer(new_keypair)
ceremony.present_offer(offer)            // shows QR, NFC, etc.

// On the existing device (A):
existing_bundle = peer.identity.load_bundle("alice")
agent_cert = peer.identity.pair_new_device(
  existing_bundle,
  new_device:  NewDeviceParams { keypair_pubkey: new_keypair.pub, label: "phone" },
  ceremony:    ceremony,
)
// On approval, an identity-cert (function=agent) attestation is created,
// signed by an existing agent under the controller cap, and persisted.
// New device pulls it via standard sync; runs identity.configure with the
// new agent_cert in its bindings; receives local-peer→controller cap.
```

The `PairingCeremony` adapter is application-defined. Common implementations:
- **Mobile QR scan**: device A renders QR; device B scans.
- **Desktop dialog**: device A prompts the user to approve; device B awaits.
- **CLI prompt**: `entity pair --offer <offer-string>` on device A, `entity pair --approve <token>` on device B.

The SDK provides the multi-step protocol; application provides the UX.

---

## 4. Connecting to a contact

Once you have an identity, you connect to other identities. The first time you encounter a contact's controller key, your peer caches it (TOFU — trust-on-first-use):

```
contact_handle = registry.resolve("bob.example.com")
// → returns { controller_pubkey, bootstrap_endpoints, attestations }
// SDK caches the controller_pubkey under system/identity/relationships/{contact_id}/...
peer.connect(contact_handle.bootstrap_endpoints[0])
```

Subsequent connections verify against the cached controller. If the contact rotates their controller (handoff or recovery), they publish the rotation attestation; your peer processes it via `:process_attestation` and updates the cache.

**Cache-miss policy** is application-configurable: TOFU (default), explicit-confirmation, registry-only (no cache; verify every time). See `core-protocol-domain/guides/GUIDE-IDENTITY.md` §6 for trust model details.

---

## 5. Operating as an agent (post-bootstrap)

After `:configure` succeeds, the peer holds a **local peer→controller cap** at `system/capability/grants/identity/peer-to-controller/{controller_hex}`. This cap is the typical caller cap for identity ops, role ops, and any other dispatched EXECUTE the agent performs.

```
// Use the controller cap as caller cap for role assignment:
peer.role.assign(
  context: "admin",
  assignee: bob_peer_id,
  caller_cap: peer.identity.local_controller_cap(),
)
```

**What the agent CAN do:**
- Issue caps under the controller's authority.
- Create new identity-cert attestations (e.g., paring more devices).
- Publish/supersede/revoke attestations (per controller authority).
- Drive role assignments / re-derive / delegate.

**What the agent CANNOT do (without K-of-N quorum):**
- Rotate the controller (requires quorum signatures).
- Rotate the identifier (4-key only; requires quorum signatures).
- Change quorum membership (requires quorum signatures).

These require the recovery ceremony (§7).

---

## 6. Rotation

### 6.1 Agent rotation

When an agent's keypair needs to change (compromised; aging; replacement device):

```
new_agent_keypair = generate_keypair()
agent_cert = peer.identity.rotate_agent(
  bundle.quorum_ref,
  old_agent_ref: current_agent,
  new_agent: new_agent_keypair,
)
```

**Effect:** Substrate `:supersede` (via REBIND_KINDS path) supersedes the old agent's identity-cert with a new one. Runtime peers' caps issued by the old agent remain valid — granter is the runtime peer, not the agent — so application caps issued downstream are unaffected.

**SDK signal:** none required at rotation time. App caps continue to work.

### 6.2 Agent retirement

When an agent is removed (lost device; security event):

```
peer.identity.retire_agent(bundle.quorum_ref, agent_ref)
```

**Effect:** Creates an `identity-retirement` attestation. The local peer→controller cap for the retired agent is revoked via PI-13 cap cascade. Other live agents' caps remain.

**SDK signal:** `agent_retired` event fires on connected peers. Apps holding caps under the retired agent see 401 on next dispatch; SHOULD re-acquire credentials by reconnecting under a live agent.

### 6.3 Controller rotation (handoff path — both old and new keys available)

When you want to rotate the controller and you still control both the old and new keys (planned rotation, aggressive cadence in 4-key shape):

```
new_controller_keypair = generate_keypair()
peer.identity.rotate_controller(bundle.quorum_ref, new_controller_keypair)
```

**Effect:** Identity-cert supersede signed by both old and new controllers (dual-sig handoff). Contacts process the rotation via the attestation channel; their caches update on receipt.

**SDK signal:** `controller_rotated` event fires; apps watching identity changes may update UI.

### 6.4 `RotationReissueOutstandingGrants`

Per ARC-FIXES IA26 (originating direction). If your deployment has long-lived caps issued downstream that need to survive a rotation, the SDK exposes:

```
reissued = peer.identity.rotation_reissue_outstanding_grants(
  rotated_peer:  old_agent_or_controller,
  new_authority: new_cap_under_new_agent_or_controller,
  filter: { min_remaining_ttl: 24*hour },   // optional
)
```

The helper iterates outstanding caps, filters per deployment policy, re-issues each, and emits each new cap for delivery to the consuming peer via the consuming extension's normal flow.

**Three deployment patterns:**
- **Short-TTL (recommended default):** caps with short expiration; rotation is a non-event because caps refresh anyway. Don't use this helper.
- **SDK-tracked:** SDK tracks outstanding caps in a registry; on rotation, helper iterates and re-issues. Use this helper.
- **Accept-loss:** deployment accepts that some caps may fail post-rotation; users re-acquire. Don't use this helper.

---

## 7. Recovery

When you've lost the controller or identifier key — or want to change the quorum itself — you run the K-of-N ceremony.

The ceremony is **inherently asynchronous**. Someone has to physically retrieve cold keys (paper, hardware token, second device) and produce signatures. This may span multiple sessions across days.

### 7.1 Multi-step API (the canonical shape)

```
// Step 1 — begin recovery
req = peer.identity.begin_recovery(
  bundle.quorum_ref,
  kind: "rotate_controller",
  proposal: { new_controller_keypair: new_controller },
)
// req persists to local store; ceremony can resume across sessions.

// Step 2 — gather signatures (one per constituent, possibly across sessions)
for constituent in retrieve_cold_keys() {
  signature = constituent.sign(req.request_entity_hash)
  req = peer.identity.add_recovery_signature(req, constituent.id, signature)
  if peer.identity.is_recovery_complete(req) { break }
}

// Step 3 — finalize
rotation_attestation = peer.identity.finalize_recovery(req)
// Constructs the rotation entity (identity-rotation-recovery, etc.),
// writes to peer's tree, triggers propagation via standard sync.
```

### 7.2 In-session adapter (optional convenience)

If your deployment has all cold keys at hand at the moment of recovery:

```
adapter = SignatureGathererImpl { /* prompts user, scans QR, etc. */ }
rotation = peer.identity.recover_in_session(
  bundle.quorum_ref,
  kind: "rotate_controller",
  proposal: { new_controller_keypair: new_controller },
  gatherer: adapter,
)
```

The adapter wraps the multi-step API; the SDK calls `gatherer.request_signature(...)` until threshold is met.

### 7.3 Recovery scenarios

| Scenario | `kind` | What happens |
|---|---|---|
| Controller key lost / compromised | `rotate_controller` | New controller keypair attested by K-of-N quorum; `identity-rotation-recovery` attestation created. Contacts process and update handle cache. |
| Identifier key lost / compromised (4-key only) | `rotate_identifier` | New identifier keypair attested by K-of-N quorum. Identifier is the contact-side published handle; rotating it disrupts contact recognition until contacts process the rotation. |
| Quorum membership change | `rotate_quorum` | Quorum-update attestation signed by current K-of-N. Constituents added or removed; threshold may change. |

### 7.4 Cold-key custody patterns

- **Paper QR.** Print constituent's private key as QR (encrypted with passphrase); store in safe. Scan + decrypt at recovery time.
- **Hardware token.** Constituent's private key on a YubiKey or similar; physically present at recovery time.
- **Trusted holder transfer.** Constituent's private key entrusted to a person (lawyer, family member); they sign on your behalf.
- **NFC handoff.** Constituent's private key on a second device; tap to sign.

The SDK doesn't dictate; the application chooses based on threat model and convenience.

---

## 8. Pitfalls

Cross-reference `core-protocol-domain/guides/GUIDE-IDENTITY.md` §13 (the architecture-spec deep guide's pitfalls section) for detailed treatment. Highlights:

- **Don't leave quorum constituent private keys colocated with the runtime peer.** Bootstrap leaves them there transiently; distribute immediately. `get_quorum_distribution_status` is the gate.
- **Don't share `peer-config` across identities on a multi-identity host.** Per `EXTENSION-IDENTITY.md` §3.8 — peer-configs MUST NOT share state across identities. Use separate `peers/{name}/` directories.
- **Don't direct-`tree:put` to `system/identity/...` paths from application code.** Use the handler ops (`system/identity:create_attestation`, etc.). Direct tree:put bypasses validation (PI-11 valid-modes; supersede REBIND_KINDS discipline).
- **Don't expose L0 access to delegated/attenuated code.** Per `SDK-OPERATIONS.md` §2.7 — L0 is for the peer owner's bootstrap code only. Code running under role-derived caps must go through dispatched EXECUTE.
- **Don't forget recovery_signal events.** When `:publish_attestation` MOVE or `:revoke_attestation` cap cascade fails partially, the SDK emits a `recovery_signal` controller-event at `system/identity/events/...`. SHOULD surface these to the controller-equivalent operator interface so they can be cleared.
- **Don't assume contact handle keys never change.** Controllers can rotate; contacts process the rotation via attestation channel and update their cache. Build for it.

---

## 9. Reference: SDK helpers ↔ handler ops

| SDK helper | Underlying ops |
|---|---|
| `bootstrap()` | L0 direct-store: write quorum entity; write identity-cert attestations; bind paths; write peer-config. |
| `bootstrap_from_existing()` | Same as `bootstrap()`, reusing existing keypair as agent. |
| `pair_new_device()` | `system/identity:create_attestation` (function=agent) signed by existing agent under controller cap. |
| `export_quorum_constituent()` | Local file/handoff operation; updates `export_status` in bundle. No protocol op. |
| `rotate_agent()` | `system/identity:supersede_attestation` (REBIND_KINDS path → substrate `:create` with new attesting/attested + supersedes). |
| `retire_agent()` | `system/identity:create_attestation` (kind=identity-retirement) → cap cascade per PI-13. |
| `rotate_controller()` (handoff) | `system/identity:supersede_attestation` (controller-cert REBIND); dual-sig (old + new). |
| `revoke_peer()` | `system/identity:revoke_attestation` per scope; per-mode propagation per `EXTENSION-IDENTITY.md` §5.2 sync surface. |
| `begin_recovery()` | Local: construct recovery-request entity; persist to local store. |
| `add_recovery_signature()` | Local: append signature to recovery-request. |
| `finalize_recovery()` | `system/identity:create_attestation` (identity-rotation-recovery / identity-cert / quorum-update per `kind`). |
| `rotation_reissue_outstanding_grants()` | Iterates `system/capability/grants/*` under rotated peer; re-issues caps under new authority; delivery via consuming-extension flow. |
| `local_controller_cap()` | Read from `system/capability/grants/identity/peer-to-controller/{controller_hex}` (set by `:configure`). |

---

## 10. Document history

- **v0.1 draft:** Initial Rev 6 companion guide. Lead with "Do you need identity?" decision tree (per the original Rev 5 §S13 design, refreshed for v3.5 vocabulary). Sections: provisioning (3-key + 4-key + V7→identity-aware migration); quorum custody patterns; pairing; connecting to contacts; agent operation; rotation (agent / retirement / controller handoff / `RotationReissueOutstandingGrants`); recovery (multi-step API; in-session adapter; scenarios; cold-key patterns); pitfalls; helper ↔ handler op reference. Companion: `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` (the SDK contract spec).
