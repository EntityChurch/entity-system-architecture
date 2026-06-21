# Applications (L5) — Domain Roadmap (canonical, living)

**Target repo:** `entity-system-architecture` (the applications sub-domain).
**What this domain is:** the **application-layer (L5) convention domain** — cross-impl standards
that let independent applications and independent front-ends (web, Godot, terminal) converge on
**one shared content format** instead of each reinventing it. The core protocol defines the
substrate; the SDK defines the bindings; this domain is the home for "a content-format
convention that all front-ends agree to render the same way."

> **Canonical & living.** Updated as the L5 conventions move; the content/papers projects pull
> from it. Maturity (M0–M6) is defined in **`STATUS-RELEASE-SURFACE-AND-MATURITY-CANONICAL.md`**;
> this is the forward, stage/phase view.

**Owning workstream:** W1 (Outer Limits / application).
**Charter:** `applications/CHARTER.md` (Draft).

---

## The charter discipline (what makes a convention a convention)

An `APP-CONVENTION-*` spec **defines FORMAT and only format** — the entity-type vocabulary
(`{type, data}` shapes, CDDL schemas, dispatch keys, on-wire/on-disk bytes). Two conformant
impls produce byte-compatible entities and render each other's. It is *not* a place for
application logic, UI, or handler behavior. "Paths are convention; the entity graph is
coherence" (GUIDE-EXTENSION-DEVELOPMENT §3.5).

## Artifacts & maturity

| artifact | version | hdr | maturity | role |
|---|---|---|---|---|
| `applications/CHARTER` | — | Draft | **M1** | the domain charter + convention discipline |
| `APP-CONVENTION-EMBED` | 0.2.3 | Draft | **M1→M2** | the generic rich-content typed-node primitive |
| `APP-CONVENTION-SEMANTIC-CONTENT-SITE` | 0.5 | Draft | **M1→M2** | content sites built on Embed (the first consumer) |

**Important:** this domain is **mostly plan, not spec.** The SITE convention exists as a draft;
SPACES and REPOS are named directions with **zero design** today. Distinguish plan from
spec-state when describing what ships.

## Stages / phases

**Stage — Embed + SITE convergence (IN FLIGHT, the high-yield work).**
`APP-CONVENTION-EMBED` (the rich-content typed node) and `APP-CONVENTION-SEMANTIC-CONTENT-SITE`
(its first consumer) had a three-substrate joint round — the spine is locked three ways
(workbench-go + Godot + egui). SITE v0.5 corrected a layer violation (sites are now free
subgraphs at publisher-chosen tree paths, not under `system/content/*`) and added the `sites`
URL-projection prefix. **Next phase:** cut joint conformance vectors → ratify. The
three-substrate round was the sweep; vectors are the lock.

**Stage — 06-21 preview (SITE read-only).** What actually ships at the preview is the **SITE
read projection only**. The Embed/SITE spine is paper-frozen and converged; the demo is a read
path over published content, with honest "unverified" UI until the registry's verification path
(REGISTRY C1 / published-root) lands cohort-wide.

**Stage — Forms (PARKED, opens post-release).** Forms reached a 5-round paper exploration but
are **paper-frozen, not in spec** — they open in a later cycle (v0.6+ post-release). Do not
describe forms as shipping.

**Stage — SPACES / REPOS (FUTURE, undesigned).** Named L5 conventions (collaborative spaces;
repository hosting) that will share the SDK `browse_*` seam with SITE — same navigation
scaffold, different destination interpretation. **Zero design today**; name-drops only.

## Relationship to the rest of the system

- **Below:** depends on EXTENSION-CONTENT (the content-hash address space) + the core tree.
- **Beside:** shares the W1 `browse_*` SDK seam (SITE/SPACES/REPOS navigate the same way).
- **Not in core, not an extension:** app conventions are deliberately outside the protocol —
  divergence is allowed; convergence is offered, not mandated.

## Notes for the content/papers project

- The framing: **"write content once, render it the same everywhere"** — the cross-front-end
  convergence story (web + game engine + terminal agree on the bytes).
- Be precise about state: **SITE read = shipping at preview; Embed/SITE format = converged
  paper; Forms = parked; SPACES/REPOS = future, undesigned.** This domain is where "plan vs
  spec" confusion is most likely — keep them distinct.
