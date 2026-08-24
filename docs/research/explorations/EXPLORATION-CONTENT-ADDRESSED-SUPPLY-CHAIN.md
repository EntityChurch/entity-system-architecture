# EXPLORATION — the content-addressed software supply chain (the capstone that unifies the stack)

**Status:** Exploration / vision synthesis — 2026-07-24. **NOT a proposal, NOT ratified.** This is the capstone
arc the operator named: *signed packages, built from signed revision repositories, run on the managed fleet — all
one content-addressed system.* It is an **assembly of primitives this repo already has**, not an invention: an
exhaustive survey (154 docs, 2026-07-24) found **every load-bearing primitive present and specified, and no
document that assembles them into a supply chain.** This doc is that assembly, with the honest gaps marked.

> **Post-review reconciliation (2026-07-24 — `ANALYSIS-SUPPLY-CHAIN-LANDSCAPE-ALIGNMENT`).** A legacy + external
> landscape review (Nix/OCI/SLSA/TUF/Sigstore/Bazel/Unison + our own v7 Nix studies) sharpened three things, folded
> through below: **(1)** distribution is **already ratified** as the Nix-substituter model in `EXTENSION-SUBSTITUTE
> v1.0` (Active) + REGISTRY §6a.7 — reconcile to it, don't reinvent; **(2)** entity-native content **has no build**
> (Unison + our legacy agree: the tree *is* the artifact), so `EXTENSION-PACKAGE` narrows to the **foreign-artifact
> on-ramp (Lane B)**; **(3)** trust is **three planes** — integrity (hash) / provenance (a *new* in-toto-shaped
> a build-provenance *kind* on the Active `EXTENSION-ATTESTATION` — already the in-toto primitive, so provenance is
> **not a new type**, just a new kind — bound to the artifact digest) / freshness (REGISTRY §6a.7, TUF-tiered). The
> **four-noun vocabulary** — tree (name→hash) · substituter (hash→bytes) · registry (name→where) · manifest (a
> *baked* name→hash cache) — is the organizing frame. See the analysis for the position on the field's five
> unsettled forks (input- vs content-addressed; keyless vs key-pinned; re-derive vs trust-signature; …).

**Why now.** W-FLEET (the managed-substrate operations lineage) gave us the *consuming* end — a fleet that runs
persistent managed services. The obvious next question is *where do the things it runs come from* — and the answer
closes the biggest loop in the system: the same content-addressing + signatures + identity that move entities are
exactly what a software supply chain is made of. Source, build, artifact, distribution, and runtime stop being
five systems glued together and become **five views of one content-addressed entity graph.**

---

## §1 The pipeline — one content-addressed graph, five stages

```
signed revision repository        →  reproducible build           →  signed package / artifact
(EXTENSION-REVISION DAG +             (deterministic compute;          (content-addressed entity +
 IDENTITY signatures = source)         source-hash → artifact-hash)      signature + capability manifest)
        │                                                                        │
        └──────────────── all one content store: tree = path→hash, dedup ────────┤
                                                                                  ▼
                          run on the fleet                        distributed by hash
                     (W-FLEET / W-HOSTING: admit on the            (CONTENT dedup + REGISTRY signed
                      manifest via `offered`, run it)               manifest + the static reef/CDN)
```

Each arrow is an **existing** mechanism (§2). The novelty is only that they line up into a single graph where the
output hash of one stage is the input hash of the next, and a signature over any node is a claim about everything
it transitively names. **The DAG *is* the bill of materials; the hash *is* the provenance; the signature *is* the
attestation.** No SBOM format, no separate provenance database, no version-number trust — those all collapse into
properties the content-addressed graph already has.

---

## §2 What already exists (the primitives — do not rebuild)

The survey's inventory, mapped onto the pipeline. This is the point of the exploration: the parts are here.

- **Source as content-addressed, versioned data — `EXTENSION-REVISION` (v3.8, Active).** A full git-like model:
  content-addressed trie roots with parent lineage, a version DAG, common-ancestor finding, three-way + CRDT
  merge, conflicts-as-entities, cross-peer sync. "Immutable once shared." This *is* "source repositories as
  content-addressed data." (`entity-repo`, the git-analog CLI over it — `GUIDE-SHELL-FRAMING` §9.2 — is specified,
  not yet built; `push` is deferred in the reference impls.)
