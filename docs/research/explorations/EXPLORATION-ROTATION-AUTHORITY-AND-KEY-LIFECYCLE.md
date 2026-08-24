# EXPLORATION — rotation authority, key lifecycle, and where they belong

**Status:** Exploration (workspace). Consolidates the identity-rotation / key-lifecycle thread
into one analysis of record and audits the core protocol against it.
**Date:** 2026-08-12.
**Pins (all read live, not summary-read):** `entity-system-architecture` `bf88be0` ·
`entity-core-protocol` `3042bd8` · `entity-browser-rust` `c183e6a`.
**Consolidates:** `DESIGN-IDENTITY-ROTATION-AND-THE-LANDSCAPE.md` (survey) ·
`PROPOSAL-IDENTITY-PRE-ROTATION.md` (14 decisions, 11 open) ·
`PROPOSAL-CORE-KEY-LIFECYCLE.md` rev 3 (boundary question) ·
`REVIEW-2026-08-11-identity-pre-rotation-first-pass.md` ·
`ROUTING-2026-08-11-c-three-of-your-open-items-were-already-closed.md`.

**The answer in one line.** `identity-rotation-handoff` drops the issuing authority of every cert
it rotates — verified by trace, §3 — and neither the fix nor anything else in this thread requires
a core protocol change.

---

## 1. Provenance — what is and is not recoverable

The thread began as a small question: does the landscape — AT Protocol in particular — have
something simpler than what we designed? **The specific field or property that prompted it is
lost.** It is not in this repo's reviews or routing packets, and it is not identifiable from the
meta-repo documents. Earlier drafts of this exploration named a candidate reconstructed from
ATProto's surface; that was inference presented as recovery and has been removed.

What *is* on the record:

- The **mechanism the thread adopted** is KERI's pre-rotation
  (`DESIGN-IDENTITY-ROTATION-AND-THE-LANDSCAPE.md` §4), substituted for did:plc's 72h recovery
  window — correctly, since that window is purely a function of a trusted total order and we have
  no directory to supply one.
- The landscape doc's §3 records two ATProto qualities it considered: the identifier being a hash
  of the genesis operation, and priority-ordered rotation keys.

**Nothing below depends on recovering the original ask.** §3's defect is established by tracing
the live spec, not by appeal to provenance. §2 records ATProto's surface as design precedent — not
as a claim about what was originally asked.

## 2. ATProto — full surface audit

Audited against the primary sources, not the survey's summary (`did:plc` spec v0.1;
`atproto.com/specs/{did,cryptography}`; the identity guide).

**Identifier layer**
- DID = `base32(sha256(genesis_op))[:24]` — a hash of the genesis operation, never a key.
- Handle (DNS/HTTP) → DID; a mutable pointer, bidirectionally verified via `alsoKnownAs`.
- Hash-chained operation log, `prev` → CIDv1 (dag-cbor, sha-256); 7500-byte per-op limit.

**Key roles — two disjoint sets, and this is the finding**
- `rotationKeys` — **1 to 5**, **priority-ordered** (lower index = higher authority), `did:key`
  serialization, `k256`/`p256` only. **These control the DID.**
- `verificationMethods` — up to **10** per DID, any syntactically valid `did:key` including
  ed25519; carries the `#atproto` signing key. The spec, verbatim: **"Cannot control the DID"** —
  used for service authentication only.

**Lifecycle operations**
- `plc_operation` — create / update.
- `plc_tombstone` — carries only `type`, `prev`, `sig`; *"clears all of the data fields and
  permanently deactivates the DID."*
- 72h nullification window — a lower-index rotation key forks the log at the last valid operation.
  **Applies to tombstones too.**

**Cryptographic qualities**
- Exactly two curves (`p256`, `k256`); compressed multikey, base58btc `z` prefix.
- **low-S required on both curves**; raw 64-byte `r‖s`, base64url, no DER.
- Strict DAG-CBOR/DRISL; verify via library routines, never raw byte comparison.

### 2.1 The precedent that matters for §3

**`verificationMethods` cannot control the DID.** The key that signs data every day has zero
authority over the identity; only `rotationKeys` can change the document, and they are a separate
list with different permitted key types and different limits.

That is why ATProto's revocation story is unremarkable: **replacing a compromised signing key is a
routine `plc_operation` signed by a rotation key.** No ceremony, no recovery path, no drama — the
stolen key never held identity authority.

Recorded as precedent for the fix in §4, **not** as a claim about the thread's origin (see §1).

