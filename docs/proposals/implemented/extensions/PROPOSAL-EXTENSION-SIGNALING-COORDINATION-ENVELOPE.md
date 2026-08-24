# PROPOSAL — SIGNALING §6.3 self-contained coordination envelope

**Status:** **✅ FOLDED 2026-08-04** into `EXTENSION-SIGNALING.md` §6.2 (blob framing repointed), §6.3 (the
`signed-blob` envelope + bucket-bound signing input + verification + four-disposition anti-downgrade rule), §6.5
(webrtc entities wrapped), §12 (`system/signaling/signed-blob` registered). **Fold trigger met:** the container
cross-verified **both ways at both current heads** — core-rust `38·0F` on Go's file (`7384c7c`), core-go `36·0F`
on Rust's (`6077229`), each direction checked by the *other* impl. Closes the standing S3 blocker.
**What folded beyond the reviewed DRAFT (cohort build, cross-verified):**
- **Bucket binding = cover-but-don't-carry** (core-rust, better than either §5 option): the signature covers
  `signing_input = "entity:sigblob:v1" ‖ 0x1F ‖ rendezvous_key(33) ‖ inner_content_hash(33)`; the key is supplied
  by the verifier from the bucket, never a wire field — so §6.5 shapes and §12's four-field set are untouched.
- **Four dispositions, not three** (core-rust argument, core-go-confirmed): step 5's "skip everything like
  undecodable" was a **downgrade bug** — "never was a container" (read-as-bare) and "container that failed to
  verify" (skip, MUST-NOT-fall-back-to-bare) are opposite dispositions; collapsing them lets a flipped signature
  byte downgrade a signed offer to an accepted unsigned one. The parse/container outcome is a fourth classification.
- **`signer_mismatch` reserved for the §6.1 claim comparison** (step 3); unsupported/unusable key material →
  `unusable_key`, so the skip stays a skip (keeps the Ed448 hardcode out).
- **The container is itself an entity** (`{type: system/signaling/signed-blob, data, content_hash}`), ECF field
  order `entity, public_key, signature, signer`; `entity` embedded verbatim.
**Earlier rev (DRAFT review, 2026-08-04):** canonical peer-id derivation not decoded-from-wire (Go blocking 1);
two-axis hash bytes corrected (Go blocking 2); local-observability SHOULD.
**Still pending (impl, not spec):** the deposit-side flip (both loops seal in one window) — see the flag-day note
in `ROUTING-2026-08-04-envelope-folded-and-s3-scope-answered-to-cohort.md`. Until it lands the browser leg is
pre-conformant and not MITM-safe — the reason is the migration, not a missing shape.
**Against:** `EXTENSION-SIGNALING.md` §6.2 / §6.3 / §6.5 / §12.
**Origin:** cohort finding — `entity-core-go`'s `2026-08-03-signaling-6.3-envelope-unpinned.md`, corroborated
by `entity-core-rust` (§6.3 unimplemented in-crate) and by the Ed448 defect Go's vector row caught in Rust.

## The gap

§6.2 MUST: the blob is the canonical entity encoding `{type, data, content_hash}`, `data` verbatim.
§6.3 MUST: the blob is *self-contained* — the entity **plus a detached signature carrying the signer's
`public_key`**. **Both cannot be true of the same bytes** — an entity encoding has no signature field — and
**§12 registers no envelope**, §6.1/§6.5 define no signature field, and §11's checklist says "sign
self-contained" without naming the container. `system/signature` cannot serve: it carries the signer as a
**hash**, needing a key lookup the rendezvous cannot perform — the exact "unreachable at rendezvous time"
condition §6.3 exists to escape.

The native path limped because §6.1's `connect-request`/`-response` carry `initiator`/`responder`, so a peer
correlates and punches while ignoring §6.3 — which is what **both impls do today**. **§6.5 removes the crutch:**
its three payloads carry no peer-id, so (a) §6.3 check "equals the claimed `initiator`/`responder`" has no
referent, and (b) §6.5's offerer MUST — justified by *"both peer-ids are now known (§6.3)"* — is unimplementable.
Eight green cross-impl crossings sit on an unimplemented security MUST: a **fourth §11.5.1 blindness instance, on
the security axis.**

## The proposal

### 1. Register one envelope type (§12) and repoint §6.2 at it

```
system/signaling/signed-blob := {           ; the self-contained §6.3 carrier — what a bucket actually holds
  fields: {
    entity:     {type_ref: "primitive/bytes"}    ; the §6.2 canonical entity encoding of the coordination
                                                 ; entity {type,data,content_hash}, embedded VERBATIM (§6.2 byte-preservation)
    signer:     {type_ref: "system/peer-id"}     ; the signer's canonical peer-id — self-describing per V7 §1.5
                                                 ; multikey (key_type ‖ hash_type ‖ digest); carries key_type, so nothing is hardcoded
    public_key: {type_ref: "primitive/bytes"}    ; the raw public key the signer commits to
    signature:  {type_ref: "primitive/bytes"}    ; detached signature over the inner entity's 33-byte content_hash
  }
}
```

