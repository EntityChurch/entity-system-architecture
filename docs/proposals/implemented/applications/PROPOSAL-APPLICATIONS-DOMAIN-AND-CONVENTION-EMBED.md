# Proposal — stand up the `applications/` domain + `APP-CONVENTION-EMBED` (v0.2)

> **Pulled in from the pre-split legacy archive, 2026-08-17, unedited.** This is a historical
> design record, carried across because `specs/applications/CHARTER.md` and
> `APP-CONVENTION-EMBED.md` both name it as their originating proposal. **Its dates use the
> legacy `cgid-10-NNN` scheme and are pre-genesis for this repo** (genesis: 2026-06-21) — read
> them as archive stamps, not as live dates. Documents it cites in `reviews/` live in that same
> archive and are not in this corpus. Its content is not re-dated or rewritten: a design record
> edited to look current is worse than one that is honestly old.

**Status:** Draft — **for cohort short re-confirm of the v0.2 corrections** (egui/Dom, workbench-go, godot),
then ratify. **Date:** cgid-10-225. **Workstream:** W1 (arch-chartered). **Author:** architecture.

> **v0.2 update (lead's critical pass — `reviews/CRITICAL-REVIEW-EMBED-BASE-MODEL-AND-HASH-AGILITY-cgid-10-225.md`).**
> The first cohort confirm caught nothing wrong with two things that *were* wrong; the lead's independent pass
> did. Two corrections, both **simplifying**: **(1) `hex33` removed** (it re-locked SHA-256 — the system is
> encoding-agnostic; now self-describing `content-hash`); **(2) the base/embed model made explicit + EmbedOutput
> trimmed** (markdown is the base document, embed is the escape hatch, EmbedOutput is a small result shape, not a
> universal page AST — prose is markdown/CommonMark, not EmbedOutput). v0.2 is **smaller** than v0.1. The
> re-confirm below is on these corrections; the rest of the v0.1 confirm stands.

> **What this proposes, and what it does NOT.** It (1) **pops out a new spec domain** `applications/` with a
> FORMAT-only charter, and (2) **authors `APP-CONVENTION-EMBED` v0.1** — the foundational rich-content primitive,
> carrying the **output-shape lock** (`G-PIN-1`/`[ASK-ARCH-OUTPUT-SHAPE]`) that all three L5 teams demanded.
> It does **not** fold the content-site spec yet (that is the *next* step — "EMBED first, then site," per the
> lead's call). It does **not** change the core protocol or any extension.
>
> **Why a proposal, not a direct standup.** A new normative spec domain + a cross-impl FORMAT contract is
> proposal-first work, and the lead chose "teams light-confirm the text before ratify." This is the vehicle. The
> *substance* is already agreed across all three teams (see provenance); this confirm is **"does the normative
> text match what we agreed,"** not a new design round.

---

## 1. The two artifacts (authored, Draft)

- **`applications/CHARTER.md`** — the domain charter. Five disciplines: FORMAT-only; no kernel/SDK machinery;
  improvise no protocol (needs → `[ASK-ARCH]` → core/extension proposal); valid floor; ships conformance vectors.
  Provisional scope boundary: format conventions here, peer-composition charters stay in `guides/` (godot's
  `[ASK-ARCH-APPLICATIONS-SCOPE]` — confirm or widen below).
- **`applications/APP-CONVENTION-EMBED.md`** — v0.1 spec. The `Embed` input entity + CDDL (`G-PIN-2`); the
  **`EmbedOutput` cross-substrate contract** (`G-PIN-1` — the lock); the two-level handler/renderer registry +
  renderer-caps/rendition-selection; the fallback ladder; the anti-graveyard contract; capability/compute + gate
  G1; required conformance vectors.

## 2. What the cohort confirms (the light-confirm checklist)

Each team confirms the **normative text matches the agreed decision** (cite a disagreement, don't redesign):

1. **`EmbedOutput` floor type (§4)** — is `Text | Image | Box{box_kind} | Raw | Fallback` with the enumerated
   `box_kind` set the shape your front-end can render, and is `Raw`-dropped-clean the right escape-hatch
   semantics? *(The one that must be byte-exact across impls.)* — **all three**, esp. godot (Control nodes),
   workbench-go (tview/Avalonia), egui (DOM).
2. **`Embed.data` CDDL (§3)** — tagged `payload` union, string-only `params`, mandatory `fallback`, no
   `data.media_type`, no `renderer_hint` string, ~16 KiB inline ceiling, substrate-neutral `sandbox` — all match?
   — **workbench-go** (raised G-PIN-2, S-2/S-3/S-4/S-6) primary.
3. **Handler discovery via `system/handler/*` (§5.1)** — correct, not a new registry namespace? — **workbench-go** (S-5).
4. **Renderer-caps + rendition selection (§5.3)** — the declared-capability + deterministic-selection contract
   work for your substrate? — **workbench-go + godot** (D-RENDERER-CAPS).
5. **G1 conditions (§7)** — C1–C5 captured correctly (esp. C3 cross-peer-refuse-by-default)? — **workbench-go**.
6. **Conformance vectors (§9)** — the right set; any missing the second impl would need?
7. **Charter scope (§CHARTER)** — confirm the provisional boundary (format conventions vs peer-composition
   charters in `guides/`), or widen it. — **godot** (`[ASK-ARCH-APPLICATIONS-SCOPE]`).

## 3. The gaps this formalization surfaced (the point of writing it)

Authoring the real text surfaced corners the synthesis left implicit — flagged here so the confirm closes them,
not so they linger:

- **`EmbedOutput` materialization (§4, §10 OPEN):** is the output ever a *stored* entity (cache/transfer), or
  always a transient handler→renderer value? v0.1 says *schema is the contract regardless*; confirm that's enough
  or whether a stored `app/embed-output/*` type is wanted.
- **`box_kind` floor completeness (§4):** the v1 set is `section/paragraph/list-*/table*/blockquote/code/columns/
  card/chart`. Is anything load-bearing missing for a real first site (e.g. `image-grid`/`gallery`, `figure` +
  caption)? Cheap to add now, awkward later.
- **Inline-directive grammar (§3, OPEN):** deferred to the first inline-authoring front-end (egui). Confirm
  child-entity transclusion carries v1 and the inline directive can lag.
- **`capability-decl` / `sandbox-constraint` shapes (§3):** sketched substrate-neutral; the *exact* CDDL for
  `requires`/`sandbox` is thin because active embeds are DEFER-behind-G1. Confirm thin-is-fine for v0.1 (full
  shape lands with the G1 build) vs. pin now.

## 4. Disposition & next steps

1. **Cohort light-confirm** (§2) → absorb (§3 gaps closed) → **ratify**: flip CHARTER + APP-CONVENTION-EMBED to
   non-Draft; the `applications/` domain is then live and managed.
2. **Then fold the content-site spec** as `APP-CONVENTION-SEMANTIC-CONTENT-SITE` (egui Rev 0.2 + the v1 Addenda
   from `SYNTHESIS-CONTENT-SITE-V1-LOCK` §2–§5, §8), consuming EMBED. *("EMBED first, then site.")*
3. **Ship conformance vectors** (EMBED §9 + the site vectors: signed `site-root` pin `G-PIN-3`,
   reproducible-publish `G-PIN-4`).
4. **Build handoff** to peers with vectors as the build target — where real implementation feedback comes.
5. **Parallel, unrelated:** the A4 EXTENSION-CONTENT ingest-guardrail erratum (egui + arch co-author) and the W2
   transport-compression flesh-out (`NOTE-COMPRESSION-SANITY-CHECK`) proceed on their own tracks.

**Not in scope here:** any core/extension change; the content-site fold; active-embed build (G1-gated).

## 5. Provenance
- `reviews/SYNTHESIS-CONTENT-SITE-V1-LOCK-cgid-10-225.md` (`e8a33c5`) — the aggregated v1 lock this formalizes.
- `reviews/REVIEW-L5-SEMANTIC-CONTENT-SITE-ARCH-FIRST-PASS-cgid-10-225.md` (`4351f73`) §5 — the domain proposal.
- Cross-team passes (all endorsed factoring EMBED): egui `FEEDBACK-TO-ARCH-CONTENT-SITE-FIRST-PASS`, workbench-go
  `FEEDBACK-CONTENT-SITE-SPEC-WORKBENCH-GO`, godot `REVIEW-L5-SEMANTIC-CONTENT-SITE-GODOT-CROSS-FRONT-END` (all cgid-10-225).
