# PROPOSAL — EXTENSION-PACKAGE — the foreign-artifact on-ramp (a build-provenance attestation kind)

**Status:** DRAFT (2026-07-24)
**Target:** a new `specs/extensions/EXTENSION-PACKAGE.md` (Tier: capability layer). It **defines no new entity
type** — it registers a **`build-provenance` attestation kind** on the Active `EXTENSION-ATTESTATION` primitive, and
specifies the build/reproducibility contract + the distribution reconciliation. Also registers the kind in
`EXTENSION-ATTESTATION` §3.2's kind-ownership table.
**Provenance:** `EXPLORATION-CONTENT-ADDRESSED-SUPPLY-CHAIN` + `-EXECUTION-AND-REPO-ARCHITECTURE`, reconciled by
`ANALYSIS-SUPPLY-CHAIN-LANDSCAPE-ALIGNMENT` (Nix/OCI/SLSA/TUF/Sigstore/Bazel/Unison + our v7 legacy). Verified
against the live surface: `EXTENSION-ATTESTATION` v1.3, `EXTENSION-SUBSTITUTE` v1.0, `EXTENSION-REGISTRY` §6a.7,
`EXTENSION-REVISION` v3.8, `EXTENSION-CONTENT` v3.6 (all Active).
**Scope:** how a **non-entity-native build output** (a cargo/podman binary, an OCI image, a WASM module) enters the
content-addressed world as a signed, provenance-bearing, distributable, admittable artifact — so it rides the
*same* store, distribution, and admission machinery as native entity subtrees. **This is the one genuinely-missing
piece; everything else already exists** (§1).

---

## 1. The one-paragraph design (why this is tiny)

A content-addressed software supply chain decomposes into three planes, and **two of the three already ship**:
**integrity** (`tree = path→hash` + `EXTENSION-CONTENT` blobs), **distribution** (`EXTENSION-SUBSTITUTE` v1.0 — the
ratified Nix-substituter model — + the `EXTENSION-REGISTRY` §6a.7 signed release manifest), and **provenance**
(`EXTENSION-ATTESTATION` v1.3 — a general in-toto-style signed claim whose `attested` subject may be *any* hash,
incl. "content … software binary"). The **only missing surface** is the *shape* of a build-provenance claim: a
registered attestation **kind** binding an artifact hash to how it was built. So a "package" is **not a new noun** —
it is a **`build-provenance` attestation over a content-addressed artifact blob.** This proposal defines that kind
and the build contract around it; it invents no entity type, no wire change.

**Load-bearing scope cut (from the review):** entity-native content has **no build** — the subtree *is* the
artifact, executable in place (Unison + our own legacy agree). So a package is needed **only for Lane B** — foreign
artifacts from a non-entity-native toolchain. **For Lane A (entity-native), there is no package** (§6).

## 2. The `build-provenance` attestation kind `[the only new normative shape]`

A package is a `system/attestation` (`EXTENSION-ATTESTATION` §3.1) with a `build-provenance` kind:

```cbor
system/attestation = {
  attesting:  <peer_hash | quorum_hash>,        ; the builder (or a K-of-N build quorum) — EXTENSION-ATTESTATION §3.1
  attested:   <content_hash>,                    ; THE ARTIFACT — the content-addressed built bytes (an EXTENSION-CONTENT blob)
  properties: {
    kind:              "build-provenance",       ; namespaced kind, owned by EXTENSION-PACKAGE (ATTESTATION §3.2)
    schema:            "1",                       ; predicate version — encoded here, since ATTESTATION has no predicate_type field (§7)
    source_ref:        { subgraph_type: "git" / "revision" / "content", hash: <content_hash> },  ; §3
    build_env_digest:  <hash>,                    ; the hermetic build environment (podman base-image digest, pinned)
    build_recipe:      { verb: tstr, toolchain: tstr },   ; e.g. { verb: "package", toolchain: "rustc-1.9x" }
    target:            tstr,                       ; "linux-x86_64" / "wasm32" / "oci-image" / ...
    artifact_size:     uint,                       ; bytes of `attested` — refs carry no size (§5); a DoS/allocation bound
    ? artifact_media:  tstr,                        ; media/type hint for the artifact bytes
    ? capability_manifest: <manifest>,              ; imports/offers — the admission contract (§4)
    ? build_inputs:    [* { subgraph_type, hash }], ; additional pinned inputs (FODs — §3.2)
  },
  ? not_before, ? expires_at, ? supersedes,       ; standard EXTENSION-ATTESTATION fields
}
; signed by `attesting` per EXTENSION-ATTESTATION (the signature IS the provenance)
```