- **The universal store — `EXTENSION-CONTENT` + `EXTENSION-TREE`.** `tree = path→hash`, content-store **dedup** as
  "the primary value," convergent chunking, identical bytes → identical hash across peers. The artifact store, the
  source store, and the package cache are **the same store** — dedup'd across all of them for free.
- **The compiled artifact as a first-class entity — `EXPLORATION-COMPUTE-PROGRAM-RUNTIME-CONTRACT` §12.4-12.5.**
  The single most on-point existing design: `compiled/{target}/{ir_hash} → { target, ir_hash, artifact_bytes,
  oracle_vectors_hash }`. Caching/distribution are "free" (the content store *is* the artifact cache); provenance
  is "intrinsic" (the key asserts "I am the compilation of `ir_hash` for `target`"); **trust = passes the oracle,
  not the version number** (§12.5). This is the supply chain's core object, already sketched — for compute IR; §4
  generalizes it.
- **Deterministic build — `EXTENSION-COMPUTE`.** Its determinism section already names "content-addressed build
  systems (deterministic from source to output)" as an enabled use case. Determinism is what makes
  source-hash → artifact-hash a *function*, hence independently reproducible. *(This is **Lane A** — an
  entity-native compute build; a native `make`+podman build is **Lane B**, reproducible by hermetic discipline
  instead — see §2a.)*
- **Signatures + provenance substrate — `EXTENSION-IDENTITY` / `-ATTESTATION` / `-QUORUM`.** The signed-graph
  layer. A signed revision, a signed package, a K-of-N release approval are all this substrate applied to a
  content hash. *(Note: `EXTENSION-REVISION` does not itself wire signing — a signed revision is IDENTITY over a
  revision root. That wire is one of the few genuinely-missing pieces, §3.)*
- **Signed distribution manifest — `EXTENSION-REGISTRY` §6a.7.** A TUF/DNSSEC-analog signed binding manifest
  (name→hash index, `coverage: partial/complete`, delegated/sharded, anti-rollback `seq`, MUST-verify against a
  pinned key). Structurally *is* a signed package-registry index — today the objects are name bindings; §4 points
  it at artifacts.
- **Content-addressed distribution in practice — `EXTENSION-SUBSTITUTE` + the static "reef."** The egui-WASM
  release path: hold a known root hash, walk refs, fetch by hash from object-storage/CDN (the Tier-0 static reef,
  `GUIDE-REFERENCE-DEPLOYMENT`). Content-addressed distribution already ships — for the app's own bundle.
- **Package admission by capability manifest — `EXPLORATION-RUN-ENVIRONMENTS-AND-WORKLOAD-ACTUATION`.** "Workload =
  a WASM component carrying a WIT world" — a build-time-auditable manifest of imported/exported capabilities;
  admission = static manifest-vs-policy allowlist *before* it runs. This is the package's admission contract, and
  it plugs straight into `offered` (W-DEVICE §7 / W-HOSTING).

**Net:** content-addressed store + dedup, a git-like signed-capable revision DAG, a signed-graph attestation
substrate, a TUF-shaped signed manifest, a compiled-artifact-as-entity model, a capability-manifest admission
model, and a working content-addressed CDN. **Seven of the pipeline's primitives already exist and are specified.**

---

## §2a The concrete build — `make <verb>` on podman → the minimal package `[the build stress-test]`

The vision's "build" arrow is not abstract: the ecosystem already has a build interface — **`make <verb>` over
podman** (AGENTS-STANDARD: stock tools, container-default, the verb vocabulary `build`/`test`/`lint`/`fmt`/`check`/
`clean` with a `-native` opt-in). Stress-testing the supply chain against it surfaces both a convergence and a gap.

**The convergence (why it feels close to mapped out).** The existing build convention is *already* most of a
content-addressed build:

