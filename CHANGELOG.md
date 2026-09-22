# Changelog

All notable changes to this project are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project aims to follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

**Measured against the published tree on 2026-09-20: 91 of the 217 published paths differ.**
Seventy were rewritten in place — 26 extension specs, 15 guides, 3 application conventions, 2 SDK
specs, `SYSTEM-ARCHITECTURE`, both authoring standards and all three roadmaps, plus the agent
guidance and the design record — and 21 no longer answer at the address a reader holds: 18
proposals promoted from `active/` to `implemented/`, plus the three retirements below. The 0.8.2
entry describes the tree as it stood on 2026-08-24, and the specification has not stopped moving
since; read this section as the delta against it.

_This section said "Nothing yet." while those 91 paths had already moved. The count is what a
reader downloads, not what the commit log says, and the two are not the same measurement._

**What "breaking" means for a repository whose product is prose, and the surface it is measured
against.** There is no API here and no binary. **The public surface is `specs/` and `guides/`** —
the normative extension, SDK, bridge-extension and application-convention text, and the implementer
guides that teach it, each document versioned by its own header. **Out: everything under `docs/`**,
including the proposal and exploration corpus. That corpus publishes, because this repo's authoring
rule keeps rationale out of normative text and sends it there — but a design document is dated, is
superseded as the corpus moves, and travels between `active/`, `implemented/`, `superseded/` and
`deferred/` as the ordinary lifecycle, so **its path is not promised. Cite the spec; a proposal is
rationale, never authority.** Also out: `ENTITY-CORE-PROTOCOL`, published separately on its own
number.

So a break here is one of exactly two things — **a published `specs/` or `guides/` address stops
answering, or a normative obligation inside that text moved under an implementation already built
against it.** Both happened in this window, which is why what follows is a breaking heading rather
than an all-additive one.

### Changed in ways that can break an existing caller

- **Three published addresses stop answering at this cut.**
  - `specs/applications/CHARTER.md` → folded into **`guides/GUIDE-APPLICATION-DEVELOPMENT.md`
    v2.0**. The applications tier is now taught in one document rather than split between a
    charter and a guide.
  - `specs/domains/DOMAIN-LOCAL-FILES.md` → **`specs/bridge-extensions/DOMAIN-LOCAL-FILES.md`**
    (v1.5). Bridge extensions — those that project a non-entity system into the entity address
    space — now have their own directory and their own development guide.
  - `specs/ENTITY-SYSTEM-REFERENCE.md` → **retired, and not replaced.** It was a condensed
    restatement of the specs; a restatement with no declared authority drifts silently from what
    it restates, and every spec it condensed is published here in full.

  **A citation survives all three; a link does not.** Nothing normative was withdrawn — the text
  is still published, at a different address or inside a different document — but a reader holding
  one of these URLs loses it at this publish.

- **The normative surface moved under anything built against the 0.8.2 text.** Twenty-six
  extension specs advanced their own version headers in this window — `EXTENSION-TREE` 4.3 → 4.12
  on its own — and fifteen guides changed with them. **Each spec's own header and its conformance
  section remain the source of truth for what it requires**; this changelog records that the text
  moved, never what any individual obligation now says. An implementation that was
  conformance-green against the 0.8.2 corpus is not thereby conformant to this one, and a green
  result is scoped to the check set that produced it. Re-read the spec you implement.

### Moved outside the surface — not breaking, and recorded because a reader cannot tell

Under the surface above these are not breaks, and a file that quietly stops being published is
indistinguishable from a mistake, so they are written down rather than left to inference.

- **Eighteen proposals moved from `docs/proposals/active/` to `docs/proposals/implemented/`.** The
  documents are unchanged and still published; only the path differs, and a citation into `active/`
  is now dead. This is the proposal lifecycle working exactly as designed — a proposal folds most
  weeks — which is why proposal paths are excluded from the surface rather than promised and then
  broken.
- **`docs/DISCIPLINE-CHARTER.md` is no longer published.** It is this team's internal rulebook and
  it stays in the tree, gated and authoritative for that rule set; it stopped being addressed to a
  reader outside this ecosystem. The contributor-facing half was always its own document —
  `CONTRIBUTING.md`, which is unchanged.

### Known limitation, re-measured — commit citations that do not resolve