- **The signature is the provenance** — `attesting` signs the claim (`EXTENSION-ATTESTATION`), so "who built this,
  from what, how" is a signed edge in the graph, **attached by reference** to the immutable artifact (OCI
  `subject`+Referrers, done entity-native). Many builders may attest the *same* artifact hash — that is exactly the
  **reproducibility signal** (§7): independent `build-provenance` attestations over one `attested` hash from
  distinct `attesting` peers = independent confirmation the build is reproducible.
- **No `predicate_type` field exists in `EXTENSION-ATTESTATION`** (verified, §7 there is storage-path-free and
  version-free). We therefore carry an explicit `schema` in `properties` — the one small normative addition this
  proposal makes to the predicate model. Register `build-provenance` in the ATTESTATION §3.2 kind-ownership table,
  owner = EXTENSION-PACKAGE.

## 3. `source_ref` — a typed reference into any hashed subgraph

`source_ref = { subgraph_type, hash }` references the source the artifact was built from, in **any content-addressed
("hashed identity") subgraph**: `revision` (an `EXTENSION-REVISION` root — Active, content-addressed, immutable-
once-shared, **unsigned**, so a *signed source* is itself an attestation over the revision root), `git` (a git
commit/tree hash — first-class, referenced/verified like our own), `content` (a bare content-store tree). **Git and
REVISION are peers, not a migration** — content-addressing is the general case that unifies them.

**§3.2 Controlled impurity (the Nix FOD lesson).** A hermetic build (§8) forbids network access. Any input that
*must* be fetched (a dependency, a base layer) is pre-committed as a `build_inputs` entry `{subgraph_type, hash}` —
the artifact's build closure is fully hash-pinned, or the build is not hermetic and its `artifact_size`/hash claim
is not reproducible.

## 4. The capability manifest — admission, reconciled to `offered`

`capability_manifest` is the artifact's **imports/offers** — what it needs and provides. It is the **admission
contract**: a host (W-HOSTING) matches it against `system/device/host/offered` (`PROPOSAL-SYSTEM-DEVICE` §7) before
running the artifact — the sense→actuate seam. For a WASM component the manifest is *also* extractable from the
artifact (a WIT world), so a host MAY verify the claimed manifest against the artifact rather than trust the
builder's claim. Manifest shape reconciles to the run-environment work (`EXPLORATION-RUN-ENVIRONMENTS`); this
proposal pins only that a package **carries** one and that admission reads it.

## 5. Distribution — reconciled, not invented `[no new distribution surface]`

A package artifact is an ordinary content blob, so distribution is **already shipped**:

- **Fetch by hash** — the artifact rides `EXTENSION-SUBSTITUTE` v1.0 (ordered priority sources, **signature-MUST on
  the entry, no transitive trust, no wildcards**, opaque polymorphic `endpoint`). A package needs *no new
  substitute type*; the named-future `nix-cache` / `peer-to-peer` types are the **bridge to external hashed
  subgraphs** when wanted.
- **Release index** — a signed `name/version → artifact_hash` index is an `EXTENSION-REGISTRY` §6a.7
  `system/registry/binding-manifest` (signed, `seq` anti-rollback, `coverage: partial|complete`, delegated/sharded)
  — **TUF-tiered** per §7. No package-specific index is invented.
