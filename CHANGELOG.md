# Changelog

All notable changes to this project are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project aims to follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

Nothing yet.

## [0.8.2] — 2026-08-24

**Prepared for the research-preview publication alongside `entity-core-protocol` 0.8.2.** This repo is
the **optional capability layer above the core** — it does not carry the core protocol, which is
published separately and is upstream of everything here.

### What is published

| Surface | Count | Notes |
|---|---|---|
| `specs/extensions/EXTENSION-*.md` | 26 | Independently versioned; the spec header is source of truth |
| `specs/sdk/SDK-*.md` | 3 | Restate extension schemas so an SDK author reads one document |
| `specs/applications/` | 4 | L5 application conventions |
| `specs/domains/` | 1 | |
| `guides/` | 35 | User-facing developer how-to, discipline docs, and two operator runbooks |

Plus the normative authoring standards (`SPECIFICATION-FORMAT.md`, `STYLE-NAMING-CONVENTIONS.md`) and
three living roadmaps.

### Maturity — read this before depending on anything here

**Extension maturity is per-spec and is not uniform.** Some extensions are implemented and
conformance-green across three independent implementations; others are specified and built nowhere.
**The spec's own version header and its conformance section are the source of truth for which** — a
document existing in `specs/` is not a claim that it ships.

- **Conformance is the contract, not the version number.** Every published conformance figure is
  reproducible and oracle-pinned; a bare percentage is not a claim this project makes.
- **`EXTENSION-REGISTRY` v1** is the surface with the most cross-implementation exercise — its v1
  release line, what is in it and what is deferred to v2, is stated in the spec and on the cohort
  ledger rather than implied by the version.
- Several backends named in the resolution model (`dns-txt`, `well-known-url`, `did-web`,
  `consensus-anchored`) are **paper** and say so where they are defined.

### Changed on the published surface

- **Four documents landed after the last release and appear here for the first time:**
  `EXTENSION-SIGNALING` (the NAT-traversal extension, 2026-07-31),
  `APP-CONVENTION-SHARE` (v0.1 DRAFT — read the version header),
  `GUIDE-REFERENCE-DEPLOYMENT` and `RUNBOOK-CDN-BROWSER-DEPLOYMENT`. All four are now also
  declared in `CANONICAL-DOCS.toml`, which had gained no document since 2026-06-30 — so the
  counts in the table above, which this file had been publishing, finally match the
  declaration.
- **Five community-health documents are declared for the first time** — `CHANGELOG.md`,
  `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `CLAUDE.md`. They were already
  public and declared nowhere, which meant a release would have **removed** them. Verified by
  running the publication filter over an exported tree, before and after.
- **The `egui` name is gone from the specs and guides.** It was this project's browser peer
  under an earlier internal name, and `egui` is an unrelated third-party Rust GUI library
  this project no longer uses — so the name pointed a reader at the wrong thing entirely.
  It now reads `entity-browser-rust` throughout. The four places where `egui` genuinely
  means that library, as one example among raylib and game loops, are unchanged.
- **The status document moved to `docs/STATUS.md`** (was `docs/status/STATUS.md`), per [ADR-0031].
  `docs/status/` now publishes nothing: the dated snapshots there are internal working memory, and
  separating the two kinds by **path** rather than by filename is the point of the move.
- **`AGENTS.md`, `AGENTS-STANDARD.md` and `METHODOLOGY.md` are published**, by operator ruling. They
  are written for any agent, not only Claude, and they carry this repo's working discipline.

### Known limitations at publication

- **Commit citations in published documents may not resolve for a reader of this repo.** Published
  history is authored fresh at the release boundary ([ADR-0027]), so an internal `dev` commit hash
  has never been reachable from public `master` — it is not degraded by the release, it was never
  valid for this reader. **82 such citations remain**, concentrated in `AGENTS.md`, `GUIDE-CONFORMANCE`
  and `EXTENSION-NETWORK`; the sweep is in progress and anchors on content digests, which are stable
  across the boundary. Measured with `spec pins`. **No such citation is load-bearing for implementing
  a spec.** Four of them, in `AGENTS.md`, are quoted *because* they are dead — they are the worked
  example of this exact failure and are not defects to fix.
- **Guides cite authoring documents that are not published here.** The proposal and exploration
  corpus lives in the authoring workspace; **255 citations across `specs/` and `guides/` resolve to
  115 documents a reader of this repo cannot open** (86 recoverable from the legacy corpus, 29 are
  renames or forward references). Measured with `spec address`; the worklist is on the board. **No
  citation is load-bearing for implementing a spec** — they are rationale pointers — but they are
  dead links until that migration lands.
- **Process narrative in normative text** is being paid down file by file under a ratcheted baseline
  (`.spec-baseline.json`, which only ever lowers). `EXTENSION-NETWORK` carries the largest remaining
  share.

### Tooling

The spec linter and gates ship separately in `entity-system-arch-tools` — stdlib-only Python, no
third-party dependencies. `spec check` (style + standards + coherence), `spec address` (citation
integrity), `spec pins` (does a citation resolve for the reader it ships to), `spec ledger`,
`spec sdksync`, `spec coverage`, `spec provenance`.