The 0.8.2 entry below reports **82** unresolvable commit citations. **That number is now 705, and
none of the growth is new citations.** Published history is authored fresh at the release boundary
([ADR-0027]), so an internal commit hash has never been reachable from public `master` — and the
measurement is scoped to what this repo *declares*, which grew from 88 documents to 324 when the
design record was declared as a published tree. Publishing the surface rather than the count alone,
since the count inherits the shape of the search that produced it:

| Where | Citations a reader cannot resolve | Files |
|---|---|---|
| `docs/proposals/` | 569 | 71 |
| `docs/research/explorations/` | 57 | 17 |
| `specs/` | 40 | 12 |
| root prose + `docs/` | 23 | 3 |
| `guides/` | 16 | 2 |

**Fifty-six of the 705 are on the surface this repo promises to keep**; the rest are in the design
record, where a dated internal hash is native to the genre and the fix is a rewrite, not a sweep.
**None is load-bearing for implementing a spec** — they are rationale pointers. Anchors on content
digests are stable across the boundary and are what the sweep converges on. Measured with
`spec pins`.

### Added

| Document | At |
|---|---|
| `specs/SYSTEM-DATA-EXCHANGE.md` | v0.3 |
| `specs/applications/APP-CONVENTION-FEED.md` | v0.3.1 |
| `specs/applications/APP-CONVENTION-REFERENCE.md` | v0.1 |
| `guides/GUIDE-APPLICATION-DEVELOPMENT.md` | v2.0 — absorbed the applications-tier charter |
| `guides/GUIDE-BRIDGE-EXTENSION-DEVELOPMENT.md` | — |

### Changed — the extensions that moved most

`EXTENSION-TREE` 4.3 → **4.12** · `EXTENSION-COMPUTE` 3.27 → **3.32** · `EXTENSION-REGISTRY`
1.21 → **1.26** · `EXTENSION-HISTORY` 1.7 → **1.11** · `EXTENSION-RELAY` 1.2 → **1.5** ·
`EXTENSION-ROLE` 2.0 → **2.3** · `EXTENSION-CONTINUATION` 1.23 → **1.25** · `EXTENSION-REVISION`
3.12 → **3.14** · `EXTENSION-QUERY` 1.7 → **1.8** · `EXTENSION-NETWORK` 1.8 → **1.9** ·
`EXTENSION-SUBSCRIPTION` 3.18 → **3.19** · `EXTENSION-TYPE` 1.2 → **1.3** · `EXTENSION-CONTENT`
3.6 → **3.7** · `EXTENSION-DISCOVERY` 1.1 → **1.2** · `EXTENSION-SIGNALING` 1.1 → **1.2** ·
`EXTENSION-TRANSACTION` 0.1 → **0.2**.

Also `SYSTEM-ARCHITECTURE` 0.4 → **0.9**, `SPECIFICATION-FORMAT` 1.1 → **1.5**, `SDK-OPERATIONS`
1.11 → **1.13**, `APP-CONVENTION-SHARE` 0.1 → **0.2.1**, `APP-CONVENTION-SEMANTIC-CONTENT-SITE`
0.5 → **0.5.2**.

**Each spec's own header remains the source of truth for its version** — this list is a copy of
those headers, and where the two ever disagree, the header wins.

### Note on maturity, restated because it matters more at this size than at the last cut

The corpus is **current, not final**, and it is deliberately published in that state. The core
protocol it layers on is at **0.8.2.32** and revisions are still landing against text that is days
old. Two specific things a reader should weigh before depending on a number:

- **Conformance results trail the specification text, and a passing result is scoped to the check
  set that produced it.** Checks are written against a pinned revision of the spec, so a green
  result is silent about any obligation that set does not drive. A spec's conformance section
  states what is exercised; the version header does not.
- **The normative surface grew substantially across this window.** Between 0.8.2 and 0.8.2.25 the
  core protocol's count of MUST / MUST NOT obligations rose by roughly 45%, and the extension text
  layered above it moved with it. Text that was reviewed against the earlier surface has not
  necessarily been re-reviewed against the later one.

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

_**This table describes the 2026-08-24 tree and four of its rows have since moved.** It is left as
written rather than corrected in place: this entry is the record of what that cut contained, and
editing a shipped count to match a later tree is how a changelog stops being one. The current
surface is `specs/applications/` **5**, `guides/` **37**, `specs/domains/` **retired** in favour of
`specs/bridge-extensions/` **1** — see `[Unreleased]` above. The directory listing is the arbiter
for any of these counts._

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
