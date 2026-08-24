# ANALYSIS — supply-chain landscape alignment (Nix/OCI/SLSA/TUF/Sigstore/Bazel/Unison + our legacy)

**Status:** Analysis / reconciliation — 2026-07-24. Grounds `EXTENSION-PACKAGE` + the content-addressed supply
chain before authoring. **Method:** two review streams — (a) our own legacy archive (the v7 Nix studies,
`EXTENSION-SUBSTITUTE v1.0`, the entity-repo/vcs/publish family) and (b) a web-grounded external landscape (14
systems). Reconciled against the current design (`EXPLORATION-CONTENT-ADDRESSED-SUPPLY-CHAIN`, `-EXECUTION-AND-REPO-
ARCHITECTURE`). Discipline: **borrow convergent semantics, not syntax** — the same move `system/device` made
against Redfish/OTel; every name stays entity-native. External status claims web-verified as of 2026-07-24.

---

## §1 The three reconciliations that reshape the design

1. **Distribution is already the Nix model, ratified — do not reinvent it.** `EXTENSION-SUBSTITUTE v1.0` (Active,
   conformance-gated) *is* the Nix-substituter design: ordered priority-driven substitute sources, "sign the
   content, trust the key, fetch from anywhere," **no transitive trust**, three publisher patterns
   (descriptors / manifest / both). The producer half exists as code (`entity-publish` computes the closure and
   emits a signed manifest + substitute-entry; `entity-vcs` = git-verbs over `system/revision`). **Our "distribute-
   by-hash / signed release index" was not new work** — it reconciles onto SUBSTITUTE + `EXTENSION-REGISTRY` §6a.7.

