# Entity System — Architecture Overview

**Version**: 0.9
**Status**: Active
**Audience:** Architecture team, paper team, implementation teams (core + platform). Future audiences as scope expands: implementers getting started, spec reviewers, developers targeting the system, paper readers cross-referencing specs.

---

## 1. What This Document Is

This is the navigable map of the entity system's architecture. It answers questions like "how do the pieces fit together," "where do I look for X," "what's structural vs. what's a particular implementation choice," "what layer am I working in."

It is **not** a spec (see `specs/` for those), **not** a tutorial (nothing here tells you how to build an application), **not** a paper (see the paper repo for the analytical treatment and public-facing material). It's a reference for people navigating the system, staying oriented across its layers, and understanding where their own work fits.

Current scope: internal. The document will expand into multi-audience guidance as the system matures — see §10.

---

## 2. Six-Layer Architecture

The system has a layered structure with clear responsibilities per layer and clear interfaces between them. Layers are stacked bottom-to-top in dependency order: each layer depends on everything below, and nothing above.

```
┌─────────────────────────────────────────────────────────────┐
│ L5: APPLICATIONS  ·  BRIDGE EXTENSIONS                      │
│   Applications: user-facing software. Games, tools,         │
│   front-ends. Combinatorial, problem-specific, infinite.    │
│   Bridge extensions: adapters onto foreign technologies.    │
│   Same radius, different bearing — applications build UP    │
│   on the system, bridges reach OUT of it. Neither is        │
│   built on the other.                                       │
├─────────────────────────────────────────────────────────────┤
│ L4: SDK LAYER 3 — SHARED PATTERNS                           │
│   Composition patterns made executable. Pipeline builder,   │
│   bridge framework, reactive bindings, sync patterns.       │
│   Must produce identical entities across implementations.   │
│   = Formalized form of guides.                              │
├─────────────────────────────────────────────────────────────┤
│ L3: SDK LAYER 2 — PROTOCOL FACADE                           │
│   Per-language ergonomic wrappers over protocol ops.        │
│   Rust builders, Go functional options, Python context      │
│   managers. Language-idiomatic — variation expected.        │
├─────────────────────────────────────────────────────────────┤
│ L2.5: SYSTEM EXTENSIONS                                     │
│   Each owns ONE `system/<name>` namespace segment and       │
│   exercises a specific pair-bundle of primitives.           │
│   Composable and deployment-selectable.                     │
│   The roster is §13.1 — deliberately not restated here.     │
├─────────────────────────────────────────────────────────────┤
│ L2: SYSTEM-COMPOSITION (COORDINATION LAYER)                 │
│   Consumer ordering on emit pipeline, cascade depth,        │
│   convergence classes, well-behaved-consumer patterns.      │
│   Specifies how multiple actualizers compose over shared    │
│   pair-surfaces (ITM, TMX, XP).                             │
├─────────────────────────────────────────────────────────────┤
│ L1: CORE PROTOCOL                                           │
│   Our particular specification of the weighted K₆ graph.    │
│   Six primitives (E, I, T, M, X, P), their pair-            │
│   relationships, dispatch, capabilities, emit primitive,    │
│   type fixed-point. Internal categories: (a) structural,    │
│   (b) structural instantiation, (c) gradient-into-          │
│   extensions.                                               │
├─────────────────────────────────────────────────────────────┤
│ L0: ALGORITHM LIBRARY                                       │
│   Standardized algorithms. CBOR canonical encoding,         │
│   SHA-256, Ed25519, Base58, content chunking,               │
│   Merkle trie operations. Shared by L1, L3, L4.             │
│   Vendored per language to reduce dependency risk.          │
├─────────────────────────────────────────────────────────────┤
│ L_native: PLATFORM BOOTSTRAP                                │
│   Evaluator, primitive I/O, platform-specific bindings.     │
│   A few hundred lines per platform. Irreducible minimum.    │
└─────────────────────────────────────────────────────────────┘
```

**The architecture team's direct responsibility is L1 + L2 + L2.5.** L0 is shared across teams (spec-fixed algorithms, architecture-team-influenced via core protocol but implemented per-language). L_native is per-implementation. L3 + L4 are being discovered by platform teams and will crystallize into an SDK spec; no dedicated SDK team yet. L5 is out of scope (and will always be).

---

## 3. Primitives, Pair-Relationships, and Triangles

The mathematical structure underlying L1. Developed in detail in:
- The primitives paper and the framework synthesis, in the paper repository (§7.2 —
  not published; the analytical treatment of this structure lives there)
- `EXPLORATION-PRIMITIVES-AND-PAIR-RELATIONSHIPS` — the architecture-side exploration,
  four amendments, in this repo's authoring workspace (§7.1 — not published)

### 3.1 Six primitives

| Symbol | Name | Shape |
|---|---|---|
| E | Entity | `{type, data}` |
| I | Identity | `hash = SHA256(ECF(type, data))` |
| T | Tree | `path → hash` |
| M | Emit | Atomic state crossing (Store + Bind + Event) |
| X | Execution | Typed dispatch via `EXECUTE` |
| P | Peer | Identity + capabilities + connection |

Dependency DAG: `I → E`, `T → I`, `M → T`, `X → T`, `P → I`, `P → X`. These constrain which subsets are internally coherent.

### 3.2 Fifteen pair-relationships

Every pair of primitives forms a structural coupling. Load distribution:

- **Heavy (11):** EI, IT, ET, IM, TM, EX, IX, TX, MX, TP, XP
- **Medium (1):** IP
- **Light (2):** EM, MP
- **Negligible (1):** EP

T and X are load-bearing primitives (5 heavy pairs each). P is lightest (2 heavy + 1 medium). Design changes to T or X ripple across 5 pair-strengths.

### 3.3 Five named triangles

Three-primitive subsets with irreducible triple-level content:

