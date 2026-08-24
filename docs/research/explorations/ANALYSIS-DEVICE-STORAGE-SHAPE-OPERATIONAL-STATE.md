# ANALYSIS — the shape question: is "storage stats" even an extension?

**Phase:** Analysis (pre-proposal). 2026-07-12. **Companion to**
`EXPLORATION-SUBSTRATE-INTROSPECTION-DEVICE-AND-STORAGE.md` — that doc surveyed *what* to
expose; this one answers *what shape it takes in our system*, prompted by the operator's
instinct: **"it seems like a big extension just to get some storage stats — is there a more
general idea?"** There is, and it collapses most of the proposal.

---

## §0 TL;DR

**Storage-stats is not an extension. It's operational state.** Our system already has a ruled,
named answer to "how does a peer expose facts about itself": **the peer writes self-describing
entities into its own tree, read through the ordinary `tree`/`query` handlers** (`GUIDE-
OPERATIONAL-STATE §1`; `GUIDE-INSPECTABILITY §1` — "why a guide and not an extension"). Under that
lens:

- **Entity/path counts** → already `system/query:count` + `EXTENSION-QUERY §2.5` prefix
  zone-summaries (`{count, min, max, null_count}`). **Zero new surface.**
- **Store accounting** (blobs, bytes, reclaimable churn) → a small **operational-state entity**
  the peer maintains at `system/…` and you read with `tree:get`. Same model as `system/connection`
  / `system/config`. The reclaimable-churn number is the exact signal `GUIDE-GC` already wants.
- **Device** is the *only* genuinely distinct case — not because it's a query extension, but
  because device facts are the one thing here that is **not entity-native**: they describe the
  substrate the peer runs on, not the peer's own state, so they need a **provider** to project
  host → tree. Even then the right shape is an **operational-state vocabulary** (`system/device/*`
  entities a provider populates), not a bespoke query handler.

**Net: the "two new §5.5 query-handler extensions" framing is wrong.** The genuinely new surface
is tiny: **(1) the availability descriptor** (a reusable field primitive), **(2) a `system/device`
entity vocabulary + a per-runtime host provider** (provider is impl-local), and optionally **(3)
one QUERY aggregate** if content-store byte totals aren't reachable from the tree indexes. No big
extension.

## §1 The instinct, stated precisely

`system/storage:stats` returning `entity_count` is a whole handler/op/entity-type/capability
surface to report a number the peer already computes locally. That asymmetry is a smell. The
question isn't "how do we spec this handler" — it's "**what existing thing is this an instance
of?**"

## §2 The system already answers "how a peer describes itself" — three ruled surfaces

1. **Operational state** (`GUIDE-OPERATIONAL-STATE §1–2`). *"The entity vocabulary a peer uses to
   describe itself… everything is entities written to the tree through the standard emit pathway…
   every peer is self-describing and observable through the same mechanisms used for everything
   else."* The five V7 §3.13 types (`transport`, `config`, `peer/status`, `connection`,
   `peer/alias`) are read via `tree:get` and subscribed to — **no bespoke stat handler**. The
   explicit design cost is "a handful of tree writes per event"; the win is uniform readability.
2. **Inspectability** (`GUIDE-INSPECTABILITY §1`). The architecture already **ruled that observing
   a peer is a guide, not an extension** — "inspectability emerges from the model." §2.2 names
   **path enumerator** and **entity reader** as read-side primitives that *need no event hooks or
   new machinery* — pure substrate reads. §4 frames the one real choice: **out-of-band vs
   entity-native vs hybrid** population.
3. **Query aggregates** (`EXTENSION-QUERY`). `count` is a Level-1 op; §2.5 **path-prefix zone
   summaries** maintain `(prefix, type, field) → {min, max, count, null_count}` incrementally.
   "How many entities under this prefix" is already answerable.

Three surfaces, all existing, all converging on the same model: **a peer's self-facts live as
entities in its tree and are read/aggregated/subscribed through the ordinary handlers.**

## §3 Storage-stats collapses into that model

| Store fact | Where it already lives / should live | New surface? |
|---|---|---|
| entity count / live-path count, per prefix | `system/query:count` + §2.5 zone-summaries | **none** |
| content-store blob count, physical bytes, **dedup savings** | below the tree indexes (content substrate) → an **operational-state accounting entity** the store maintains (populate via emit; read via `tree:get`); *maybe* one QUERY aggregate if a byte total must be computed | **small** (an entity type + populate; possibly 1 aggregate op) |
| reclaimable / save-state churn | same accounting entity — and it **is** the `GUIDE-GC` retention-pressure signal; don't invent a second one | **none new** — reuse the GC signal |

So `system/storage` as an *extension with a handler op surface* is the wrong shape. The store's
self-accounting is **operational state** — a `system/store/…` (or `system/storage/…`) entity the
peer maintains like `system/connection`, read remotely via `tree:get` (capability-gated by path,
exactly the privacy model we want), aggregated via QUERY. **Remote-readability — the whole
motivating requirement — is satisfied by `tree:get` being cross-peer dispatchable; it never needed
a `:stats` op.**