2. **Entity-native content has no build — confirmed from three independent sides.** Our own legacy
   ("executable-in-place; the tree *is* the artifact; no build step; capabilities travel with it"), **Unison** ("the
   hash *is* the typechecked artifact; no build step"), and the whole content-addressing thesis converge. So
   **`EXTENSION-PACKAGE`'s real scope is narrow: the foreign-artifact on-ramp (Lane B)** — wrapping a non-entity-
   native build output (a cargo/podman binary, an OCI image, a WASM module) as a content-addressed entity so it
   distributes and runs *uniformly* with native subtrees. For **Lane A** (entity-native), there is no package: the
   subtree is the artifact.

3. **Trust is three orthogonal planes, and we were thin on two.** The field's single recurring thesis:
   content-addressing **dissolves integrity at the content layer but re-concentrates trust and freshness at the
   naming/pointer layer — and every system's sharpest failures live in that gap.** The three planes:
   - **Integrity** (hash names bytes) — we have this solid (`tree = path→hash`, invariant #1). ✓
   - **Provenance** (signed statement: *who* built *what*, *how*, *from what*) — **we lumped this into "signed
     package"; it is under-specified.** This is the real new work.
   - **Freshness / anti-rollback** (is this current, non-reverted, non-mixed) — we have it *partially* (REGISTRY
     §6a.7 signed manifest, seq); the field says make it explicit and tiered.

---

## §2 What our design already gets right (confirmed across the field)

- **Content-address = integrity, never provenance** — universal across all 14 systems, and identical to our
  load-bearing invariant #1 + the CDN-corridor meta-rule. We never conflate them. ✓
- **Bind assertions to the digest, never the name.** OCI's central *unpatched* weakness is the mutable tag (a March
  2026 Trivy-image repoint is the live example). We are content-addressed throughout; the name layer is a thin,
  separately-authenticated index. ✓
- **Separate the content index from the name/result index.** Bazel **CAS-vs-Action-Cache**, OCI **digest-vs-tag**,
  Unison **hash-vs-name**, Nix **drvPath-vs-outPath** are one pattern — and it is exactly our own Nix study's
  **four-noun vocabulary**: **tree** (name→hash) · **substituter** (hash→bytes) · **registry** (name→where) ·
  **manifest** (a *baked* name→hash cache — an optimization, not a fourth concept). Adopt it explicitly as the
  organizing frame. ✓
- **Fetch-by-hash after a local miss + re-hash on receive + no transitive trust** (SUBSTITUTE) = Bazel
  `FindMissingBlobs` + the universal content-trust rule. ✓
- **Three trust postures** (accept-signed / re-run-vectors / re-derive-locally, §12.5 of the runtime-contract
  exploration) = the field's signing-vs-re-derivation planes, already anticipated. ✓

## §3 What to borrow — convergent semantics, entity-native (not syntax)

| Source | The semantic worth borrowing | Entity-native adoption |
|---|---|---|
| **OCI Descriptor** `{mediaType, digest, size}` | one universal typed edge; carrying `size` is a quiet DoS guard | ensure package/attestation refs carry **`size`** alongside the content-hash |
| **OCI `subject` + Referrers** | attach signatures/SBOM/provenance to immutable content *by reference*, answer "what refers to X?" | provenance is a separate **attestation entity** with `subject: content-hash`; "what attests X" is a graph/reverse-edge query — natural for entities |
| **in-toto Statement** (subject → predicateType → predicate, DSSE envelope) | a signed, typed claim *bound to a digest*; the envelope is a dumb signed container, semantics live in a versioned predicate-type | a `system/attestation` type: `subject = artifact_hash`, a **typed predicate** (`build-provenance` / `test-vectors` / `sbom`), signed by IDENTITY — SLSA-provenance is just one predicate |
| **TUF role/freshness split** | offline high-value content keys vs online low-value freshness keys; a compromised freshness key can neither forge nor roll back content | REGISTRY §6a.7 adopts the **tier split**; resolver enforces **highest-seen seq** |
| **Nix Fixed-Output Derivation** | one controlled-impurity escape hatch: any network access pre-declares its content hash | Lane-B builds: any in-build fetch **pre-commits to a content hash** (else the build is not hermetic) |
| **Guix fast-forward-only signed history** | anti-rollback as monotonic signed history, not a separate server | when source is git/REVISION, anti-rollback rides the signed revision DAG's fast-forward rule |
| **Sigstore transparency-log timestamp** | converts "trust this key forever" → "trust this identity at instant T" | **key-pinned, not keyless** (see §4); `EXTENSION-HISTORY` (append-only) is our transparency-log analog *if* non-repudiation is wanted |
| **Guix full-source bootstrap** (357-byte seed) | the only real answer to Thompson's "trusting trust" | aspirational; note the bootstrap axis exists for Lane-B toolchains |
| **IPFS multihash agility** | self-describing hash function → survive a hash break without renumbering | already ours (self-describing `content-hash` `format_code`, no fixed width) ✓ |

## §4 The unsettled forks — our position (stated, with rationale)

The field has **not** settled these; an entity-native design must take a position, and ours falls out of the
substrate we already have:

1. **Input-addressed vs content-addressed outputs.** *We are already past the fork for Lane A.* Entity content
   **is** a content-addressed output by construction — the thing Nix has chased for ~6 years (CA-derivations, still
   experimental in Nix 2.34) is *free* because the tree is the artifact (Unison's model). For **Lane B** we are
   input-recipe-pinned (`source_ref` + `build_env_digest`) with a content-addressed output blob, and reproducibility
   is **empirical, not structural** (see §5).
2. **Keyless-transparency vs key-pinned signing.** **Key-pinned** (`EXTENSION-IDENTITY` + `-QUORUM` for m-of-n),
   *not* Sigstore-keyless — we have an identity+quorum substrate, not OIDC/CA/public-log machinery, and importing
   that would add an OIDC dependency, a monitoring burden, and availability coupling. Borrow only the *insight*
   (timestamped non-repudiation), served by our own append-only `EXTENSION-HISTORY` if wanted.
3. **Re-derive vs trust-signature.** **Both — and re-derivation is *free* for Lane A** (the compute-determinism
   corpus, 330/330 three-way, already proves independent builders agree byte-for-byte). Almost no one else can make
   re-derivation a real gate; we can, for entity-native artifacts. For Lane B we are with everyone else: signature +
   *optional* empirical second-builder (`make verify`).
4. **Address-determinism: normative vs best-effort.** **Normative + cross-impl-tested.** IPFS (same bytes → *different*
   CID across tools, forcing IPIP-0499 import profiles) and REAPI's mandated canonical serialization are the field's
   scars; our CDN-corridor meta-rule already says a claim about what a hash resolves to isn't validated until a
   cross-impl conformance test exercises it. Canonical encoding of the package, `source_ref`, and manifest is a
   **MUST**, exercised by the cohort. (Canonical ECF / byte-preservation §1.8 is our NAR.)
5. **How far down to push content-addressing.** **All the way down, structurally** (Unison-like) for native content
   — entities are content-addressed by construction — and wrap foreign artifacts as content-addressed blobs
   (Nix-output-like) at the Lane-B boundary. This *is* the entity-native identity; state it as the position, not an
   accident.

## §5 Pitfalls to guard — the field's hard-won lessons → our MUSTs

1. **Non-hermeticity → false cache hits → fleet poisoning.** Bazel's production scars (#16179: same `clang` path,
   different headers, same action key → wrong object; IEEE study: 0/70 projects fully hermetic) come from an
   undeclared input not being in the key. **Lane B MUST:** sandbox reads *and* writes, strip the environment, pin
   `timeout`/a salt into any build-cache key, and **cache nothing on unclear success.** Lane A is hermetic by
   construction (the compute engine).
2. **Do not over-claim reproducibility.** Honest field numbers: Guix ~75%, nixpkgs ~69–91%, OCI ~few-% out of the
   box. For **Lane B**, bit-reproducibility is an **empirical, per-artifact, second-builder-verified** property,
   never a structural guarantee. For **Lane A** it is structural (corpus-backed) — state the two differently.
3. **Referrers-aware GC.** OCI's pre-1.1 signature-as-tag hack let routine registry GC delete signatures as orphans
   (Harbor broke cosign-signed images). Our GC (`GUIDE-GC`) **MUST NOT** reclaim an attestation while its subject is
   pinned — pin the subject↔attestation edge.
4. **Anti-rollback = signed monotonicity + resolver highest-seen.** IPNS's CVE (sequence not covered by the v1
   signature; resolvers checking only expiry, not highest-seen, served valid-but-older records) is the exact trap.
   The `seq` **MUST** be inside the signature, and the resolver **MUST** reject a seq below the highest seen.
