# Entity System — Architecture Overview

**Version**: 0.4
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
│ L5: APPLICATIONS / DOMAINS                                  │
│   User-facing software. Games, tools, front-ends.           │
│   Combinatorial, problem-specific, infinite.                │
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
│   Inbox, Continuation, Subscription, Clock, Query, History, │
│   Revision, Transaction, Content, Compute, Network, Type,   │
│   Role. Each exercises a specific pair-bundle of primitives. │
│   Composable and deployment-selectable.                     │
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
├── ENTITY-SYSTEM-REFERENCE.md               ← condensed working reference
├── SPECIFICATION-FORMAT.md                  ← spec authoring standard
├── STYLE-NAMING-CONVENTIONS.md              ← identifier naming standard
├── extensions/                          ← L2.5 extension specs
├── sdk/                                 ← L3-L4 SDK specifications
├── applications/                        ← L5 conventions + the domain CHARTER
└── domains/                             ← domain specs
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

Irreducible add-ons — a peer fundamentally needs these to run. Together with Tier 0 and V7, they define what a peer is. Eleven members today:

| Member | Status | Role |
|---|---|---|
| `EXTENSION-TREE` | Draft v3.13 | Snapshots / diffs / merges / view-trees over the core-protocol tree. (Top-level in `specs/`, not nested under `extensions/` — a placement choice reflecting its tight coupling to V7's tree handler.) |
| `EXTENSION-CONTENT` | Draft v3.4 | Content store ingestion. |
| `EXTENSION-INBOX` | Draft v5.9 | Async delivery handler. |
| `EXTENSION-SUBSCRIPTION` | Draft v3.11 | Publish-subscribe / cross-peer reactivity. |
| `EXTENSION-CONTINUATION` | Draft v1.12 | Chained execution / cross-peer dispatch. |
| `EXTENSION-COMPUTE` | Draft v3.18 | Embedded expression language; programs-as-entities; the standard-IR floor (nav primitives, MUST collection stdlib, pinned integer model). Foundational for entity-native compute. |
| `EXTENSION-QUERY` | Draft v1.7 | Query operations over the tree. |
| `EXTENSION-REVISION` | Draft v3.2 | Versioning + three-way merge / cross-peer merge convergence. |
| `EXTENSION-HISTORY` | Draft v1.6 | Path-level transition recording. |
| `EXTENSION-TYPE` | Draft v1.0 | Value-level constraints layered above the Tier 0 type system. Currently deferred in implementation because the team is close enough to the ground to enforce manually; the proper-form-of-types in the system needs it. |
| `EXTENSION-CLOCK` | Draft v1.3 | System time — wall-clock primarily; logical / vector / HLC sub-pieces are reference research, kept in the spec. |

These freeze in the first wave alongside V7 and Tier 0 (§13.4).

#### Tier 2 — Operational extensions

Needed for **production multi-peer deployments**. A single peer can run without these on V7 + bare-key-pair identity. Required for identity management, peer networking, and operational management. Decomposes into three sub-groups; **most of this tier is now authored** (identity/authority/membership in 2a; NETWORK + DISCOVERY + RELAY in 2b), and the **remaining gaps are in operational management (2c — GC, persistence)**.

##### Tier 2a — Operational user-identity

How identity, authority, and membership are managed above raw key-pair identity.

| Member | Status | Role |
|---|---|---|
| `EXTENSION-IDENTITY` | Stable v3.10 | Peer identity, controllers, rotation. |
| `EXTENSION-ATTESTATION` | Stable v1.2 | Substrate primitive *within the operational tier* — generic attestation graph consumed by IDENTITY, QUORUM, ROLE, GROUP. |
| `EXTENSION-QUORUM` | Stable v1.2 | K-of-N node primitive within the operational tier. |
| `EXTENSION-ROLE` | Draft v2.0 | Role-based authority. |
| `EXTENSION-GROUP` | Stable v1.4 | Membership / registries. |

##### Tier 2b — Operational network

Cross-peer connection, sync, discovery, relay. Architecturally established for some time; **now authored** — NETWORK, DISCOVERY, and RELAY all carry normative specs.

| Member | Status | Role |
|---|---|---|
| `EXTENSION-NETWORK` | Draft v1.3 | Sync, peer connection, bootstrap conventions. |
| `EXTENSION-DISCOVERY` | Active | Peer discovery beyond ad-hoc connection (mDNS / local + registry-assisted resolution). Authored as a normative extension. |
| `EXTENSION-RELAY` | Active | Peer-to-peer message relay through intermediaries — four canonical modes (Forward / Store-and-poll ship in v1; Aggregate / Circuit named-but-deferred). Reauthored against the v7-substrate model. |

##### Tier 2c — Operational management

Operational concerns above networking. Currently mostly gaps; this is where the architecture team's "we know we need these eventually" thinking lives.

| Member | Status | Role |
|---|---|---|
| **GC (garbage collection)** | Spec gap; nature unsettled. | Operational storage management. **Borderline question:** is this an extension or implementation-management guidance? Theoretically with unbounded storage a peer never needs GC, so it's optional in a strict sense. The architecture team plans to outline it either way to set expectations; whether it lands as an extension or as a guide is a downstream decision. |
| **Persistence** | Partial — `GUIDE-PERSISTENCE.md` covers the SDK side (configuration directory / per-peer state / runtime state in tree). No core-protocol-domain extension. | Local-storage substrate. SDK-side guidance exists; core-protocol-domain treatment is the gap. Overlap with GC is real and to be resolved when either lands. |
| `PROPOSAL-EXTENSION-ENCRYPTION` | Early sketch — no Status header. | The user has confirmed encryption is required ("there's no way around it"). May belong in operational management or as its own tier. To be decided when the proposal lands. |

These sub-tiers freeze in the second wave (§13.4), after substrate stabilizes in cross-impl.

#### Tier 3 — Extras / first-pass grounding

Major CS concepts the community will expect. May converge or diverge as the community uses the system. Defensible as *anti-fragmentation* work (§13.2 reason 3) — better to have a coherent first pass than have downstream implementers invent four competing semantics.

| Member | Status | Role |
|---|---|---|
| `EXTENSION-TRANSACTION` | Draft v0.1 (pre-review) | Atomic multi-binding writes + observation boundary. The isolation-level apparatus (`snapshot` / `serializable`) is RDBMS pattern-matching and is a narrow-candidate; the extension itself is real anti-fragmentation work that bridges toward the eventual Raft / consensus roadmap. |
| Future extras may emerge | — | If a major CS concept surfaces with expected community demand and no existing-extension coverage, it lands here. |

May never freeze in this repo — converge through community use.

#### Tier 4 — Exploratory

Preserved reference design after a retraction or before a driver is identified. **No deployment is required to install.** Status header explicitly marks these as *exploratory · optional · not actively developed*.

| Member | Status | Role |
|---|---|---|
| `EXTENSION-DURABILITY` | v0.1 Exploratory | Retracted from spec. The lifted material is preserved verbatim; the durability contract is not normative for any tier. |

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
│       └── EXTENSION-RELAY             (Tier 2b; under NETWORK; Active)
│
├── EXTENSION-CONTINUATION              (Tier 1; chained execution; independent of INBOX spec-wise; composes operationally)
│   └── EXTENSION-NETWORK               (cross-dep)
│
├── EXTENSION-CONTENT                   (Tier 1; content store ingestion; independent)
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
└── EXTENSION-ROLE                      (Tier 2a; depends on V7 directly; composes-with IDENTITY but does not strictly depend on it; SUBSCRIPTION + CONTINUATION optional)
    └── EXTENSION-GROUP                 (cross-dep)
```

**Reading the DAG:**

1. **V7 (Tier 0) is the root.** Every Tier 1 substrate extension depends directly on V7.
2. **TREE is the early branch.** REVISION, TRANSACTION, and several other extensions build on TREE's snapshots/diffs/merges.
3. **The async delivery sub-tree** is INBOX → SUBSCRIPTION → NETWORK (+ CONTINUATION as a sibling). These have real spec-level dependencies on each other.
4. **The user-identity chain** is ATTESTATION → QUORUM → IDENTITY → GROUP, with ROLE sitting independently at the V7 level but composing-with IDENTITY/GROUP. The "core primitive" pair (ATTESTATION + QUORUM) sits below the named identity surfaces, exactly as user described: attestation + quorum are foundation; identity + group depend on them.
5. **The network sub-tree** has NETWORK at the root and discovery + relay positioned underneath. Both are spec gaps; the dependency structure is established by the conceptual model even though normative specs are missing.
6. **Several extensions are spec-independent** of others — CONTENT, COMPUTE, HISTORY, CLOCK only depend on V7. These compose freely without inter-extension constraints.
7. **The cross-dependencies** make the DAG slightly web-like — QUERY depends on both TYPE-SYSTEM and SUBSCRIPTION; TRANSACTION depends on TREE with optional REVISION; NETWORK depends on INBOX + CONTINUATION + SUBSCRIPTION jointly. These are real but few in number; the graph stays tree-readable.

**Implications for freeze sequencing:**

- A node freezes only when everything it depends on is frozen. V7 must freeze before TREE; TREE before REVISION; etc.
- The substrate tier (Tier 1) can freeze in waves: first the no-dependency leaves (CONTENT, COMPUTE, HISTORY, CLOCK), then TREE, then SUBSCRIPTION (after INBOX), then NETWORK (after its three deps), then QUERY/TRANSACTION (which have cross-deps).
- The operational tier (Tier 2) waits on substrate. Within Tier 2a, ATTESTATION + QUORUM freeze first; IDENTITY then; GROUP last.
- GC / persistence (the remaining Tier-2c gaps) need spec authoring before they can freeze at all. (Discovery + relay are now authored — `EXTENSION-DISCOVERY` / `EXTENSION-RELAY`, Active.) These are explicitly the *post-publishing* community-and-team work surface.

### 13.1a Reading the classification

A working note on how to use this taxonomy:

1. **It's not strict.** A given extension may straddle two tiers; the classification puts it in the tier that best captures its freeze trajectory.
2. **It shows the spec-surface gaps.** The gap rows in Tier 2 (discovery, relay, GC, persistence as a core-protocol-domain extension) are real **spec gaps**, not concept gaps. The team has discussed these for a long time — relay has a prior Rust implementation in a legacy project; discovery has a deferred proposal and a registry-and-discovery landscape review; GC is borderline extension/management; persistence has SDK-side coverage. The work the team has not yet done is authoring the current normative specs. Calling them out here means we know what's missing on the published surface, not that the concepts are novel.
3. **Tier movement is expected.** As the community uses the system, items in Tier 3 (extras) may converge to Tier 2 (operational) if they earn that, or stay in Tier 3, or migrate to Tier 4 (exploratory). The classification is a snapshot.
4. **CLOCK stays in Tier 1** as a single extension; the wall-clock surface is irreducible and the logical/vector/HLC research notes coexist with it. (The team has decided not to split CLOCK — keeping it as one spec is cleaner than fragmenting on a research question.)

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

### 13.6 Disposition for the next 3–4 weeks

The work the architecture team should plan to complete before publishing:

- **Status-header sweep.** Every extension and proposal gets its Status header explicitly tiered per §13.1. Today many specs say only "Draft" or "Stable"; the tier (substrate / operational / extras / exploratory) belongs in every header.
- **V7 modification audit.** Sweep V7's recent amendments (v7.40 onward) against §13.3's independent-justification test. The v7.47 reversion is one outcome of that test; there may be others.
- **Exploratory-tier framing.** Items that should land as exploratory (rather than be retracted entirely) get framed cleanly with a Status header and a preserved-design-record statement.
- **Migration plan for retained rationale.** Some of the design exploration that drove current specs (the 30+ explorations, the durability retrospective, the substrate split history) is appropriate as a *legacy archive* — the architectural rationale is preserved, but not in the same surface as the published specs and guides. The published release should be "specs + guides + SDK specs + SDK guides + reference implementation," fresh and clean. Rationale lives in the archive for those who want to understand the system's history.
- **Implementation-team handoff.** Each freeze wave (substrate-first, then operational) involves coordinating with the impl teams on the freeze boundary. The teams need to know *what they can rely on not changing* before they commit to the published-version interop.

---

## 14. Document History

- **v0.4:** Added §13.5a "Constraint regime and design divergence" — two informative axes beside §13.1's tiers. Axis A (constraint regime: substrate / design space / accumulated technology) names what governs a decision and therefore whose complexity it is; a regime applies to a decision rather than to an extension, and an extension's structure and motivation routinely sit in different ones. Axis B (design divergence, bands A–D) names how many defensible designs the landscape contains for an area, and pairs each band with the review method that fits it — correctness for the convergent core, landscape grounding for the divergent network family, cross-impl contract only for L5. Added because the tier axis alone cannot express why a wire decision and a transport decision are different kinds of decision, which is the "the boundary is blurry" observation the team keeps re-deriving. Informative; no normative change. Full treatment in `docs/research/explorations/EXPLORATION-THE-THREE-CONSTRAINT-REGIMES.md`.
- **v0.3:** Added §13 "Extension Classification and Stability Discipline" pulling together the four-tier extension classification (substrate / operational / extras / exploratory), the three legitimate reasons to spec (irreducible / operational / anti-fragmentation grounding), the V7-frozen-by-discipline rule, the publishing-horizon trajectory, the spec-vs-reference-implementation framing, and the disposition for the publishing window. Renumbered Document History to §14. The rationale and full retrospective backing this section live in `EXPLORATION-SCOPE-OVERREACH-RETROSPECTIVE.md`; the operational checklist is `GUIDE-EXTENSION-BUILDER.md`. Motivated by the durability retraction and the surfacing of the recurring scope-overreach pattern.
- **v0.2:** Updated for SDK domain and directory reorganization. Repository map reflects core-protocol-domain/ and sdk-domain/ split. Implementation inventory updated (Go and Rust have all 7 standard extensions complete). SDK specs added to "Where to Look." Open threads reflect SDK work. Actor roles updated (SDK coordination emerging).
- **v0.1:** Initial draft. Created to provide a navigable architecture map spanning core protocol, SYSTEM-COMPOSITION, extensions, and the SDK-track convergence.

---

*End of document. Internal working draft.*