- **podman / OCI images are content-addressed by digest** — the build *environment* is already a content-addressed
  object; pin the base image by **digest, not tag** and the environment becomes a hash.
- **`make <verb>` is the fixed, deterministic recipe** — a small verb vocabulary, no bespoke toolchain manager
  (the ecosystem deliberately avoids `mise`/`just`).
- So the build *inputs* (source-revision hash + base-image digest + toolchain version) and the *recipe* (the verb)
  are both already pinnable by hash. **The build is a function of hashes the moment you pin the base image by
  digest** — which is exactly what the supply chain needs.

**The minimal build packaging.** For a native service (the connection node), the smallest package that satisfies
the vision is:

```
package = {                      ; type = system/package  — the LANE-B foreign-artifact on-ramp (post-review scope)
  source_ref,                    ; { subgraph_type, hash } — a typed ref into a hashed subgraph (git | revision | …), §3a
  build_env_digest,              ; the podman base-image digest (pinned, content-addressed)
  build_recipe,                  ; the make verb + toolchain version
  target,                        ; linux-x86_64 | wasm32 | oci-image | ...
  artifact,                      ; Descriptor { type, hash, size } — content-hash + size (OCI DoS guard)
  capability_manifest,           ; imports / offers — the admission contract (offered, run-env)
}
; PROVENANCE (who/how) is NOT here — it is a separate signed system/attestation bound to artifact.hash (§3, §4)
```

It names its inputs by hash, its recipe, and its output by hash (with `size`). **Provenance moves out** to a
separate `system/attestation` (in-toto-shaped, `subject = artifact.hash`) — so "who built this, how" is an
attach-by-reference claim, not a field, and many attesters can vouch for one artifact. A **second builder** with the
same `source_ref` + `build_env_digest` + `build_recipe` MUST reproduce `artifact.hash` — the free-verification
property (§4). **For Lane A (entity-native) there is no package** — the subtree *is* the artifact (§3a); `system/
package` exists only to bring *foreign* build outputs into the content-addressed world.