**Secondary finding worth recording:** their `plc_tombstone` is *itself* nullifiable inside the
recovery window. So ATProto already answers the tombstone-DoS that `PROPOSAL-CORE-KEY-LIFECYCLE.md`
§6.3 flags as an unresolved cost — with the directory, which we cannot take. The problem is
pre-solved there rather than novel, which is useful to know when pricing R2/R3.

## 3. The defect, established by trace

### 3.1 The vocabulary, grounded

There is **one entity type**. `EXTENSION-ATTESTATION` §3.1 defines `system/attestation` with
`attesting` (`system/hash` — *"who's making the claim (peer hash; or quorum hash for K-of-N)"*),
`attested` (the subject), `properties` (open map), `supersedes`, `not_before`, `expires_at`.

- A **cert** is not a distinct type. It is that entity with `properties.kind = "identity-cert"`
  and a `properties.function`. It is an **edge**: *peer A claims peer B holds function F*.
- **A peer is a key.** `ENTITY-CORE-PROTOCOL` §1.5: `peer_id = f(public_key)`. So `attesting` and
  `attested` are keys.
- **Rolling a cert therefore means moving a function from one key to another.**

### 3.2 Issuance authority vs rotation authority

`identity_topology_for` (§3.6) dispatches on `(kind, function, is_quorum(attesting))`. Collated
with §4.1's kind table:

| Cert / event | Authority to **issue** | Authority to **rotate** (handoff) |
|---|---|---|
| `identity-cert` controller, top-level | **K-of-N from the quorum** | dual-sig (old key + new key) |
| `identity-cert` controller, sub | issuing controller (single-sig) | dual-sig (old key + new key) |
| `identity-cert` agent | controller (3-key) / identifier (4-key) | dual-sig (old key + new key) |
| `identity-cert` identifier | issuing controller (single-sig) | dual-sig (old key + new key) |
| `identity-rotation-recovery` | — | K-of-N from the quorum |
| `identity-retirement` | — | K-of-N from the quorum |
| `revocation` of a cert | — | **quorum at the chain root** (`identity_is_authorized_revoker`) |

The handoff arm is **`("identity-rotation-handoff", _, _)` — a wildcard on function**, returning
`{mode: "dual", signers: [target.attested, att.attested]}`. It silently overrides all four
issuance arms above it.

**So: the authority required to *grant* a function is dropped at rotation time and replaced by
"whoever currently holds the key."** Recovery and retirement are quorum-gated; revocation is
quorum-gated; only handoff is not.

The severe case is the top-level controller cert: **K-of-N from the quorum to grant the controller
function, dual-sig to move it.**

### 3.3 The trace — why the chain walk does not catch it

`identity_verify_cert` step 5 walks `attesting` back to the quorum and re-verifies every link, so
it is fair to ask whether that closes the hole. It does not:

1. A thief holds stolen key `K_a`, which holds `cert(attesting=K_c, attested=K_a, function=agent)`.
2. The thief mints `handoff(attesting=K_a, attested=K_thief, target_cert=cert)` and signs with
   `K_a` and `K_thief`. Step 4's dual-sig requirement is satisfied.
3. Step 5 walks from `handoff.attesting = K_a` and finds `cert(K_c → K_a)` — genuinely valid,
   signed by the real controller, live, chain intact to the quorum.
4. Every link validates. **`identity_verify_cert` returns OK.**

The chain walk verifies **that the outgoing key legitimately held the function**. It never
verifies **that anyone authorized moving it**. The same trace runs for a controller key, where the
bypassed authority is the quorum.

Per §4.3's function-inheritance rule, no fresh `identity-cert` is required afterwards for the new
key to be chain-walkable — so the thief's key is a first-class holder of the function immediately.

## 4. The fix, and why it is two rules

The principle the audit yields: **rotation must not require less authority than issuance.**

**Rule 1 — authority parity.** A handoff carries the target cert's *issuance* topology, plus the
incoming key. This is a change to one wildcard arm of `identity_topology_for`, dispatching on the
target's function instead of ignoring it. No new primitive, no new entity, no core change.

**Rule 2 — the pre-committed exemption.** Rule 1 alone is too strong at the top: a top-level
controller cert is issued K-of-N, so parity collapses handoff into `identity-rotation-recovery`
and the graceful path for the controller key disappears. The exemption restores it — **a handoff
MAY substitute a redeemed pre-rotation commitment for the issuer's signature.** A thief cannot
produce a key matching the pre-committed digest, so self-service roll stays safe without a
ceremony.