§6.2's MUST is repointed: **what the carrier stores is the `signed-blob` encoding**; the inner coordination
entity rides in `entity` **verbatim** (the existing byte-preservation MUST now protects the signed bytes end to
end — re-encoding the inner entity still silently invalidates the signature, §6.2/§1.8). The carrier stays opaque
and byte-preserving over the whole `signed-blob`.

**Why `signer` (the peer-id), not a bare `key_type` field** (Go's sketch offered the latter): the peer-id is
self-describing (it *encodes* key_type + hash_type per §1.5), so it answers the parametric question for free; and
it is the exact referent §6.5's offerer rule and §3.2's pair/glare sort consume by name. **The peer-id is
verified-then-used** (Go review, blocking 1): `signer` is a wire field and therefore forgeable, so it is **never
trusted as given** — step 2 derives the id canonically from `(public_key, key_type)` and compares it to `signer`,
and only the *verified* value flows downstream. That still buys what motivated `signer` — no call site re-derives
it (re-derivation is where the 2026-08-04 id-encoding bug lived); the re-use just begins after step 2. Carrying
both `signer` and `public_key` is deliberate: the signature needs the pubkey (unrecoverable from a hash), the
identity needs the id, and step 2 binds them.

### 2. Verification — one procedure, uniform for §6.1 and §6.5 `[security — MUST]`

A verifier, on a `signed-blob`:

1. **Parse `signer`** as §1.5 multikey → `(key_type, hash_type, digest)`. `key_type` selects the algorithm;
   **`hash_type` is not an input to verification** — it is covered by the comparison in step 2. No hardcoded `0x01`.
2. **Bind key to id.** Check `public_key`'s length matches `key_type`. Compute
   `digest′ = Hash_{canonical_hash_type(key_type)}(public_key)`, where `canonical_hash_type` is the §1.5 mapping
   already fixed per key type (Ed25519 → identity `0x00`; Ed448 → SHA-256 `0x01`). Require `signer` to equal
   `varint(key_type) ‖ varint(canonical_hash_type(key_type)) ‖ digest′` **in full**. Any mismatch — **including a
   well-formed `hash_type` that is not the canonical one for `key_type`** — → **`unusable_key`**.

   *A verifier that instead dispatched on the envelope's `hash_type` would accept two distinct `signer` values for
   one key (`0x01‖0x00‖pk` and `0x01‖0x01‖SHA-256(pk)` both satisfy such a check), letting a peer choose its own
   identity per message. Every §6.5 decision is a sort over that id — §3.2's `pair_key`, §6.5's offerer rule, §6.4's
   skip-own — so a chosen id is a chosen glare role, a split rendezvous bucket, and a peer that no longer recognizes
   its own entity. (Go review, blocking 1; measured on one key, and what `TestPeerIDIsDerivedCanonicallyNotDecodedFromTheWire` pins.)*

   *(This is §6.3 check (a), generalized: the id is derived-and-bound, not compared to a separate claim.)*
3. **Where the inner entity carries an `initiator`/`responder`** (§6.1): check the verified `signer` equals it → else
   **`signer_mismatch`**. **Where it carries none** (all of §6.5): `signer` **is** the identity — step 2's binding is
   the whole of check (a). *(This is the one-sentence §6.3 clarification Go asked for.)*
4. **Verify `signature`** over the inner entity's **33-byte content_hash** (§3 below), dispatching the algorithm on
   `key_type`. Fail → **`bad_signature`**.
5. **Any failure MUST be skipped exactly as an undecodable blob (§6.4)** — never acted on, never an error on the
   wire. The taxonomy in steps 2–4 is for **diagnostics and conformance vectors**, not a wire-visible distinction.
   **The skip SHOULD be observable locally** — a log line or counter carrying the taxonomy label and the offending
   `key_type`. *Silence on the wire is required; silence inside the implementation is how the Ed448 hardcode survived
   until a conformance vector crossed it (Go review).*

### 3. What is signed — the 33-byte content_hash `[cross-peer seam — MUST]`

The signature is over the inner entity's **33-byte** `content_hash` = `content_hash_format ‖ digest` (the wire form
where the format byte always travels), **not** the bare 32-byte digest. This matches every other signature in the
tree (`mint`, `connect`) and the invariant-pointer rule. State it normatively: it is the 66-vs-64-hex / 33-vs-32
trap in a stranger-verifiable place, where a convention-derived answer is worth least.

