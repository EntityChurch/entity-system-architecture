# EXTENSION-ENCRYPTION

**Version**: 1.0
**Status**: Active

The original conceptual core (entity-level framing, three-mode split, don't-reuse-identity-key, encryption-as-defense-in-depth-with-capabilities) is preserved verbatim in §2 + §13; every cross-extension / wire / cap claim is the substrate-aligned form.

## §1 Scope and v1 framing

The user direction for v1: **storage encryption and basic relay-encrypted sending are MUST. Group and rotation are best-effort. Sessions / chat / multi-peer key-agreement are OUT.** This spec delivers an honest v1 around that priority order — under-promising on what we don't deliver (no PFS claim in peer mode, no group-PFS, no session ratchet) so impl teams build to the threat model we actually meet.

**In scope for v1.0:**
- `mode: "self"` — at-rest symmetric encryption with key derived from a local secret. The storage primary. PRIMARY v1.
- `mode: "peer"` — non-interactive single-shot hybrid encryption to one recipient. The "encrypted send through a relay" primitive. PRIMARY v1.
- Three algorithm-byte registries (`enc_key_type`, `aead_id`, `kdf_id`) following the v7.65 – v7.69 multikey-style varint discipline.
- Encryption-key certs published as `identity-cert` attestations with `properties.function = "encryption"`, per the v3.10 cert-chain substrate.
- Substrate-aligned rotation and revocation at three deployment tiers (§4.0): **Tier A** V7-only (encryption-owned `handoff` + `revocation` entity types; self-rooted V7 keypair as trust anchor); **Tier B** +ATTESTATION (supersedes chain + universal `revocation` kind); **Tier C** +IDENTITY (`identity-rotation-handoff` + controller-authorized revocation, multi-device). ENCRYPTION does **not** require IDENTITY; lower tiers are valid configurations.
- AEAD additional-data binding (normative), pinned nonce semantics per AEAD, sender authentication via attached signature (default).
- Defense-in-depth at-rest key storage (Tier 1 platform secure storage + Tier 2 passphrase-wrapped backup with pinned Argon2id parameters; Tier 3 Shamir is a reserved-slot future workstream item).
- Resource bounds inheriting V7 §4.10 floor.
- Composition with frame-level (V7 §4.5), relay, inbox, tree, content store, capability.
- 8 conformance vectors (see §16).

**Best-effort for v1 (ship if it lands cleanly; defer to v1.5 otherwise):**
- `mode: "group"` — static key-wrap for small groups (≤256 members default). No group-PFS, no MLS. The "share a thing with a fixed set of people" primitive.
- `enc_key_type = 0x04` hybrid X25519+ML-KEM-768 slot — allocated in v1 even if no impl ships it (so the discipline holds; first PQ impl has a target).

**In scope but not fully implemented this cycle (designed; theories of shape in §19):** these items are explicitly owned by this extension's design space — we have a position on shape; we did not have time to fully spec normative pseudocode this cycle. Marking them "deferred" would read as "we didn't think about it"; that is not the case. Each carries a "theory of shape" subsection in §19.

- **Sealed-sender / recipient-hiding (`mode: "peer-anonymous"`).** Slot reserved; cryptographic shape sketched.
- **Padding / size hiding.** Design direction identified (length-bucket padding under AAD); no normative spec this cycle.
- **Hybrid post-quantum peer mode (`enc_key_type = 0x04`, X25519 + ML-KEM-768).** Byte allocated; KEM combiner shape sketched; first conformant impl can write the vector.
- **Shamir N-of-K secret-sharing (Tier 3 backup).** Full design sketch (§19.4) — entity shape, distribution, reconstruction, refresh, custody model. **Designed; impl is W2-scheduled, not gating v1.0 ratification but the design is ours.** Companion work to vault/custodians per `WORKSTREAMS.md` W2.

**Sibling extension: `EXTENSION-ENCRYPTED-SESSION` (planned).** Session/streaming/chat encryption (Signal double ratchet, Noise XK, MLS) is a structurally distinct extension built on this extension's shared substrate. Architectural call + scope split in §20. The decision is "two sibling extensions sharing substrate," not "one extension covering both" — the boundary is **stateful interactive (sibling) vs stateless single-shot (this extension).** Naming it now so v1 impls know it's coming + how it composes.

**OUT entirely (different problem, not this extension's domain):**
- Searchable encryption / structured queries on encrypted content (separate research direction; tied to query extension, not encryption).
- Transport-level frame encryption (V7 §4.5, already exists).

The v1 cipher-suite floor is: `enc_key_type=0x01` (X25519) + `aead_id=0x01` (XChaCha20-Poly1305) + `kdf_id=0x01` (HKDF-SHA-256). Every conformant peer MUST support this suite for every mode it implements; other suites are negotiated per encryption-key cert.

---

## §2 The core insight (preserved verbatim from v1)

Encryption modeled at the entity level unifies two concerns:

1. **Encrypted communication** — end-to-end through relays; untrusted intermediaries can't read content.
2. **Encrypted storage** — at rest in the tree or content store; compromise of storage doesn't expose data.

Same entity, same model. An encrypted entity is secure wherever it lives — in memory, on disk, in transit, on a remote peer. The encryption is a property of the entity itself, not of the transport or storage layer.

**Why not just transport encryption?** Transport encryption (TLS-equivalent, V7 §4.5 frame-level) protects link-by-link only. Relay sees plaintext; storage sees plaintext; an attacker compromising the relay or the storage layer reads everything. Entity-level encryption survives every intermediary because the entity carries its own ciphertext.

**Why not reuse the identity key?** The identity key (Ed25519, `key_type=0x01`) is for **signing** — proving who you are. Using it for encryption is bad practice:
- **Compromise blast-radius.** If the identity key is compromised, the attacker reads ALL encrypted content ever produced AND impersonates the peer. Separate keys limit blast radius.
- **No forward secrecy on long-lived signing keys.** A long-lived encryption key means all past ciphertext is retroactively readable on compromise.

Standard practice: signing keys and encryption keys are separate. PGP has master + subkeys. Signal has identity + prekeys. Same pattern here, expressed via the v3.10 identity cert-chain substrate.

**Normative key-separation requirement (R6, MUST).** The encryption keypair (`enc_key_type`, e.g. X25519) **MUST** be generated independently of the identity signing keypair (Ed25519, `key_type=0x01`). An impl **MUST NOT** use the identity key material as the encryption key, and **MUST NOT** derive the encryption key from the identity key by any deterministic transform — including the birational Ed25519→X25519 map (the age / libsodium `crypto_sign_ed25519_*_to_curve25519` pattern). Rationale: any such derivation re-couples the two keys, so compromise of the identity key compromises all encrypted content — defeating the blast-radius containment this section and §9.4 promise. This is the spec's stated threat model made enforceable. Gated by `ENC-KEY-SEPARATION-1` (§16) — it cannot be observed from the pinned-seed KATs (which use independent seeds by construction), so it is a BLOCK-1 validation-round gate against real key generation.

**Encryption + capabilities are complementary; defense in depth.** Capabilities control access (who can fetch a path; who can dispatch an op). Encryption controls comprehension (who can read plaintext). A peer with capability over a path can fetch the encrypted entity but cannot read its content without the decryption key. The cap-axis is retired in the release shape ("public = standing public grant filtered at publish; privacy = encryption" — `WORKSTREAMS.md` W2); this extension owns the privacy primitive.

---

## §3 Algorithm registries

Per the v7.65 – v7.69 multikey-style discipline: every algorithm choice is a varint-encoded byte under a documented registry. Future-compatible by construction. Three registries are owned by this extension.

### §3.1 `enc_key_type` — encryption keypair algorithm

| Byte | Algorithm | Status | Notes |
|---|---|---|---|
| `0x00` | reserved | reserved | Mirrors v7.66 `key_type=0x00` carve-out |
| `0x01` | X25519 | **PRODUCTION** | v1 floor. 32-byte pubkey. Curve25519 ECDH per RFC 7748 |
| `0x02` | X448 | reserved (next validate slot) | 56-byte pubkey. Pairs with v7.67's Ed448 / SHA-384 validate slot |
| `0x03` | ML-KEM-768 | reserved (PQ KEM) | 1184-byte pubkey, 1088-byte ciphertext. NIST FIPS 203 |
| `0x04` | X25519 + ML-KEM-768 hybrid | reserved (recommended PQ upgrade path) | Concatenated key material; both shared secrets HKDF'd together. IETF draft `hpke-pq` style |
| `0x05` | ML-KEM-512 | reserved | Smaller PQ KEM |
| `0x06` | ML-KEM-1024 | reserved | Larger PQ KEM |
| `0x07–0xEF` | reserved | reserved | Production allocations |
| `0xF0–0xFD` | experimental | experimental | Implementations may use without spec coordination |
| `0xFE` | test-only | allocated for cross-impl agility tests | Mirrors v7.66 `key_type=0xFE` slot |
| `0xFF` | reserved for protocol use | reserved | |

`enc_key_type` is encoded as varint per V7 §1.5 / §1.2 multikey-style LEB128.

### §3.2 `aead_id` — symmetric AEAD

| Byte | Algorithm | Status | Notes |
|---|---|---|---|
| `0x00` | reserved | reserved | |
| `0x01` | XChaCha20-Poly1305 | **PRODUCTION** | v1 floor. 256-bit key, 192-bit nonce, 128-bit tag. RFC 8439 + XSalsa20 nonce extension. Safe under random nonces |
| `0x02` | AES-256-GCM | reserved (peer-mode-only in v1) | 256-bit key, 96-bit nonce, 128-bit tag. Hardware-accelerated. v1: ALLOWED in peer mode (per-message ephemeral key avoids nonce reuse); FORBIDDEN in self / group modes for v1 (nonce-reuse hazard with stable keys) |
| `0x03` | ChaCha20-Poly1305 (IETF, 96-bit nonce) | reserved | Same nonce constraint as AES-GCM. Same v1 restriction |
| `0x04` | AEGIS-256 | reserved | Future high-performance AEAD |
| `0x05–0xEF` | reserved | reserved | |
| `0xF0–0xFE` | experimental / test | per `enc_key_type` convention | |

### §3.3 `kdf_id` — key derivation

| Byte | Algorithm | Status | Notes |
|---|---|---|---|
| `0x00` | reserved | reserved | |
| `0x01` | HKDF-SHA-256 | **PRODUCTION** | v1 floor. RFC 5869 |
| `0x02` | HKDF-SHA-512 | reserved | Larger output |
| `0x03` | HKDF-SHA-384 | reserved | Pairs with v7.67 SHA-384 validate slot |
| `0x04` | Argon2id (KDF mode, not just password hash) | reserved | Memory-hard; used by `self` mode (see §6.2) |
| `0x05–0xEF` | reserved | reserved | |

For self-mode passphrase-derivation, Argon2id is mandated (§6.2) — that's a separate path from `kdf_id`, which selects the post-keying-material derivation. Self-mode uses both: Argon2id passphrase → master secret, HKDF (`kdf_id`) → per-entity key.

### §3.4 Cipher-suite advertisement

Every encryption-key cert (§4) carries its supported AEAD + KDF set:

```
supported_aead_ids:  array_of(varint)  ; AEADs this key can be used with
supported_kdf_ids:   array_of(varint)  ; KDFs this key supports
```

A sender encrypting to this recipient picks the first AEAD / KDF in the cert's advertised order that matches the sender's own supported set. This mirrors V7 §4.5's `key_types` accept-set discipline (each peer's own algorithm must be in the other's set).

Conformance: every peer MUST advertise at least `{aead_id: 0x01, kdf_id: 0x01}` on every encryption-key cert it publishes. Other entries are optional.

---

## §4 The encryption key

### §4.0 Configuration progression — three tiers

ENCRYPTION runs on V7 alone. Adding ATTESTATION or IDENTITY layers up. The encryption / decryption flow (modes, AEAD, AAD, nonce) is **identical at every tier** — those operate on byte-string keys. What varies by tier is how the encryption-pubkey is **published, attested, rotated, and revoked**. Tier choice is per-deployment; the wire format is tier-aware via discriminator fields on the encrypted entity and the pubkey-publishing path.

Analogous to `EXTENSION-IDENTITY` §1.1's configuration progression: lower tiers are valid configurations, not "broken" ones.

| Tier | Requires | Encryption-pubkey publishing | Rotation | Revocation | Multi-device |
|---|---|---|---|---|---|
| **A — V7 floor** | V7 only | Signed directly by peer's V7 keypair via `system/signature` invariant pointer | Encryption-owned `system/encryption/handoff` entity | Encryption-owned `system/encryption/revocation` entity | No (single-peer single-key model — same as IDENTITY §1.1 item 1) |
| **B — +ATTESTATION** | V7 + ATTESTATION | `system/attestation` with `attester = peer_id`, `attested = pubkey`, `properties.kind = "encryption-key"` | Substrate `supersedes` chain | Universal `revocation` kind | No additional (same as Tier A; the benefit is cross-extension graph tooling) |
| **C — +IDENTITY (full)** | V7 + ATTESTATION + IDENTITY (+ QUORUM transitive) | `identity-cert` attestation with `properties.function = "encryption"`, attesting up to a controller or sub-controller | `identity-rotation-handoff` chain | Universal `revocation` kind with controller-or-above authority | **Yes** — multiple agents under one logical identity each publish their own encryption-cert |

**Tier A** is genuinely valuable: it's the right shape for headless single-purpose peers (CLI tools, storage daemons, simple servers, CDN-only publishers). A peer running V7-only ENCRYPTION can still do self-mode storage encryption and peer-mode relay-encrypted sending; they just lose multi-device convergence.

**Tier B** is the "I want substrate-grade revocation observability but I'm not running a multi-device identity" middle ground. ATTESTATION is small (one entity type + six graph ops + the universal `revocation` kind); installing it is cheap and gets you cross-extension dashboards / introspection / generic-tooling support.

**Tier C** is the "I want one logical identity across my phone + laptop + server" shape — the default in the release-shape deployment. IDENTITY's cert-chain framework provides multi-device + controller-can-issue-for-agents + rotation-via-handoff + authority-rules-for-revocation.

**Sender + recipient can be at different tiers.** The wire format for peer mode (§7) carries NO tier discriminator — `recipient_key` is uniformly the inner `system/encryption-pubkey` content_hash (the same value at every tier), and resolution (§4.4) walks the recipient namespace highest-tier-first. A Tier-A sender encrypting to a Tier-C recipient resolves the recipient's cert chain down to the inner pubkey hash and binds that; a Tier-C sender encrypting to a Tier-A recipient binds the V7-signed pubkey's hash directly. Both derive identical AEAD keys from identical inputs. Interop is normative across tiers; ENC-TIER-INTEROP-1 gates it.

**Cross-tier interop requires the SAME authored inner pubkey entity (F2-3).** "Identical inputs" holds only because every tier binds the *byte-identical* `system/encryption-pubkey` entity. Per §4.1, `content_hash(system/encryption-pubkey)` is a pure function of `(enc_key_type, public_key, supported_aead_ids, supported_kdf_ids, created, expires)` — so a re-minted *equivalent* pubkey (e.g. a Tier-A republish authored at a different `created` timestamp, or a different suite-array ordering) produces a **different** hash → different HKDF `info` → different key → interop fails. This is the F-GO-1 class one layer up (cert-hash-vs-pubkey-hash → pubkey-hash-vs-reminted-pubkey-hash). A peer that publishes the same encryption key at multiple tiers MUST re-publish the **one authored** inner pubkey entity, not re-mint an equivalent. ENC-TIER-INTEROP-1 (§16) shares one inner pubkey entity across both tiers and asserts the `recipient_key` hash is byte-equal.

### §4.1 Inner pubkey entity

```
system/encryption-pubkey := {
  fields: {
    enc_key_type:        {type_ref: "primitive/uint"}        ; varint, per §3.1
    public_key:          {type_ref: "primitive/bytes"}        ; length determined by enc_key_type
    supported_aead_ids:  {array_of: {type_ref: "primitive/uint"}}
    supported_kdf_ids:   {array_of: {type_ref: "primitive/uint"}}
    created:             {type_ref: "primitive/uint"}        ; ms since epoch
    expires:             {type_ref: "primitive/uint", optional: true}
  }
}
```

`content_hash(system/encryption-pubkey)` is a pure function of `(enc_key_type, public_key, supported_aead_ids, supported_kdf_ids, created, expires)`. The pubkey entity is content-addressed and immutable; the cert (§4.2) attests to it.

### §4.2 Publishing per tier

The pubkey from §4.1 is published according to the highest tier installed.

#### §4.2.a Tier A — V7 floor (no ATTESTATION, no IDENTITY)

Publish the `system/encryption-pubkey` entity at:

```
/{peer_id}/system/encryption/pubkey/{pubkey_content_hash}
```

Sign it with the peer's V7 keypair via `system/signature` at the invariant pointer per V7 §5.2 / §7.4:

```
/{peer_id}/system/signature/{hex(pubkey_content_hash)}
```

A sender resolves Tier-A recipient pubkeys by reading `system/encryption/pubkey/{h}` at the recipient's namespace and verifying the invariant-pointer signature with the recipient's V7 peer_id (which is the trust anchor — the V7 peer keypair is self-rooted identity per V7 §1.5 / IDENTITY §1.1 progression item 1).

Discovery: enumerate the `system/encryption/pubkey/` subtree at the recipient's peer namespace; filter out revoked entries per §11; pick most recent live by `created`.

#### §4.2.b Tier B — +ATTESTATION (no IDENTITY)

Publish the `system/encryption-pubkey` entity at the same path as Tier A. Wrap publication authority in a `system/attestation`:

```
system/attestation := {
  attester:    <peer_id>                       ; the publisher
  attesting:   <hash of peer's system/peer entity>   ; the V7 peer keypair as trust anchor
  attested:    <hash of the system/encryption-pubkey>
  properties: {
    kind:      "encryption-key"                ; THIS extension's kind, registered in the ATTESTATION kind-ownership table
  }
  signature:   <peer's V7 signature>
}
```

Path: `/{peer_id}/system/encryption/attestation/{attestation_hash}`.

Discovery: `find_attestations_targeting(attested=pubkey_hash)` per ATTESTATION §5; filter revocations via universal `revocation` kind; pick most recent live.

Tier B is genuinely equivalent to Tier A in features — what it adds is **substrate-grade observability**: graph-tooling that walks `system/attestation` entities for any purpose now sees encryption-key publications uniformly with every other signed-edge claim in the system. Generic dashboards / cross-extension queries / introspection tools work. Tier-A peers are equally functional; they're invisible to substrate-graph tooling because their authority lives in invariant-pointer signatures, not attestations.

#### §4.2.c Tier C — +IDENTITY (full cert-chain)

Publish via `identity-cert` attestation per `EXTENSION-IDENTITY` v3.10 §4.2:

```
system/attestation [identity-cert wrapper] := {
  attester:    <peer issuing the cert; controller or sub-controller>
  attesting:   <controller cert hash; chain terminates at the identity's quorum>
  attested:    <hash of the system/encryption-pubkey inner entity>
  properties: {
    kind:      "identity-cert"
    function:  "encryption"                ; app-defined function per IDENTITY §2.3
    mode:      <publication mode per IDENTITY §4.2a>
  }
  signature:   <attester's signature>
}
```

Per `EXTENSION-IDENTITY` §3.6 / §4, the cert is validated by walking `attesting` back to the identity's quorum and verifying each link. Revocation is checked via `find_revocations_for(cert_hash)` per `EXTENSION-ATTESTATION` §5.

**Authority rules for issuing an encryption-cert (Tier C):**
- Top-level encryption-cert (attesting directly to a controller): the controller MUST sign.
- Sub-controller-authorized encryption-cert: the sub-controller signs; chain walks back through the sub-controller's own cert to the quorum.
- The encryption-key holder (`attested.public_key`'s private-key holder) is typically the same peer as the cert's `attester` (self-attestation under controller chain), but the substrate allows separation (a controller can attest an encryption key on behalf of a sub-peer).

### §4.3 Path conventions

Path layout depends on tier.

**Tier A / Tier B** — encryption-extension's own subtree under the peer namespace:

| Audience | Path | Purpose |
|---|---|---|
| Public (the discoverable handle) | `system/encryption/pubkey/{pubkey_hash}` | Senders encrypt to this |
| Per-key backup | `system/encryption/key-backup/{pubkey_hash}` | Tier 2 passphrase-wrapped backup (§9.2) |
| Revocation | `system/encryption/revocation/{revocation_hash}` (Tier A) or universal `revocation` kind in ATTESTATION's subtree (Tier B) | Marks a pubkey as revoked |

**Tier C** — IDENTITY's audience layout (§5.1):

| Audience | Path | Purpose |
|---|---|---|
| Internal (per-agent private) | `system/identity/internal/cert/{cert_hash}` | This agent's own encryption-cert (its decryption private key lives off-tree per §9; this is the cert) |
| Public (contact-discoverable) | `system/identity/public/encryption/{cert_hash}` | The discoverable handle for senders to encrypt to this identity |
| Per-relationship | `system/identity/relationships/{contact_id}/encryption/{cert_hash}` | Per-contact encryption key for forward-secrecy hygiene |
| Per-key backup | `system/identity/internal/key-backup/{cert_hash}` | Tier 2 passphrase-wrapped backup |

For **self-mode storage** (encrypting your own data), the local internal cert is the one referenced.

For **peer-mode sending** (to another identity / peer), the recipient's public cert is the one referenced.

For **per-relationship hygiene** (rotating the contact-side key independently per relationship), the relationships path applies (Tier C only).

### §4.4 Discovery (tier-aware)

A sender resolves recipient encryption-pubkeys by reading at the recipient's namespace. The sender doesn't need to know the recipient's tier in advance — the sender walks the namespace and finds whichever publication shape the recipient uses.

**Resolution order (sender does this):**

1. Try Tier C: read `system/identity/public/encryption/` at the recipient. If non-empty, enumerate `find_attestations_with_kind(kind="identity-cert", function="encryption")` filtered through identity's authority chain; filter revoked; pick most recent live.
2. Try Tier B: read `system/encryption/attestation/` at the recipient. If non-empty, enumerate `find_attestations_targeting(attested in system/encryption/pubkey/...)` with `kind="encryption-key"`; filter revoked via universal `revocation` kind; pick most recent live.
3. Try Tier A: read `system/encryption/pubkey/` directly at the recipient. Verify each pubkey's `system/signature` at the invariant pointer against the recipient's V7 peer_id; filter revoked via encryption-owned `system/encryption/revocation/`; pick most recent live.
4. None resolved → `403 encryption_recipient_unknown`.

The sender resolves to whichever tier is highest-installed at the recipient.

**Wire-format discriminator (peer mode):** the encrypted entity's `recipient_key` reference is content-addressed (it points at the pubkey entity, regardless of tier); recipient resolves by content_hash through its own local store + signature verification. No wire-level tier discriminator needed — content-addressing handles it.

**Multi-device discovery (Tier C only):** one logical identity may have several agents, each with its own encryption-cert. The sender picks ONE cert and the receiving agent is the agent that holds the private key for that cert. For broadcast to all agents of an identity, use group mode with each agent's cert as a member. Tier A / Tier B peers do not surface multi-device — each peer is its own identity.

---

## §5 The encrypted entity

### §5.1 Common fields

Every `system/encrypted` entity carries:

```
system/encrypted := {
  fields: {
    mode:             {type_ref: "primitive/string"}    ; "self" | "peer" | "group"
    enc_key_type:     {type_ref: "primitive/uint"}      ; varint, per §3.1 (0x00 for self mode)
    aead_id:          {type_ref: "primitive/uint"}      ; varint, per §3.2
    kdf_id:           {type_ref: "primitive/uint"}      ; varint, per §3.3
    nonce:            {type_ref: "primitive/bytes"}     ; per §5.3
    ciphertext:       {type_ref: "primitive/bytes"}     ; AEAD output (ciphertext || tag)
  }
}
```

Per-mode additional fields are defined in §6, §7, §8.

**Sender authentication is NOT a field on this entity** — it lives at the V7 invariant pointer `system/signature/{hex(content_hash)}` per §7.4. A field holding the entity's own content_hash cannot live inside that entity (self-reference: adding the field changes the hash). This was a v2.2-era contradiction (F-GO-3); v2.3 corrects it to the invariant-pointer-only form, consistent with V7 §5.2 post-v7.74 discipline.

The ciphertext, when decrypted, yields a canonically-encoded ECF inner entity — type, data, the whole thing. The inner entity is complete and self-contained.

The content_hash of the encrypted entity is the hash of the outer entity (type + data including ciphertext). It is a real entity with a stable hash. It can live in the tree, in the content store, in envelopes, anywhere.

### §5.2 AAD construction (normative)

AEAD MUST use additional-data binding to prevent context confusion (replay into a different mode / recipient / cipher suite). AAD is **ECF per `ENTITY-CBOR-ENCODING.md`** (RFC 8949 §4.2 length-first canonical ordering, the form already authoritative across the cohort for every other byte-pinned surface). The legacy "canonical CBOR" phrasing in v2.2 admitted two key orderings (length-first vs older bytewise-lexicographic) which broke byte-equality across CBOR libraries; v2.3 pins ECF exclusively (F-GO-2).

**Per-mode key sets (normative, all-keys-present discipline).** Each mode has a FIXED key set. Keys not applicable to a mode are emitted as empty bytes, NOT omitted. Omission vs present-empty is the v7.67 Phase-2 byte-pin trap; v2.3 closes it.

**Self mode** — 8-key AAD:
```
ECF{
  "mode":          "self"
  "enc_key_type":  <varint, always 0 for self mode>
  "aead_id":       <varint>
  "kdf_id":        <varint>
  "nonce":         <24 bytes for XChaCha20-Poly1305>
  "kdf_salt":      <the per-entity Argon2id salt — same bytes as the outer entity's kdf_salt>
  "kdf_params":    <the system/encryption/kdf-params sub-shape as a NESTED CBOR map (R5/F-PY-ENC-1 — nested map, not flattened), ECF-encoded>
  "recipient_key": <empty bytes — no recipient in self mode>
}
```
The `kdf_salt` + `kdf_params` keys bind the derivation parameters to the ciphertext (F2-4). v2.3 omitted them, leaving the primary storage path weaker-bound than the §9.2 cold-backup path (which already binds them) — an attacker with local write access could flip the params to force a wrong-key tag-failure (DoS-of-your-own-data; not a confidentiality break, since self mode has no signature to catch it either way), and the inconsistency is a cross-impl byte-pin trap. Binding them here closes both. The receiver reads the same `kdf_salt`/`kdf_params` from the outer entity (§6.4) to re-derive, so the AAD is reconstructed identically.

**Peer mode** — 7-key AAD:
```
ECF{
  "mode":          "peer"
  "enc_key_type":  <varint, per §3.1>
  "aead_id":       <varint>
  "kdf_id":        <varint>
  "nonce":         <24 bytes>
  "recipient_key": <content_hash of recipient's inner system/encryption-pubkey entity — uniform at every tier; F-GO-1>
  "ephemeral_key": <sender's ephemeral public key, length per enc_key_type>
}
```

**Group outer-ciphertext AAD** — 7-key (self-shape + key-commitment):
```
ECF{
  "mode":          "group"
  "enc_key_type":  <varint, 0 for outer — the outer key is the random group_aead_key, not a keypair>
  "aead_id":       <varint>
  "kdf_id":        <varint>
  "nonce":         <24 bytes>
  "commitment":    <32 bytes — SHA-256(group_aead_key); the key-commitment tag, F2-1>
  "recipient_key": <empty bytes — no single recipient at the group-outer level>
}
```
**Key-commitment (F2-1).** XChaCha20-Poly1305 (Poly1305) is **not key-committing**: without the `commitment` field a malicious group author could craft one outer `(ciphertext, nonce, AAD)` that opens under two *different* `group_aead_key` values wrapped to two different members, delivering divergent plaintexts under one signature (invisible-salamanders / partitioning-oracle, Len–Grubbs–Ristenpart USENIX'21). Binding `commitment = SHA-256(group_aead_key)` into the outer AAD makes **only the single committed key** open the outer ciphertext: a second victim reconstructs the AAD with `SHA-256(their_key)`, gets a different AAD, and AEAD.Open fails by construction. Equivocation is structurally impossible. The receiver already holds `group_aead_key` (recovered from their wrap, §8.4), so it recomputes the commitment locally and binds it — no separate outer entity field is needed. With the cap-axis retired, ENCRYPTION is *the* release-shape privacy primitive, so a multi-recipient mode must not ship with an unstated non-committing-AEAD assumption.

**Group per-wrap AAD** — 7-key (peer-mode-shaped, one per member), domain-separated:
```
ECF{
  "mode":          "group-wrap"      ; F2-2: distinct from "peer" so a recovered group key cannot be
                                     ; confused with a recovered inner entity, and a lifted wrap blob
                                     ; cannot be replayed as a standalone peer-mode message
  "enc_key_type":  <varint, per that member's enc_key_type>
  "aead_id":       <varint>
  "kdf_id":        <varint>
  "nonce":         <that wrap's wrap_nonce>
  "recipient_key": <that member's pubkey-entity content_hash>
  "ephemeral_key": <that wrap's ephemeral key>
}
```
**Domain separation (F2-2).** v2.3 labeled the per-wrap AAD `mode:"peer"`, which both contradicted §5.2's own "AAD prevents replay into a different mode" claim and made a `wrapped_aead_key` blob a *valid standalone peer-mode ciphertext* (plaintext = the 32-byte group key) replayable to that member as a peer message. The distinct `"group-wrap"` mode label closes this without a content_hash circularity (the wraps are inside the outer entity, so the outer `content_hash` is not yet known when the wraps are AEAD'd).

Reference AAD hex per mode is given in §16 (ENC-SELF-KAT-1 / ENC-PEER-KAT-1 / ENC-GROUP-KAT-1 conformance vectors). Cohort impl MUST byte-match the reference hex.

The AAD is reconstructed identically by sender (during encryption) and receiver (during decryption). Tampering with any AAD field by an attacker between encryption and decryption MUST cause the AEAD tag to fail verification.

On AEAD failure, decryption MUST fail with error `400 encryption_aead_failed` (§15).

### §5.3 Nonce semantics per AEAD

Nonce derivation depends on the AEAD:

| `aead_id` | Nonce length | v1 derivation | Allowed in self/group? |
|---|---|---|---|
| `0x01` XChaCha20-Poly1305 | 24 bytes | Random per message | Yes |
| `0x02` AES-256-GCM | 12 bytes | Random per message — SAFE ONLY because peer mode uses a fresh key per message; forbidden in self / group modes for v1 | **No** for self/group |
| `0x03` ChaCha20-Poly1305 (IETF) | 12 bytes | Same constraint as AES-GCM | **No** for self/group |

For `0x02` / `0x03` in self / group modes, the impl MUST refuse encryption and the receiver MUST refuse decryption with `400 encryption_unsupported_suite`. (v1.5 may relax this with a deterministic-counter construction; that's out of scope for v1.)

For `0x01` XChaCha20-Poly1305: nonce is 24 random bytes per encryption. Collision probability under birthday bound is ≈ 2⁻⁹⁶ after 2⁴⁸ messages per key — safe for any realistic deployment.

---

## §6 Mode: self (storage encryption) — v1 PRIMARY

`self` mode encrypts content for the encrypting peer's own later decryption. The key is symmetric, derived from a local secret (passphrase, keyfile, platform keychain entry, hardware token). No public-key crypto involved.

**Use cases:** encrypt files at rest on disk; encrypt entities published to a CDN where the publisher is also the consumer; encrypt local backup snapshots.

### §6.1 Wrapper shape

```
system/encrypted [mode = "self"] additional fields: {
  key_id:      {type_ref: "primitive/string"}          ; selects which local secret to use
  kdf_salt:    {type_ref: "primitive/bytes"}           ; per-entity random salt (16 bytes minimum)
  kdf_params:  {type_ref: "system/encryption/kdf-params"}   ; R4: named sub-type (below), not an anonymous {fields:…}
}
```

The `kdf_params` sub-shape is a **named type** (R4 — anonymous inline `{fields:…}` is not registrable in the type system; the cohort confirmed this when Rust declined to fabricate an internal name without sign-off, while Go (`0a1351f`) + Python pragma-named it):
```
system/encryption/kdf-params := {
  fields: {
    argon2_version: {type_ref: "primitive/uint"}     ; Argon2 version byte (v1 floor = 0x13 / v19)
    memory_cost:    {type_ref: "primitive/uint"}     ; Argon2id m (KiB)
    time_cost:      {type_ref: "primitive/uint"}     ; Argon2id t
    parallelism:    {type_ref: "primitive/uint"}     ; Argon2id p
    output_len:     {type_ref: "primitive/uint"}     ; bytes
  }
}
```

The `enc_key_type` field is `0x00` (reserved/unused) in self mode — no keypair, only symmetric. Field names in `kdf_params` are normative (`memory_cost`, `time_cost`, `parallelism` — NOT `m`, `t`, `p` abbreviations); cohort impls MUST emit those keys for ECF byte-equality (F-GO-9).

### §6.2 Key derivation (Argon2id baseline)

The user-side secret (passphrase, keyfile bytes, keychain entry) is converted to the AEAD key via:

```
master_secret = utf8(passphrase)              ; UTF-8 encoding, NO trailing NUL byte (F-GO-9)
                                              ; for keyfile / keychain bytes: raw bytes as-stored
kek           = Argon2id(
                  password    = master_secret,
                  salt        = kdf_salt,
                  version     = 0x13,         ; Argon2 v1.3 / v19 — v1 floor (F-GO-9)
                  memory_cost = m,            ; baseline 64 MiB (65536 KiB)
                  time_cost   = t,            ; baseline 3
                  parallelism = p,            ; baseline 1
                  output_len  = 32 bytes
                )
aead_key      = HKDF-SHA-256(
                  ikm   = kek,
                  salt  = nonce,
                  info  = utf8("entity-core/self/") || utf8(key_id),  ; ASCII prefix + UTF-8 key_id, no separator, no NUL
                  L     = AEAD key length
                )
```

**Baseline parameters (pinned by spec; configurable per impl up; portability via `kdf_params`):**
- Argon2 version `0x13` (v1.3 / v19) — pinned for v1; matches Go's `x/crypto/argon2` stdlib + Rust + Python library defaults.
- Argon2id memory cost `m = 65536` (64 MiB, in KiB per RFC 9106 §3.1 convention).
- Argon2id time cost `t = 3`.
- Argon2id parallelism `p = 1`.
- Argon2id output length `32 bytes`.
- Random `kdf_salt`, ≥ 16 bytes.

These match IETF RFC 9106 baseline recommendations.

Stored `kdf_params` carries the actual `(argon2_version, memory_cost, time_cost, parallelism, output_len)` used so a different peer (or this peer post-rotation) can re-derive given the same user secret. Field names normative per §6.1.

### §6.3 Encryption flow (self)

1. Compose the inner entity in ECF.
2. Generate random nonce (per §5.3) and `kdf_salt`.
3. Look up the local secret by `key_id` (impl-specific — keychain entry, prompted passphrase, hardware token).
4. Derive `aead_key` per §6.2.
5. Compute AAD per §5.2 (`recipient_key` empty, `ephemeral_key` absent).
6. `ciphertext = AEAD_encrypt(aead_key, nonce, AAD, inner_entity_ecf)`.
7. Construct the `system/encrypted` outer entity. Store / publish.

### §6.4 Decryption flow (self)

1. Inspect outer entity's `key_id`, `kdf_salt`, `kdf_params`.
2. Look up the local secret by `key_id`. If missing → `403 encryption_key_unavailable` (user-side cap layer rejected, missing keychain entry, etc.).
3. Re-derive `aead_key` per §6.2.
4. Reconstruct AAD per §5.2.
5. `inner_entity_ecf = AEAD_decrypt(aead_key, nonce, AAD, ciphertext)`. On tag failure → `400 encryption_aead_failed`.
6. Decode the inner entity and return.

### §6.5 Self-mode storage portability

A peer with the same `key_id` and the same user secret can decrypt the same self-encrypted entity. This is the multi-device case: agent A on the desktop, agent B on the phone, both under one identity, both with the same passphrase configured — they read each other's encrypted storage.

`key_id` is a logical identifier (the same key_id on different devices selects the same logical secret). It is opaque to the protocol — `"user-passphrase"`, `"device-keyfile-1"`, `"yubikey-slot-9c"`. The protocol pins the derivation; the impl pins the user-side mechanism per `key_id`.

---

## §7 Mode: peer (relay-encrypted sending) — v1 PRIMARY

`peer` mode encrypts content for one specific recipient (identified by their encryption-cert), in a non-interactive single-shot hybrid construction. The sender does ECDH with the recipient's encryption pubkey, derives a per-message symmetric key, and AEAD-encrypts the inner entity.

**Use cases:** send an encrypted entity through an untrusted relay; encrypt content to a known recipient's CDN; encrypted mailbox / inbox delivery.

### §7.1 Honest framing (read before designing)

This is **non-interactive single-shot hybrid encryption**, structurally equivalent to age, NaCl `crypto_box`, or libsodium sealed boxes (with sender authentication). It provides:

- **Receiver authentication.** Only a peer holding the recipient encryption-cert's private key can decrypt.
- **Sender ephemerality.** The sender's per-message ephemeral key is discarded after encryption; the sender cannot themselves decrypt their own past sends.
- **Sender authentication.** The sender publishes a `system/signature` at the invariant pointer `system/signature/{hex(content_hash(encrypted_entity))}` (§7.4 / V7 §5.2 post-v7.74 discipline; F-GO-3 — NOT a field on the encrypted entity). The recipient verifies who sent it.

It does **NOT** provide:

- **Forward secrecy against recipient-key compromise.** If the recipient's encryption private key is later compromised, an attacker holding the persisted (`ephemeral_pub`, `ciphertext`) pair can recover the shared secret using the compromised private + the public ephemeral, and decrypt.
- **Interactive forward secrecy / future secrecy** of the kind a session ratchet provides.

This is honest framing because the threat model peer mode protects against is **passive observation of the relay / storage path**, not **post-decryption compromise of the recipient**. For the latter, see the future session-encryption extension (out of v1 scope).

### §7.2 Wrapper shape

```
system/encrypted [mode = "peer"] additional fields: {
  ephemeral_key:   {type_ref: "primitive/bytes"}        ; sender's ephemeral public key (per enc_key_type)
  recipient_key:   {type_ref: "system/hash"}            ; content_hash of the recipient's INNER system/encryption-pubkey entity — uniform at every tier (F-GO-1)
}
```

`recipient_key` is the inner pubkey-entity hash, NOT a cert / attestation hash. The pubkey entity exists identically at every tier (§4.1); cert / attestation wrapping is tier-specific (§4.2.a/b/c) and not bound into the key schedule. Resolution (§4.4) walks the recipient namespace highest-tier-first and projects whatever it finds (V7 invariant-pointer signature / single ATTESTATION / IDENTITY cert chain) down to the inner pubkey entity hash before binding. This is what makes Tier-A ↔ Tier-C cross-tier interop work by construction.

The cipher suite is determined by the recipient's encryption-cert / pubkey entry. The sender picks the first `(enc_key_type, aead_id, kdf_id)` triple advertised by the recipient's `system/encryption-pubkey` that the sender also supports; if intersection empty, error `400 encryption_no_common_suite`.

### §7.3 Encryption flow (peer)

1. Look up recipient's encryption-pubkey entry per §4.4. Pick cipher suite per §3.4. Record `recipient_pubkey_hash = content_hash(recipient's system/encryption-pubkey entity)` (uniform across tiers — F-GO-1).
2. Generate ephemeral keypair `(eph_priv, eph_pub)` of the chosen `enc_key_type`.
3. ECDH:
   - `enc_key_type = 0x01` (X25519): `shared_secret = X25519(eph_priv, recipient_pubkey)`.
   - `enc_key_type = 0x02` (X448): `shared_secret = X448(eph_priv, recipient_pubkey)`.
   - `enc_key_type = 0x03` (ML-KEM-768): `(shared_secret, ciphertext_kem) = ML-KEM-768.Encaps(recipient_pubkey)`; ciphertext_kem becomes `ephemeral_key`.
   - `enc_key_type = 0x04` (X25519 + ML-KEM-768 hybrid): `ss_classical = X25519(...); (ss_pq, ct_pq) = ML-KEM-768.Encaps(...)`; concatenate `shared_secret = ss_classical || ss_pq`; ephemeral_key = `eph_pub_classical || ct_pq`.
4. Derive AEAD key:
   ```
   aead_key = HKDF(kdf_id,
                   ikm   = shared_secret,
                   salt  = nonce,
                   info  = utf8("entity-core/peer/") || recipient_pubkey_hash,    ; uniform across tiers (F-GO-1)
                   L     = AEAD key length)
   ```
   `recipient_pubkey_hash` here (and as the `recipient_key` AAD/wire value) is the **full 33-byte content_hash** — the 1-byte multihash format prefix (`0x00` for SHA-256) + the 32-byte digest — **NOT** the bare 32-byte digest (R5 / F-PY-ENC-2). The cohort converged on this 3-way; the spec now pins it so the HKDF `info` (and therefore the derived key) is byte-identical across impls.
5. Compute AAD per §5.2 (peer-mode 7-key shape; `recipient_key` = `recipient_pubkey_hash`; `ephemeral_key` = `eph_pub`).
6. `ciphertext = AEAD_encrypt(aead_key, nonce, AAD, inner_entity_ecf)`.
7. Construct the `system/encrypted` outer entity from steps 1–6.
8. Sign per §7.4 — publish a `system/signature` entity at the V7 invariant pointer `system/signature/{hex(content_hash(encrypted_entity))}`. **No field on the encrypted entity** (F-GO-3); the signature is discovered at the invariant pointer.
9. Discard `eph_priv` and `shared_secret`.

### §7.4 Sender authentication (default ON in v1)

Sender authentication is provided by a `system/signature` entity published at the V7 invariant-pointer path:

```
system/signature/{hex(content_hash(encrypted_entity))}
```

per V7 §5.2 (post-v7.74 discipline). The signature entity itself:

```
system/signature := {
  fields: {
    target:    <content_hash(system/encrypted)>
    signer:    <sender's identity-cert hash (Tier C) or sender's peer_id (Tier A/B)>
    algorithm: <Ed25519 / Ed448 / ML-DSA-65 per V7 §1.5>
    signature: <bytes>
  }
}
```

**NO field on the encrypted entity references the signature** (F-GO-3 closeout). The receiver discovers the signature by computing `content_hash` of the received encrypted entity and resolving the invariant pointer. A field holding the entity's own content_hash is structurally impossible — adding it changes the hash.

The recipient verifies:
1. Resolve `system/signature/{hex(target)}` invariant pointer.
2. Verify signature with `signer`'s identity-cert pubkey per V7 §5.2 / §7.4.
3. Confirm `signer` is an authorized sender for the expected identity (per receiving peer's policy — typically "any agent under controller X").

**Anonymous peer mode** (sender authentication off) is opt-in via `mode: "peer-anonymous"`. Reserved as a future mode value; not delivered in v1. v1 ships sender-signed-by-default.

### §7.5 Decryption flow (peer)

1. Compute `content_hash(received encrypted entity)`. Resolve `system/signature/{hex(content_hash)}` invariant pointer. If absent → `403 encryption_unsigned_sender`. If signature verification fails → `403 encryption_signature_invalid` (§7.4).
2. Look up local `system/encryption-pubkey` entity by `recipient_key` content_hash (direct content-store lookup — F-GO-1, uniform across tiers). If not held locally → `403 encryption_recipient_unknown`.
3. Recompute shared_secret using own encryption-private key + sender's `ephemeral_key`:
   - X25519: `X25519(my_priv, ephemeral_key)`.
   - ML-KEM-768: `ML-KEM-768.Decaps(my_priv, ephemeral_key)`.
   - Hybrid: parallel both, concat (same as §7.3 step 3).
4. Derive AEAD key per §7.3 step 4 — `info` uses `recipient_pubkey_hash` (the same `recipient_key` value from step 2). Tier-A and Tier-C decryptors arrive at identical keys.
5. Reconstruct AAD per §5.2 (peer-mode 7-key shape).
6. `inner_entity = AEAD_decrypt(aead_key, nonce, AAD, ciphertext)`. On tag failure → `400 encryption_aead_failed`.

### §7.6 Cross-content-hash-format references

Per V7 §1.8 + v7.69 §4.5a: `recipient_key` is the cert's **authored** content_hash under the recipient's home content_hash_format. The sender MUST NOT re-derive the cert hash under the sender's home format. If sender and recipient are in different content-address-spaces (rare; not v1-floor), the sender uses the recipient's authored hash exactly as the recipient published it.

---

## §8 Mode: group (static key-wrap) — v1 BEST-EFFORT

`group` mode encrypts content readable by a fixed set of recipients (members). Random symmetric key per encrypted entity, wrapped per-member using peer-mode-style hybrid encryption.

**Use cases:** share an encrypted entity with a fixed small set of peers (≤256 members).

### §8.1 Honest framing

- **Member set is plaintext.** Each `wrapped_keys[*].recipient_key` is observable; the membership of the group is visible to anyone holding the entity.
- **No group-PFS.** Removing a member by re-keying protects FUTURE entities only; the removed member still has the old key and can decrypt past entities they previously had access to.
- **Static groups only.** Dynamic groups with frequent membership churn want MLS / TreeKEM; that is out of v1 scope and lives in a future extension.

### §8.2 Wrapper shape

```
system/encrypted [mode = "group"] additional fields: {
  wrapped_keys: {array_of: {type_ref: "system/encryption/wrapped-key"}}   ; R4: named element type (below)
}
```

The `wrapped_keys` element is a **named type** (R4, same rationale as `kdf-params` — Go `0a1351f` is the reference):
```
system/encryption/wrapped-key := {
  fields: {
    recipient_key:    {type_ref: "system/hash"}       ; member's INNER system/encryption-pubkey content_hash (F-GO-1, uniform with peer mode)
    enc_key_type:     {type_ref: "primitive/uint"}    ; per-member, varint
    ephemeral_key:    {type_ref: "primitive/bytes"}   ; per-member sender ephemeral
    wrapped_aead_key: {type_ref: "primitive/bytes"}   ; the symmetric key, AEAD-encrypted to this member
    wrap_nonce:       {type_ref: "primitive/bytes"}   ; per-wrap nonce
  }
}
```

Each `wrapped_keys[i]` is structurally peer-mode hybrid encryption to member `i`, applied to the random `group_aead_key` as the inner plaintext. The outer `ciphertext` (§5.1) is AEAD-encrypted with `group_aead_key` itself. **No `sender_signature` field on a wrapped_key** (F-GO-3 + F-GO-6); the outer entity is signed once at the invariant pointer per §7.4 and that authenticates the entire group entity including all wraps.

### §8.3 Encryption flow (group)

1. Generate random `group_aead_key` (32 bytes for XChaCha20-Poly1305).
2. Compute `commitment = SHA-256(group_aead_key)` (F2-1).
3. Compute the **outer-ciphertext AAD** per §5.2 (group outer 7-key shape, `commitment` bound).
4. AEAD-encrypt the inner entity with `group_aead_key` + outer nonce + outer AAD. Result = the outer `ciphertext`.
5. For each member `M`:
   - Resolve M's `system/encryption-pubkey` per §4.4 → `M.pubkey_hash` (uniform across tiers).
   - Run **peer-mode encryption flow §7.3 steps 1–6 only** (F-GO-6 — NO step 7/8/9; the wrap is not separately signed). Plaintext = `group_aead_key`. Inputs: M's pubkey, fresh per-wrap ephemeral keypair, fresh per-wrap `wrap_nonce`. AAD = **group per-wrap AAD** per §5.2 (`mode:"group-wrap"`, 7-key, with M's pubkey_hash as `recipient_key`, that wrap's ephemeral as `ephemeral_key`, `wrap_nonce` as `nonce`). Output = `wrapped_aead_key` for M.
   - Discard per-wrap `eph_priv` and intermediate shared_secret.
6. Construct the `system/encrypted` outer entity (mode="group") with `wrapped_keys[]` populated.
7. Sign the outer entity ONCE per §7.4 (single `system/signature` at the invariant pointer for the outer entity's content_hash). This authenticates the entire group entity including all wraps.

### §8.4 Decryption flow (group)

1. Verify outer signature.
2. Scan `wrapped_keys` for an entry whose `recipient_key` matches a cert held locally.
3. If found: decrypt the `wrapped_aead_key` (using the `mode:"group-wrap"` per-wrap AAD, §5.2) to recover `group_aead_key`.
4. Reconstruct the outer AAD with `commitment = SHA-256(group_aead_key)` (F2-1) and AEAD-decrypt the outer `ciphertext`. If the author committed a different key to a different member, this member's reconstructed AAD will not match and AEAD.Open fails with `encryption_aead_failed` — the equivocation is rejected, not silently decrypted to a divergent plaintext.

### §8.5 Add / remove member

- **Add member:** publish a new `system/encrypted` superseding the old (per `EXTENSION-REVISION`) with the additional member's wrapped_keys entry. The `group_aead_key` does NOT change; existing members re-use the same key. This is cheap.
- **Remove member:** generate a fresh `group_aead_key`, re-encrypt the inner content, re-wrap for the remaining members. The removed member retains the OLD key and can still decrypt OLD entities; only NEW entities are protected. Honest framing: this is forward secrecy at the **group-snapshot** level, not at the message level.

### §8.6 Resource bound

Default ceiling: 256 `wrapped_keys` per entity (configurable per impl up to V7 §4.10's payload bound). Exceeding → `413 encryption_wrapped_keys_too_many` (§15). Rationale: real groups are usually small (≤50); 256 is generous; impls that want larger groups configure up explicitly.

---

## §9 At-rest key storage (defense in depth)

**The encryption private key is a high-value secret. v1 ships a normative defense-in-depth model: two tiers per agent.** A single-fault loss (lost device, forgotten passphrase, corrupted keychain) MUST NOT lock the user out of their own encrypted storage permanently.

### §9.1 Tier 1 — Primary (hot)

Platform secure storage where available:
- **macOS**: Keychain (Secure Enclave-wrapped where available).
- **Windows**: Credential Manager (DPAPI-wrapped); Windows Hello hardware key where available.
- **Linux**: libsecret (GNOME Keyring / KWallet); systemd-credential-store on systems that support it.
- **Hardware token**: YubiKey, NitroKey, TPM, OS-provided secure enclave.

The hot storage allows transparent decryption while the user is logged in. SHOULD be unlocked at session start (OS-authenticated user) and re-locked on session end.

Where no platform secure storage is available (server, headless deployment), Tier 1 collapses to a passphrase prompt or a sealed configuration file.

### §9.2 Tier 2 — Backup (cold)

A passphrase-wrapped copy of the encryption private key, stored on disk. Path is **tier-relative** to match §4.3 (F-GO-7):

| Tier | Backup path |
|---|---|
| Tier A / Tier B | `/{peer_id}/system/encryption/key-backup/{pubkey_hash}` |
| Tier C | `/{peer_id}/system/identity/internal/key-backup/{cert_hash}` |

Tier A has no `system/identity` subtree (no IDENTITY installed); the encryption-extension's own subtree is used. The `{pubkey_hash}` / `{cert_hash}` selector matches the publishing path's hash kind at that tier.

Structure:

```
system/encryption/key-backup := {
  fields: {
    pubkey_ref:  {type_ref: "system/hash"}         ; (Tier A/B) the encryption-pubkey this backs up
                                                   ; (Tier C: replace with cert_ref pointing at the identity-cert)
    kdf_salt:    {type_ref: "primitive/bytes"}     ; per-backup random salt (16 bytes)
    kdf_params:  {fields: {                        ; per §6.1 normative field names (F-GO-9)
                    argon2_version: {type_ref: "primitive/uint"}
                    memory_cost:    {type_ref: "primitive/uint"}
                    time_cost:      {type_ref: "primitive/uint"}
                    parallelism:    {type_ref: "primitive/uint"}
                    output_len:     {type_ref: "primitive/uint"}
                }}
    wrap_nonce:  {type_ref: "primitive/bytes"}     ; XChaCha20-Poly1305 nonce (24 bytes)
    wrapped_key: {type_ref: "primitive/bytes"}     ; encrypted private key bytes
  }
}
```

(Tier C: rename `pubkey_ref` → `cert_ref`; otherwise structurally identical. The entity type at Tier C is `system/identity/key-backup` per the identity subtree.)

Derivation:

```
kek         = Argon2id(
                password    = utf8(passphrase),                ; UTF-8, no trailing NUL (F-GO-9)
                salt        = kdf_salt,
                version     = 0x13,                             ; v1.3 / v19
                memory_cost = 65536,                            ; baseline 64 MiB
                time_cost   = 3,
                parallelism = 1,
                output_len  = 32
              )
wrap_key    = HKDF-SHA-256(
                ikm   = kek,
                salt  = wrap_nonce,
                info  = utf8("entity-core/key-backup/") || pubkey_ref_hash,  ; binds backup to the specific key
                L     = 32
              )
backup_AAD  = ECF{                                              ; ECF, F-GO-2 discipline
                "pubkey_ref":     <pubkey hash>,
                "argon2_version": 0x13,
                "memory_cost":    65536,
                "time_cost":      3,
                "parallelism":    1,
                "output_len":     32
              }
wrapped_key = XChaCha20-Poly1305.encrypt(wrap_key, wrap_nonce, AAD=backup_AAD, plaintext=private_key_bytes)
```

The user can always recover Tier 1 from Tier 2 by re-entering the passphrase. The backup entity is a normal tree entity; it is replicated and persisted with normal storage discipline.

### §9.3 Tier 3 — Shamir N-of-K (designed, W2 impl)

Shamir N-of-K split across devices / people / safety-deposit-boxes is **fully designed in §19.4** (entity shape, distribution flow, reconstruction flow, refresh, custody model). Impl is W2 vault/custodians workstream per `WORKSTREAMS.md`.

Reserved paths (tier-relative, matching §4.3):

| Tier | Share path |
|---|---|
| Tier A / Tier B | `/{peer_id}/system/encryption/key-share/{share_id}` |
| Tier C | `/{peer_id}/system/identity/internal/key-share/{share_id}` |

Cross-reference §19.4 for the full design.

### §9.4 Three different keys, three different storages

| Key | Purpose | Storage strategy |
|---|---|---|
| Identity signing key (Ed25519, `key_type=0x01`) | Signing peers / certs | IDENTITY's purview; typically Tier 1 platform storage |
| Encryption private key (X25519, `enc_key_type=0x01`) | Decrypting peer / self / group entities | This extension; Tier 1 + Tier 2 (this section) |
| At-rest passphrase-derived KEK | Wrapping Tier 2 backup | Never persisted; derived per-session from user input |

Compromise of one MUST NOT trivially compromise the others. Specifically:
- Compromise of the identity signing key does NOT decrypt past content (separate key).
- Compromise of Tier 1 (live keychain dump) recovers the encryption private — but not the identity key, and not past Tier-2 backups if the passphrase is independent.
- Compromise of Tier 2 alone does not yield the private key without the passphrase.

---

## §10 Rotation (tier-aware)

Rotation publishes a new encryption-pubkey and links old → new so senders can follow the chain. Substrate-aligned at every tier; no mutation.

### §10.1 Tier A — V7 floor rotation

This extension defines `system/encryption/handoff` entity:

```
system/encryption/handoff := {
  fields: {
    previous_pubkey: {type_ref: "system/hash"}    ; old encryption-pubkey
    next_pubkey:     {type_ref: "system/hash"}    ; new encryption-pubkey
    created:         {type_ref: "primitive/uint"} ; ms since epoch
  }
}
```

Path: `/{peer_id}/system/encryption/handoff/{handoff_hash}`.

Signed by **both** the old encryption-pubkey holder AND the new (dual-sig invariant pointer signatures at `system/signature/{hex(handoff_hash)}`). The dual-sig protects against an attacker with one key (old or new) silently rotating to a key they control.

Senders walk the handoff chain forward: starting from a known old pubkey, find the most recent `system/encryption/handoff` where `previous_pubkey = old`; follow `next_pubkey`; repeat until no further handoff exists. The terminal pubkey is the current one.

### §10.2 Tier B — +ATTESTATION rotation

Use the ATTESTATION `supersedes` field on the encryption-key attestation:

```
new_attestation.supersedes = <old attestation's content_hash>
```

Per `EXTENSION-ATTESTATION` §5.6, `walk_supersedes_chain` finds the live head. No separate handoff entity needed; the supersedes graph IS the rotation chain.

**Dual-sig target = the supersedes-attestation's content_hash** (F-GO-5; corrects v2.2 collision on the pubkey-hash invariant pointer). The new attestation is published with its own self-signature (new pubkey holder, signing the new attestation per Tier B publishing in §4.2.b). A **second** signature — from the OLD encryption-pubkey holder, targeting the new-attestation's content_hash — is published at the V7 invariant pointer `system/signature/{hex(new_attestation_hash)}` slot for the OLD signer. Per V7 §5.2, invariant pointers are keyed by `(target_hash, signer)`, so the new-attestation's content_hash CAN host two signatures from two distinct signers without collision. The pubkey-hash slot is not touched. This mirrors Tier A's discipline (dual-sig the handoff entity's hash, not the pubkey hash).

Pre-v2.3 draft text said the dual-sig went over `new_pubkey_content_hash`, which collided with the new key holder's self-publish signature at the same invariant pointer (single-occupant per `(target, signer)` — but in v2.2's misreading we had two different signers attempting the same target with the new pubkey hash). v2.3 pins the dual-sig target at the **attestation hash**, not the pubkey hash.

### §10.3 Tier C — +IDENTITY rotation

Issue `identity-rotation-handoff` attestation per `EXTENSION-IDENTITY` §4.3:

```
properties.kind   = "identity-rotation-handoff"
attesting         = <old encryption-cert hash>
attested          = <new encryption-cert hash>
signatures        = dual: signed by old cert holder AND new cert holder
```

This is the substrate-native rotation primitive. Same dual-sig discipline; IDENTITY pinned it in §4.3.

### §10.4 Retention discipline (all tiers)

The old encryption private key MUST be retained as long as the holder wants to decrypt OLD content. Retirement (deleting the old private key) is the user's call:
- For **storage** (self mode, encrypted backups), old private key kept indefinitely or until content migrated to new key.
- For **communication** (peer mode), old private key may be deleted to provide post-compromise security against future compromise of the storage; old content becomes unreadable to anyone.

There is no normative retention schedule; this is a deployment policy.

### §10.5 Receiver-side fallback resolution (all tiers)

Sender encrypts to recipient's `current` cert/pubkey. Recipient may have rotated since sender's last view. Receiver-side decryption attempts in order:
1. Look up `recipient_key` in local pubkey store.
2. If `recipient_key` matches an old pubkey the receiver retains the private key for: decrypt directly.
3. If not: error `403 encryption_recipient_unknown`. The sender re-resolves the receiver's current pubkey via §4.4 discovery + re-encrypts.

---

## §11 Revocation (tier-aware)

Revocation marks an encryption-pubkey as no longer trusted; senders MUST stop encrypting to revoked pubkeys. Substrate-aligned at every tier; no mutation of the original pubkey entity.

### §11.1 Tier A — V7 floor revocation

This extension defines `system/encryption/revocation` entity:

```
system/encryption/revocation := {
  fields: {
    revokes:  {type_ref: "system/hash"}        ; encryption-pubkey being revoked
    reason:   {type_ref: "primitive/string", optional: true}
    created:  {type_ref: "primitive/uint"}
  }
}
```

Path: `/{peer_id}/system/encryption/revocation/{revocation_hash}`.

Signed by the peer's V7 keypair via `system/signature` invariant pointer. The trust anchor is the V7 peer keypair (self-rooted identity per IDENTITY §1.1 progression item 1).

Authorized revoker: the peer that owns the V7 keypair (i.e., themselves). Tier A has no multi-peer authority chain; the V7-peer is the authority.

Discovery: senders enumerate `system/encryption/revocation/` at the recipient's namespace before encrypting; if any revocation targets the chosen pubkey → reject.

### §11.2 Tier B — +ATTESTATION revocation

Use the universal `revocation` attestation kind per `EXTENSION-ATTESTATION` §5.6:

```
properties.kind = "revocation"
attested        = <encryption-pubkey hash being revoked>
attester        = <peer_id>
```

Signed by the peer's V7 keypair. Same authority as Tier A (peer-self); benefits from substrate-graph tooling visibility.

Senders MUST call `find_revocations_for(pubkey.content_hash)` per ATTESTATION §5 before encrypting. A live revocation → reject.

### §11.3 Tier C — +IDENTITY revocation

Use the universal `revocation` kind, with authority governed by IDENTITY's `identity_is_authorized_revoker` predicate (`EXTENSION-IDENTITY` §3.6 / §6.4):

```
properties.kind = "revocation"
attested        = <encryption-cert hash being revoked>
attester        = <authorized revoker, per identity_is_authorized_revoker>
```

Authorized revokers: the controller-or-above in the identity graph. Sub-controller chain governs who can revoke whom — same as IDENTITY's agent-cert revocation in §6.4.

Senders MUST check `find_revocations_for(cert.content_hash)` before encrypting. A live revocation → reject; do not encrypt to a revoked cert.

### §11.4 Receiver hygiene (all tiers)

Receivers MAY decrypt content encrypted before revocation, depending on whether the private key is retained. A receiver SHOULD destroy the private key on revocation (post-compromise hygiene), but the substrate does not force this — local impl policy.

---

## §12 Resource bounds

Inherits V7 §4.10 floor:

- **Per-payload size:** the encrypted entity (outer wrapper + ciphertext) inherits V7 §4.10(a) `payload_too_large` (default 16 MiB; configurable per impl with safe defaults).
- **Group-mode `wrapped_keys` ceiling:** 256 default (§8.6); configurable up. Exceeding → `413 encryption_wrapped_keys_too_many`.
- **Argon2id memory parameter:** capped at impl-defined ceiling to prevent DoS via outsized passphrase-derivation requests. Default `kdf_params.memory_cost ≤ 1 GiB`; exceeding → `400 encryption_kdf_params_excessive`.

---

## §13 Composition

### §13.1 Frame-level (V7 §4.5) vs entity-level (this extension)

Orthogonal layers. Both MAY be active simultaneously (defense in depth):
- **Frame-level** secures the link against passive wire observation. Negotiated per V7 §4.5 `encryption` field (single-active-value per v7.69).
- **Entity-level** secures the entity against everyone except the intended decryptor — including the relay, the storage layer, and any intermediate handler.

Neither replaces the other. Frame-level disabled is acceptable when transport is already encrypted (TLS at the transport layer below); entity-level disabled is acceptable when content is genuinely public.

### §13.2 Relay (`EXTENSION-RELAY`)

Relay peers forward entity envelopes without reading content. For `system/encrypted` entities:
- Outer type (`system/encrypted`) is plaintext — routing machinery sees it.
- `recipient_key` (peer / group mode) is plaintext — usable as a routing hint into the recipient's inbox.
- `ciphertext` is opaque to the relay.
- Capability over the relay path is independent of the encryption (the cap protects access to forwarding; the encryption protects comprehension of content).

### §13.3 Inbox (`EXTENSION-INBOX` v5.9)

An encrypted entity arriving at an inbox is matched by `recipient_key` against the receiver's encryption-cert store. The decryption handler unwraps and re-injects the plaintext inner entity into the receiving handler chain. The inner entity's type determines its onward routing.

Inbox does NOT need to know the encryption format; it just delivers the wrapper. The decryption handler is the bridge.

**No replay protection beyond content-hash dedup (F2-5).** Encryption provides confidentiality + integrity + sender-auth (§7.4); it does NOT provide replay or freshness protection. An attacker who captures an encrypted entity can re-deliver it; it decrypts identically. The only built-in dedup is content-hash equality (a byte-identical re-delivery dedups in the content store). A receiving handler whose semantics are sensitive to replay (e.g. "apply this command once") MUST be idempotent per inner-entity `content_hash`, or carry an out-of-band sequence/nonce inside the inner entity. This is a deliberate scope boundary: per-message freshness/ordering is a session-extension concern (§20), not a stateless single-shot one.

### §13.4 Tree (`EXTENSION-TREE`)

Encrypted entities live at normal paths. A path like `local/files/secret.txt` could hold a `system/encrypted` instead of a `file`. The tree stores by hash; it doesn't care.

A handler reading that path could:
- Decrypt (if key available) and return the inner entity.
- Return the encrypted entity as-is (client decrypts).
- Return an error (no key access).

View extensions can provide transparent decryption — the view is configured with a key reference, and reads through the view auto-decrypt. Raw tree has ciphertext; view presents plaintext.

### §13.5 Content store

Encrypted chunks dedup on ciphertext. Same plaintext + same key + same chunking → same ciphertext → dedup. Different keys → no dedup, which is correct (different keys are different conceptual ciphertexts).

For backup: CDC the plaintext, encrypt each chunk, store encrypted chunks. The blob manifest references encrypted chunk hashes. Restore: fetch chunks, decrypt each, reassemble.

### §13.6 Capability (`EXTENSION-CAPABILITY` / V7 §5)

Capabilities control access; encryption controls comprehension. Two layers of defense:
- A peer with `get` capability on a path can fetch the encrypted entity.
- Only a peer with the decryption key can read the plaintext.
- A peer without capability can't get the ciphertext at all.

The cap-axis retirement (`cap?: public | private` removed) makes encryption the only privacy mechanism in the release shape — `public = standing public grant filtered at publish`; `private = encrypted at publish`.

---

## §14 Privacy properties per mode

Honest accounting of what each mode reveals to a passive observer of the wrapper:

| Field | Self | Peer | Group |
|---|---|---|---|
| Outer type (`system/encrypted`) | plaintext | plaintext | plaintext |
| `mode` discriminator | plaintext | plaintext | plaintext |
| Cipher suite (`enc_key_type`, `aead_id`, `kdf_id`) | plaintext | plaintext | plaintext |
| `recipient_key` (single) | absent | **plaintext** | absent (per-member) |
| Per-member `recipient_key` (group) | absent | absent | **plaintext, all members observable** |
| `ephemeral_key` | absent | plaintext | plaintext (per-member) |
| `kdf_salt`, `kdf_params`, `key_id` (self) | **plaintext** | absent | absent |
| Nonce | plaintext | plaintext | plaintext |
| Sender signature (signed mode) | absent | **plaintext, signer identity observable** | **plaintext** |
| Ciphertext length | plaintext (no padding in v1) | plaintext | plaintext |
| Inner entity type | private | private | private |
| Inner entity content | private | private | private |

**Implications:**
- Group mode reveals the full member set on the wire. This is acceptable for trusted-relay deployments; for hostile-relay deployments, prefer peer mode per-member.
- Signed peer mode reveals sender identity to anyone holding the entity. For sender-anonymous use cases, await `mode: "peer-anonymous"` (v1.5+).
- Ciphertext length leaks plaintext length. Padding is a v1.5+ feature.

---

## §15 Error codes (encryption's own domain)

Per V7 §3.3, extensions own their error-code domain. This extension defines:

| Status | Code | When |
|---|---|---|
| 400 | `encryption_aead_failed` | AEAD tag verification failed (wrong key, tampered ciphertext, tampered AAD) |
| 400 | `encryption_unsupported_suite` | Cipher suite combination forbidden in v1 (e.g., AES-GCM in self/group) |
| 400 | `encryption_no_common_suite` | Sender and recipient cert have empty intersection of supported suites |
| 400 | `encryption_kdf_params_excessive` | KDF parameters (e.g., Argon2id memory) exceed impl ceiling |
| 400 | `encryption_invalid_wrapper` | Outer entity malformed |
| 403 | `encryption_recipient_unknown` | No matching local encryption-cert for `recipient_key` |
| 403 | `encryption_key_unavailable` | Self-mode key_id not registered locally |
| 403 | `encryption_key_revoked` | Encryption-cert has a live revocation attestation |
| 403 | `encryption_signature_invalid` | Sender signature verification failed |
| 403 | `encryption_unsigned_sender` | Signed-by-default required in v1; invariant-pointer signature absent at `system/signature/{hex(content_hash)}` |
| 413 | `encryption_wrapped_keys_too_many` | Group mode exceeds wrapped_keys ceiling |

Inherits V7 §4.10 status codes verbatim for resource-bound exceedance (`payload_too_large`, `chain_depth_exceeded`).

---

## §16 Conformance vectors

The keystone "vectors are the contract" discipline (v7.65 – v7.75) applies. Ten gating vectors for v1.0 ratification, each with **fully-pinned byte inputs** so all impls produce identical bytes from identical inputs (F-GO-2 / F-GO-8 / F-GO-9 absorbed; F2-1 group-commitment added v2.4).

### §16.1 Vector index

| Vector | What it tests | Mode | v1 floor? |
|---|---|---|---|
| `ENC-SELF-KAT-1` | Self-mode KAT under default suite. Byte-equal ciphertext + reference AAD hex | self | **floor** |
| `ENC-PEER-KAT-1` | Peer-mode KAT under default suite. Byte-equal ciphertext + reference AAD hex | peer | **floor** |
| `ENC-PEER-KAT-2` | Peer-mode KAT hybrid suite `enc_key_type=0x04`. Validate slot, no impl required v1 | peer | validate-only |
| `ENC-GROUP-KAT-1` | Group-mode KAT: 3 members, fixed seeds, fixed plaintext → fixed outer ciphertext (commitment-bound, F2-1) + 3 fixed wraps + reference per-wrap AAD hex | group | best-effort |
| `ENC-GROUP-COMMIT-1` | Group key-commitment (F2-1): one outer `(ciphertext, nonce, AAD)`, two wraps to two members under *different* `group_aead_key` values, divergent-plaintext attempt MUST be rejected (second member's reconstructed-AAD AEAD.Open fails `encryption_aead_failed`) | group | **floor for group mode** |
| `ENC-AAD-1` | AAD verification: tampering with each key in §5.2's per-mode AAD set MUST fail decryption with `encryption_aead_failed` | all | **floor** |
| `ENC-CERT-LIFECYCLE-1` | Publish encryption-key → rotate → revoke → send to revoked → MUST `encryption_key_revoked`. One vector per tier (A/B/C); MUST pass at the tier-set the peer claims | all | **floor at peer's claimed tier** |
| `ENC-TIER-INTEROP-1` | Tier-A sender encrypts to Tier-C recipient (and vice versa); peer-mode round-trip succeeds via §4.4 resolution order. Validates F-GO-1 uniform pubkey-hash binding | peer | **floor** |
| `ENC-ROUNDTRIP-FORMAT-1` | Cross-format identity-reference: encrypt on SHA-256 peer, decrypt on SHA-384 peer; verify §1.8 / v7.69 §4.5a discipline | peer | floor when SHA-384 active |
| `ENC-RESOURCE-BOUNDS-1` | Oversized `wrapped_keys` → `encryption_wrapped_keys_too_many`; oversized ciphertext → V7 §4.10 `payload_too_large` | group | **floor** |
| `ENC-KEY-SEPARATION-1` | Key separation (R6): published encryption-pubkey is NOT the identity key and NOT a deterministic transform of it (birational Ed25519→X25519 forbidden); encryption keypair generated independently | all | **floor (BLOCK-1, against real keygen)** |

The KAT vectors (`ENC-*-KAT-*`, `ENC-AAD-1`, `ENC-GROUP-COMMIT-1`) are the **BLOCK-0** primitive-byte gate — locked 3-way (self + peer byte-equal; group AAD + commitment byte-equal; group ciphertext/wraps lock on Python's re-run with R1+R3 pinned). The lifecycle/interop/separation vectors (`ENC-CERT-LIFECYCLE-1`, `ENC-TIER-INTEROP-1`, `ENC-KEY-SEPARATION-1`) + the §3 BLOCK-1 scenarios are the **BLOCK-1** end-to-end gate that actually closes v1.0 — byte-equality of the primitive is necessary, not sufficient, for a privacy primitive.

### §16.2 ENC-SELF-KAT-1 — pinned inputs

```
mode             = "self"
enc_key_type     = 0x00
aead_id          = 0x01                                      ; XChaCha20-Poly1305
kdf_id           = 0x01                                      ; HKDF-SHA-256
nonce            = 24 bytes of 0x42                          ; pinned (F-GO-8): all 0x42 for reproducibility
kdf_salt         = 16 bytes of 0x43                          ; pinned
passphrase       = utf8("entity-core/test/self-kat-1")       ; UTF-8, no trailing NUL (F-GO-9)
key_id           = "test-key-1"                              ; UTF-8
kdf_params       = {
                     argon2_version: 0x13,                   ; v1.3 / v19 (F-GO-9)
                     memory_cost:    65536,                   ; 64 MiB (KiB units)
                     time_cost:      3,
                     parallelism:    1,
                     output_len:     32
                   }
plaintext        = ECF(ENC-KAT-INNER)                        ; R3: a REAL inner entity's ECF bytes, NOT a bare string. Canonical fixed entity below.

expected_AAD_hex = <LOCKED — cohort transcribes>             ; self-mode 8-key ECF; 3-way byte-equal achieved (197B), cohort writes the hex here
expected_ciphertext_hex = <LOCKED — cohort transcribes>      ; the AEAD output (ciphertext || tag); 3-way byte-equal (27B) — re-derives against ENC-KAT-INNER per R3
```

**Canonical KAT inner entity `ENC-KAT-INNER` (R3, arch-pinned).** The plaintext for all three KATs is the ECF of one fixed real entity — not a raw UTF-8 string. This exercises the real path (decrypt → typed entity → re-inject into the handler chain, §13.3) and pins the true ciphertext length. Arch authors + holds the reference ECF; cohort re-derives `expected_ciphertext_hex` against it.
```
ENC-KAT-INNER := system/note {
  body:    "entity-core encryption KAT inner entity"   ; primitive/string, UTF-8, no trailing NUL
  created: 0                                            ; primitive/uint, fixed
}
; ECF per ENTITY-CBOR-ENCODING (RFC 8949 §4.2 length-first). Arch pins the exact ECF hex + content_hash
; alongside the SEEDS file in the lock cycle. The SAME ENC-KAT-INNER is the plaintext for self/peer/group
; (the `"hello <mode>"` placeholders in v2.4 are superseded — R2/R3).
```

Go's v2.3 prototype derived a 6-key reference AAD hex (below). **v2.4 supersedes this**: the self-mode AAD is now 8-key (F2-4 added `kdf_salt` + `kdf_params`), so this hex must be re-derived during the joint byte-pin authoring. Length-first key order for the 8-key set: `mode`(4) < `nonce`(5) < `kdf_id`(6) < `aead_id`(7) < `kdf_salt`(8) < `kdf_params`(10) < `enc_key_type`(12) < `recipient_key`(13).
```
; v2.3 6-key shape — SUPERSEDED by the 8-key set above; retained only to show the ECF encoding style:
a6 646d6f6465 6473656c66  656e6f6e6365 5818 <24 bytes 0x42>  666b64665f6964 01  6761656164 5f6964 01  6c656e635f6b65795f74797065 00  6d726563697069656e745f6b6579 40
```
Arch + Go to jointly author + byte-pin the 8-key hex in the v2.4 → v1.0 ratification cycle (same `v767/SEEDS.md` pattern). Once expected_AAD_hex + expected_ciphertext_hex are byte-equal across Go + Rust + Python, the KAT locks.

### §16.3 ENC-PEER-KAT-1 — pinned inputs

```
mode                  = "peer"
enc_key_type          = 0x01                                  ; X25519
aead_id               = 0x01
kdf_id                = 0x01
nonce                 = 24 bytes of 0x44                      ; pinned
recipient_seed        = 32 bytes of 0x45                      ; pinned X25519 private key seed for recipient
sender_eph_seed       = 32 bytes of 0x46                      ; pinned X25519 ephemeral private key seed for sender
plaintext             = ECF(ENC-KAT-INNER)                    ; R3: the same canonical inner entity (§16.2), not a bare string

derived recipient_pubkey      = X25519.derive(recipient_seed)          ; receiver-side, 32 bytes
derived ephemeral_pub         = X25519.derive(sender_eph_seed)         ; sender-side, 32 bytes
derived recipient_pubkey_hash = content_hash(system/encryption-pubkey{
                                  enc_key_type:0x01,
                                  public_key: <recipient_pubkey>,
                                  supported_aead_ids: [0x01],
                                  supported_kdf_ids:  [0x01],
                                  created: 0,
                                  expires: absent
                                })                                      ; the recipient_key value used everywhere (F-GO-1)

recipient_pubkey_hash  = full 33-byte content_hash (1-byte multihash format prefix 0x00 + 32-byte digest), NOT bare digest (R5 / F-PY-ENC-2)
expected_AAD_hex       = <LOCKED — cohort transcribes>                  ; peer-mode 7-key ECF; 3-way byte-equal (171B)
expected_ciphertext_hex = <LOCKED — cohort transcribes>                ; 3-way byte-equal (31B) — re-derives against ENC-KAT-INNER per R3
```

### §16.4 ENC-GROUP-KAT-1 — pinned inputs

3 members, each with pinned X25519 seed (`0x50` / `0x51` / `0x52`). Outer nonce = 24 bytes of `0x53`; **per-wrap ephemeral seeds = `0x70+i`** (R1 — pinned on Go+Rust independent 2-of-3 convergence; Python aligns from `0x60+i`); per-wrap nonces = 24 bytes of `0x60+i` (distinct from the ephemeral seeds). Plaintext = `ECF(ENC-KAT-INNER)` (R3, the same canonical inner entity). Outer AAD = §5.2 group-outer **7-key** shape (with `commitment = SHA-256(group_aead_key)` bound, F2-1); per-wrap AAD = §5.2 group-per-wrap 7-key shape (`mode:"group-wrap"` per F2-2, each member's `recipient_pubkey_hash` per F-GO-1). The KAT pins `group_aead_key` itself (32 bytes of `0x54`) so `commitment` is reproducible — 3-way byte-equal on outer AAD (135B) + commitment already achieved. With R1 + R3 pinned, the outer-ciphertext + wraps lock on Python's re-run; cohort transcribes the `expected_*` hex.

### §16.5 Authoring + lock protocol

1. Arch lands this spec with TBD `expected_*` placeholders (done).
2. Go produces reference bytes from prototype against §5.2 pinned shapes.
3. Arch byte-verifies independently against this spec text (re-deriving ECF bytes from §5.2 first principles).
4. Rust + Python produce their own reference bytes; cohort compares.
5. If 3-way byte-equal → lock. If divergence → byte-equal failure is the v2.3 → v2.4 absorption signal (someone got the AAD shape wrong; cohort identifies which key disagrees; arch arbitrates).
6. Same shape as `v767/SEEDS.md` Phase-2 authoring (the cross-impl byte-fixture cycle).

Cross-impl 3-way + Keystone-generated peer green on every floor vector is the v1.0 lock signal. Same shape as v7.74 cohort close.

---

## §17 Design decisions — resolved, with cohort confirmation points

Seven design decisions. All are decided (arch position stated + Go-confirmed from the Go seat in v2.3); none blocks implementation. The items below carry a **cohort confirmation point** for Rust + Python + Keystone to validate during the first impl round — if a seat dissents, the finding folds in place (no re-ratification). Two items (4, 5) want a specific seat's sign-off as noted.

1. **Architectural call — one extension or two?** Arch position is **two sibling extensions**: this extension (`ENCRYPTION`) covers per-entity stateless encryption + shared substrate (algorithm registries, encryption-cert publishing, AAD discipline, at-rest key storage); the sibling `ENCRYPTED-SESSION` covers stateful interactive (handshake, ratchet, MLS group). The boundary is structural: stateless single-shot vs stateful interactive — same shape as the IDENTITY-over-ATTESTATION+QUORUM split (consumer over substrate). §20 lays out the split + shared-substrate boundary. **Go confirms.** Rust + Python + Keystone confirm.

2. **Tier model — does ENCRYPTION require IDENTITY?** Arch position is **NO**: ENCRYPTION runs at three tiers (§4.0). Tier A is V7-only floor (single-peer single-key; encryption-owned `handoff` + `revocation` entity types; V7 keypair as trust anchor). Tier B adds ATTESTATION for substrate-graph observability. Tier C adds IDENTITY for multi-device cert-chain. **The encryption / decryption flow is identical across tiers; only the publishing / rotation / revocation discipline varies.** Tier A is genuinely useful for headless single-purpose peers (CLI tools, storage daemons, CDN publishers). Cross-tier interop is normative (Tier-A sender ↔ Tier-C recipient resolves via §4.4). **Go confirms Tier A is feasible in Go on V7 alone, no IDENTITY transitive pull, prototype-validated.** Rust + Python confirm. Keystone confirm generated-peer support for Tier A is feasible without pulling IDENTITY transitively.

3. **AES-256-GCM in `peer` mode for v1, or wait until v1.5 with a deterministic-counter construction?** Arch lean: ALLOWED in peer (per-message ephemeral key avoids nonce reuse), FORBIDDEN in self / group (would risk nonce reuse with stable keys). This spec ships that posture. **Go confirms** (notes the per-message-ephemeral-key derivation makes the nonce hazard genuinely avoided). Rust + Python + Keystone confirm.

4. **Tier 2 key-backup entity placement.** Resolved per F-GO-7 absorption: backup path is **tier-relative** (Tier A/B → `system/encryption/key-backup/{pubkey_hash}`; Tier C → `system/identity/internal/key-backup/{cert_hash}`). Matches §4.3. Tier A self-contained without `system/identity` subtree. **Go confirms** tier-relative answer. IDENTITY-side signoff still wanted from a Tier-C deployment perspective (path placement under `system/identity/internal/` — confirm or rename).

5. **`mode: "peer-anonymous"` slot.** Reserved + design-sketched per §19.1. Confirm naming now or revise.

6. **Group-mode `wrapped_keys` default ceiling (256 in §8.6).** Right number? Real groups are usually < 50; 256 is generous; some deployments will want larger. Note this is **static-group** mode; dynamic-group with MLS is a sibling ENCRYPTED-SESSION concern (§20.2.c). **Go confirms** 256 is fine; trivially configurable; no opinion to force.

7. **Argon2id baseline parameters (m=64 MiB, t=3, p=1).** Conform to RFC 9106 baseline. Aggressive enough for typical desktop, may be heavy for low-end mobile. Should v1 ship a lighter "mobile profile" (m=32 MiB, t=2)? Lean: no — one baseline; impls configure up for high-security deployments; configure-down at deployment time is OK but `kdf_params` carries the actual values so portability is preserved. **Go confirms** single baseline suffices.

---

## §18 Scope and maturity profile

This extension owns a coherent design space. v1.0 ships a subset; the rest carries explicit design positions, not handwaves. Every item below is "ours to think about" — what differs is how fully spec'd / implemented this cycle.

| Item | Status v1.0 | Spec home |
|---|---|---|
| `mode: "self"` (storage encryption) | **IMPLEMENTED-V1.0** — normative spec + KAT vector | §6 |
| `mode: "peer"` (non-interactive single-shot) | **IMPLEMENTED-V1.0** — normative spec + KAT vector | §7 |
| `mode: "group"` (static key-wrap) | **IMPLEMENTED-V1.0 best-effort** — normative spec + KAT vector; ship if it lands cleanly | §8 |
| AEAD AAD binding | **IMPLEMENTED-V1.0** — normative | §5.2 |
| Algorithm byte registries (`enc_key_type`/`aead_id`/`kdf_id`) | **IMPLEMENTED-V1.0** — production + reserved + experimental + test allocations | §3 |
| Encryption-cert via `identity-cert` w/ `function="encryption"` | **IMPLEMENTED-V1.0** — substrate-aligned | §4 |
| Rotation via `identity-rotation-handoff` | **IMPLEMENTED-V1.0** — substrate-aligned | §10 |
| Revocation via universal `revocation` kind | **IMPLEMENTED-V1.0** — substrate-aligned | §11 |
| Sender authentication (signed by default) | **IMPLEMENTED-V1.0** — normative | §7.4 |
| At-rest key storage Tier 1 + Tier 2 | **IMPLEMENTED-V1.0** — normative two-tier defense in depth | §9.1 + §9.2 |
| Resource bounds | **IMPLEMENTED-V1.0** — inherits V7 §4.10 + extension-owned ceilings | §12 |
| Composition (frame-level / relay / inbox / tree / content / cap) | **IMPLEMENTED-V1.0** — composition rules | §13 |
| 11 conformance vectors (incl. `ENC-GROUP-COMMIT-1`, `ENC-KEY-SEPARATION-1`) | **BLOCK-0 KATs locked 3-way; BLOCK-1 lifecycle/interop/separation vectors gate v1.0 close** | §16 |
| `mode: "peer-anonymous"` (sealed sender) | **DESIGNED, NOT FULLY SPEC'D** — theory of shape; mode slot reserved | §19.1 |
| Padding / size hiding | **DESIGNED, NOT FULLY SPEC'D** — theory of shape; AAD-bound length-bucket scheme | §19.2 |
| Hybrid PQ peer mode (`enc_key_type = 0x04`) | **DESIGNED, NOT IMPLEMENTED** — byte allocated; KEM combiner shape sketched | §19.3 |
| Shamir N-of-K Tier 3 backup | **DESIGNED, NOT IMPLEMENTED** — full design sketch; impl is W2 (vault/custodians) workstream | §19.4 |
| Session encryption (Signal/Noise/MLS) | **SIBLING EXTENSION SCOPE** — `EXTENSION-ENCRYPTED-SESSION`, planned | §20 |
| Group-PFS / TreeKEM / MLS groups | **SIBLING EXTENSION SCOPE** — covered by ENCRYPTED-SESSION's MLS surface | §20.2.c |
| Searchable encryption | **OUT — different extension's domain** (query extension territory) | n/a |
| Transport-level frame encryption | **OUT — V7 §4.5 already exists** | V7 §4.5 |

The four DESIGNED-NOT-IMPLEMENTED items in §19 carry enough design pressure that a future impl team is following a sketch, not starting from scratch. The ENCRYPTED-SESSION sibling in §20 establishes the boundary so impls do not accidentally start encoding session-shape state into this extension's wrappers.

---

## §19 Theories of shape (in scope; designed, not fully spec'd this cycle)

Each subsection sketches a position concrete enough that an impl team can follow it, without locking normative pseudocode this cycle.

### §19.1 Sealed-sender mode (`mode: "peer-anonymous"`)

**Problem.** v1 peer mode reveals `recipient_key` in plaintext (necessary for relay routing) and exposes the sender identity via the invariant-pointer signature at `system/signature/{hex(content_hash)}` (necessary for recipient authentication). For high-privacy use cases (whistleblower, anonymous-tip, anti-correlation in adversarial-relay deployments) we want **both hidden from the relay**.

**Sketch.**
- **Recipient hiding.** Replace `recipient_key` with a delivery-token derived from the recipient's encryption-pubkey hash + a per-relay rotating salt (analogous to Signal's sealed-sender + the "stealth address" pattern). The relay matches the delivery-token to its known recipients without learning which pubkey it identifies.
- **Sender hiding.** Move the signature inside the ciphertext (encrypted to the recipient as part of the inner entity), so it's no longer at the public invariant-pointer slot. The recipient learns sender identity AFTER decryption, not before. Forward correlation by signer identity is impossible without recipient key.
- **Reply path.** Sealed-sender messages carry an encrypted return-path token; recipients reply by encrypting to that token rather than to the sender's public encryption-cert.
- **Mode value.** `mode: "peer-anonymous"`. Slot reserved in v1; sibling impl when first sealed-sender use case lands.

**What this requires we don't have yet:** delivery-token derivation protocol (one round; specifies how relays index recipients without seeing identity); rotating per-relay salt distribution; encrypted-return-path entity shape. Sketch enough that the next impl team picks it up from a position; full spec is one focused arch+keystone round.

### §19.2 Padding / size hiding

**Problem.** v1 ciphertext length leaks plaintext length. For storage of small structured entities (e.g. boolean flags, short strings), the ciphertext distribution reveals plaintext distribution.

**Sketch.**
- **Length-bucket padding.** Plaintext padded to the next bucket in a fixed bucket sequence — `{32B, 128B, 512B, 4KiB, 64KiB, 1MiB, ...}` — with PKCS#7-style trailing length byte.
- **Bucket choice as part of AAD.** The chosen bucket boundary is bound to AAD so an attacker cannot truncate-pad-extension attacks.
- **Padding is per-entity opt-in.** A new field `padded_to: primitive/uint, optional` declares the bucket; absent = no padding. Impl-default for new self / peer encryptions is on; legacy interop accepts un-padded.
- **Tradeoff explicit.** Padding costs storage; deployments that don't care about size-correlation can disable.

**What this requires we don't have yet:** bucket-sequence normative table; AAD extension to bind padded_to; one new conformance vector showing padded vs un-padded entries decrypt identically; impact-on-content-store-dedup analysis (padding by content_hash bucket changes dedup characteristics — likely better than naive padding, but needs measurement).

### §19.3 Hybrid PQ peer mode (`enc_key_type = 0x04`)

**Problem.** Pure X25519 is broken by a sufficiently large quantum computer (Shor). Pure ML-KEM-768 has no fielded track record at internet scale. Hybrid = both, with security floor at max of the two.

**Sketch.**
- **Composite public key.** `public_key = X25519_pubkey || ML-KEM-768_pubkey` (32 + 1184 = 1216 bytes). Encoded as a single `primitive/bytes`; structural split by fixed offset.
- **Composite encapsulation.** Sender runs both: `ss_classical = X25519(eph_priv_x25519, recipient_x25519_pub); (ss_pq, ct_pq) = ML-KEM-768.Encaps(recipient_mlkem_pub)`. Composite shared_secret = `KDF(ss_classical || ss_pq)` per IETF draft `hpke-pq` combiner.
- **Composite `ephemeral_key`.** `ephemeral_key = eph_pub_x25519 || ct_pq` (32 + 1088 = 1120 bytes).
- **Forward compatibility.** When ML-KEM matures and X25519 becomes obsolete, future deployments can move to `enc_key_type = 0x03` pure-PQ. Hybrid is the transition cipher.

**What this requires we don't have yet:** pinned KEM-combiner construction (HKDF-Extract over `ss_classical || ss_pq` per IETF draft; need to follow draft finalization); cohort cryptographic-library readiness (CIRCL has ML-KEM; pqcrypto has it; Python's `cryptography` lib likely landing in 2026); the `ENC-PEER-KAT-2` vector at §16 captures the test target.

### §19.4 Shamir N-of-K secret-sharing (Tier 3 backup)

**Problem.** Tier 1 + Tier 2 protect against device loss + corruption + remote attack, but a single-passphrase Tier 2 has a single human point of failure (forgotten passphrase = all encrypted data lost; coerced passphrase = compromise). Shamir N-of-K distributes trust across N share-holders such that any K can reconstruct, fewer than K reveal nothing.

**Use cases (the user's framing — "Secret Sharing Sharon is awesome"):**
- Personal: 5 shares to 5 trusted custodians (spouse, parent, two friends, safe deposit box); 3-of-5 reconstruct. Survives any 2 share losses + resists any 2 colluding custodians.
- Organizational: 7 shares to 7 board members; 4-of-7 reconstruct. Embeds governance into key recovery.
- Hybrid: 4 shares = (your phone, your laptop, your safety-deposit, your lawyer); 2-of-4. Survives device loss + lawyer is your dead-man's-switch.

**Entity shape:**

```
system/identity/key-share := {
  fields: {
    cert_ref:        {type_ref: "system/hash"}        ; encryption-cert this is a share of
    scheme:          {type_ref: "primitive/string"}   ; "shamir-gf256" (default) | "shamir-gf2-128"
    n:               {type_ref: "primitive/uint"}     ; total shares issued
    k:               {type_ref: "primitive/uint"}     ; threshold
    share_index:     {type_ref: "primitive/uint"}     ; 1..N
    share_bytes:     {type_ref: "primitive/bytes"}    ; this share's payload
    custodian_ref:   {type_ref: "system/hash"}        ; cert of the custodian holding this share
    custodian_hint:  {type_ref: "primitive/string", optional: true}  ; human label: "phone-2", "spouse", "safe-deposit"
    epoch:           {type_ref: "primitive/uint"}     ; for proactive refresh — see below
    created:         {type_ref: "primitive/uint"}     ; ms since epoch
  }
}
```

Path: `/{peer_id}/system/identity/internal/key-share/{share_id}` — local record that this share exists. The actual share-bytes are NOT stored at the originating peer; they live at the custodian's identity-tree.

**Issuance flow.**
1. The secret-owner generates `(share_1, ..., share_n)` from the encryption private key via Shamir over GF(2^8) (Shamir-GF256, default — interoperable; small share size).
2. For each share `i`, the owner encrypts share_i to custodian `i` via **peer-mode encryption** (full circle — peer mode delivers the share). Custodian receives `system/encrypted [mode="peer"]` wrapping `system/identity/key-share` at index i.
3. Custodian stores the decrypted share at `system/identity/internal/key-share/{share_id}` in their own tree.
4. Owner retains a `system/identity/internal/key-share-register` ledger of (custodian_ref, share_index) pairs for later reconstruction-coordination, NOT the share material itself.

**Reconstruction flow.**
1. Recovery-trigger event: owner has lost Tier 1 + Tier 2 access (lost device, forgotten passphrase) but retains their identity-cert chain (so custodians can verify they are talking to the legitimate identity).
2. Owner contacts custodians (out-of-band human channel + entity-system message). Each willing custodian re-encrypts their share via peer-mode back to the owner's recovery encryption-cert (which may be a freshly-issued cert under `identity-rotation-recovery` per `EXTENSION-IDENTITY` §4.4).
3. Owner collects ≥ K shares, runs Lagrange interpolation, recovers the encryption private key.
4. Owner re-publishes (and re-issues fresh shares for the next cycle).

**Refresh (proactive secret sharing).**
1. The `epoch` field allows refresh-without-secret-change: owner generates a fresh set of shares for the SAME underlying secret at a new epoch.
2. Custodians replace their old share with the new one.
3. Old shares (from earlier epochs) are invalid after K custodians acknowledge new-epoch. Defeats attackers who slowly compromise shares over time (e.g., 2 shares this year, 1 share next year — without refresh, attacker accumulates toward K; with refresh, they reset to 0 each epoch).
4. Refresh cadence is policy: monthly, annually, on-demand.

**Custody model.**
- `custodian_ref` is a regular `identity-cert`; the share is encrypted to that identity's encryption-cert. The custodian's authority over the share is exactly the authority over their own encryption-cert (under their own controller's quorum). No special "custodian role" needed.
- `custodian_hint` is human-readable label for out-of-band coordination ("my phone", "spouse", "lawyer Jim", "safety deposit 12"). The hint is private (lives in the owner's tree only); custodians do not learn each other's labels.

**Composition with Tier 1/2.**
- Tier 1 (hot keychain) = day-to-day access.
- Tier 2 (passphrase-wrapped) = independent backup for the user themselves.
- Tier 3 (Shamir) = recovery against passphrase loss / coercion + organizational governance.
- All three coexist; user chooses which combinations to deploy.

**What's deferred:**
- Concrete Shamir-GF256 byte format (need a specific implementation — `shamirs-secret-sharing` library family across cohort, cross-impl byte vector).
- Verifiable secret sharing (Feldman / Pedersen VSS — proves shares are correctly generated; defeats malicious dealer). v1.5 work — the user is dealing themselves, so dealer-trust is moot in personal use; matters more for organizational.
- Threshold ECDSA / threshold Ed25519 for splitting the identity signing key (different problem than splitting the encryption key — signing has multiplicative properties that simple Shamir doesn't preserve). Out of this extension's scope; lives in a future `EXTENSION-THRESHOLD-SIGNING` if we go there.

**Why this section is fully designed but not v1.0-implemented:** the design is sound and well-grounded in 50 years of literature (Shamir 1979, proactive secret sharing 1991, VSS 1985); we can specify it; the cost is engineering time (custodian flow + reconstruction-coordination + cross-impl byte format) which we don't have this cycle. The W2 vault/custodians workstream picks it up; this extension's §19.4 IS the design doc that workstream consumes.

---

## §20 Sibling extension: `EXTENSION-ENCRYPTED-SESSION` (planned)

### §20.1 Architectural ruling

Session encryption (Signal double ratchet, Noise XK, MLS) is a **sibling extension**, not part of this one. The boundary:

| Concern | This extension (`ENCRYPTION`) | Sibling (`ENCRYPTED-SESSION`) |
|---|---|---|
| State | Stateless (per-entity wrapper, no channel) | Stateful (channel state, ratchet chain, MLS group state) |
| Interaction shape | Single-shot encrypt + send | Handshake → ongoing exchange (Noise XK / X3DH / MLS welcome) |
| Wire shape | One `system/encrypted` wrapper | Handshake messages + tick messages + state-update messages |
| Forward secrecy | None (recipient-key compromise reveals all past) | Yes (ratchet provides session-PFS + post-compromise security) |
| Group | Static key-wrap, ≤ 256 members | MLS / TreeKEM; O(log N) member ops; dynamic groups |
| Conformance vector set | 8 per-entity KATs | Separate session-handshake KATs |

Same shape as the IDENTITY-over-ATTESTATION+QUORUM split: this extension is the **substrate**; ENCRYPTED-SESSION is the **consumer**.

### §20.2 Shared substrate (what ENCRYPTION provides to ENCRYPTED-SESSION)

ENCRYPTED-SESSION builds on this extension's substrate; it does NOT reinvent:

- **Algorithm byte registries (§3).** Session AEADs use the same `aead_id` registry. Session KDFs use the same `kdf_id` registry. Session key-exchange algorithms use the same `enc_key_type` registry.
- **Encryption-cert publishing model (§4).** Session-capable peers advertise session-protocol support via the SAME `identity-cert` with `function="encryption"`. The cert's `supported_aead_ids` extends to include session-mode AEADs; session protocol version is an additional optional field on the inner pubkey entity.
- **AAD discipline (§5.2).** Session messages reuse the canonical-CBOR AAD pattern (with session-specific fields added).
- **Sender authentication (§7.4).** Session messages are authenticated by chained MAC keys derived from session keys; the initial handshake reuses the identity-cert signing model from this extension.
- **At-rest key storage (§9).** Session state (ratchet chains, MLS group state) is stored under the same Tier 1 + Tier 2 model. Long-lived ratchet chains can be Tier 2-backed up; ephemeral per-message chain keys are never persisted.
- **Revocation (§11).** Session-capable cert revocation tears down all sessions established under that cert.
- **Composition (§13).** Session messages flow through the same relay / inbox infrastructure; relays see session-message envelopes the same way they see `system/encrypted` envelopes.

### §20.3 What ENCRYPTED-SESSION owns (out of THIS extension's scope)

- **Session establishment.** X3DH / Noise XK / MLS welcome — handshake to derive shared state.
- **Ratchet state machine.** Symmetric ratchet (per-message key derivation), DH ratchet (forward secrecy across DH ratchet steps), post-compromise recovery.
- **Out-of-order / missed-message handling.** Skipped-key cache, replay window, lost-message recovery.
- **Group-PFS via MLS.** TreeKEM construction, `Update` / `Add` / `Remove` proposals, commit messages, application messages, the full MLS RFC 9420 wire format and state machine.
- **Session entity types.** `system/encrypted-session/handshake-init`, `system/encrypted-session/handshake-response`, `system/encrypted-session/tick`, `system/encrypted-session/mls-group-state`, etc.
- **Session error code domain.** Distinct from this extension's `encryption_*` codes.

### §20.4 Candidate architecture (informative, for ENCRYPTED-SESSION authors)

Current literature offers three flagship candidates:

- **Noise framework** (Trevor Perrin). Well-studied protocol family; Wireguard pedigree. Mutual auth via static keys + ephemeral PFS. Stateless from the post-handshake perspective (no out-of-order recovery built in; layer it). Best fit for *two-party live sessions*.
- **Signal protocol (X3DH + Double Ratchet).** Current chat-app standard. Asynchronous (works with prekey bundles for offline send); PFS + post-compromise security; 1:1 native; group via per-pair sessions or sender-keys.
- **MLS (RFC 9420).** IETF standard. Group-first design with TreeKEM (O(log N) member ops); group-PFS by construction; 1:1 is degenerate group-of-two. Most flexible; highest complexity.

Arch lean for ENCRYPTED-SESSION: **MLS as the unifying construct** (handles 1:1 + group + future federation under one mechanism); Signal-style is the closest competitor but its 1:1-first design forces awkward group constructions. Final call belongs to ENCRYPTED-SESSION's own pre-impl review when it opens.

### §20.5 When ENCRYPTED-SESSION opens

Sequencing call: ENCRYPTED-SESSION proposal opens AFTER this extension lands v1.0 (substrate must exist for sibling to consume). Likely cycle: v1.0 ENCRYPTION ratifies + cohort impl → ENCRYPTED-SESSION proposal first-pass → cohort + Keystone review → ENCRYPTED-SESSION v1.0 ratify. Estimate: 2-3 weeks after ENCRYPTION v1.0 close, depending on the MLS-vs-Signal-vs-Noise architectural call.

The W2 workstream entry for ENCRYPTED-SESSION will be added concurrently with this extension's v1.0 landing.

---

## §21 Conformance & landing status

This spec is **landed**. Implementation is dispatched to the cohort; what remains is the close-out gate, not more design review.

1. **Cohort impl pass** — Go / Rust / Python implement `self` + `peer` (PRIMARY) against this spec; `group` best-effort. Keystone vendors + generates a v1.0 peer at Tier A.
2. **Byte-pin lock (§16.5)** — Rust + Python author the §16 `expected_*` reference bytes; 3-way byte-equal with Go + arch's independent re-derivation. Divergence is an absorption signal (a §5.2 AAD key disagrees), folded in place.
3. **Close-out signal** = cross-impl 3-way + Keystone-green on every §16 floor vector. That completes v1.0.
4. **Deltas fold in place** — any cohort finding is absorbed into this spec directly, no new ratification cycle (REGISTRY/DISCOVERY/RELAY discipline). Surface, don't self-fold.

**Release scoping:** v1.x, paired with the Tier-2 dispatch-fallback seam — off the v1 release critical path. **Independent of RELAY** (RELAY transports opaque envelopes; ENCRYPTION operates on entity contents; no dependency between the two — the encrypted-inner-opaque-to-relay path is the composition, §13.2). **Sibling sequencing:** `EXTENSION-ENCRYPTED-SESSION` (§20) opens only after this extension's v1.0 closes.

---

*Landed from `proposals/implemented/PROPOSAL-EXTENSION-ENCRYPTION.md`. The original conceptual core is preserved verbatim in §2 + §13; everything around it is the substrate-aligned form. Provenance arc: the source proposal + the pre-impl analysis/review + the V2.3 second-pass adversarial review + the substrate spec versions cited in `Depends:`.*