- **`size` gap (verified):** a content reference is a bare 33-byte `system/hash` with **no byte size**. So the
  package carries `artifact_size` explicitly (§2) as the allocation/DoS bound a consumer needs before fetching.
  *(The general "refs carry no size" gap is logged for core, not fixed here.)*

## 6. Lane A vs Lane B — a package is a Lane-B thing `[the scope ruling]`

- **Lane A — entity-native.** The source subtree (`EXTENSION-REVISION` / content) **is** the runnable artifact —
  content-addressed, self-verifying, executable-in-place, capabilities travel with it (Unison's model; our legacy
  "no build step"). **No package, no build-provenance-of-a-foreign-output is required.** A build-provenance
  attestation MAY still be minted to record *which* transform produced a derived subtree, but there is no separate
  artifact to wrap.
- **Lane B — foreign toolchain.** cargo/rustc/podman produces bytes the entity system did not evaluate. The
  `build-provenance` attestation is the **on-ramp**: it lifts those bytes into the content store and binds them, by
  signature, to their source + recipe + environment + manifest. **This is the whole reason `EXTENSION-PACKAGE`
  exists.**

## 7. Trust — three planes, our position on the forks `[from the alignment analysis]`

- **Integrity** = the artifact hash (bytes only). **Provenance** = this signed attestation (who/how). **Freshness**
  = REGISTRY §6a.7 (`seq` + resolver **highest-seen** enforcement; the `seq` MUST be inside the signature — the IPNS
  rollback-CVE lesson), TUF-tiered (offline content key / online freshness key).
- **Key-pinned, not keyless.** Trust roots in `EXTENSION-IDENTITY` / `-QUORUM` (K-of-N via `attesting = quorum_hash`),
  **not** Sigstore-style OIDC/CA/public-log — we have the former substrate, not the latter. `EXTENSION-HISTORY`
  (append-only) is our transparency-log analog if non-repudiation is wanted.
- **Re-derivation is a *free* verification gate for Lane A** (the compute corpus, 330/330 three-way, proves
  independent evaluators agree byte-for-byte) — a gate almost no other system can offer. For **Lane B**,
  reproducibility is **empirical, not structural**: two independent `attesting` peers agreeing on one `attested`
  hash is the signal (§2), verified by `make verify`, never asserted.

## 8. The build contract — hermeticity MUSTs `[the pitfalls, pinned]`