### 4. key_type is parametric — Ed25519 is the floor, not the ceiling `[MUST]`

A coordination signer is **not** restricted to Ed25519. `key_type` is the §1.5 multicodec varint carried
self-describingly in `signer`; verification dispatches on it. **Ed25519 (`0x01`) is the MUST-implement floor.** A
**well-formed but unsupported** `key_type` MUST be treated as an undecodable blob and **skipped** (MUST-ignore,
ADR-0002 forward-compat) — **never hardcode-rejected as `signer_mismatch`.** *(This is the Ed448 defect Go's row
caught in Rust: a hardcoded `if key_type != ED25519 { SignerMismatch }` silently locked out an identity Rust's own
crypto crate mints, on a fail-closed path. Parametric-with-skip is the only reading that neither locks out a valid
future key nor accepts an unverifiable one.)*

### 5. Bucket binding — RECOMMENDED, for cohort confirmation `[security]`

Raised by the Go review: nothing binds a `signed-blob` to the rendezvous key it was deposited under, so a valid
blob observed in one bucket **replays verbatim into any other and verifies**. Not a channel hijack — the SDP
fingerprint still binds DTLS (§6.5) — but a peer in an unrelated bucket can be induced to attempt a connection to a
signer that never addressed it (an unsolicited-connection / reflection nuisance against the signer, wasted work for
the replayer's victim). **Recommendation: bind it** — the signed material covers the 33-byte `rendezvous_key`, so a
blob replayed into a different bucket fails verification. The signer always knows the key at signing time (it is
establishing *at* that key), and the "cost" Go names — blobs become non-portable across buckets — is a **property,
not a regression**: a coordination blob is meaningfully scoped to its rendezvous, and cross-bucket portability has
no wanted use. Deciding it **now** is the point: Go flags — correctly — that retrofitting a signed field after the
fold is expensive. **Open for the cohort to confirm the mechanism** (cover `rendezvous_key` in the signed content
vs. add it as a signed envelope field) and to veto if a cross-bucket use exists that this would break.

## Cohort review — resolved (Go, 2026-08-04)

1. **`signer` vs `key_type`-only — `signer`, accepted**, with the **verified-then-used** framing folded into §1/§2:
   the field is forgeable, derived-and-compared in step 2, and only the verified value flows on.
2. **`unusable_key` label — confirmed**, three names as written (`signer_mismatch` / `unusable_key` /
   `bad_signature`); maps 1:1 onto Go's existing three classes (only the *names* had collided). Go renames to these.
3. **hash_type — corrected (was wrong on both axes).** Two distinct axes, colliding code values: the `signer`
   **pubkey-hash** axis is identity-multihash `0x00` (Ed25519 floor) / SHA-256 `0x01` (Ed448); the signed
   **`content_hash_format`** axis is SHA-256 `0x00` / SHA-384 `0x01`. `0x01` means SHA-256 on the first and SHA-384
   on the second — **MUST NOT conflate.** Now stated in §2/§3; the old "SHA-256 = 0x01 for both" was the stale
   SPEC-AMBIGUITIES #67 form (see *Related* below).

**Related — corpus hygiene (not this fold):** `ENTITY-SYSTEM-REFERENCE.md:75/601` still spell the Ed25519 peer-id
as `Base58(0x01‖0x01‖SHA-256(pk))` — the stale non-canonical form (#67); the canonical form is identity-multihash
(`EXTENSION-REGISTRY.md:568`, V7 §1.5). Route as a separate reference-doc correction so this proposal's fold isn't
gated on it.

## Honesty / validation gate `[§11.5.1]`

This is wire-visible crypto: per the CDN-corridor meta-rule it is **not validated until both ground-up impls build
the `signed-blob` parser + emitter and cross-verify a `signed-blob` conformance vector both ways** (the §6.3
verification logic already crossed on scaffolded triples; the *container* has not). Fold on that crossing, the same
bar the WebRTC fold itself was held to — not on this prose. Until then S3 may build against this shape, knowing the
field set could move one notch in review.

## References
- Ask: `entity-core-go/docs/validation/spec-issues/2026-08-03-signaling-6.3-envelope-unpinned.md`.
- Corroboration: `entity-core-rust` §6.3-unimplemented note; the Ed448 defect (`68e3b0b`) + fix.
- Consumes: `EXTENSION-SIGNALING.md` §6.2 (blob framing), §6.3 (signature), §6.5 (offerer rule), §12
  (types installed); V7 §1.5 (multikey key_type/hash_type), §9.1 (crypto floor); ADR-0002 (MUST-ignore unknowns).