**The gap — hermeticity ≠ reproducibility.** A `make build` in a podman container is *hermetic* (isolated from
host state) but **not automatically bit-reproducible**: containers embed timestamps, absolute paths, dependency
ordering, and can fetch from the network mid-build. The vision's "reproducible → independently verifiable" rests on
closing this, and it is **discipline, not a new mechanism**: pin the base image by digest; pin the toolchain;
`SOURCE_DATE_EPOCH` + strip timestamps/paths; **no network in the build step** (vendored / lock-pinned deps —
already the ecosystem's supply-chain-conscious default, AGENTS-STANDARD); deterministic, stripped output. This is
the §6 "assumed-unverified" flag made concrete and actionable.

**Two build lanes, one package (a refinement to §2/§4).** "Build" covers two paths with two reproducibility bases;
the supply chain handles both and they emit the *same* `package` entity:

- **Lane A — entity-native compute build.** Compute IR → materialized artifact *inside* the deterministic compute
  engine (§12.4). Reproducible **by construction** (corpus-proven; no podman needed). The endgame for compute-IR
  artifacts.
- **Lane B — native toolchain build.** `make <verb>` (cargo/rustc) on podman → a native binary / WASM / OCI image.
  Reproducible **by hermetic discipline + a second builder**, not by the compute engine. **The connection node is
  Lane B.**

The `package` entity is the stable interface across both — so the trajectory *podman-hermetic-build now →
entity-native-build later* is an implementation migration under one contract, not a redesign.

**One new verb.** The supply chain adds a single verb to the vocabulary: **`make package`** — build (Lane B) then
emit the signed `system/package` entity into the content store. Optionally **`make verify`** — rebuild and assert
`artifact_hash` matches (the second-builder / reproducibility check). Everything else is unchanged; `package` is
`build` + hash + manifest + sign.

---

## §3 What is genuinely new (the assembly + a few missing wires)

Honestly small, given §2 — this is why the vision is "coming into view," not "starting from scratch":

1. **The assembly itself** — declaring that source-hash → build → artifact-hash → distribute → run is **one graph
   with one trust model**, and pinning the seam between stages (what entity type each stage emits, what the next
   stage reads). No such document exists; this is it.
2. **`system/attestation` — the provenance plane (the genuinely-new work, post-review).** A signed, in-toto-shaped
   claim bound to a content digest: `{ subject: content-hash, predicate_type, predicate, size }`, IDENTITY-signed.
   The `subject` is the `artifact.hash` (a Lane-B package) *or* a `source_ref` (a signed revision) *or* any entity;
   the `predicate_type` is versioned (`build-provenance` / `test-vectors` / `sbom`). This is **attach-by-reference**
   (OCI `subject`+Referrers): provenance is a separate claim about immutable content, not a mutation of it, and many
   attesters can vouch for one artifact. `EXTENSION-REVISION` stays the (Active, unsigned) source mechanism; a signed
   revision is just an attestation whose subject is the REVISION root — uniform across subgraph types (§3a).
3. **`EXTENSION-PACKAGE` (`system/package`) — narrowed to the foreign-artifact on-ramp (Lane B).** It wraps a
   *non-entity-native* build output (cargo/podman binary, OCI image, WASM module) as a content-addressed entity +
   Descriptor + inputs + manifest (§2a) so it distributes and runs *uniformly* with native subtrees. **Provenance is
   NOT in it** (§2, moved to `system/attestation`). It lands as a **system component** (`system/*`, converging with
   `system/device` / `system/signaling` — infrastructure is system components, L5 is for apps). **For Lane A there
   is no package** — the subtree is the artifact.
4. **Distribution reconciles to `EXTENSION-SUBSTITUTE` (Active) — not new.** A package is a blob the substituter
   serves (the Nix-substituter model, already ratified: ordered sources, sign-content/trust-key/fetch-anywhere, no
   transitive trust). A signed **release index** is a REGISTRY §6a.7 manifest (`name/version → package_hash`),
   TUF-tiered (offline content key / online freshness key), resolver enforces **highest-seen `seq`**. The named-future
   `nix-cache` / `peer-to-peer` / foreign substitute types are the **bridge to external hashed subgraphs**.
5. **The `REPOS` L5 convention — the named-but-undesigned roadmap slot.** `ROADMAP-APPLICATIONS` `REPOS`
   ("repository hosting… zero design today; name-drops only") is the producing-end UX home (repos + releases + the
   browse seam) — the *app* layer over the `system/*` supply-chain components. This exploration is its design intake.

Everything else (containers/microVMs as run-environments, the scheduler, deploy) is **W-FLEET / run-env work
already in flight** — the supply chain *feeds* it; it does not re-specify it. **The four-noun frame** (tree ·
substituter · registry · manifest; `ANALYSIS-SUPPLY-CHAIN-LANDSCAPE-ALIGNMENT` §2) keeps these separated.

## §3a Building it outside-in — the outer loop first, the source end last `[the enabler for "start now"]`

The pipeline (§1) reads left-to-right, but it is **built right-to-left (outside-in)** — and that is what lets us
start building/running packages *now*:

- **The `package` names its source by a *typed hash reference*, not a fixed format.** `source_ref` (§2a) is
  `{ subgraph_type, hash }` — a reference into **any content-addressed ("hashed identity") subgraph.** Git and
  entity-native `EXTENSION-REVISION` are two supported types: a git commit/tree hash and a REVISION root are *both*
  content-addressed DAG nodes. Others (a bare content-store tree, an external DAG) can be added. **Git is
  first-class-supported, not a crutch to be replaced** — the system references and verifies an external hashed
  subgraph as readily as its own.
- **Hashed subgraphs unify because content-addressing is the general case.** `tree = path→hash` + dedup is exactly
  what git, REVISION, and the content store each *are*; the entity system is the superset. So "source in git" and
  "source in REVISION" are the **same kind of thing** — a signed reference into a hash DAG — and the package format
  is stable across all of them. `EXTENSION-REVISION` is **already Active** (the mechanism — content-addressed
  version DAG, lineage, merge, sync); a git-analog CLI ("entity-repo", `GUIDE-SHELL-FRAMING` §9.2) is only **thin
  UX over it, already partly present in `workbench-go`**. So entity-native source is *near*, not a distant new
  system — the difference between "entity-repo" and REVISION is CLI polish, not mechanism.
- **Therefore the outer loop goes entity-native first, and the source end is pluggable, not sequential.** The new,
  high-value part (`system/package` + distribute-by-hash + run) depends only on the content store (Active) + the
  package wire. *Which* hashed subgraph the source lives in — git, REVISION, both — is a **per-repo choice**, not a
  migration the running system waits on.
- **Trust holds across all subgraph types.** A hash (git or REVISION) + an IDENTITY signature is the same
  attestation; reproducibility (a second builder → same `artifact_hash`) is independent of the source subgraph.

The de-risking move: **stand up build → sign → distribute → run on existing primitives, prove the loop, and let the
source reference *any* hashed subgraph — git today, REVISION natively, both — with the package format unchanged.**
The concrete staging is `EXPLORATION-SUPPLY-CHAIN-EXECUTION-AND-REPO-ARCHITECTURE`.

---

## §4 The trust model — the ecosystem's own doctrine, applied end-to-end

The supply chain does not need a new trust story; it **inherits the one the ecosystem already lives by** (ADR-0012,
"conformance is the contract, not the version number"):

- **Verify by hash + signature + oracle, never by version.** A package is trustworthy iff (a) its bytes hash to
  the claimed `artifact_hash`, (b) a signature you trust covers it, and (c) it passes its pinned oracle/test
  vectors. "v1.2.3 from vendor X" is replaced by "the artifact whose hash is H, signed by K, that passes vectors
  V." This is §12.5's four trust postures (accept-signed / re-run-vectors / re-derive-locally) generalized.
- **Reproducible build = independent verification, for free.** Deterministic build (§2) means *two independent
  builders of the same `source_revision_hash` produce the same `artifact_hash`.* That is the **already-LOCKED
  "reproducible publish" property** (`APP-CONVENTION-SEMANTIC-CONTENT-SITE` `G-PIN-4`: two publishers → identical
  root hash) generalized from content sites to software. A second builder disagreeing on the hash is a
  supply-chain alarm with zero extra mechanism — the same "cohort-consistent vs independent convergence"
  discipline the ecosystem already applies to conformance (AGENTS-STANDARD).
- **Intrinsic provenance — the graph is the SBOM.** Because every input is named by hash, a package transitively
  names its exact source revision, which names its exact parents, which name their content. The **bill of
  materials is the transitive closure of the hash graph**; you don't emit an SBOM, you *walk* one. A signature at
  any node attests everything below it.
- **Dedup is the cache and the CDN.** Source, build inputs, and artifacts share one content store, so a byte
  present anywhere is present everywhere once — the artifact cache, the mirror, and the CDN are the same dedup'd
  store (§2).

---

## §5 Closing the loop with W-FLEET

This is the producing end; **W-FLEET is the consuming end**, and they compose into the full circle:

```
REPOS (signed revision repo) → build → signed package → REGISTRY signed release index → CONTENT/SUBSTITUTE (fetch by hash)
                                                                                              │
   W-FLEET: system/device.offered admits the package's capability manifest → run-env/W-HOSTING runs it → managed over the connection
```

So the connection node (W-FLEET's first managed service, and a service *built from source*) becomes the first
end-to-end demonstrand of the whole circle: **its own binary is a Lane-B `system/package` — `make package` in a
digest-pinned podman image over a signed revision of the node repo (off core-rust) — distributed by hash, admitted
on its manifest, and run + managed on the fleet.** The smallest real instance of the entire vision is: *one repo,
one hermetic reproducible build, one signed package, one box* — and it is buildable on today's `make`+podman
convention plus the small `system/package` wire (§3).

---

## §6 The honest ledger

- **Exists (§2):** the seven primitives — verified present by the 2026-07-24 survey (paths in §2). Not to be
  rebuilt.
- **New but small (§3):** the assembly + signed-revision wire + the `package` entity + artifact-targeted signed
  manifest + the `REPOS` convention. Each is a **composition or a named-slot fill**, not a research problem.
- **Deferred / not-yet (do not pull in early):** the run-environment driver family (process/container/microVM/WASM)
  and the scheduler are **W-FLEET / run-env** work, consumed here, not specified here; multi-builder reproducibility
  attestation quorums (`EXTENSION-QUORUM` applied) are a hardening tier, not v1; a full DSL/frontend is premature
  (`ANALYSIS-COMPUTE-CONSTRUCTION` Rec A — finish the lowering toolkit first).
- **Assumed-unverified (flag on sight):** "deterministic build → identical artifact hash" holds only as far as the
  build is actually deterministic across environments — the same cross-impl determinism the compute corpus
  proved for IR evaluation must be shown for whatever build path emits native/WASM artifacts. For **Lane B**
  (make+podman) this is **hermetic-build discipline**, not a guarantee (§2a: pin base image by digest, no network,
  `SOURCE_DATE_EPOCH`, stripped output). Reproducibility is a *claim to be tested by a second builder* (`make
  verify`), not an assumption.
- **The discipline (the CDN-corridor meta-rule):** none of this is real until a cross-impl run exercises it — two
  independent builders producing a bit-identical artifact hash from one signed source revision, that a third party
  admits on its manifest and runs. Prose review does not catch a reproducibility gap.

---

## §7 Where this sits

- **New unifying arc — the supply chain — connects the producing end (`REPOS`, REVISION, build, package) to the
  consuming end (W-FLEET / W-HOSTING).** It is not a new lineage disconnected from the movement; it is the
  **capstone that the content-addressing + identity + revision + compute + hosting + fleet tracks were all
  building toward.** Tracked as a registered arc in `WORKSTREAMS.md`.
- **First milestone rides W-FLEET's:** the connection node as the first signed-package-built-from-a-signed-repo
  service on the DO box — the whole circle at the smallest scale.
- **Design intake home:** the `REPOS` L5 convention (`ROADMAP-APPLICATIONS`) — this exploration is its seed.

## §8 References (in-repo)

**Primitives (Active/specified):** `EXTENSION-REVISION` (v3.8, the source DAG), `EXTENSION-CONTENT` /
`EXTENSION-TREE` (store + dedup), `EXTENSION-IDENTITY` / `-ATTESTATION` / `-QUORUM` (signatures), `EXTENSION-REGISTRY`
§6a.7 (signed manifest), `EXTENSION-SUBSTITUTE` (fetch-by-hash distribution), `EXTENSION-COMPUTE` (determinism).
**Design records:** `EXPLORATION-COMPUTE-PROGRAM-RUNTIME-CONTRACT` §12.4-12.5 (compiled-artifact-as-entity + trust
postures), `EXPLORATION-RUN-ENVIRONMENTS-AND-WORKLOAD-ACTUATION` (capability-manifest admission),
`EXPLORATION-EXECUTION-SUBSTRATE-ORCHESTRATION` (run-env family), `APP-CONVENTION-SEMANTIC-CONTENT-SITE` `G-PIN-4`
(reproducible publish). **Consuming end:** `EXPLORATION-P2P-CONNECTIVITY-AND-OVERLAY` + W-FLEET / W-HOSTING
(`WORKSTREAMS.md`). **Roadmap slot:** `ROADMAP-APPLICATIONS` `REPOS`. **Guides:** `GUIDE-SHELL-FRAMING` §9.2
(`entity-repo`), `GUIDE-REFERENCE-DEPLOYMENT` (the static reef).

*The one sentence: every primitive for a content-addressed software supply chain — a signed git-like revision DAG,
a dedup'd content store, a signed-graph attestation substrate, a TUF-shaped signed manifest, a compiled-artifact-
as-entity model, and a working content-addressed CDN — already exists in this repo; the capstone is to assemble
them so that source-hash → reproducible-build → signed-package → distribute-by-hash → admit-and-run-on-the-fleet is
one content-addressed graph whose hash is its provenance, whose signature is its attestation, and whose second
independent builder is its verification.*