## §4 Device is the real distinction — and *why* is the interesting part

Operational state works for connections/config/transport because those **are** the peer's own
state, naturally entity-shaped. Device facts are the one thing in this design that **isn't
entity-native**: they describe the *host substrate* (OS, RAM, disk, GPU), which is not the peer's
state and does not arise in the tree on its own. The genuinely new mechanism is therefore a
**provider that reads the host and projects it into entity form** — impl-local, per-runtime
(Rust `sysinfo`, a Go/Py equivalent, browser `navigator.*`).

But the **shape** should still be operational-state, not a query extension:

- A provider **populates `system/device/{domain}` entities** (`platform`, `cpu`, `memory`,
  `storage`, …) — the same way `system/transport/{protocol}` is populated. You **read them with
  `tree:get`** and **subscribe** to them. No new op surface, no new handler contract beyond the
  provider that writes them.
- **Static facts** (platform, arch, core count) → populate once at startup: cheap, always-there.
- **Dynamic facts** (free disk, memory pressure) → the `GUIDE-INSPECTABILITY §4.3` **hybrid**:
  populate on-demand / refresh-on-read rather than eagerly, because they're rate/gauge-volatile
  (§6 of the exploration — the sysinfo diff lesson). This is a known, named pattern, not a new one.
- **Capability/privacy** falls out for free: reading `system/device/{domain}` is gated by ordinary
  path-scoped read grants — *opt-in-per-domain* is literally "grant read on `system/device/gpu`."
  Cleaner than op-grants.

So device = **entity vocabulary + host provider + the availability descriptor**. Lighter than "an
extension," and consistent with how the peer already describes itself.

## §5 What's actually new (the minimal surface)

1. **The availability descriptor** `{state: value|unsupported|denied|unknown, value?, fidelity?}` —
   a tiny reusable field primitive (exploration §4). The one genuinely novel, genuinely reusable
   thing. Lands first, on its own.
2. **The `system/device` entity vocabulary + per-runtime provider.** The vocabulary (entity types
   per domain, fields, units) is cross-impl-observable → spec'd as an operational-state addition
   (sibling to V7 §3.13's five). The provider is **impl-local** → not spec'd.
3. **A `system/store` accounting entity** (blobs/bytes/reclaimable) — small operational-state
   vocabulary addition; entity/path counts reuse QUERY; reclaimable reuses the GC signal.
4. **(maybe) one QUERY aggregate op** if a content-store byte total genuinely can't be derived from
   the existing indexes. Confirm before adding — prefer reuse.

No new "storage extension." The device piece is a vocabulary + provider, not a handler-op extension.

## §6 Consequence for the proposal

Re-shape the forthcoming `PROPOSAL-*` accordingly — it is **not** "two new §5.5 query-handler
extensions." It is:

- **the availability descriptor** (reusable primitive, first);
- **`system/device` as an operational-state vocabulary** (V7 §3.13 sibling) + the honest-provider
  contract, read via `tree:get`/subscribe, path-grant-gated;
- **store self-accounting** as a small operational-state entity, counts via QUERY, churn via GC;
- explicitly **reuse** `tree`, `query`, `subscription`, operational-state, and GC rather than
  standing up a parallel stat surface.

This is dramatically smaller than the intake's two-handler framing, more consistent with the
system's own "self-describing peer" model, and answers the operator: **you don't need a big
extension for storage stats — storage stats were never an extension; device is a thin provider +
vocabulary.**

## §7 One honest open tension (for the proposal to resolve)

**Populate-as-entities (operational-state) vs answer-on-query (a handler op).** `GUIDE-INSPECTABILITY
§4` frames exactly this: entity-native population is uniform and subscribable but costs writes and
can stale; on-demand query is fresh but bespoke. For volatile device gauges (free disk, memory
pressure) the honest answer is likely **hybrid** — a `refresh-on-read` populate, or a thin
`system/device:refresh` that repopulates the entities — but this is the real design decision to pin,
with the gauge-vs-rate split (exploration §6) as its input. Not a blocker; the *shape* (operational
state, not a new extension) holds either way.

## §8 Next

- Fold this reframe into the main exploration's shape/level sections (they currently say "extension-
  tier handler contracts" — refine to "operational-state vocabulary + provider + descriptor").
- When drafting `docs/proposals/PROPOSAL-…`, lead with the **availability descriptor**, then the
  **`system/device` operational-state vocabulary**, then **store self-accounting via QUERY + GC** —
  and state the reuse explicitly so no one re-adds a stats extension.
- Sanity-check with the cohort: can content-store byte totals be derived from existing indexes, or
  is one QUERY aggregate genuinely needed? (Prove before adding.)

## Sources (in-repo)

`guides/GUIDE-OPERATIONAL-STATE.md` §1–2 · `guides/GUIDE-INSPECTABILITY.md` §1, §2.2, §4 ·
`specs/extensions/EXTENSION-QUERY.md` §1.2 (`count`), §2.5 (zone summaries) · `guides/GUIDE-GC.md`
(retention-pressure signal) · V7 §3.13 (the five operational-state types).