| Triangle | What it is |
|---|---|
| **EIT** | Self-description fixed point (`system/type` of type `system/type`) |
| **ITM** | Emit triangle — Store-event (IM) and Tree-change event (TM) over tree-binding-to-content (IT) |
| **TMX** | Reactive dispatch — the compute cascade closing M→X→M |
| **IXP** | Cryptographic capability transfer — content-addressed capabilities across peers |
| **TXP** | Distributed dispatch — peer-namespaced tree walk (HTTP's structural shape) |

**Two triangles are over-subscribed:**
- **ITM**: HISTORY, QUERY, REVISION, SUBSCRIPTION (trigger), COMPUTE (trigger) all observe it.
- **TMX**: COMPUTE (closes the loop), SUBSCRIPTION (partial), CLOCK (adjacent).

Over-subscription explains where L2 SYSTEM-COMPOSITION engineering concentrates.

---

## 4. Transferability Classes

From the SDK exploration. Classifies what content can be moved between implementations/platforms.

| Class | Name | Meaning | Layer |
|---|---|---|---|
| **N** | Native | Platform-specific bootstrap. Non-transferable. | L_native |
| **S** | Spec-fixed | Standardized universal algorithms. Must be vendored per-language but must produce identical results. | L0 |
| **T** | Transferable | Entity-native computation. Runs anywhere given N + S + COMPUTE. | L2.5+ |
| **B** | Bridge | The COMPUTE extension. Enables Class T transferability. | L2.5 |

**Structural claim:** a peer with L_native + L0 + L1 + L2 + COMPUTE (Class B) can acquire everything else through entity exchange. The transferable-computation claim (Class T) depends on:
1. COMPUTE being present (Class B).
2. Both peers agreeing on Class S instantiation (CBOR, SHA-256, Ed25519, etc.).

**This is why L0 matters so much.** Category (b) content in the core protocol spec — the specific algorithmic and encoding choices — is the Class S interop floor. Not "confusion avoidance"; interoperability ground.

---

## 5. Multi-Level Interop

Interop is not a single property. It's a ladder; each rung gives something distinct.

| Level | Boundary | What agreement gives you |
|---|---|---|
| Mathematical coherence | L1 primitive structure | Same analytical picture; framework applies consistently |
| Class S interop | L0 + L1 category (b) | Data interchange — hashes match, wire frames parse, signatures verify |
| Class T interop | + Class B (COMPUTE) + L2 | Running-program transfer — send an entity-native program, run it elsewhere |
| Developer interop | + L3 SDK facade | Consistent programming experience per language (not cross-language) |
| Pattern interop | + L4 SDK patterns | Standard compositions produce identical entities across impls |
| Secondary convergence | L5 applications | Applications may compose if patterns followed (not guaranteed) |

**Key point:** isomorphism at the primitive level (mathematical coherence) doesn't give interoperability. Agreement on the concrete choices in category (b) / Class S gives data interchange. Each higher level requires additional agreement. Don't overclaim from mathematical structure alone.

---

## 6. Core Protocol Internal Layering

L1 has internal categories that should be kept distinct:

| Category | What it is | Examples |
|---|---|---|
| **(a) Structural core** | Specifies a primitive or pair-relationship directly. Without it, K₆ collapses. | §1.1 Entity, §1.4 URI/Path, §1.6 Wire Frame, §1.7 Storage, §2.1 Type Definition, §3.1–3.9 Protocol Types, §5 Capability model, §6.1/6.3/6.5–6.8 Handler model |
| **(b) Structural instantiation** | Concrete choices that make the structure usable. Other instantiations possible; we picked these. | §1.2 SHA-256/format codes, §1.3 ECF v1 CBOR rules, §1.5 Base58, §4 three-round-trip connection, §7 algorithms |
| **(c) Gradient-into-extensions** | Pedagogical/onramp content. Leads into extensions without being structurally required. | §1.9 Namespace design, §2.11 Type conformance levels, §3.10/§3.13 Types/operational-state types, §6.2 System handler conventions, §6.4 Domain handlers, §6.9 Bootstrap |

The core protocol is our **particular instantiation** of the K₆ structure, not the pure mathematical substrate. It deliberately includes (c) content because pure minimalism would make the spec unlearnable. The internal layering keeps the boundary between structural requirement and pedagogical onramp readable.

See `reviews/core/EXPLORATION-PRIMITIVES-AND-PAIR-RELATIONSHIPS.md` §12 for the full analysis.

---

## 7. Repository Map

High-level only. Things will move. Use file search over this map for current state.

### 7.1 Architecture repository (this repo)

*Post-split layout. The published surface is `specs/` + `guides/`; `docs/` is the authoring
workspace and the archive, and does not ship with the release (§13.6).*

```
specs/                                   ← the published normative surface
├── SYSTEM-ARCHITECTURE.md                   ← this file (arch-doc, informative)
├── SYSTEM-COMPOSITION.md                    ← L2 coordination layer
├── SYSTEM-IDENTITY-COMPOSITION.md           ← identity navigation doc (informative)
├── ARCHITECTURE-IDENTITY-INFRASTRUCTURE.md  ← identity architecture (informative)
├── SPECIFICATION-FORMAT.md                  ← spec authoring standard
├── STYLE-NAMING-CONVENTIONS.md              ← identifier naming standard
├── extensions/                          ← L2.5 extension specs — the entity system's own surface
├── bridge-extensions/                   ← extensions that reach OUT: one per foreign technology
├── sdk/                                 ← L3-L4 SDK specifications
└── applications/                        ← L5 conventions (tier standard: guides/GUIDE-APPLICATION-DEVELOPMENT.md)
guides/                                  ← user-facing GUIDE-*.md
ROADMAP-{EXTENSIONS,SDK,APPLICATIONS}.md ← the living roadmaps
docs/                                    ← workspace + archive; NOT published, one exception
├── STATUS.md                                ← the rolling log; the ONE published doc here
├── research/                                ← the design record
│   ├── explorations/                            ← research and analysis
│   ├── reviews/                                 ← cross-impl absorption
│   └── INDEX.md                                 ← gated by `spec ledger`
├── proposals/                               ← state = directory, tier = subdirectory
│   ├── active/{core,extensions,applications,process}/   ← work is owed
│   ├── implemented/                             ← the spec edit landed
│   ├── deferred/                                ← parked on purpose
│   ├── superseded/                              ← retired without landing
│   └── INDEX.md                                 ← gated by `spec ledger`
├── status/                                  ← dated handoffs / status; ephemeral
├── archive/                                 ← closed docs, with an INDEX.md breadcrumb
└── DISCIPLINE-CHARTER.md                    ← internal: how this team works, not part of the spec
```

**The four specification directories group by what a specification is ABOUT, not by document
kind.** Three of them name a layer in §2's stack; `bridge-extensions/` does not, because it is
not on that axis. **A bridge extension is an extension by artifact class** — it contributes
entity types, handler operations, storage conventions and a conformance section — **and it sits
beside the application tier by layer.** The separation from `extensions/` is deliberate and is
about the reader: **`extensions/` is the entity system's own surface; `bridge-extensions/` is how
the system reaches a technology that is not it.** A reader opening the first should find the
system; a reader opening the second should immediately see a spread of foreign technologies and
know that is what the directory is for. See `guides/GUIDE-BRIDGE-EXTENSION-DEVELOPMENT.md`.

**The upstream core spec is a sibling repo.** `ENTITY-CORE-PROTOCOL.md`,
`ENTITY-CBOR-ENCODING.md` and `ENTITY-NATIVE-TYPE-SYSTEM.md` live in
`entity-core-protocol`, not here. Citations to them
resolve only when an analyzer is given that root — see `AGENTS.md` on `spec address
--namespace-root`. *(`ENTITY-CORE-MACHINE-SPEC.md` was a fourth; it is **retired** and is not a
citable source — see `SPECIFICATION-FORMAT.md` §8.4.3.)*

**Pre-split note.** This repo previously nested everything under
`docs/architecture/v7.0-core-revision/` with `core-protocol-domain/` and `sdk-domain/`
subtrees. Those prefixes appear in no post-split checkout; citations carrying them were
normalized to the canonical `DOC.md` form (`SPECIFICATION-FORMAT.md` §11.2) on 2026-08-17.

### 7.2 Paper repository (sibling)

The paper corpus — the primitives paper, the framework synthesis, the core-boundary
analysis and the SDK ergonomics exploration — lives in a sibling repository that is
**not part of this publication**. Its documents are analysis of the system's structural
properties, not specification, and nothing here is sourced from them. They are named
where an argument depends on one; there is no path a reader of this repo can resolve.

### 7.3 Implementation repositories

The reference implementations and the application tier, each with an independent
lifecycle ([ADR-0010]):

- `entity-core-protocol` — the upstream core specification (§7.1 above)
- `entity-core-rust` — Rust core protocol + extensions
- `entity-core-go` — Go core protocol + extensions; also hosts the `validate-peer`
  conformance instrument
- `entity-core-py` — Python core protocol + extensions
- `entity-core-keystone` — the canonical conformance anchor
- `entity-core-formalization` — the formal models
- `entity-browser-rust` — the browser peer (Rust → WASM), the L5 web front-end
- `entity-workbench-go` — the Go workbench (TUI + GUI) and the Go SDK, `entitysdk`
- `entity-system-arch-tools` — the `spec` toolkit that gates this corpus

A Godot binding was an earlier third-implementation experiment and is archived.

---

## 8. Prototype Implementation Inventory

Current state (April 2026):

| Layer | Go | Rust | Python |
|---|---|---|---|
| L_native bootstrap | complete | complete | complete |
| L0 algorithm library | complete | complete | complete |
| L1 core protocol | complete | complete | substantial |
| L2 SYSTEM-COMPOSITION | complete | complete | partial |
| L2.5 extensions (7 standard) | complete | complete | partial |
| L2.5 extensions (transaction, compute, network) | not yet | not yet | not yet |
| L3 SDK facade | entitysdk package (Workbench) | EntitySDK/PeerContext (entity-browser-rust) | PeerBuilder (CLI) |
| L4 SDK patterns | emerging (Workbench) | emerging (entity-browser-rust, Godot) | not yet |
| L5 applications | Workbench (TUI + GUI) | entity-browser-rust, Godot demos | CLI tools |

The seven standard extensions (inbox, continuation, subscription, history, query, clock, revision) are implemented in Go and Rust with comprehensive test coverage. Go's `validate-peer` tool provides a 15-category conformance test suite. SDK specs (SDK-OPERATIONS.md, SDK-EXTENSION-OPERATIONS.md) formalize the converged L3 interface.

Details live in the respective implementation repositories; this document tracks only the high-level inventory.

---

## 9. Development Workflow

The prototype-to-spec process. Five actor roles today; two additional roles emerge as the system matures.

```
prototype → feedback → analyze → iterate → three impls → look for ambiguity → reconverge
     │                                          │
     └──────────── specs tighten ───────────────┘
```

### 9.1 Actor roles

**Implementation layer — two sub-layers:**

1. **Core + extension implementation teams** (Go, Rust, Python) — implement L_native, L0, L1, L2, L2.5 against specs. Hit ambiguities; raise questions via reviews and proposals.

2. **Platform / application-building implementation teams** — build standard-peer packages and front-ends (Workbench, entity-browser-rust, Godot) on top of core implementations. Take L2.5 and package into usable systems. Currently discovering the SDK by making design decisions they recognize should be standardized. *Not the SDK team* — they are implementation teams whose work is uncovering SDK needs.

**Coordination layer:**

3. **Architecture team** — maintains specs, answers ambiguities via proposals, audits extension orthogonality.
4. **Paper team** — analyzes the system's structural properties, maintains framework synthesis and public-facing papers.
5. **User thesis / steering** — catches drift, articulates framing, sets direction.

**Emerging actors:**

6. **SDK coordination** — SDK specs now exist (SDK-OPERATIONS.md, SDK-EXTENSION-OPERATIONS.md in `specs/sdk/`). SDK work is cross-team — platform teams discover patterns, architecture team formalizes them. A dedicated SDK team may form as the specs stabilize and implementation feedback accumulates.
7. **Application developers** — building L5 applications against the SDK + platform implementations. Currently platform teams are also building applications; segmentation happens as the SDK stabilizes.

### 9.2 Feedback channels

- **Implementations ↔ architecture:** proposals, reviews, ambiguity reports. Implementation ambiguities trigger spec tightening.
- **Architecture ↔ paper:** framework notes, exploration documents, synthesis. Analysis refines specs; specs ground analysis.
- **Platform teams ↔ SDK discovery:** the SDK is emerging from platform work. Guides are the current written form of what will become SDK Layer 3/4.
- **Everything ↔ steering:** the user thesis catches drift across all tracks.

### 9.3 Process evolution

The internal five-actor workflow can accommodate post-release RFC-style external input without structural change. External contributors become another source of proposals feeding the existing cycle. The documentation of the process becomes more important once external contributors are writing proposals; that's one reason this document exists.

---

## 10. How This Document Evolves

The current scope is internal — architecture + paper + implementation teams catching up on each other. As the system matures, this document expands into multi-audience guidance:

| Audience | What they need here |
|---|---|
| **Architecture team (current)** | Shared picture of where we are, what the open threads are, cross-team context |
| **Implementers getting started (near-term)** | Layer diagram, dependency order, where to start, what Class S means for their implementation |
| **Spec reviewers (near-term)** | Orthogonality rubric, drift signals, where a proposal's change should land |
| **Platform/application developers (medium-term)** | Layer map focused on L3/L4/L5, prototype inventory, transferability classes |
| **Paper readers cross-referencing (ongoing)** | Navigation between the abstract framework and concrete specs |

The release-time version will include a more detailed prototype inventory (§8), more detailed repository map (§7), and potentially an extracted public-safe version.

Candid internal material (extension-orthogonality pattern from the exploration §13, drift points from §7) stays in this version until extracted for public release.

---

## 11. Where to Look (By Question)

| If you want to… | Start here |
|---|---|
| Understand what the system is, philosophically | `papers/00-the-entity-system/content/paper.md` |
| Understand the pair-relationship framework | `papers/shared/notes/core/framework-synthesis.md` |
| Understand the core protocol boundary | `papers/shared/notes/core/core-protocol-boundary.md` |
| Review the architecture-side analysis | `reviews/core/EXPLORATION-PRIMITIVES-AND-PAIR-RELATIONSHIPS.md` |
| Implement the core protocol | `ENTITY-CORE-PROTOCOL.md` |
| Implement an extension | `specs/extensions/EXTENSION-*.md` |
| Author or propose a new extension | `GUIDE-EXTENSION-DEVELOPMENT.md` |
| Understand how extensions compose | `SYSTEM-COMPOSITION.md` |
| Build an SDK for a new language | `SDK-OPERATIONS.md` + `SDK-EXTENSION-OPERATIONS.md` |
| Understand the SDK access model | `reviews/EXPLORATION-SDK-ACCESS-MODEL.md` |
| Understand SDK usage patterns | `GUIDE-SDK-PATTERNS.md` |
| Understand peer organization | `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` |
| Understand a protocol composition pattern | `guides/` |
| See what spec changes are in flight | `docs/proposals/active/{core,extensions,applications,process}/` (owed) and `docs/proposals/implemented/` (applied) |
| See the cross-team convergence story | The full cross-team convergence synthesis (paper-team summary note) |

---

## 12. Open Threads (Architecture Side)

Active items the architecture team is working on or needs to work on.

**SDK domain (active):**
- SDK specs (SDK-OPERATIONS.md v1.1, SDK-EXTENSION-OPERATIONS.md v0.2) ready for implementation team review
- SDK access model exploration complete — three access levels, peer spectrum, extension composition model
- Transaction extension SDK surface drafted (v0.1, needs implementation validation)
- Capability management SDK API (delegate/revoke/inspect) — needs GrantScope formalization and design
- Role/group/alias semantics — underdeveloped, needs spec work before SDK surface
- Per-identity grant resolution mechanism — needed for multi-user, not yet designed

**Composition and deployment (active):**
- ⭐ **`PROFILE` — the install-set vocabulary, and it does not exist.** `Depends` states what must exist *beneath* an extension; nothing states what must exist *beside* it, so *"install this set to get this capability"* is unsayable. **Installing one member of a family alone accomplishes nothing** — a content-fallback path needs the substitute chain, the content store, the tree, a fetch convention and the capability system, none of which are `Depends` edges between them. §13.1b note 8 states the mechanics; the artifact is owed. **Also the missing half of `EXTENSION-SUBSTITUTE` §9.2's ruling** that *"required for v1 binds the implementation, not every deployment"*, which presupposes a way to describe a deployment.
- **Family membership is undeclared.** §13.1a notes 5–6: grouping cannot live in extension names and the `SYSTEM-*` composition tier is where it belongs; no extension currently declares which family it is in.

**Core protocol (maintenance):**
- Cascade depth thresholds — empirical, works, could be grounded in TMX cycle structure (R2 in backlog)
- Extension-design rubric — codify orthogonality test + drift signals (A1 in backlog)

**Cross-team coordination:**
- Guide → SDK L3 promotion path — SDK specs now exist, platform teams can review
- Class S interop floor review — verify L0 algorithm inventory is complete
- Vocabulary alignment — "extension" in architecture docs, "actualizer" in paper contexts

**Paper coordination:**
- "Particular instantiation" framing into Paper 0
- Six-layer architecture adoption in Paper 0 and Paper 7

---

## 13. Extension Classification and Stability Discipline

*Added to consolidate the architecture team's working classification of extensions, the rationale for what belongs where, and the discipline that maintains the system's "we're going to be done at some point" trajectory. See `EXPLORATION-SCOPE-OVERREACH-RETROSPECTIVE.md` for the full retrospective that motivated this section and `GUIDE-EXTENSION-BUILDER.md` for the pre-merge checklist that operationalizes it.*

### 13.1 Five-tier classification

The L2.5 extension layer (per §2's layer diagram) plus the core-protocol-internal handlers decompose into five tiers. **The boundaries are not strict either-or** — there is genuine overlap, and the tiering will move as the team learns what belongs. This is the architecture team's working classification.

#### Tier 0 — Core-protocol substrate (V7-internal)

Handlers and primitives built directly into the core protocol. These are not extensions; they are pre-loaded during system initialization (per V7 §6.5) and are required for *any* extension to function above them.

| Member | What it provides |
|---|---|
| `system/tree` handler | Basic tree `get` / `put` operations (V7 §3.8). The substrate the tree extension builds on. |
| `system/handler` handler | Handler registration (V7 §6.2) — `register`/`unregister`. How every other handler enters the system. |
| `system/type` handler | Type validation and registration (V7 §6.2). Anchored by `ENTITY-NATIVE-TYPE-SYSTEM.md`. |
| `system/protocol/connect` handler | Connection bootstrap (V7 §3 protocol surface). |

These freeze with V7 itself in the first wave (§13.4).

#### Tier 1 — Core substrate extensions

Irreducible add-ons — a peer fundamentally needs these to run. Together with Tier 0 and V7, they define what a peer is. Twelve members today:

| Member | Status | Role |
|---|---|---|
| `EXTENSION-TREE` | Draft v4.12 | Snapshots / diffs / merges / view-trees over the core-protocol tree. (Top-level in `specs/`, not nested under `extensions/` — a placement choice reflecting its tight coupling to V7's tree handler.) |
| `EXTENSION-CONTENT` | Draft v3.7 | Content store ingestion. |
| `EXTENSION-INBOX` | Draft v5.9 | Async delivery handler. |
| `EXTENSION-SUBSCRIPTION` | Draft v3.19 | Publish-subscribe / cross-peer reactivity. |
| `EXTENSION-CONTINUATION` | Draft v1.25 | Chained execution / cross-peer dispatch. |
| `EXTENSION-COMPUTE` | Draft v3.32 | Embedded expression language; programs-as-entities; the standard-IR floor (nav primitives, MUST collection stdlib, pinned integer model). Foundational for entity-native compute. |
| `EXTENSION-QUERY` | Draft v1.8 | Query operations over the tree. |
| `EXTENSION-REVISION` | Draft v3.14 | Versioning + three-way merge / cross-peer merge convergence. |
| `EXTENSION-HISTORY` | Draft v1.11 | Path-level transition recording. |
| `EXTENSION-TYPE` | Draft v1.3 | Value-level constraints layered above the Tier 0 type system. Currently deferred in implementation because the team is close enough to the ground to enforce manually; the proper-form-of-types in the system needs it. |
| `EXTENSION-CLOCK` | Draft v1.3 | System time — wall-clock primarily; logical / vector / HLC sub-pieces are reference research, kept in the spec. |
| `EXTENSION-SUBSTITUTE` | Active v1.3 | Ordered substitute-source chain consulted on CONTENT's local-miss path, plus the HTTP-as-storage-transport convention. **Placement is the spec's own declaration** (*"Tier 1, CDN release v1 critical path"*) and is the looser fit in this tier: installing it is additive and not installing it leaves CONTENT's 404 unchanged, which is Tier-2 shaped. Its own header says *"Operational — Tier 1"*, which is the ambiguity rather than a resolution of it. |

These freeze in the first wave alongside V7 and Tier 0 (§13.4).

#### Tier 2 — Operational extensions

Needed for **production multi-peer deployments**. A single peer can run without these on V7 + bare-key-pair identity. Required for identity management, peer networking, and operational management. Decomposes into three sub-groups; **most of this tier is now authored** (identity/authority/membership in 2a; NETWORK + DISCOVERY + RELAY in 2b), and the **remaining gaps are in operational management (2c — GC, persistence)**.

##### Tier 2a — Operational user-identity

How identity, authority, and membership are managed above raw key-pair identity.

| Member | Status | Role |
|---|---|---|
| `EXTENSION-IDENTITY` | Stable v3.10 | Peer identity, controllers, rotation. |
| `EXTENSION-ATTESTATION` | Stable v1.3 | Substrate primitive *within the operational tier* — generic attestation graph consumed by IDENTITY, QUORUM, ROLE, GROUP. |
| `EXTENSION-QUORUM` | Stable v1.2 | K-of-N node primitive within the operational tier. |
| `EXTENSION-ROLE` | Draft v2.3 | Role-based authority. |
| `EXTENSION-GROUP` | Stable v1.4 | Membership / registries. |

##### Tier 2b — Operational network

Cross-peer connection, sync, discovery, relay. Architecturally established for some time; **now authored** — NETWORK, DISCOVERY, and RELAY all carry normative specs.

| Member | Status | Role |
|---|---|---|
| `EXTENSION-NETWORK` | Draft v1.9 | Sync, peer connection, bootstrap conventions. |
| `EXTENSION-DISCOVERY` | Active v1.2 | Peer discovery beyond ad-hoc connection (mDNS / local + registry-assisted resolution). Authored as a normative extension. |
| `EXTENSION-RELAY` | Active v1.5 | Peer-to-peer message relay through intermediaries — four canonical modes (Forward / Store-and-poll ship in v1; Aggregate / Circuit named-but-deferred). Reauthored against the v7-substrate model. |
| `EXTENSION-REGISTRY` | Active v1.26 | Name → target bindings, the peer-local resolver chain and its backend vocabulary, supersession and revocation. **Placement is the spec's own declaration.** |
| `EXTENSION-ROUTE` | Active v1.0 | The next-hop *decision* — a tree-bound, cap-scoped routing table. **Relay forwards one hop; routing decides the hop, and the algorithm is never relay's.** Placement and sibling set are the spec's own declaration. |
| `EXTENSION-SIGNALING` | Draft v1.2 | Connectivity establishment — candidates, observed addresses, punch coordination — composed through NETWORK's live-establishment seam. Placed here by dependency; the spec declares no tier of its own. |

##### Tier 2c — Operational management

Operational concerns above networking. Currently mostly gaps; this is where the architecture team's "we know we need these eventually" thinking lives.

| Member | Status | Role |
|---|---|---|
| **GC (garbage collection)** | Spec gap; nature unsettled. | Operational storage management. **Borderline question:** is this an extension or implementation-management guidance? Theoretically with unbounded storage a peer never needs GC, so it's optional in a strict sense. The architecture team plans to outline it either way to set expectations; whether it lands as an extension or as a guide is a downstream decision. |
| **Persistence** | Partial — `GUIDE-PERSISTENCE.md` covers the SDK side (configuration directory / per-peer state / runtime state in tree). No core-protocol-domain extension. | Local-storage substrate. SDK-side guidance exists; core-protocol-domain treatment is the gap. Overlap with GC is real and to be resolved when either lands. |
| `EXTENSION-ENCRYPTION` | **Active v1.0** | At-rest and in-transit confidentiality, peer and group key-agreement modes, key backup. **Landed, and this row is the one placement in the section that is a judgment rather than a transcription** — the spec declares no tier, its only unconditional dependency is the core protocol, and its tiered internal structure (Tier A…) is its own and unrelated to this classification's tiers. Placed in operational management because a deployment chooses it; revisit if it earns its own sub-group. |

These sub-tiers freeze in the second wave (§13.4), after substrate stabilizes in cross-impl.

> ⭐ **Why five rows changed at once in September 2026, and what it says about this section.** Four
> extensions — `REGISTRY`, `SUBSTITUTE`, `ROUTE`, `SIGNALING` — were absent from this classification and
> from §13.1b's dependency DAG entirely, while **three of the four declare their tier in their own
> headers and cite this section by number as the authority for it.** One names its sibling set
> (*"sibling of RELAY / NETWORK / REGISTRY / DISCOVERY"*) more completely than this table did. **So the
> placements existed and were declared by the members; the map was the copy that drifted** — a citation
> resolving to a section that does not list the citing document. Thirteen version numbers here were
> simultaneously stale by up to eleven minor revisions, and one landed extension was described as an
> early sketch. **The cause is structural rather than careless: this is a restatement of every spec's
> header, nothing in either document says so, and until 2026-09-14 it was the one version roster in the
> corpus that no gate read.** It is gated now.

#### Tier 3 — Extras / first-pass grounding

Major CS concepts the community will expect. May converge or diverge as the community uses the system. Defensible as *anti-fragmentation* work (§13.2 reason 3) — better to have a coherent first pass than have downstream implementers invent four competing semantics.

| Member | Status | Role |
|---|---|---|
| `EXTENSION-TRANSACTION` | Draft v0.2 (pre-review) | Atomic multi-binding writes + observation boundary. The isolation-level apparatus (`snapshot` / `serializable`) is RDBMS pattern-matching and is a narrow-candidate; the extension itself is real anti-fragmentation work that bridges toward the eventual Raft / consensus roadmap. |
| Future extras may emerge | — | If a major CS concept surfaces with expected community demand and no existing-extension coverage, it lands here. |

May never freeze in this repo — converge through community use.

#### Tier 4 — Exploratory

Preserved reference design after a retraction or before a driver is identified. **No deployment is required to install.** Status header explicitly marks these as *exploratory · optional · not actively developed*.

| Member | Status | Role |
|---|---|---|
| `EXTENSION-DURABILITY` | v0.1 Exploratory | Retracted from spec. The lifted material is preserved verbatim; the durability contract is not normative for any tier. |

#### Bridge extensions — beside this classification, not a tier in it

**The five tiers above classify the entity system's own extensions: what a peer is, what a
production deployment needs, what the community will expect.** A bridge extension answers a
different question — *how does the system reach a technology that is not it* — so it takes **no
row in Tiers 0–4 and is not an omission from them.**

| | |
|---|---|
| **Home** | `specs/bridge-extensions/`, one document per foreign technology |
| **Naming** | `EXTENSION-BRIDGE-<TECHNOLOGY>` |
| **Artifact class** | an extension — entity types, handler operations, storage conventions, a conformance section |
| **Layer** | beside the application tier (§2) |
| **Roster** | `ROADMAP-EXTENSIONS.md`; the cross-family discipline is `guides/GUIDE-BRIDGE-EXTENSION-DEVELOPMENT.md` |

**There is no umbrella bridge extension and there will not be one.** *Bridge is a development-time
category, not a runtime surface* — unification at the all-bridges level fractures on four axes at
once (different trust models, different idempotency, different capability flows, different failure
surfaces). **What the family shares is knowledge, and knowledge lives in a guide.**

**Members today: one, and it is named for where it was first prototyped rather than for what it
is.** `DOMAIN-LOCAL-FILES` predates the family framing and is the prior art the framing was derived
from. Its placement here is the correction; its name and its `local/files` namespace are a
deliberate holdover, and the successor is additive rather than a rename — see
`GUIDE-BRIDGE-EXTENSION-DEVELOPMENT.md` §6.

#### Tiers above the architecture team's direct scope

Listed for context — these are not in the L2.5 extension layer but in the SDK and application layers above:

- **SDK Layer 3** (per §2): per-language protocol facade. SDK specs are a separate concern (`specs/sdk/`).
- **SDK Layer 4** (per §2): shared patterns / standard-library user-space extensions. These are what the team pushes forward as standard-library conventions but they live above the system extensions — UI/UX, application-level conventions, deployment patterns.
- **L5 applications** (per §2): combinatorial, problem-specific, out of scope.

These are mentioned only to clarify that the L2.5 classification above stops at "system extensions" and the SDK / standard-library / application layers are intentionally separate concerns with their own organizing principles.

### 13.1b Dependency DAG (Depends-based)

The classification in §13.1 captures *what tier* each extension lives in. This subsection captures *what depends on what* — the dependency DAG derived from the `Depends:` headers in each extension's spec. The DAG is roughly tree-shaped with some cross-dependencies; it is shown here as an indented sketch rather than a strict graph (graph rendering is a downstream concern).

```
ENTITY-CORE-PROTOCOL (Tier 0 substrate; v7.48)
│   carries: system/tree, system/handler, system/type,
│            system/protocol/connect handlers
│
├── ENTITY-NATIVE-TYPE-SYSTEM (foundational type substrate)
│   ├── EXTENSION-TYPE                  (Tier 1; value-level constraints)
│   └── EXTENSION-QUERY                 (Tier 1; also depends on SUBSCRIPTION)
│
├── EXTENSION-TREE                      (Tier 1; tree snapshots/diffs/merges)
│   ├── EXTENSION-REVISION              (Tier 1; also depends on SYSTEM-COMPOSITION)
│   │   └── EXTENSION-TRANSACTION       (Tier 3 extras; also depends on TREE directly; REVISION optional)
│   └── EXTENSION-TRANSACTION           (cross-dep — Tier 3 extras; TREE required, REVISION optional)
│
├── EXTENSION-INBOX                     (Tier 1; async delivery)
│   ├── EXTENSION-SUBSCRIPTION          (Tier 1; depends on INBOX)
│   │   ├── EXTENSION-QUERY             (cross-dep; tree change events)
│   │   └── EXTENSION-NETWORK           (cross-dep; subscription restoration)
│   └── EXTENSION-NETWORK               (Tier 2b; depends on INBOX + CONTINUATION + SUBSCRIPTION)
│       ├── EXTENSION-DISCOVERY         (Tier 2b; under NETWORK; Active)
│       ├── EXTENSION-RELAY             (Tier 2b; under NETWORK; Active)
│       ├── EXTENSION-SIGNALING         (Tier 2b; under NETWORK — §6.7 reachability facts,
│       │                                §10.3 live-establishment seam)
│       └── EXTENSION-REGISTRY          (Tier 2b; depends on V7 + ATTESTATION directly —
│                                        grouped under NETWORK by its role in the family, not by Depends)
│
├── EXTENSION-CONTINUATION              (Tier 1; chained execution; independent of INBOX spec-wise; composes operationally)
│   └── EXTENSION-NETWORK               (cross-dep)
│
├── EXTENSION-CONTENT                   (Tier 1; content store ingestion; independent)
│   └── EXTENSION-SUBSTITUTE            (Tier 1; consulted on CONTENT's local-miss path; CONTENT required)
├── EXTENSION-COMPUTE                   (Tier 1; programs as entities; independent)
├── EXTENSION-HISTORY                   (Tier 1; path-level transitions; independent)
├── EXTENSION-CLOCK                     (Tier 1; system time; independent)
│
├── EXTENSION-ATTESTATION               (Tier 2a; foundation of operational user-identity)
│   ├── EXTENSION-QUORUM                (Tier 2a; depends on ATTESTATION)
│   │   └── EXTENSION-IDENTITY          (Tier 2a; depends on ATTESTATION + QUORUM)
│   │       └── EXTENSION-GROUP         (Tier 2a; depends on IDENTITY + ROLE)
│   └── EXTENSION-IDENTITY (direct)
│
├── EXTENSION-ROUTE                     (Tier 2b; depends on V7 directly — the next-hop DECISION;
│                                        RELAY and any other propagation primitive are its consumers)
│
├── EXTENSION-ENCRYPTION                (Tier 2c; depends on V7 unconditionally, ATTESTATION optional)
│
└── EXTENSION-ROLE                      (Tier 2a; depends on V7 directly; composes-with IDENTITY but does not strictly depend on it; SUBSCRIPTION + CONTINUATION optional)
    └── EXTENSION-GROUP                 (cross-dep)
```

**Reading the DAG:**

1. **V7 (Tier 0) is the root.** Every Tier 1 substrate extension depends directly on V7.
2. **TREE is the early branch.** REVISION, TRANSACTION, and several other extensions build on TREE's snapshots/diffs/merges.
3. **The async delivery sub-tree** is INBOX → SUBSCRIPTION → NETWORK (+ CONTINUATION as a sibling). These have real spec-level dependencies on each other.
4. **The user-identity chain** is ATTESTATION → QUORUM → IDENTITY → GROUP, with ROLE sitting independently at the V7 level but composing-with IDENTITY/GROUP. The "core primitive" pair (ATTESTATION + QUORUM) sits below the named identity surfaces, exactly as user described: attestation + quorum are foundation; identity + group depend on them.
5. **The network sub-tree** has NETWORK at the root with DISCOVERY, RELAY, SIGNALING and REGISTRY grouped underneath, and ROUTE beside them. **All five are authored** — the sentence here previously called discovery and relay *"spec gaps"* and was written before either was written. **Two of the five are grouped by family rather than by `Depends`:** REGISTRY and ROUTE depend on the core protocol directly and would sit at the root of the DAG on dependency alone. That is the first place this sketch stops being a `Depends` graph, and it is deliberate — see note 8.
6. **Several extensions are spec-independent** of others — CONTENT, COMPUTE, HISTORY, CLOCK only depend on V7. These compose freely without inter-extension constraints.
7. **The cross-dependencies** make the DAG slightly web-like — QUERY depends on both TYPE-SYSTEM and SUBSCRIPTION; TRANSACTION depends on TREE with optional REVISION; NETWORK depends on INBOX + CONTINUATION + SUBSCRIPTION jointly. These are real but few in number; the graph stays tree-readable.

8. ⭐ **`Depends` answers *what must exist BENEATH me*, and never *what must exist BESIDE me* — so this DAG cannot answer the question people bring to it.** A functioning content-fallback path needs SUBSTITUTE **and** CONTENT **and** TREE **and** a substitute convention **and** the capability system; a functioning name-resolution path needs REGISTRY **and** NETWORK **and** at least one backend. **None of those are `Depends` edges in the inheritance sense, and installing any one member alone does nothing.** ⇒ **The install-set vocabulary is a genuine gap, not a presentation problem** — it is filed in §12 as `PROFILE`, and until it exists, a sentence like *"required for v1 binds the implementation, not every deployment"* (`EXTENSION-SUBSTITUTE` §9.2) presupposes a vocabulary for describing deployments that the corpus has never written.

9. **The corpus's forward roadmap is partly expressed as citations to extensions that do not exist** — a cross-spec reference sweep resolves nine such names, `EXTENSION-GC` being the most-cited. That is legitimate as a signal of intent and illegitimate as a dependency: **a mechanism cited by five documents and specified by none cannot be relied on by a design**, which matters as soon as any extension's steady-state operation produces collectable garbage.

**Implications for freeze sequencing:**

- A node freezes only when everything it depends on is frozen. V7 must freeze before TREE; TREE before REVISION; etc.
- The substrate tier (Tier 1) can freeze in waves: first the no-dependency leaves (CONTENT, COMPUTE, HISTORY, CLOCK), then TREE, then SUBSCRIPTION (after INBOX), then NETWORK (after its three deps), then QUERY/TRANSACTION (which have cross-deps).
- The operational tier (Tier 2) waits on substrate. Within Tier 2a, ATTESTATION + QUORUM freeze first; IDENTITY then; GROUP last.
- GC / persistence (the remaining Tier-2c gaps) need spec authoring before they can freeze at all. (Discovery + relay are now authored — `EXTENSION-DISCOVERY` / `EXTENSION-RELAY`, Active.) These are explicitly the *post-publishing* community-and-team work surface.

### 13.1a Reading the classification

A working note on how to use this taxonomy:

1. **It's not strict.** A given extension may straddle two tiers; the classification puts it in the tier that best captures its freeze trajectory.
2. **It shows the spec-surface gaps.** The remaining gap rows in Tier 2 are GC and persistence-as-a-core-protocol-domain-extension; discovery and relay were gaps when this note was written and are now authored. They are real **spec gaps**, not concept gaps. The team has discussed these for a long time — relay has a prior Rust implementation in a legacy project; discovery has a deferred proposal and a registry-and-discovery landscape review; GC is borderline extension/management; persistence has SDK-side coverage. The work the team has not yet done is authoring the current normative specs. Calling them out here means we know what's missing on the published surface, not that the concepts are novel.
3. **Tier movement is expected.** As the community uses the system, items in Tier 3 (extras) may converge to Tier 2 (operational) if they earn that, or stay in Tier 3, or migrate to Tier 4 (exploratory). The classification is a snapshot.
4. **CLOCK stays in Tier 1** as a single extension; the wall-clock surface is irreducible and the logical/vector/HLC research notes coexist with it. (The team has decided not to split CLOCK — keeping it as one spec is cleaner than fragmenting on a research question.)

5. ⚠⚠ **A naming CONVENTION exists and it is not a rule — this note previously said it was, and the claim was withdrawn on operator correction `[2026-09-14]`.** The tendency is real: most extensions take their name from a `system/<name>` namespace segment they principally own. **It is a habit with reasons, not an invariant, and the corpus contains its own counter-examples with their reasons recorded** — which is what a healthy convention looks like rather than a broken law:

   | | |
   |---|---|
   | **depth is ordinary** | 122 distinct three-segment namespace paths are in live use across `specs/` and `guides/` — `system/type/constraint/*`, `system/substitute/sources/{hex}`, `system/peer/transport/quic`, `system/capability/revocations/{hash}`. **Nothing is flat and nothing needs to be** |
   | ⭐⭐ **`system/peer/*` is a SUBJECT INDEX, not an overloaded topic namespace** | **Corrected twice in one day `[2026-09-14]`, and the second correction is the durable one.** This row first read *"an extension can own a namespace that is not its namesake"* (filed as an exception); it was then rewritten as a misalignment to realign; **both were wrong, because the corpus has TWO organizing axes.** A namespace groups **by topic** (*what mechanism is this*) or **by subject** (*who is this about*) — and `system/peer/*` is the second: `status/{peer_id}`, `session/{peer}`, `transport/{peer_id}/{protocol}`, `published-root/{peer}`, `identity/{peer_id}`, **with `self` as the slot for *me* in an index otherwise keyed by others, which proves the axis.** §3.13 states the pairing outright — a peer's own transports at `system/transport/{protocol}`, a remote peer's at `system/peer/transport/{peer_id}/{protocol}`, **same type, two locations, one per axis.** ⇒ **regrouping by mechanism would break the property that everything known about one peer sits under one prefix.** The open question is **which facts belong on which axis** (duplication across axes is already answered *yes*, for transports). Advisory check: `spec shape` — ⚠ **and its first cut measured only the topic axis and reported this namespace as the corpus's worst lookup failure, which is why the count it prints is not the finding** |
   | ~~*(superseded by the row above)*~~ | **The intermediate version said:** `EXTENSION-NETWORK` does declare `system/peer/session`, `system/peer/status` and six `system/peer/transport/*` profiles — **but that is not an exception proving the convention optional; it is the one place in the corpus where a reader holding a path cannot reach the specifying document.** Measured: of 185 declared type paths, **168 resolve to a document by first segment**, 17 sit under a core-owned namespace, and **10 of those 17 are `system/peer/*`, declared across three extensions plus the core.** ⇒ **the convention is the corpus's lookup index and it works 168 times; this is its strongest catch, and scoring it as an excuse is how a comprehensibility defect became evidence that comprehensibility has no rules.** The realignment is real, is priced (85 references and a pinned path segment inside a three-way-green ordering for `transport` alone), and is a **coordinated cut rather than a rename to do casually** — advisory check: `spec shape` |
   | **and can deliberately keep names outside any prefix** | `ENCRYPTION` keeps flat `system/encrypted` / `system/encryption-pubkey`, ruled that way because a rename would rebind the hash of every encrypted entity at rest and change every `recipient_key` |
   | **placement is a judgment the team exercises** | `INBOX` moved `system/protocol/inbox/delivery` → `system/inbox/delivery` as a ratified decision, weighing two conditions — not by applying a naming law |

   **The withdrawn claim, stated so it is not re-derived:** a grep found the namesake string in 26 of 26 extension specs and that was promoted to *"an invariant — renaming the document renames the namespace, grouping cannot live in extension names."* **The measurement was the presence of a string, not ownership or exclusivity**, and three of the four rows above contradict the conclusion. ⇒ **The convention is worth keeping and worth stating as a convention.** §13.5b is where the class of decision it belongs to is described.

6. **Grouping is available at more than one place, and the document classes are one of them.** `EXTENSION-*` is conventionally *a handler surface*; `SYSTEM-*` is conventionally *a composition* — `SYSTEM-DATA-EXCHANGE` describes its own tier as *"system composition, an emergent property no single extension owns."* **That is a useful distinction and a chosen one.** Namespace depth, family documents, tier grouping and document class are all available and they are not exclusive; **which one carries a given grouping is a §13.5b decision, decided on whether it helps a reader hold the system, and not settled by this note.** What is genuinely absent is any *declaration* of family membership: the network family is named in prose in §13.5a and mapped in `GUIDE-NETWORKING-MODEL`, and is invisible from each of its members.

### 13.2 Three legitimate reasons to spec something

For any new extension or amendment, the proposer must answer *which of three* applies:

1. **Irreducible technical requirement.** The system fundamentally cannot function without it (substrate tier).
2. **Operational requirement.** Production multi-peer deployments need it (operational tier).
3. **Anti-fragmentation grounding.** A major CS concept the community will expect the system to address (extras tier). *Requires both:* expected community demand **and** no existing coverage through composition of existing extensions.

A proposal that fails all three is *exploratory at best* and a candidate for retraction-before-merge.

The durability thread is the cautionary example: it could not pass test 1 (no irreducibility), could not pass test 2 (no production driver), and could not pass test 3 because the property was already covered by `TRANSACTION` + `REVISION`+sync + `INBOX`+`CONTINUATION`. The retraction took the surface to exploratory.

The transaction thread is the contrasting example: it can pass test 3 in the anti-fragmentation form (large-scale-consensus / Raft is on the long-term roadmap, and the community will expect a transaction story), even if it does not pass test 1 (a peer can run without it). Test 1 / Test 2 / Test 3 are distinct gates, not a single gate.

### 13.3 V7 is frozen-by-discipline before it is frozen-in-fact

A discipline observation that emerged from the durability retraction: the team controls V7, which makes V7 modifications feel cheap during design ("quick change to V7, this extension needs it"). The cost is hidden until V7 freezes — at which point every modification we made because-it-was-easy becomes a community cost every implementer absorbs.

**Going-forward discipline:** treat V7 as if it were already frozen. Modifications to V7 require an independent justification — not "because this extension needs it," but "V7 needs this change *independently* because some other class of extension or use case requires it as well."

The durability thread's V7 v7.47 changes (no-silent-ignore in §3.2, 412 in §3.3, 202 documentation in the core table) all failed this independent-justification test. V7 was reverted to v7.46 with the retraction.

This discipline is now part of the pre-merge checklist in `GUIDE-EXTENSION-BUILDER.md` §3.8.

### 13.4 Publishing horizon and freeze trajectory

The system is approximately **3–4 weeks from publishing**. The trajectory:

```
T-0  (now)     T+3w (publish)    T+12m (community use)    T+24m+
─────────────────────────────────────────────────────────────────
V7              ──────  frozen ──────────────────────────────────
substrate ext   ──────  freeze first wave  ──────────────────────
operational ext ───────────  freeze second wave ─────────────────
extras / parity ─────────────────────  may freeze / may evolve ──
exploratory     ─────────────────────  community-driven ─────────
```

After publishing:

- **V7 freezes in fact.** §13.3's discipline becomes absolute. Any V7 modification is a community-cost event.
- **Substrate extensions freeze first.** The 10 substrate-tier extensions (§13.1) stabilize against amendment churn.
- **Operational extensions follow.** The 6 operational/user-identity extensions stabilize as the substrate stops moving.
- **Extras may never converge.** That is expected. The community may pick up some, ignore others, design parallel alternatives. The architecture team's deliverable is the first-pass grounding, not the final form.

**The goal is eventually being done — not continuing to build forever.** The architecture team's central job in the publishing window is the consolidation that makes "done" possible:

1. Pulling the rationale together (this section is part of that).
2. Cleaning up the spec surface (apparatus retraction; status-header discipline).
3. Classifying every spec by tier (§13.1) so freeze-order is unambiguous.
4. Marking deferred / borderline work as exploratory with its design intact.
5. Acknowledging that some "extras" may be wrong and that's fine — the community will sort them.

### 13.5 Spec versus reference implementation

A related framing the team should keep clear. The architecture deliverable is **the spec**. The reference implementations (`entity-core-go`, `entity-core-rust`, `entity-core-py`) are *proof that the spec is implementable*. They are not canonical:

- A spec change is a spec change — it affects every implementation.
- An implementation bug is an implementation bug — it affects one.
- A security gap in the spec is a security gap in every conforming peer.
- A security gap in an implementation is a security gap in that implementation.

A community implementation may eventually emerge that is faster, simpler, better-designed, or higher-performing than the reference. That is the expected outcome of a clean spec + open ecosystem — the reference is reference, not canonical.

**Implication for the team's signal interpretation.** Cross-impl ratification proves implementations agree on the spec text. It does *not* prove the spec text belongs. Three impl teams ratified durability Amendment 1 the week durability was retracted. That distinction — convergence on text vs. justification of text — is what the cold re-read step (`GUIDE-EXTENSION-BUILDER.md` §4) exists to enforce.

### 13.5a Constraint regime and design divergence — two axes beside the tiers

*Informative. §13.1's tiers sort by **how necessary** a surface is. That axis cannot express the thing
the team keeps re-deriving as "the boundary is blurry": that a decision near the wire and a decision in
the network family are **different kinds of decision**, reviewed correctly by different methods. Two
further axes, kept here because they change how review is run. Full treatment:
`docs/research/explorations/EXPLORATION-THE-THREE-CONSTRAINT-REGIMES.md`.*

#### Axis A — constraint regime: what governs a *decision*

| Regime | Governed by | Freedom | Whose complexity |
|---|---|---|---|
| **I — Substrate** | mathematics and computer science | none in structure | nobody's; irreducible. Simplifying it means being wrong |
| **II — Design space** | coherence; we choose | real | **ours** — the only regime where complexity is fully ours to remove |
| **III — Accumulated technology** | the deployed world (NAT, ICE, the browser sandbox, CDN semantics) | none, for a *contingent* reason | not ours, and not optional. It can be placed and quarantined, not deleted |

**A regime applies to a decision, not to an extension.** Every extension is a mixture, and the two
halves that matter are usually different: an extension's **structure** may be Regime II while its
**motivation** is Regime III. `EXTENSION-RELAY` is the worked case — its mode set, envelope shape and
capability model are ours to choose, while the fact that intermediaries are needed at all is forced by
a world where reachability is not universal. *"Is RELAY core-like or network-like" is two questions.*

Regime I structures routinely contain Regime II parameters. That a hash is collision-resistant is
necessary; *which* hash is arbitrary — which is why hash agility and the `content-hash`
`(format_code, digest)` shape exist, and why a fixed-width assumption is a lint error.

**The standing rule: Regime III complexity is kept out of Regime I and II structure.** Transport
profiles are entities rather than branches in a dispatcher; routing sits behind a resolver seam rather
than inside relay; the reachability-class taxonomy turns the browser's inability to listen into one
row of a dispatch table. Each is a place where an external fact was made into data so it could not
become structure.

#### Axis B — design divergence: how many defensible designs the landscape contains

Where an *area's* centre of mass sits. This is the grouping the team means by "core plus standard
extensions is one thing, and network and L5 are another."

| Band | Area | What the landscape looks like | How to review it |
|---|---|---|---|
| **A — Convergent** | V7 core, Tier 0, Tier 1 standard extensions | Independent designers arrive at the same answers. Content addressing, Merkle structure, capability chains, three-way merge — a survey finds agreement, not variety | For **correctness**. A disagreement usually resolves to a fact, and one side is wrong. Conformance vectors are the gate; prose review does not catch these |
| **B — Bounded choice** | Tier 2a — identity, attestation, quorum, role, group | Real alternatives, enumerable, trade-offs documented in the literature (K-of-N topologies, cert chains vs. webs of trust) | For **coherence with the choice already made**. A small number of defensible options; pick the one that composes with the rest |
| **C — Divergent** | Tier 2b — network, relay, route, signaling, discovery, registry | **Wide, and every model in it works.** libp2p, Nostr, AT Protocol, Matrix, SMTP/NNTP, IPFS and Tor solve overlapping problems with genuinely different architectures | For **fit and landscape grounding**. A design is argued against the survey, not from first principles alone. Disagreement is data rather than defect, and closing a design space wrongly costs more than leaving a knob |
| **D — Opinion-dominated** | Tier 4 — L5 application conventions | Taste, ecosystem and front-end reality dominate; several conventions coexist in every comparable ecosystem | For **the cross-impl contract only** — the bytes and the format. Rendering, UX and wiring are per-front-end and are not the spec's to settle |

**Four things change as an area moves down the bands**, and they are why the same review method does
not work throughout:

1. **Degrees of freedom rise.**
2. **A landscape survey gains value; a first-principles derivation alone loses it.** In Band A,
   deriving the answer is sufficient. In Band C, deriving *an* answer proves only that it is
   expressible — it says nothing about what the field has learned or what the system needs.
3. **Legitimate disagreement rises.** In Band A a dispute usually has a right answer. In Band C it may
   resolve to a preference, and forcing a resolution destroys options that were deliberately open.
4. **The normative posture inverts.** Band A pins tightly — a `MAY` whose two conformant readings
   diverge across a peer boundary is a latent interop bug. Band C exposes knobs and states trade-offs.

**The tension this makes visible, rather than resolves.** The pin-the-cross-impl-observable-surface
instinct is a Band A instinct, and it is right there. In Band C it still governs the *observable
surface* but not the *model* — and the two are easy to conflate. The live instance is aggregation
ordering: the order a consumer sees a union of sources in is cross-peer observable (so Band A logic
applies), while the merge policy that produces it is design space (so Band C logic applies). **Naming
the band does not settle such a case; it shows why two correct rules are pointing in opposite
directions.**

**The question this adds to review, asked before "is this right?":** *which band is this in, and which
regime is the decision?* A Band A correctness argument applied to a Band C design space is how a
design space gets pruned; a Band C "it is all trade-offs" applied to Band A is how a wire invariant
gets negotiated.

### 13.5b Axis C — whose understanding, and the degrees of freedom we own

*Informative, and the newest of the three axes `[2026-09-14]`. Axis A asks what governs a decision;
Axis B asks how many defensible answers the landscape holds. **Neither says what a Regime II decision
is optimizing for**, and §13.5a's answer — "economy and coherence" — is incomplete in a way that has
cost real work. Full treatment:
`docs/research/explorations/EXPLORATION-THE-ORGANIZATION-IS-A-DEGREE-OF-FREEDOM-AND-COMPREHENSION-IS-WHAT-IT-BUYS.md`.*

#### The claim

> **A large class of decisions in this system is underdetermined by the mathematics, forced on us
> anyway, and optimized for how cheaply another mind can build a correct model of the system.** That
> class includes almost everything a newcomer meets first: document names and name classes, namespace
> shape, which document a rule lives in, tier numbering, family groupings, the extension decomposition
> itself. **None of it is derivable from the substrate. All of it decides whether the substrate can be
> understood.**

**Coherence is a property of the artifact. Comprehensibility is a property of the artifact relative to
a reader, and the two come apart** — a maximally economical decomposition can be harder to learn than a
slightly redundant one, and "fewest documents" is not the same objective as "fastest correct model."

#### Why this belongs in the architecture and not in a style guide

**Because the system's central claim is verifiability, and verifiability that nobody can trace is
faith.** A reader is asked to rely on content addressing, detached signatures, capability chains and
root sequences. Reliance without understanding is exactly the posture this system exists to remove. So
the ladder — *application convention → the extension it depends on → the core protocol → the
mathematics* — **has to be walkable by a person**, or the central claim is auditable in principle and
unauditable in practice.

⇒ **Organization is how the trust claim is delivered.** It is a functional requirement with an
unusually concrete test, below, and it is not decoration.

#### ⭐⭐⭐ Whose understanding — there is not one reader, and the classes navigate differently

*Added `[2026-09-14]`. This section has been titled for this question since it was written and did not
answer it: everything above says "a reader", "another mind", "a person". That is why the axis could
describe a trade-off and never settle one — **two organizational choices that serve different readers
look equally defensible until you say which reader.** Full treatment:
`docs/research/explorations/EXPLORATION-THERE-IS-NO-SINGLE-READER-THE-AUDIENCE-MODEL-WAS-ALREADY-ON-DISK-AND-THE-PARTY-IT-NEVER-NAMES.md`.*

**The model was already distributed across the corpus and had never been collected.** 27 documents open
with an `**Audience:**` declaration; nothing prescribes that field and nothing checks it. Collated, the
questions reduce to six — and **the column that decides organization is not who anyone is, but what shape
of question they arrive with.**

> ⛔⭐⭐ **These are CONTEXTS, not classes of person, and the distinction is the whole correction
> `[2026-09-14]`.** The same individual moves between them within an hour: understanding the system, then
> building on it, then running a deployment, then just trying to get a file onto another machine. **A
> reader is never *an* audience — they are in a context, and the context is what has a need.** Nobody is
> excluded from any document; **the failure is a document that answers the wrong question first**, which
> is a placement defect and never a permissions one. ⇒ **do not stamp a class on a document and do not
> read this table as a taxonomy of people.** It exists to make *which question does this answer first*
> an askable question.

| Context | Arrives asking | Navigates by |
|---|---|---|
| **Deployment operator** | *who is this peer, what does it hold, what did I grant it, can I reach it?* | ⭐ **SUBJECT** |
| **End user** | *did my thing work?* | — reads none of this |
| **Application developer** | *what do I need to do X?* | **capability** — outcome first, mechanism later |
| **Extension / specification author** | *what mechanism is this, what does it own?* | ⭐ **TOPIC** |
| **Peer implementer** | *what exactly must I produce?* | the normative surface, **exhaustively** |
| **Reviewer, auditor, newcomer** | *is this claim true, and can I check it myself?* | **the ladder** |

> ⭐⭐ **The two axes this corpus organizes on are two of these readers made structural.** *What mechanism
> is this* and *who is this about* are not a taxonomic curiosity — they are the author's question and the
> operator's question, each given a namespace.

**Three consequences that change how an organization argument is conducted:**

1. **The implementer is not a vote.** An implementer reads everything regardless, so regrouping buys them
   almost nothing and omission costs them a conformance failure. **Counting them as a stakeholder in a
   grouping argument inflates the apparent cost of every proposed move.**
2. **The developer wants neither axis.** *What do I need to do X* is not answered by any grouping of
   documents — it is answered by a named set of components that travel together, which is a different
   artifact and is additive rather than a relocation.
3. ⚠ **One party is named constantly and addressed nowhere.** *User* appears in hundreds of lines of the
   published surface, is defined in no glossary, and is the declared audience of nothing. **That is
   arguably correct — no specification is addressed to an end user — but it means a design argument that
   invokes "the user" is using a word nothing here can check**, which is how an undefined party once
   carried a normative rule through eight revisions.

#### ⭐⭐ The tiebreak: rank by which comprehension failure is RECOVERABLE

Where the classes do not conflict — most of the time — there is nothing to decide. **Where they do, a
ranking is needed, and comprehensibility alone cannot supply one, because both sides of such an argument
are comprehensibility arguments.**

> **Deployment operators, and through them end users, take priority over the reader working in the
> internals.**

**As a preference that would be arbitrary. It is not arbitrary, because the classes differ in something
observable — what a wrong model costs, and whether it can be undone:**

| Class | Cost of a comprehension failure | Recoverable? |
|---|---|---|
| Extension author, implementer | **time** — a wrong model, found and corrected while building | ✅ **and the build is what corrects it** |
| Application developer | **time**, plus a design they revise | ✅ |
| **Deployment operator** | **a wrong authorization** — a capability granted to the wrong party, a scope wider than intended, a reachability record trusted that should not have been | ❌ an exercised capability cannot be un-exercised; fetched bytes cannot be recalled |
| **End user** | the consequence of the above, **having read nothing and chosen nothing** | ❌ and they had no opportunity to prevent it |

⇒ **The tiebreak is not "who matters more." It is that one class's comprehension failure is a security
outcome borne by a third party, and the others' is a delay borne by themselves.**

⚠ **Two clauses are part of the rule rather than caveats on it.** It applies **only where the interests
genuinely conflict** — where a deployment has no exposure, such as an authoring convention or a
document's internal section order, the author's convenience is the only signal present and wins by
default. And **it is a lean, not a law**: it says which way to fall when the arguments are otherwise
balanced, it does not license an expensive change for a thin deployment-side benefit, and **it leaves the
cost side of the ledger exactly as it was.** Hardening it into an unarguable rule would be this section's
own documented failure, committed by the section that documents it.

#### The test: a grouping earns its cost by making the best use of a BOUNDED WORKING SET

**Hierarchy's function is to let a mind spend a fixed budget well.** A grouping is worth what it costs
when a reader can work at one level and *correctly hold nothing* from the others — not merely when it
looks tidy. **The resource is bounded attention plus retrieval cost**, and naming it that way is what
makes the test arguable: *bounded* is measurable, and **a structure that lets a reader put a subject down
but not find it again has moved the cost rather than removed it** (which is where an index earns its keep
and a taxonomy alone does not).

⭐ **And the constraint is not human-only, which is the correction that makes this more than ergonomics.**
A frontier model running with a very large context is nearly indifferent to organization; **a constrained
one is not — every context token is contested, and pulling in one branch, working, and releasing it is
exactly how a bounded context is managed.** ⇒ **the structure that serves a human working set serves a
small context window, for the same reason.** Organization is the interface to a bounded working set,
whatever kind of mind holds it.

#### The aesthetic dimension, and it is not a separate criterion

> ⭐⭐ **Beauty here is understanding compressed into a form pristine enough to transfer directly from one
> mind to another.** On that reading the aesthetic response is not a nice-to-have correlated with
> comprehensibility — **it is the felt signal of a successful compression**, which is why it arrives before
> the explanation.

**Which fixes what kind of artifact this axis produces:** the same kind as accumulated design guidance —
proportion, symmetry, the rule of three — **rules of thumb derived from practice, with real authority and
acknowledged exceptions, which long predate any cognitive account of why they work.** Not theorems, not
preferences. ⚠ **And clarity is our scoped objective rather than a universal one**: difficulty and
disorientation are legitimate artistic ends, and we are not pursuing them — we are guiding a reader toward
comprehension of a system built from mathematical principles and intentionally designed to be understood.

⚠ **Stated the other way, it is a live diagnosis of this corpus: we have accumulated a great deal of
true dependency and very little licensed ignorance.** Twenty-six extensions, five application
conventions and four composition documents all citing each other means that isolating one piece for
refinement requires holding most of the rest. **That is the cost Axis C measures, and nothing else in
§13 measures it.**

#### The asymmetry, and the trap it sets for the authors

**The machine is indifferent.** Rename every extension, handler and type to a UUID and every hash,
proof and dispatch still resolves; the conformance suite stays green. **So organization has no
machine-side signal at all** — which means it cannot be validated the way the rest of the corpus is,
and the absence of a failing test is not evidence that it is working.

⭐⭐ **And an author operating under ABUNDANCE cannot feel the cost it is imposing.** The failure is not
*machine versus human* — it is *abundant versus constrained*, and it catches a well-resourced human author
who knows the corpus by heart just as surely as a large-context agent. **The specific failure that follows
is manufacturing necessity for a choice: measure a convention, find it consistent, and promote it to an
invariant** — because a rule is cheaper to carry than a judgment, and a law needs no justification while a
choice does. ⇒ **state which conditions a cost claim was measured under**, the same discipline this repo
already applies to a build-state claim.

> **This section's own §13.1a note 5 was that failure, in the same week it was written.** A grep found a
> naming pattern in 26 of 26 specs; the note then asserted an invariant with three *"cannot"* clauses,
> against three counter-examples already in the corpus. **The correction is not "be more careful with
> claims" — it is that Regime II decisions must be recorded AS decisions**, with their reasons, so the
> next reader can disagree with them. **A convention re-described as a law becomes unarguable, and an
> unarguable aesthetic is the worst of both kinds.**

#### Both failure directions are live, so the rule is neither "converge" nor "leave it open"

| Direction | What it costs |
|---|---|
| **Leave the freedom unexercised** | somebody else exercises it. Variety of aesthetic choice is a documented route to ecosystem fragmentation — **§13.2's reason 3 (anti-fragmentation) applied to organization rather than to mechanism** |
| **Exercise it and then describe it as necessary** | it calcifies. The choice stops being reviewable, and a later reader inherits a constraint nobody chose and nobody can argue with |

> **So: converge deliberately, and mark the convergence as chosen.** A convention with its reasons
> written down is both usable and arguable. That is the whole disposition, and it is cheap.

#### ⭐⭐⭐ The radial frame — the three axes are one coordinate

**§13.1's tiers, §13.5a's regimes and §13.5a's divergence bands have been three tables. They are one
coordinate: distance from the mathematics.**

```
            ·  anyone's applications, built on all of it
         ·  application conventions          (not under `system/` — the radius showing)
      ·  the outer families — connectivity · data exchange · data location
   ·  the core extensions — flat, orthogonal, minimally interconnected
·  the core protocol
◦  the mathematics
```

**Read outward and four things move together:** degrees of freedom rise · the regime shifts I → II →
II-with-III-quarantined · the review method goes correctness → coherence → landscape fit → cross-impl
contract only · and ⭐ **the ladder back to the mathematics lengthens.**

> **That last one is what the other two axes cannot say: the cost of being far out is paid by the READER,
> in rungs.** ⇒ **the outer rings are where organization matters most — longest climb, widest freedom —
> and they are exactly where this corpus has been flattest.**

**Two consequences:**

1. ⭐ **The core extensions are flat because they are nearly orthogonal, and that is a property rather than
   a style.** `CONTENT`, `COMPUTE`, `HISTORY` and `CLOCK` depend on nothing but the protocol; the real
   interconnection is one cluster (`INBOX → SUBSCRIPTION → NETWORK`, with `CONTINUATION` beside it). **A
   flat namespace is the honest shape for a ring whose members are independent** — the flatness there is
   not a defect and should not be "fixed."
2. ⭐⭐ **The outer ring is where families are real, and there are two:** connectivity, and data — with
   `SYSTEM-DATA-EXCHANGE` already at the composition tier and its *source* layer owed. **That ring is
   where a composition document earns its place**, and where a flat shape stops being honest.

#### Flat was the right first answer; the question is whether the deferral has expired

**Premature taxonomy is worse than the disorder it replaces** — it encodes a wrong model *as structure*,
where it is expensive to remove and is inherited downstream. **A flat namespace defers the choice by not
making it**, which is correct while the organizing principles are not yet apparent.

> ⇒ **So the question is not "was flat wrong" but "has the deferral expired?"** It expires when the
> boundaries, mechanisms and style guidance become apparent — because past that point the deferral buys no
> option value and only withholds an index from the reader.

**Signals that it is expiring:** the outer families have stabilized enough to be named; a composition tier
exists and is in use; and the namespace audit now returns a **bounded, priced, concrete** misalignment
rather than a vague unease. ⚠ **Counter-signal, kept because it is real:** the largest misalignment found is
also the most expensive to execute. **Deferral does not expire everywhere at once.**

#### ⭐⭐ The threshold is high AND rising, and the expiry date is not ours to set

*Added `[2026-09-14]`, because the two halves above are easy to read as an argument for waiting and that
is the one conclusion they do not support.*

**A deferred organizational choice is not a free option.** The cost of exercising it rises monotonically,
on three compounding counts: every new normative cross-reference widens the sweep; every independent
implementation adds a tree that must be told; and **every conformance check that encodes a shape makes
that shape harder to change than the prose that specifies it.**

> ⛔ **The last one is the sharpest and it is not hypothetical anywhere in this field: a check set can
> lock in a defect.** Once a behaviour is exercised by a suite that peers are built to pass, the suite is
> what everyone reads the specification against — so a mistake acquires a workaround, the workaround
> acquires dependents, and the pair ships forever. That is the ordinary history of deployed protocols,
> and it is why conformance is the contract rather than a convenience.

**And the window closes on somebody else's schedule.** The freedom to reorganize is a property of having
no installed base. It ends when a third party implements, forks or ships against the current shape — an
event nobody here schedules and nobody will be told about. ⇒ **the honest frame is not *is this worth
doing* but *is this worth doing while it is still cheap*, and the second question has a deadline the
first one does not.**

⚠ **This does NOT lower the bar for a change — it raises the bar for a deferral.** "Leave it and see"
is a decision with a rising price, and it must be argued for on the same evidence as a move: **record the
deferral as a decision, with its reason and what would change it.** An unrecorded deferral is
indistinguishable from nobody having looked, which is how a choice becomes a constraint nobody chose.

> ⭐ **The compensation, and it is the same mechanism seen from the other side.** The interconnection that
> makes a late change expensive is also what makes the system hard to fragment: a fork that dislikes one
> extension inherits its binding points, the extensions around it, and eventually the core protocol.
> **Cohesion is the asset and the constraint at once** — which is precisely why it is worth spending the
> remaining freedom deliberately rather than letting it lapse.

#### What is in this class, today

**An inventory, offered so the class is recognizable rather than as a settled list:** the
`EXTENSION-` / `SYSTEM-` / `GUIDE-` / `SDK-` / `APP-CONVENTION-` name classes and what each
connotes (**and `DOMAIN-`, which is closed at one member and is a holdover — §13.1's bridge note**) · the habit of naming an extension after a namespace segment it owns (§13.1a note 5, **a
convention**) · namespace depth and where a type is homed · the tier numbering and its sub-tier letters
· family groupings and which document carries one · the one-extension-one-concern decomposition · which
of several valid homes a rule is stated in. **Every item is ours; none is forced; all of them are load-
bearing for a reader and invisible to a test.**

### 13.6 Disposition for the next 3–4 weeks

The work the architecture team should plan to complete before publishing:

- **Status-header sweep.** Every extension and proposal gets its Status header explicitly tiered per §13.1. Today many specs say only "Draft" or "Stable"; the tier (substrate / operational / extras / exploratory) belongs in every header.
- **V7 modification audit.** Sweep V7's recent amendments (v7.40 onward) against §13.3's independent-justification test. The v7.47 reversion is one outcome of that test; there may be others.
- **Exploratory-tier framing.** Items that should land as exploratory (rather than be retracted entirely) get framed cleanly with a Status header and a preserved-design-record statement.
- **Migration plan for retained rationale.** Some of the design exploration that drove current specs (the 30+ explorations, the durability retrospective, the substrate split history) is appropriate as a *legacy archive* — the architectural rationale is preserved, but not in the same surface as the published specs and guides. The published release should be "specs + guides + SDK specs + SDK guides + reference implementation," fresh and clean. Rationale lives in the archive for those who want to understand the system's history.
- **Implementation-team handoff.** Each freeze wave (substrate-first, then operational) involves coordinating with the impl teams on the freeze boundary. The teams need to know *what they can rely on not changing* before they commit to the published-version interop.

---

## 14. Document History

- **v0.9:** §13.5b's six are restated as **CONTEXTS rather than classes of person** — the same individual moves between them within an hour, so a reader is never *an* audience; they are in a context, and **the failure is a document that answers the wrong question first**, which is a placement defect and never a permissions one. Also lands the **rising-threshold** frame: a deferred organizational choice is not a free option, because every week of interconnection and adoption raises the price of exercising it, and the expiry is set by third-party adoption rather than by us. Informative; no normative token moved.
- **v0.8:** §13.5b answers the question it is titled for. **There is not one reader — there are six classes, and the model was already distributed across 27 `Audience:` declarations where nothing could collect it.** The column that decides organization is the shape of question each class arrives with, and ⭐ **the corpus's two organizing axes turn out to be two of those readers made structural**: *what mechanism is this* is the extension author's question, *who is this about* is the deployment operator's. **Adds the tiebreak for when they conflict, and its basis is not a preference: rank by which comprehension failure is RECOVERABLE.** An author's wrong model costs time and the build corrects it; a deployment operator's is a misauthorization, which cannot be undone and lands on the party who read nothing. **Two clauses are part of the rule — it applies only to genuine conflicts, and it is a lean rather than a law that leaves the cost side of the ledger unchanged.** Also notes the party named constantly and addressed nowhere: *user* is defined in no glossary and is the declared audience of no document, which is why a design argument invoking it is using a word nothing here can check. Informative; no normative token moved.
- **v0.7:** §13.1a note 5 corrected again, and the second correction supersedes the first. `system/peer/*` is a **SUBJECT INDEX** rather than an overloaded topic namespace: the corpus has **two organizing axes** — by topic (*what mechanism is this*) and by subject (*who is this about*) — and §3.13 states the pairing outright, a peer's own transports at `system/transport/{protocol}` and a remote peer's at `system/peer/transport/{peer_id}/{protocol}`, same type, two locations, one per axis. `system/peer/self` is the slot for *me* in an index otherwise keyed by others, which is what proves the axis. ⇒ **regrouping those members by mechanism would break the property that everything known about one peer sits under one prefix**, and the open question is which facts belong on which axis rather than whether anything should move. ⚠ **A realignment recommendation was drafted and withdrawn on this**, and the advisory checker that produced it had measured one axis and reported the other's paths as the corpus's worst lookup failure — fixed, and noted, because a purpose-built instrument reporting an artifact of its own model is a new failure shape. Informative; no normative token moved.
- **v0.6:** §13.5b amended on three corrections, each of which changed a conclusion. **The test is restated as *making the best use of a bounded working set*** rather than *licensing ignorance* — the resource is bounded attention plus retrieval cost, and **the constraint is not human-only: a constrained context window is the same constraint, and hierarchy is how it is managed**, so the previous claim that *the machine is indifferent* was measured against one deployment's abundance. The failure is *abundant versus constrained*, not machine versus human. **Adds the aesthetic dimension** — beauty as understanding compressed enough to transfer directly between minds, which makes this axis's output the same kind of artifact as accumulated design guidance: rules of thumb with real authority and acknowledged exceptions. **Adds the radial frame: the tiers, the regimes and the divergence bands are one coordinate, distance from the mathematics** — and the cost of being far out is paid by the reader, in rungs, which is the thing the other two axes cannot say. **Adds the deferral argument:** a flat namespace defers the taxonomy by not making it, correctly, and the question is whether the deferral has expired. **§13.1a note 5's second exception row is corrected** — `system/peer/*` was filed as an exception proving the convention optional; measured, it is the one place a reader holding a path cannot reach the specifying document (168 of 185 paths resolve by first segment; 10 of the 17 that do not are that one namespace, across three extensions plus the core). Advisory check `spec shape` ships with it. Informative; no normative token moved.
- **v0.5:** §13 reconciled against every spec header, and §13.1a gained the naming invariant. Four extensions absent from both the classification and the dependency DAG are placed — REGISTRY, ROUTE and SUBSTITUTE at the tier each declares in its own header, SIGNALING by dependency — and ENCRYPTION's row, which described an early sketch, now records the landed extension. Thirteen version numbers corrected (stale by up to eleven minor revisions), DAG reading note 5 corrected (it called two authored extensions spec gaps), and §2's L2.5 box stopped restating the roster and now points at §13.1. New: §13.1a notes 5–6 — **the single-word extension name is the namespace, measured 26 of 26**, so grouping cannot live in extension names and belongs to the `SYSTEM-*` composition tier; §13.1b notes 8–9 — `Depends` cannot express an install set, and nine cited extensions have no file. §12 gains `PROFILE`. Informative throughout; no normative token moved.
- **v0.4:** Added §13.5a "Constraint regime and design divergence" — two informative axes beside §13.1's tiers. Axis A (constraint regime: substrate / design space / accumulated technology) names what governs a decision and therefore whose complexity it is; a regime applies to a decision rather than to an extension, and an extension's structure and motivation routinely sit in different ones. Axis B (design divergence, bands A–D) names how many defensible designs the landscape contains for an area, and pairs each band with the review method that fits it — correctness for the convergent core, landscape grounding for the divergent network family, cross-impl contract only for L5. Added because the tier axis alone cannot express why a wire decision and a transport decision are different kinds of decision, which is the "the boundary is blurry" observation the team keeps re-deriving. Informative; no normative change. Full treatment in `docs/research/explorations/EXPLORATION-THE-THREE-CONSTRAINT-REGIMES.md`.
- **v0.3:** Added §13 "Extension Classification and Stability Discipline" pulling together the four-tier extension classification (substrate / operational / extras / exploratory), the three legitimate reasons to spec (irreducible / operational / anti-fragmentation grounding), the V7-frozen-by-discipline rule, the publishing-horizon trajectory, the spec-vs-reference-implementation framing, and the disposition for the publishing window. Renumbered Document History to §14. The rationale and full retrospective backing this section live in `EXPLORATION-SCOPE-OVERREACH-RETROSPECTIVE.md`; the operational checklist is `GUIDE-EXTENSION-BUILDER.md`. Motivated by the durability retraction and the surfacing of the recurring scope-overreach pattern.
- **v0.2:** Updated for SDK domain and directory reorganization. Repository map reflects core-protocol-domain/ and sdk-domain/ split. Implementation inventory updated (Go and Rust have all 7 standard extensions complete). SDK specs added to "Where to Look." Open threads reflect SDK work. Actor roles updated (SDK coordination emerging).
- **v0.1:** Initial draft. Created to provide a navigable architecture map spanning core protocol, SYSTEM-COMPOSITION, extensions, and the SDK-track convergence.

---

*End of document. Internal working draft.*