For Lane-B builds (the field's hard-won lessons → MUSTs):

- **Hermetic or it doesn't count.** Pin the build environment by **digest** (`build_env_digest`, not a tag); **no
  network in the build step** (all inputs are `source_ref` / `build_inputs` FODs, §3.2); `SOURCE_DATE_EPOCH` +
  strip timestamps/paths; deterministic, stripped output. A non-hermetic build's reproducibility claim is void.
- **No false cache hits.** If a build result is cached, the cache key MUST cover the full declared closure +
  `build_env_digest` + a salt; **cache nothing on unclear success** (the Bazel poisoning class).
- **`make package` / `make verify`.** One new build verb — `make package` = build (hermetic) → put the artifact
  blob → mint the signed `build-provenance` attestation. `make verify` = rebuild and assert the `attested` hash
  matches (the second-builder gate). Extends the `build`/`test`/`lint`/… vocabulary (AGENTS-STANDARD); everything
  else unchanged.
- **Referrers-aware GC.** `GUIDE-GC` MUST NOT reclaim a `build-provenance` attestation while its `attested` artifact
  is pinned (the OCI/cosign orphan-signature failure). Pin the subject↔attestation edge.

## 9. Security

- **Forgery** — the attestation is IDENTITY-signed by `attesting`; an unsigned/mis-signed claim is invalid (per
  `EXTENSION-ATTESTATION`). A false `attested` hash simply names different bytes (content-addressing).
- **Artifact substitution** — impossible without changing `attested` (the hash *is* the artifact); SUBSTITUTE's
  signature-MUST + no-transitive-trust prevents a relay forging a source claim.
- **Reproducibility is a claim, not a proof** — a single builder's attestation attests *its* build; only
  independent re-derivation (multiple `attesting` over one `attested`) makes it verifiable. State it honestly;
  never over-claim (Guix ~75% / nixpkgs ~69–91% are the field's real numbers).
- **Manifest lie** — a builder could claim a false `capability_manifest`; a host SHOULD verify against the artifact
  where the artifact is self-describing (WASM WIT), and admission is still gated by `offered` regardless.

## 10. Conformance & posture

- **No wire change; no new entity type.** New surface = one attestation `kind` (+ its `properties` schema + `schema`
  version field), one build verb, and normative build/GC MUSTs. Entities are existing `system/attestation` /
  `system/content` / `system/registry/binding-manifest`.
- **Cross-impl-observable surface to pin (MUST):** (1) the canonical encoding of the `build-provenance` `properties`
  (so the attestation hash is well-defined across impls — the address-determinism lesson); (2) `artifact_size`
  matches the artifact; (3) resolver **highest-seen `seq`** on the release manifest; (4) the FOD rule (a hermetic
  build with an unpinned network fetch is non-conformant).
- **Validation gate (CDN-corridor meta-rule):** not real until a cross-impl run **builds a Lane-B artifact
  hermetically, mints + signs the `build-provenance` attestation, distributes the artifact by hash via SUBSTITUTE,
  and a *second independent builder reproduces the same `attested` hash.`** Prose does not prove a build pipeline.

## 11. Open / deferred

- **OCI-manifest-style bundling object** — if an artifact becomes multi-part or multi-arch (a binary + assets; a
  fat/multi-target set), a `system/package` *bundling* entity (one hash → {artifacts, manifest, attestations}) may
  earn its place, à la the OCI image index. **Deferred until multi-part demands it** — v1 is one attestation over
  one artifact (avoid speculative structure).
- **Verified sub-blob range-fetch** — `EXTENSION-CONTENT` verifies whole chunks (flat list, no chunk-Merkle); a
  BitTorrent-v2 / BLAKE3 verified-range layer for large artifacts is a **distribution-efficiency** future, not a
  package concern. Logged.
- **`predicate_type` versioning in `EXTENSION-ATTESTATION`** — we encode `schema` in `properties`; if versioned
  predicates become common across kinds, promoting it to a first-class ATTESTATION field is a future core note.
- **`nix-cache` substitute type** — the concrete bridge to Nix's own store; named-future in SUBSTITUTE §6, a
  natural follow-on to reference external hashed subgraphs.

## 12. References

- **Reused (Active):** `EXTENSION-ATTESTATION` v1.3 (§3.1 the claim, §3.2 kind-ownership), `EXTENSION-CONTENT` v3.6
  (the artifact blob), `EXTENSION-SUBSTITUTE` v1.0 (distribution), `EXTENSION-REGISTRY` §6a.7 (release manifest),
  `EXTENSION-REVISION` v3.8 (source subgraph), `EXTENSION-IDENTITY`/`-QUORUM` (signing), `PROPOSAL-SYSTEM-DEVICE` §7
  (`offered` admission).
- **Design:** `EXPLORATION-CONTENT-ADDRESSED-SUPPLY-CHAIN`, `-EXECUTION-AND-REPO-ARCHITECTURE`,
  `ANALYSIS-SUPPLY-CHAIN-LANDSCAPE-ALIGNMENT` (the fork-positions + pitfalls).
- **Consuming end:** W-FLEET / W-HOSTING (`WORKSTREAMS.md`); the connection node as the first demonstrand.

*The one sentence: a package is not a new noun — it is a signed `build-provenance` attestation (existing
`EXTENSION-ATTESTATION`) binding a content-addressed foreign-build artifact (existing `EXTENSION-CONTENT`) to its
source, environment, recipe, and manifest, distributed by the existing substituter and release manifest — so the
only genuinely-new surface is one attestation kind plus a hermetic-build contract, and reproducibility is proven,
for entity-native artifacts, by the re-derivation gate almost no other supply chain can offer.*
