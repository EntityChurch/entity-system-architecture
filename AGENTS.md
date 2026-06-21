
# entity-system-architecture

Read **AGENTS-STANDARD.md** first. This file adds entity-system-architecture (the spec) specifics.

## Overview

This repo is the **conceptual architecture + specification development** for the
Entity Core Protocol — the optional capability layer above the core: the extension
family (`EXTENSION-*`), the SDK conventions, the L5 application conventions, and the
developer guides. It is for *designing* the system, not implementing it; final specs
transfer to the implementation repos. The spec-style tooling that keeps the corpus
coherent (the stdlib-only Python `spec` linter) ships separately in the
`entity-system-arch-tools` repo. This repo is the **spec authority**:
ratifications, retractions, and corrections happen here; the meta-repo only
observes (don't back-sync to meta). You + this session are the architecture team —
make the spec/op-set/host-type calls; the only external handoff is to the
implementation cohort for cross-impl review + build.

## Setup / environment & build & test

- **No build here.** This repo is prose specs + guides; there is no compiler or app
  test-runner. Spec work is authored and reviewed as text.
- **Spec linter / gates live in the sibling `entity-system-arch-tools` repo** — a
  stdlib-only Python `spec` CLI (`make check` / `make style` / `make standards`, a
  hermetic `make check-podman`, and readers `make render`/`tree`/`topology`). Point it
  at this corpus to lint style/standards/topology; it is the contributor pre-submit and
  the post-V8 CI conformance gate. See that repo's README for invocation.
- **Naming/format are still enforced** — `specs/STYLE-NAMING-CONVENTIONS.md` and
  `specs/SPECIFICATION-FORMAT.md` are the normative authoring rules the linter checks.

## Code style (spec authoring)

- **proposal → ratify → fold** lifecycle (see AGENTS-STANDARD): a change starts as a
  DRAFT proposal, ratify = edit the spec **and** mark the proposal implemented; a
  well-reviewed proposal **lands as a versioned spec file** (e.g.
  `specs/extensions/EXTENSION-REGISTRY.md v1.0`) — don't manufacture extra review
  cycles. "Ratified ≠ folded": the spec edit is the second half, and the spec header
  is source of truth. Cohort impl findings **fix the spec in place** (no rev bump)
  once landed. (The proposal/review workspace lives in the internal authoring repo;
  this mirror carries the folded specs.)
- **Naming convention** (`STYLE-NAMING-CONVENTIONS.md`, normative; the linter
  enforces it) — domain-invariant split: **kebab** for the namespace (entity-type
  path segments, operation names, enum/string values); **snake_case** for
  data-structure keys (field/map-key names, error & status codes);
  **SCREAMING_SNAKE** for wire message constants (`HELLO`/`EXECUTE`); **UPPERCASE**
  for pseudocode terminals (`DENY`/`ALLOW`). Rule of thumb: key is left-of-colon
  snake, value is kebab.
- **Terminology** — say **"extension"** / "system extension" (never "actualizer");
  **"core"** not "kernel" in normative text (kernel is the newcomer intuition only);
  reserve **"bootstrap"** for true self-host/type-system bootstrapping — use
  startup/init/hydrate/rebuild for ordinary peer/extension start-up.
- **No backward compatibility.** No installed base — write every change as the only
  design; no legacy paths, migration windows, dual-kind acceptance, or mirror-writes.
- **Stay in the spec lane.** STATUS/proposals record the spec delta + the MUST/SHOULD
  + the owning team for any follow-on — not impl-execution checklists. Don't track
  what impl teams owe.
- **Pin the cross-impl-observable surface, leave internals to converge.** A `MAY`/
  `SHOULD` whose two conformant readings diverge *across a peer boundary* is a latent
  interop bug — lean MUST (prefer a general determinism MUST over a point-scope).
- Decisions made in conversation must be tracked **on disk in executable form**
  (target file + section + verbatim edit) before any deferral or context-clear —
  transcripts are not durable.

## Project structure

This published mirror is the flattened spec surface — the working directories of the
internal authoring repo (proposals / reviews / explorations / versioned spec lines)
are not part of it. What ships here:

- `specs/` — the core model docs plus `specs/extensions/EXTENSION-*.md` (the extension
  family), `specs/sdk/SDK-*.md`, `specs/applications/` (L5 conventions), and
  `specs/domains/`. Extension specs are **versioned** (e.g. `EXTENSION-REGISTRY.md v1.0`);
  the spec header is source of truth.
- `guides/` — **user-facing only** developer how-to and discipline docs (`GUIDE-*`).
- `specs/SPECIFICATION-FORMAT.md` + `specs/STYLE-NAMING-CONVENTIONS.md` — the normative
  authoring standards; `ROADMAP-{EXTENSIONS,SDK,APPLICATIONS}.md` — the living roadmaps.

The **proposal → ratify → fold** lifecycle (above) plays out in the internal authoring
repo; this mirror carries the folded result — the specs themselves.

## Boundaries — do NOT modify

- **Ratified spec decisions are the historical record** — supersede via a new spec
  revision, don't silently rewrite a landed decision.
- The **locked V7 wire core** is never renumbered (see AGENTS-STANDARD / ADR-0002).
- Superseded spec lines (`v1.0`/`v2.0`/`v3.0`) are frozen reference history; current
  work is V7.

## Load-bearing invariants (FOREGROUND)

Before touching any spec or reasoning about wire/storage/path behavior, internalize the
five load-bearing invariants implementers (human and LLM) keep re-deriving and
mis-implementing — each has already caused a cross-impl bug that **passed prose review**
and was caught only by an implementation or a conformance test:

1. `tree = path → hash` (**not** path → entity)
2. content-store **dedup**
3. universal address space + **local-view authority** model
4. **absolute paths at every layer** (don't double-qualify)
5. the **two HTTP mechanisms** (CDN is *not* a special category)

The model is correct but *scattered* — the canonical normative homes are V7 §1.4/§1.7,
`specs/extensions/EXTENSION-TREE.md` §1, `specs/extensions/EXTENSION-NETWORK.md` §6.5,
and `guides/GUIDE-EXTENSION-DEVELOPMENT.md` §3.7.

**Meta-rule (the CDN-corridor lesson):** a normative claim about what bytes a route
returns / what a path resolves to / what dedups is **not validated until a
cross-impl conformance test exercises it** — prose review does not catch these.
Spec changes are exercised by the multi-language **cohort** (Go / Rust / Python +)
via a conformance run before/at fold.

**Recurring failure shapes** to expect and guard against — keep the durable catalogs
in-repo, not in memory:

- **Cross-peer seam equivalence-collapse:** an equivalence/discovery claim that
  holds locally and for the issuer silently collapses distinctions that spring apart
  at a cross-peer / cross-consumer seam — *masked* because the conformance tests
  assert the issuer's convention. State the invariant **once, generally, with the
  local collapse called out**; never patch instances. (Landed instances: V7 §5.2
  three-slot model, §3.5 discovery-locality, §5.8 cross-peer chain registry.)
- **CVE-style cross-impl bug classes (A–F):** the cohort maintains a living ledger of
  these in the internal authoring repo. When one impl finds an instance, **every** impl
  audits its own surface against that class's prompt; spec-defects route back as a spec
  revision.
- **Cross-impl interop pitfalls** (durable home: `guides/GUIDE-EXTENSION-DEVELOPMENT.md`)
  — check when a spec change touches cross-peer behavior or wire
  encoding: (a) a `MAY`/`SHOULD` that diverges across a peer boundary → pin it;
  (b) **re-encoding on receive/forward is THE interop hazard** — prefer byte
  preservation (§1.8); a re-encoder MUST produce bit-identical canonical ECF;
  (c) cumulative state in matrix testing creates false signals — run fresh peers per
  directional pair; (d) cbor2 float16 minimization is a known Python Rule-4 gap —
  flag before shipping any float-carrying entity type.
