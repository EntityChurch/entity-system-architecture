# `applications/` — the L5 Application-Convention domain — CHARTER

**Status**: Draft
`proposals/PROPOSAL-APPLICATIONS-DOMAIN-AND-CONVENTION-EMBED.md`).
**Owning workstream:** W1 (Outer Limits / application), arch-chartered.

---

## What this domain is

A home for **cross-impl, application-layer (L5) conventions** — standards that let independent applications and
independent front-ends converge on **one shared format** instead of each reinventing it. The first members are
`APP-CONVENTION-EMBED` (the rich-content typed-node primitive) and `APP-CONVENTION-SEMANTIC-CONTENT-SITE` (its
first consumer).

This domain exists because the spec taxonomy had no home for it. The core protocol (`core-protocol-domain/`) and
its extensions define the **substrate**; the SDK domain (`sdk-domain/`) defines the **bindings**; neither is the
place for "a content-format convention that web, Godot, and terminal front-ends all agree to render the same
way." Our standing stance (`GUIDE-EXTENSION-DEVELOPMENT` §3.5 — *"paths are convention; the entity graph is
coherence"*) said app-layer conventions belong **outside** the core protocol, but never gave them a canonical,
convergence-supporting home. This is that home.

## The charter discipline (what makes a convention a convention, not a dumping ground)

An `APP-CONVENTION-*` spec:

1. **Defines FORMAT — and only format.** The cross-impl contract is the **entity-type vocabulary**: the
   `{type, data}` shapes, their CDDL schemas, the dispatch keys, the on-wire/on-disk bytes. Two conformant impls
   that implement the convention produce byte-compatible entities and interpret each other's.

2. **Does NOT define kernel or SDK machinery.** *Format is the contract; rendering, overlay UX, and
   front-end wiring are per-front-end and live in the application, never here.* A convention adds **no** kernel
   features and **no** required SDK surface.

3. **Improvises no protocol.** Where a convention needs something the substrate lacks, it files an `[ASK-ARCH]`
   and the answer is a **core/extension proposal** (proposal-first) — never a normative invention at the
   convention layer. A convention may *recommend* an SDK helper or an extension, but it does not *define* one.

4. **Has a valid floor.** A convention must degrade to the substrate floor (a peer with less capability is a
   valid participant, not a degraded one). Capability *adds*; it is never *assumed*.

5. **Ships conformance vectors.** Because a convention is a byte-level cross-impl contract, it is not validated
   until vectors exercise it (the PRIMER meta-rule). Each convention ships example entities + expected hashes.

6. **Encoding-agnostic — never lock a hash width.** A convention references content by the **self-describing
   `content-hash`** `(format_code, digest)` per V7 §1.2/§1.4 — variable-length, format follows the leading
   varint. It MUST NOT bake a fixed-width form (e.g. a "33-byte / SHA-256" hash type) into its schema. The system
   supports SHA-256 / SHA-384 / BLAKE3 / future codes; a convention that assumes one re-locks the cage the
   v7.66–v7.70 agility arc removed. *(This discipline exists because EMBED v0.1 shipped `hex33` and three
   independent reviews missed it — the math/CS invariants, not cohort consensus, are the check.)*

## Scope boundary (provisional — confirm in the charter proposal)

`applications/` houses **format / content-vocabulary conventions** (`APP-CONVENTION-*`). **Peer-composition
charters** (operational wiring recipes — the named compositions: operational peer, observer, service pool,
hub-and-spoke, recovery cluster, bridge, compute pool) are a **different genus** (deployment recipes, not
cross-impl wire contracts) and stay in `guides/`. This boundary is **provisional** — godot raised it
(`[ASK-ARCH-APPLICATIONS-SCOPE]`); it is pinned here so the charter proposal can confirm or widen it, and so the
domain doesn't have to revisit it later.

## Members

| Spec | Role | Status |
|---|---|---|
| `APP-CONVENTION-EMBED` | Foundational — the generic rich-content typed node + two-level registry + output shape | Draft v0.2.3 — spine locked 3-way; **next: vectors** |
| `APP-CONVENTION-SEMANTIC-CONTENT-SITE` | First consumer — content sites built on Embed (document / content / compute anatomy) | Draft v0.4.2 — spine locked 3-way; `pages` cut; ordering floor pinned, semantic feeds open; v1 = manifest/page/nav/`.list`; **next: vectors** |

## Provenance

- Arch first pass: the L5 semantic-content-site architecture first-pass review (commit `4351f73`) §5 proposed this domain.
- Cross-team synthesis / v1 lock: the content-site v1-lock synthesis (commit `e8a33c5`) §6 ratified it.
- All three L5 teams (egui/Dom, workbench-go, godot) endorsed factoring `APP-CONVENTION-EMBED` as foundational.