This is what makes the two halves of the thread one design rather than two proposals:

| Target | Closed by | Why the other does not suffice |
|---|---|---|
| agent / sub-controller / identifier | **Rule 1** — the issuer co-signs | the issuer is a single online key; a commitment is unnecessary ceremony |
| top-level controller | **Rule 2** — pre-rotation | the issuer is the quorum; parity alone means a K-of-N ceremony per roll |

**The honest cost of Rule 1**: rolling an agent key now requires the controller to be reachable.
That is the same availability trade ATProto makes by holding `rotationKeys` apart from the PDS's
signing key, and it should be stated rather than discovered.

Open questions this raises, carried to the proposal: whether the outgoing key must still sign at
all under Rule 1 (it proves consent, but a compromised key's consent is worthless); and which
event's commitment is in force under Rule 2 — already `PROPOSAL-IDENTITY-PRE-ROTATION.md`
decision 12, and a correctness ruling rather than a preference.

## 5. Does any of this touch core? No.

Core is `ENTITY-CORE-PROTOCOL.md` **0.8.0**, protocol line **V8**, M6. It is unchanged by
everything in §4 and §7.

**Pre-rotation needs no core change — three independent confirmations:**

1. `EXTENSION-ATTESTATION` §3.2 — `properties` is a consumer-extensible map, *"the attestation
   primitive doesn't interpret keys"*; the type is `properties: {map_of: {type_ref:
   "primitive/any"}}`. A new key is additive by construction.
2. `EXTENSION-IDENTITY` §3.6 — the reject-on-unknown gate ranges over `valid_functions()` and
   `identity_lifecycle_kinds()`. It does **not** range over `properties` keys.
3. `EXTENSION-IDENTITY` §1, explicit: *"This extension does not define: **V7 chain verification
   changes — chains stay pure V7.**"*

**Rotation-authority separation needs no core change** for the same reason plus one more: the
signer-set rule lives entirely in identity's own `identity_topology_for` validator.

**Both pass the `§5.10` test.** The rotation predicate is evaluated in `identity_verify_cert` over
an attestation, never in `verify_request` over a capability chain — so it adds **no Layer-1
predicate**. Downstream capability effects are `§6.4`-style *writing* of Layer-1 state, which
`§5.10` permits and for which `:revoke_attestation` (PI-13) is the standing precedent.

The one core dependency — §4.7's `content_hash(system/peer)` construction — rests on `§4.5a`
**item 1a (v7.77)**, which is **already landed** (`entity-core-protocol` `fc54930`). The
`Depends:` raise is a line in the extension's own header, not an edit to core.

## 6. Core protocol audit

Read live at `entity-core-protocol` `3042bd8`. **Clean; nothing surprising.**

- **Shape.** 4,216 lines, ten sections (Foundations · Type System · Protocol Type Definitions ·
  Connection Establishment · Capability System · Handler Model · Algorithms · Constants ·
  Conformance · Related Specifications). Proportionate; no section has silently become a dumping
  ground.
- **No identity-lifecycle leakage.** `rotat*` = **1** hit (the §1.4 / §1.5a cross-reference to
  `EXTENSION-IDENTITY` as the layer providing "rotatable presence"); `successor` = **0**. Core
  makes no claim about key lifecycle, which is exactly the posture §5 depends on.
- **Layering direction is correct.** Core's four `EXTENSION-IDENTITY` references are one
  descriptive pointer (§1.5a IAM) and three occurrences inside `§5.10`, where core *constrains*
  extensions (naming `IdentityBindingChecker` as the canonical Layer 2 hook). Core constrains
  down; it does not depend up. Reference counts across the family (CONTINUATION 19, COMPUTE 14,
  TREE 13, SUBSCRIPTION 13, ROLE 10, IDENTITY 4) track how much wire surface each extension
  touches, not how much core relies on them.
- **`--profile core` is genuinely self-contained.** Sixteen categories; extension handlers are
  "matched-if-present, not-a-FAIL-if-absent"; extension-specific carve-outs in `security`/`authz`
  are named with diagnostics rather than silently skipped, and §9.1's floor applies under both
  profiles. The scoping discipline holds.
- **`§5.10` is the load-bearing boundary and it is stated once, generally** — Layer 1 / Layer 2,
  the extension contract, the convergent-input admission criterion with its tier clause and
  `revocation_propagation_bound`. This is the surface any future key-lifecycle work must satisfy,
  and it is in good shape.

### 6.1 One systemic finding — `Depends:` floors are not maintained

Every extension in the identity stack declares the **same** core floor, and it is stale:

| Extension | Declares |
|---|---|
| `EXTENSION-IDENTITY` | `ENTITY-CORE-PROTOCOL.md (v7.40+)` |
| `EXTENSION-ATTESTATION` | `ENTITY-CORE-PROTOCOL.md (v7.40+)` |
| `EXTENSION-QUORUM` | `ENTITY-CORE-PROTOCOL.md (v7.40+)` |
| `EXTENSION-REGISTRY` | `ENTITY-CORE-PROTOCOL.md (v7.40+)` |

Four extensions with very different core surface usage cannot all have the same true floor, and
the corpus is at **v7.77**. The extension→extension dependencies *are* maintained — REGISTRY
declares `EXTENSION-ATTESTATION.md (v1.3+)` **with a stated rationale** — so the discipline exists
and is simply not applied to the core dependency.

Two concrete under-declarations, both load-bearing:

- **REGISTRY's self-certifying binding pins to core `§1.5`** (*"the Base58-encoded peer-id per V7
  §1.5 multikey form (key_type ‖ hash_type ‖ digest, encoded)… This is the V7 §1.5 alignment
  pin"*). `§1.5` moved at **v7.64/v7.65** (the digest-IS-the-public-key form; `peer_id` exiting
  the hashable basis). REGISTRY declares v7.40+ — under-declared by roughly 25 amendments on its
  single most identity-critical pin.
- **IDENTITY §9.5's sync-latency-bounded convention** is cited *from core* in the v7.76
  `revocation_propagation_bound` text, so the two are entangled at v7.76 while IDENTITY declares
  v7.40+.

**This is hygiene with teeth**: a stale floor is exactly the kind of latent under-declaration that
becomes load-bearing the moment a new construction rests on it — which is what happened to §4.7,
where the fix was mis-scoped as "raise IDENTITY to v7.77" when the real defect is that the whole
stack's floors stopped being maintained. Fixing it is a per-extension audit of what each actually
uses, not a blanket bump.

## 7. The identifier question — and a conflict nobody caught

`did:plc`'s real advantage was never its rotation machinery; it is that **the identifier is a hash
of the genesis record, so keys rotate underneath a stable name.** Ours is the key
(`§1.5`: `peer_id = f(public_key)`), so no amount of rotation machinery gives us that property.

Core has already ruled where the fix belongs. `§1.5` (v7.69), verbatim: *"cross-form correlation
otherwise lives in the IDENTITY/REGISTRY/POLICY layers, **never the address or capability layer of
core**."* So this is a REGISTRY question by core's own ruling — **no core change.**

**The conflict.** `PROPOSAL-IDENTITY-PRE-ROTATION.md` §7 proposes registry names of the form
`base58(hash(genesis_attestation))`. But `EXTENSION-REGISTRY` has **already landed and implemented
a self-certifying binding form that is the opposite**: *"`target_peer_id` is an identity, not a
content-hash… Self-certifying naming uses this string directly (`name == target_peer_id`), **NOT**
`hex()` of a hash."* It is a landed trust-anchor variant with a cohort-convergence pin (§2.4.1),
`kind: "self-certifying"`. §7 is therefore not a greenfield addition — it contradicts a shipped
v1 pin.

**It is still cheap, because REGISTRY §3.0a is a forward-compat rule:** an unknown binding `kind`
**MUST be ignored with a warning**, and the binding remains valid on the wire. So the genesis-hash
form lands as a **new binding kind beside `self-certifying`**, never as a redefinition of it. Old
peers skip it; new peers resolve it. No core change, no wire break, no conflict with the landed
pin.

**This is the item with a real deadline**, because publishing freezes what consumers pin.

## 8. Key death in Layer 1 — parked, with the reason

`PROPOSAL-CORE-KEY-LIFECYCLE.md` asks a legitimate and different question: what does a
`--profile core` peer get without adopting the identity stack? Its spine is sound — `§5.10` makes
key death core-or-nothing, and R1 is best understood as `EXTENSION-IDENTITY` §6.4's
`:revoke_attestation` cascade generalized past its enumeration limit.

It is parked, for three reasons:

1. **It is the only component that would touch core**, and core is at the M6 floor.
2. **It costs a permanent tier flag.** `ctx.supports_revocation` is a single boolean and core has
   no general feature-negotiation surface; `§5.10` scopes the determinism MUST to *"peers of the
   same revocation-support tier."* Two peers both advertising `true`, one implementing a new
   marker family and one not, diverge on Layer 1 while claiming the same tier — the exact leak
   `§5.10` forbids. Every new Layer-1 revocation input therefore costs one flag, permanently, and
   the cost lands at first adoption.
3. **ATProto is evidence against putting it there.** Their relays and appviews resolve a DID; they
   do not run a key-death predicate inside request verification. Key death lives above the wire.

Two of its findings are worth keeping regardless of disposition: R1's `by-grantee/{peer_hash_hex}`
path is a **cross-issuer DoS** as written (`is_revoked` reads `ctx.entity_tree.get(marker_path)`
against the verifier's own synced tree with nothing binding the author), and the admissibility
criterion generalizes to a rule worth stating once — **a Layer-1 revocation input is admissible
iff its authority is determinable from the marker itself.** Cap-hash markers pass, issuer-scoped
grantee markers pass, bare grantee markers fail, and a self-signed retirement passes because the
signer *is* the subject.

## 9. Where we are

**Ruled (2026-08-11, `f2b0c63` / `5606254`):** decision #4 yes and scoped to three items ·
decision #10 withdrawn (the hazard does not exist — item 1a) · genesis/release timing no
(§4.4 stickiness is forward-sticky) · §4.7's "crypto-agility inherited" bullet is false, verdict
unchanged.

**Open:** the 11 remaining pre-rotation decisions, all extension-scoped.

**Newly established here:** the §3 authority-parity defect, by trace against the live spec. It is
independent of the 11 open decisions and is the sharper statement of what pre-rotation was
reaching for. Written up as `PROPOSAL-IDENTITY-ROTATION-AUTHORITY-PARITY.md`.

**On-disk gap:** the thread exists in this repo as one review and one routing packet. Neither
proposal has been pulled into `docs/proposals/`, there is no STATUS entry, and
`ROADMAP-EXTENSIONS.md` still carries `EXTENSION-IDENTITY 3.10` with no pending revision.

**Independent of every ruling above:** `entity-browser-rust`'s publish path
(`resolve_publish_keypair` → `persistence::publisher_keypair_in`, read live at `c183e6a`)
load-or-generates a bare keypair and mints no attestation at any point. `explicit_publish_seed`
already accepts `--identity-seed=<64 hex>`, so a deliberately chosen, recoverable seed needs no
code change — but a genesis with no cert has no rotation path under any design in this document.

## 10. Sources

**Internal (live worktrees, 2026-08-12)**
- `ENTITY-CORE-PROTOCOL.md` §1.4, §1.5 (v7.64/v7.65/v7.69), §1.5a, §3.5, §4.5a item 1a (v7.77),
  §5.1 (`is_revoked`), §5.5 (grantee resolution, PR-3), **§5.10** (Layer 1 / Layer 2, the
  extension contract, the convergent-input criterion), §9.0/§9.1 — `entity-core-protocol`
  `3042bd8`
- `EXTENSION-IDENTITY.md` v3.10 §1, §3.6 (`valid_functions`, `identity_lifecycle_kinds`,
  `identity_topology_for`, `identity_is_authorized_revoker`, `identity_verify_cert`), §4.1, §4.3,
  §4.4, §4.5, §6.4 (PI-13), §9.5 — `entity-system-architecture`
  `bf88be0`
- `EXTENSION-ATTESTATION.md` v1.3 §3.2 · `EXTENSION-QUORUM.md` v1.2 · `EXTENSION-REGISTRY.md` v1.2
  §2.4/§2.4.1, §3, §3.0a
- `entity-browser-rust` `src/content_site/publish.rs`, `src/persistence.rs` — `c183e6a`

**External (re-audited against primary sources for this document)**
- did:plc specification v0.1 — https://web.plc.directory/spec/v0.1/did-plc
- AT Protocol DID spec — https://atproto.com/specs/did
- AT Protocol cryptography spec — https://atproto.com/specs/cryptography
- AT Protocol identity guide — https://atproto.com/guides/identity
- KERI pre-rotation (KID0005) — https://identity.foundation/keri/kids/kid0005Comment.html
- Adversarial ATProto PDS migration —
  https://www.da.vidbuchanan.co.uk/blog/adversarial-pds-migration.html