5. **Reproducible-address determinism is a cross-impl conformance gate, not prose** (CDN-corridor meta-rule) — the
   one lesson we already know, re-confirmed by IPFS/OCI/REAPI.

## §6 The refined design (the deltas to apply)

1. **Three planes, three homes.** *Integrity* = tree/content (Active). *Provenance* = a **new `build-provenance`
   *kind* on the Active `EXTENSION-ATTESTATION` v1.3** (verified: it is already the general in-toto primitive —
   `attested` may be any hash incl. a software artifact; `kind` is an open, consumer-extended vocabulary). So
   provenance is **not a new entity type** — just a registered kind + its `properties` schema (a `schema` version
   field, since ATTESTATION has no `predicate_type`). *Freshness* = REGISTRY §6a.7, TUF-tiered, resolver
   highest-seen. **Drafted: `PROPOSAL-EXTENSION-PACKAGE`.**
2. **`EXTENSION-PACKAGE` narrowed** to the **foreign-artifact on-ramp** (Lane B). It wraps a build output as a
   content-addressed entity + Descriptor (`type, artifact_hash, size`) + `source_ref` + `build_env_digest` +
   `build_recipe`; **provenance and signatures move OUT to `system/attestation`** (bound to `artifact_hash` by
   reference). For Lane A there is **no package** — say so normatively.
3. **Adopt the four-noun vocabulary** (tree / substituter / registry / manifest) as the supply chain's organizing
   frame; each already has an entity home.
4. **Distribution reconciles to `EXTENSION-SUBSTITUTE`** — do not author a parallel index. A package is a blob the
   substituter serves; the release index is a REGISTRY §6a.7 manifest. The named-future `nix-cache` / `peer-to-peer`
   / foreign substitute types are the **bridge to external hashed subgraphs** (the never-written `EXTENSION-BRIDGE-
   NIX`/git-bridge — ingress-first, needs a hash-translation table).
5. **Position the forks (§4)** in the design docs; they were implicit and are now explicit.

## §7 Honest ledger

- **Legacy grounded in-repo:** SUBSTITUTE v1.0 Active (Nix-substituter, conformance-gated); the entity-repo /
  entity-vcs / entity-publish family is preliminary-design + shipped producer code; two named legacy spec-gaps
  (continuation transform can't compute path strings; cross-peer `dispatch_capability` provenance) are orthogonal
  but real.
- **External web-verified** with status corrections that matter: **Nix CA-derivations still experimental** (2.34,
  mid-2026); **SLSA v1.2** (Nov 2025, Source track approved, VSAs first-class); **OCI 1.1** (subject/Referrers);
  **Sigstore Rekor v2 GA** (Oct 2025); **Notary v1 / DCT retiring** (shutdown Dec 2026 → Sigstore). *Don't cite
  stale "SLSA 1–4" or "NixOS 100% reproducible."*
- **This is a reconciliation, not a fold:** the deltas land on the two supply-chain explorations + a narrowed
  `EXTENSION-PACKAGE` scope + a new `system/attestation` type — all DRAFT/exploration, none folded.
- **Reproducibility claims:** the field's numbers are other-systems/empirical; **our Lane-A structural claim rests
  on the compute corpus (proven); our Lane-B claim is empirical and unproven until we measure a second builder.**

*The one sentence: the review confirms our spine (content-address = integrity; bind to the digest, not the name;
distribution already ratified as the Nix-substituter model in `EXTENSION-SUBSTITUTE`) and sharpens the rest —
narrow `EXTENSION-PACKAGE` to the foreign-artifact on-ramp because entity-native content has no build (Unison + our
own legacy agree), split trust into three planes with provenance as a new in-toto-shaped `system/attestation`, adopt
the four-noun vocabulary, and take an explicit position on the field's five unsettled forks — where our compute-
determinism corpus lets us make re-derivation a real gate for native artifacts that almost no one else can.*
