# PROPOSAL — EXTENSION-ENCRYPTION v1.0: the three-mode entity encryption layer

**Status:** **IMPLEMENTED — reference proposal, reconstructed after the fold. See §0.**
**Target:** `EXTENSION-ENCRYPTION.md` (the whole spec; landed v1.0).
**Tier:** `extensions/` — validated by go · rust · py.
**Scope:** the design decisions behind the landed spec, its authoring/lock protocol, and its landing
status. **Not** the namespace/resolution arc — that is
`active/extensions/PROPOSAL-ENCRYPTION-NAMESPACE-AND-RECIPIENT-RESOLUTION.md`.

---

## 0. Why this file exists

`EXTENSION-ENCRYPTION.md` closed with *"Landed from `proposals/implemented/PROPOSAL-EXTENSION-ENCRYPTION.md`"* —
a citation to a document that **exists in no post-split checkout.** The original did not cross the repo
split. The corpus gate reports it as `proposal-citation-unresolved`, which is the same cited-but-absent
class as `system/peer/published-root`'s missing definition.

With no proposal on disk, the spec had become the only home for its own rationale: three sections of
design-decision narrative, cohort sequencing, and lock protocol sat inside a normative document that is
supposed to be implementable by someone who has never heard of this cohort. **Reconstructing the
proposal is what lets that material leave the spec** rather than being deleted.

**Reconstructed from the spec's own text**, verbatim where it was already written down. Nothing here is
a new decision, and nothing is back-dated. Where the original recorded a confirmation that cannot now be
re-derived from a source, it is marked as such rather than restated as fact.

---

## 1. The seven design decisions (moved verbatim from the spec's §17)

All were decided before v1.0 landed; each carried a cohort confirmation point for the implementing seats
to validate during the first build round. They are recorded here because they are **rationale for the
rules**, not rules.

1. **One extension or two?** Two sibling extensions — `ENCRYPTION` (stateless, single-shot; self / peer /
   group) and a planned `ENCRYPTED-SESSION` (stateful, ratcheted). The stateless primitive is complete on
   its own and useful without a session layer; merging them would make every consumer pay for session
   state it does not use.

2. **Does ENCRYPTION require IDENTITY?** **No.** Three tiers (§4.0): Tier A is the V7 floor
   (single-peer single-key; encryption-owned `handoff` + `revocation` types; the V7 keypair as trust
   anchor), Tier B adds ATTESTATION for substrate-graph observability, Tier C adds IDENTITY for
   multi-device cert chains. The encrypt/decrypt flow is **identical at every tier**; only publishing,
   rotation and revocation discipline vary. Tier A is the right shape for headless single-purpose peers
   (CLI tools, storage daemons, CDN publishers), and cross-tier interop is normative.

3. **AES-256-GCM in `peer` mode for v1?** Deferred. XChaCha20-Poly1305 is the v1 default; a
   deterministic-counter construction for AES-GCM is a v1.5 question, because GCM nonce reuse is
   catastrophic and the v1 model has no per-recipient counter state to make it safe.

4. **Group mode's key-commitment binding** — the outer entity commits to the group key, so a member
   cannot be shown a different plaintext than its peers (the F2-1 property, gated by
   `ENC-GROUP-COMMIT-1`).

5. **Rotation as handoff, not silent replacement** — a rotation is an authored entity dual-signed by the
   old and new key holders, targeting the handoff/attestation hash rather than the pubkey hash.

6. **Signed-by-default in v1** — sender authentication is not optional; an unsigned encrypted entity is
   rejected. The signature lives at the V7 invariant pointer, never as a field on the encrypted entity
   (a field holding the entity's own `content_hash` is structurally impossible).

7. **Replay is out of scope** — content-hash dedup is the only built-in; per-message freshness belongs to
   the session extension. Recorded as a deliberate boundary so consumers do not assume it.

## 2. Authoring + lock protocol (moved from the spec's §16.5)

The byte-fixture cycle that produced the pinned `expected_*` values in §16:

1. Arch lands the spec with TBD `expected_*` placeholders.
2. One seat produces reference bytes from a prototype against the §5.2 pinned shapes.
3. Arch byte-verifies independently against the spec text, re-deriving ECF bytes from §5.2 first
   principles rather than accepting the emitted values.
4. The remaining seats produce their own reference bytes; the cohort compares.
5. Three-way byte-equal → lock. Divergence is an absorption signal: identify which AAD key disagrees,
   arch arbitrates.

**The lock signal for v1.0** is cross-impl three-way plus a keystone-generated peer green on every floor
vector. The independent re-derivation in step 3 is the load-bearing step — without it the cohort locks
whatever the first emitter produced.

## 3. Landing status at v1.0 (moved from the spec's §21)

The spec landed with implementation dispatched to the cohort. What remained at landing:

- **Implementation pass** — `self` + `peer` modes are PRIMARY; `group` and rotation are best-effort.
- **§16 floor vectors** are the gate, not green unit tests.
- **Close-out** = cross-impl three-way + keystone-green on every floor vector.
- **Deltas fold in place** — a cohort finding corrects the spec directly, without a rev bump, per the
  ecosystem's *"ratified ≠ folded"* discipline.

*(Build state as of this reconstruction lives in the arc-D proposal §6 and in `WORKSTREAMS.md`'s
oracle-coverage table, both of which carry live pins. It is deliberately **not** restated here: a build
claim in a reference proposal is a dated measurement with nothing to expire it.)*

## 4. What this reconstruction does not recover

- **The original's own review record.** Whatever cohort discussion produced decisions 1–7 is not on disk
  in this repo; the decisions survived in the spec, the deliberation did not.
- **Which seat confirmed which decision.** The spec recorded "Go-confirmed" against several items with
  no pin. Rather than repeat an unsourced attribution, it is recorded here as: the confirmations
  happened, the record of who gave them did not cross the split.
- **Anything about §4.3/§4.4 namespace or resolution** — that arc is separately proposed and reviewed.
