# APP-CONVENTION-EMBED — the generic rich-content typed node — v0.2.3 DRAFT

**Status**: Draft
+ refined per the v0.2.1 base-plurality-and-output-basis refinement
+ the v0.2.2 raw-role-and-reserved-kinds refinement
+ **joint cross-team round folded** per the joint embed/site convergence synthesis (PF-2/5/6/7;
spine locked three ways by workbench-go + godot + egui). Pairs with `APP-CONVENTION-SEMANTIC-CONTENT-SITE` v0.4.
**Next: cut joint conformance vectors → ratify** (the three-substrate round was the high-yield sweep; the vector
sprint is the lock). Proposal: `proposals/PROPOSAL-APPLICATIONS-DOMAIN-AND-CONVENTION-EMBED.md`.
**Domain:** `applications/` (first member — see `CHARTER.md`). **Charter class:** FORMAT-only.
**Consumers:** `APP-CONVENTION-SEMANTIC-CONTENT-SITE` (first); workbench-go panels; Godot panels.

> **v0.2 corrections (lead's critical pass):** (1) **`hex33` removed** — it re-locked SHA-256; the system is
> encoding-agnostic, so all hash references are the self-describing **`content-hash`** `(format_code, digest)` per
> V7 §1.2 (§3). (2) **base/embed model explicit + `EmbedOutput` trimmed** to a small result-shape, not a page AST.
>
> **v0.2.1 refinements (lead + both peers, prose only — CDDL byte-stable):** (3) **base format is the consuming
> convention's choice, not EMBED's** — v1 recommends markdown for reach, but EMBED is base-format-independent
> (§1.1). (4) **The output basis & the extension point are now stated** (§4): EmbedOutput is a **closed basis of
> display primitives — four leaves (`text/image/raw/fallback`) + one container (`box`)**; you **extend by writing
> a handler** (open, input layer) that **lowers** your new type into the basis — you do **not** add output kinds.
> "Why image and not chart?" image is irreducible; a chart composes → a chart is a handler, not a primitive.
>
> **v0.2.2 refinements (lead's pass, prose + one reserved-slot note — CDDL byte-stable for the required five):**
> (5) **the five are reframed as a CLOSED DISPATCH VOCABULARY filling four roles** (2 content atoms + 1 container +
> 1 floor + 1 escape), not "five co-equal display primitives" — fixes the self-contradiction that `raw` was called
> a "primitive every renderer must implement" when it is precisely the one kind a renderer MAY drop. **`raw` is the
> escape/FFI-to-native, not a primitive** (§4.0). (6) **`media` + `interactive` are now RESERVED candidate kinds**
> (§4.0a), the V7 §1.2/§1.5 reserved-range discipline applied to the display basis — named/sketched/slot-held,
> **experimental, not required in v0.2, not locked** — so "the basis is closed" no longer means "we pretend no
> other irreducible kind exists."

> **What this is.** The single mechanism for **all** rich content beyond plain text in an entity-system L5
> application: images, charts, tables, forms, video, "whatever a guy dreams up." It is **one typed entity + one
> registry + one fallback contract** — it invents no markup language and no new kernel machinery. An image is the
> trivial floor instance; a chart is the same shape with a different type tag and a handler.
>
> **The load-bearing idea (B1, grounded).** Every extensible markup system in 30 years of practice converged on
> the same node shape — `{type, attributes, payload, fallback, raw-escape}`. **That shape is structurally an ECF
> entity** `{type, data}`. So rich content is not a new format problem; it is the entity model applied at the
> presentation layer. *Adding a new media type = a new `type` tag + a handler. The grammar never changes.*

---

## 1. Scope, layering, floor

This convention defines **format**: the `Embed` input entity (§3), the `EmbedOutput` structured value (§4 — the
cross-substrate lock), the two-level handler/renderer registry (§5), and the degradation contract (§6). It adds
**no** kernel feature and **no** required SDK surface (CHARTER discipline 2).

### 1.1 The base/embed model `[LOCKED — read this first]`
Three layers, kept strictly distinct (this is what keeps the convention small and prevents the "is everything an
embed?" confusion):

| Layer | What | Rendered by | Cross-substrate contract |
|---|---|---|---|
| **Base — the document** | the document body in a **base format chosen by the consuming convention** (v1 recommended: markdown) | each front-end's **own base-format renderer** | the base format's own standard (v1: **CommonMark / GFM**) |
| **Embed — the escape hatch** | a typed node for what the base **cannot** express (image, chart, form, video, live table, 3D scene, compute widget), inserted into a document or transcluded | handler → `EmbedOutput` → renderer | the `Embed` entity (§3) |
| **EmbedOutput — an embed's result** | the small renderer-neutral shape an embed *handler* returns | each front-end's renderer | `EmbedOutput` (§4) — a closed basis |

**Base format is the consuming convention's choice — not EMBED's.** A document has a **base format**; in v1 the
recommended base is **CommonMark/GFM markdown** — it renders on every substrate and has a public conformance
suite, so it is the **universal-reach baseline**. Other base formats (HTML at the web tier behind G2; future:
Jupyter, canvas, slide decks) are permitted as the consuming convention's choice, **at the cost of reach** (a
substrate that can't render that base can't render the document). This is the same shape as V7 §1.2a content-format
*standard-compliance vs. price-of-divergence*: markdown = the SHOULD-baseline for reach; alternatives = permitted,
priced. **EMBED's contract is base-format-independent** — an embed is a typed insertion into *whatever* base the
consumer chose.

**The base is NOT an embed.** The base renderer is the **base renderer**, not an embed; embeds are typed
**insertions**. **EmbedOutput is the shape of an embed's result — never the lowering target for the base
document's prose.** In the markdown base, a paragraph with a link, emphasis, a list, a GFM table is **markdown**,
rendered by the base floor; it does **not** live in `EmbedOutput`. (Distinguish **markdown-as-base** — the
document body — from **markdown-as-content** — a transcluded `app/embed/markdown` fragment; different layers, not
a contradiction.) An embed may *contain* sub-content (a figure = image + caption) via bounded `box` composition
(§4), but the document never recurses through itself. This is the web's own split (HTML base; `<img>`/`<video>`/
`<canvas>` as embeds). *We are a renderer, not a format converter — so, unlike Pandoc, we need no universal
Block/Inline AST; the base carries prose, EmbedOutput carries only what the base can't.*

**Two-level model.** `media_type → handler → EmbedOutput → renderer → presentation`. Two steps, two trust
domains: the **handler** (untrusted code, data→output) and the **renderer** (trusted per-front-end code,
output→presentation). The same `{type → dispatch}` discipline applies twice: `media_type` routes to a handler;
`EmbedOutput` kind routes to a renderer (§4).

**Floor.** A front-end with no handler for an `Embed`'s `media_type` renders its **mandatory `fallback`** (§6).
A passive image/av `Embed` has a trivial pass-through handler — no untrusted code at all — so passive media
"just works" at the floor. No capability above the floor is required to participate.

## 2. The two surfaces at a glance

```
INPUT  (authored, content-addressed, travels in the subtree)
  Embed entity   type = app/embed/{media_type}    data = §3 CDDL

         │  handler (media_type → output): untrusted, content-addressed,
         │  capability-gated, sandboxed; discovered via system/handler/* (§5)
         ▼
OUTPUT  (structured value; renderer-neutral; the cross-substrate contract)
  EmbedOutput    Text | Image | Box{kind} | Raw{format} | Fallback   (§4)

         │  renderer (output-kind → presentation): trusted, per-front-end
         ▼
PRESENTATION   web→DOM · Avalonia→controls · tview→cells · Godot→Control nodes
```

## 3. The `Embed` input entity `[LOCKED shape / G-PIN-2 schema]`

An `Embed` is an ECF entity. **The type tag is the dispatch key** (`app/embed/{media_type}`); there is **no
`data.media_type` field** (it would be a redundant second source of truth — dropped per cross-team S-4).

```cddl
; --- shared atoms (CDDL-complete; resolves workbench-go PF-1 / godot "define hex33") ---
content-hash = bstr                          ; self-describing (format_code, digest) per V7 §1.2 — the leading
                                             ; varint is the content_hash_format; DIGEST LENGTH FOLLOWS THE CODE.
                                             ; NOT fixed-width. (SHA-256 → 33 B is one instance, used in vectors;
                                             ; SHA-384, BLAKE3, future codes are equally valid. Unknown code →
                                             ; unsupported_content_hash_format, V7 §4.7.) **No hex33.**
path         = tstr                          ; an absolute entity path /{peer_id}/... (V7 §1.4)

; type = "app/embed/" .cat media-type        ; e.g. app/embed/image/png, app/embed/form
embed-data = {
  payload:        embed-payload,             ; the content — tagged union, see below
  fallback:       tstr,                      ; MANDATORY, non-empty: authored markdown/text degradation (§6)
  ? params:       { * tstr => any },         ; open attribute bag; keys are STRINGS ONLY (no int keys)
  ? renditions:   [* rendition],             ; responsive/format variants, lazily dereferenced (§5.3)
  ? requires:     [* capability-decl],       ; declared caps; KERNEL-enforced — active embeds only (§7) [OPEN per G1]
  ? sandbox:      sandbox-constraint,        ; substrate-neutral containment decl — active embeds only (§7) [OPEN per G1]
}

embed-payload = inline-payload / pointer-payload / child-payload    ; TAGGED — no untagged ambiguity
inline-payload  = { tag: "inline",  bytes: bstr .size (1..16384) }  ; ≤16 KiB (icons/SVG); in-tree (PF-2: .size pinned)
pointer-payload = { tag: "pointer", hash: content-hash }           ; content-store blob (every real image/video)
child-payload   = { tag: "child",   ref:  (path / content-hash) }  ; entity-native transclusion (a sibling Embed)

rendition = { pointer: content-hash, media_type: tstr, capability_tags: [* tstr] }   ; see §5.3 selection

sandbox-constraint = {                       ; substrate-NEUTRAL (no "iframe"/"wasm" literals — S-3)
  ? read_only_within: path,                  ; the prefix the embed's compute may read (default: its own subtree)
  ? deny_external_handlers: bool,            ; default TRUE — refuse handler_targets outside the subtree (§7, C3)
}
```

**Normative notes:**
- **`content-hash` is self-describing and variable-length** (V7 §1.2). The convention is **encoding-agnostic** —
  it never assumes SHA-256 or any fixed width. (The removed `hex33` was a SHA-256 lock-in; v0.2 fix.)
- `payload` is a **tagged union** — the tag disambiguates inline vs pointer vs child. Decoders MUST reject an
  untagged/ambiguous payload (the silent-divergence risk G-PIN-2 closes).
- `params` keys are **strings only**. `params`/`attrs` values are ECF values (canonical CBOR per V7 §1.3).
- `fallback` is **mandatory and non-empty** (anti-graveyard §8; ladder §6).
- There is **no `renderer_hint` free string** (dropped per S-2 — violates declarative-not-code). If ever needed
  it returns as a `content-hash` pointer to a renderer-policy entity, never a free string.
- **Inline ceiling ~16 KiB** (icons / SVG only); above that use `pointer` (single-chunk or chunked blob). Inline
  bytes inflate trie nodes + `.list`; keep them tiny (workbench-go's >10 MB inline-failure history — S-6). *The
  16 KiB inline ceiling supersedes the site spec's earlier 256 KiB; the 16–256 KiB range is a single-chunk
  pointer.*
- **`requires`/`sandbox` are thin in v0.2 (active embeds are DEFER-behind-G1, §7). v0.2 consumers MUST refuse to
  render any `Embed` carrying a non-empty `requires` or `sandbox`** (passive-only at the format layer — prevents a
  partial-honor impl silently rendering with declared-but-unenforced caps; workbench-go/godot). Decoders MUST
  tolerate unknown keys (forward-compat).

**Embedding modes:** (a) **child-entity transclusion** — the host references a sibling `Embed` by path/hash
(`child` payload; entity-native; **the v1 mode**, sufficient for the panel consumers); (b) **inline directive** in
a host document body. **The inline-directive grammar belongs to the consuming convention (e.g. the SITE
convention), NOT to EMBED** (godot) — and **an inline directive MUST lower to a `child` payload; it is sugar, not
a parallel format** (workbench-go/egui round-trip pin: "edit in egui, view in workbench" requires the directive
and the child entity be the same thing). v1 embeds **append** (no interleave) until the site convention defines an
inline directive.

## 4. `EmbedOutput` — the cross-substrate contract `[LOCKED — the spine; G-PIN-1]`

> **This is the lock.** A handler's output is a **typed, structured value — never a byte fragment.** The same
> dispatch the convention applies at the input (`media_type → handler`) applies one level down at the output
> (**`EmbedOutput` kind → renderer**). A handler that must emit SVG/HTML may do so **only** inside the `raw`
> escape hatch, which a non-supporting substrate **drops cleanly**. This structurally prevents the network from
> converging on web-native output (SVG/HTML) and forcing every other substrate to build a web engine. It is a
> hard cross-impl contract, not a recommendation, because cross-substrate interop *fails* without it.
>
> **EmbedOutput is the result-shape of an embed, NOT a page AST (v0.2).** Prose — paragraphs, links, emphasis,
> lists, GFM tables, headings, code blocks — is the **base** (§1.1), rendered by the base floor; it does **not**
> appear here. EmbedOutput carries only what the base can't: an image, a generic layout box of sub-results, an
> escape, a fallback. So it is **small** — five kinds across four roles (§4.0).

### 4.0 The basis & the extension point `[LOCKED — read this to understand the five]`
The single most-asked questions — *"where's the extension point? do I add an output kind? why image and not
chart? what even IS `raw`?"* — have one answer:

- **EmbedOutput is a CLOSED dispatch vocabulary**, not an open set. It is the fixed set of output *kinds* every
  renderer must **recognize and have a defined behavior for** — render it, or degrade it per §6. It is closed *on
  purpose*: a renderer can only act on kinds it knows, so a small fixed vocabulary is what makes the *same* output
  render predictably on every substrate. **You do NOT extend EmbedOutput to add an embed type.**
- **The extension point is the HANDLER (the input/`media_type` layer), and it is OPEN.** A new embed type = a new
  `app/embed/{type}` + a **handler that lowers your type into the vocabulary.** Chart, form, diagram, applet,
  "whatever a guy dreams up" — *all new handlers, zero output-vocabulary change.* That is the entire extensibility
  story, and it is the same `{type → handler}` dispatch as the input layer, applied as `{type → lower-to-vocab}`.

**The five kinds are NOT five co-equal "things you display" — they fill four distinct ROLES.** This is what
`raw` and `fallback` are (the most-asked confusion), and why `image` earns a slot but `chart` does not:

| Kind | Role | What it is |
|---|---|---|
| `text` | **content atom** | the symbolic-content atom — the one kind every substrate truly renders. Not composable from the others. |
| `image` | **content atom** | the raster/vector-content atom — opaque bytes. **You cannot compose a picture from text+box**, so it is its own kind. (A substrate that can't show pixels *declares* so and degrades — §5.3.) |
| `box` | **container** | composition: group / lay out child outputs. The one recursive kind. |
| `fallback` | **degradation floor** | the authored "show this instead" — always renderable; the bottom of the ladder (§6). |
| `raw` | **escape hatch** | native substrate bytes (SVG / HTML / BBCode) a handler emits when it has nothing better. The one kind a renderer is *allowed not to render*: known `format` → render; unknown → **drop clean** and show `fallback`. |

**What `raw` is — the confusion, answered.** `raw` is **not a display primitive**; it's the deliberate
**escape / FFI-to-native** so a handler *may* emit web-native output (an SVG, an HTML snippet) **without forcing
every other substrate to build a web engine.** The contract is one rule: *render it if you know the format, else
drop it clean (never show source text), and the mandatory `fallback` covers the gap.* That single rule is what
structurally prevents the network from converging on SVG/HTML output and stranding Godot / the terminal — the
central cross-substrate goal (§4 header). So `raw` is the escape that keeps the *other four* honest; you never
author a `raw`, a handler emits it as a last resort. (`text` and `image` are the two content atoms — symbolic vs.
visual; `box` composes; `fallback` is the floor; `raw` escapes.)

**"Why image and not chart?"** Image is an **irreducible content atom**; a chart is **composed**. A bar chart
lowers to `box[ box(bar)×N, text(labels) ]`, or the handler rasterizes it to an `image`, or emits `raw{svg}` +
`fallback` — so **a chart is a handler, not a kind**, and it renders on Godot/terminal *because* it lowered to the
vocabulary those substrates already implement. Image is **not** privileged "for us" — it's the passive *visual
atom*, and the v1 floor because images were the first ask.

**"Someone wants applets/live-code as the top-level thing."** An applet is an **input type**
(`app/embed/applet`, `requires` compute/WASM — active, deferred behind G1), not an output kind: when it
runs it still produces something a renderer draws (box/image/raw today, or the reserved `interactive` kind
behind G1 — §4.0a). Nobody is shut out — they're routed to the right layer (handler = open; vocabulary =
closed/amendment-rare).

### 4.0a Reserved candidate kinds — `media`, `interactive` `[EXPERIMENTAL — not required in v0.2; reserved, not locked]`
The vocabulary is closed, but **closed does not mean we pretend the only irreducible kinds are the five we need
today.** Two further kinds look genuinely irreducible — nothing composes them — so we **reserve** them now, the
same way V7 §1.2/§1.5 reserves `key_type`/`content_hash_format` codes it has not yet required: named, sketched,
slot held, **not burned, not required, not yet locked.** A v0.2 conformant impl implements **only the five**; it
MUST tolerate (and degrade) an unknown output `kind` per the §4 dispatch contract, so a future peer emitting a
reserved kind degrades cleanly rather than breaking. These are **drafted for review, deliberately not part of the
v0.2 conformance set**:

| Reserved kind | Confidence | Why it looks irreducible | Sketch (NOT locked) |
|---|---|---|---|
| `media` | **strong** | audio/video is a **temporal** stream — playback, seeking, duration. `image` is a single static frame; `box`+`image` cannot express time. A distinct atom. | `{ kind:"media", src: img-src, media_type, ? poster: img-src, ? duration }` |
| `interactive` | **tentative — may not survive** | an active surface that responds to input. *May decompose* into `box` + an active-behavior binding rather than being its own atom; tied to the active-embed / compute story. | TBD behind G1 — do not design until G1 (§7) lands. |

**Discipline (the V7 reserved-range rule applied here):** a candidate is reserved **only** if nothing in the
basis composes it (the `media` test passes: time is not composable; the `interactive` test is *unresolved*).
Things that compose — chart, table, form, diagram — are **never** reserved; they are handlers (§4.0). Promotion
from reserved → required is a **deliberate amendment** of this convention with its own conformance vectors, not a
v0.2 decision. Until then: a `media` embed lowers to `image` (poster frame) + `raw` (native `<video>`/`<audio>`) +
`fallback`; an `interactive` embed is an active embed behind G1. The reserved type prefixes
`app/embed-output/media` and `app/embed-output/interactive` are **held** so a future amendment does not collide.

```cddl
embed-output =                               ; logically an ECF value; type = app/embed-output/{variant}
    text-output                              ; text MAY itself be markdown (rendered by the base floor, §1.1)
  / image-output
  / box-output
  / raw-output
  / fallback-output

text-output     = { kind: "text",  text: tstr }
image-output    = { kind: "image", src: img-src, alt: tstr, ? media_type: tstr }
box-output      = { kind: "box",   ? layout: box-layout, children: [* embed-output] }  ; grouping/layout ONLY
raw-output      = { kind: "raw",   format: tstr, body: bstr }   ; ESCAPE HATCH — dropped clean if unsupported
fallback-output = { kind: "fallback", text: tstr }             ; the authored degradation (§6 step 2)

img-src    = { tag: "hash", hash: content-hash } / { tag: "inline", bytes: bstr }   ; tagged (godot — a hash IS bytes)
box-layout = "group" / "columns" / "card" / "figure"           ; minimal layout floor (NOT markdown structure)
```

**What is deliberately NOT here (v0.2 de-creep — see the critical review):**
- **No `section`/`paragraph`/`list-*`/`table*`/`blockquote`/`code`/`heading`/inline-marks** — *all markdown*,
  rendered by the base floor (§1.1). GFM tables lower structurally **at the markdown layer** (godot's concern is
  handled there, not here).
- **No `chart`** — a chart has no agreed sub-grammar; left as a free `box_kind` it silently diverges across
  renderers (egui + godot). A chart returns as its **own** sub-convention `APP-CONVENTION-EMBED-CHART` (real
  schema) when there is a real use case. Likewise a rich/sortable table → `APP-CONVENTION-EMBED-TABLE` (distinct
  from a GFM table, which is markdown).
- **No `figure` primitive** — a figure is **composition**: `box{layout:"figure", children:[image, text(caption)]}`.

**Dispatch contract (normative):**
- A renderer dispatches on `kind`, then (for `box`) on `layout`.
- **Unknown top-level `kind`** (e.g. a future reserved kind §4.0a a v0.2 renderer doesn't know) → **degrade via
  the §6 ladder** (render the Embed's authored `fallback`); MUST NOT break and MUST NOT show the value as source
  text. This is the forward-compat guarantee that lets the closed vocabulary grow by amendment without stranding
  older renderers.
- **Unknown `layout`** → render `children` in a plain container (the default `group` behavior). An unknown layout
  MUST NOT break a renderer (it never falls silently to nothing).
- **`raw` with a `format` the substrate cannot render** → **dropped clean** (Pandoc `RawBlock` model): the `raw`
  node is omitted from presentation; it is **never** shown as visible source text. The *only* place HTML/SVG may
  live; opt-in, isolated, degradable.

**Materialization.** `EmbedOutput` is the value a handler returns to a renderer; it MAY be materialized as an ECF
entity (e.g. a per-frame render cache) but need not be — the **schema is the contract.** If materialized, the
`app/embed-output/*` type prefix is reserved.

**Extending.** The **five variants are the stable grammar** (with `media`/`interactive` reserved, §4.0a). The
`box.layout` set may grow via this convention's amendment process. Genuinely-structured rich content (chart,
diagram, rich table) gets its **own** sub-convention with a real schema — it is **not** crammed into `box.layout`.
A new *media type* needs only a new `type` tag + a handler; it does **not** need a new output kind.

**The amendment process (PF-5):** "this convention's amendment process" = the `applications/`-domain proposal flow
(`CHARTER.md` + a proposal in `proposals/`, cohort review → fold → ratify — the same flow that produced v0.1→v0.2.3).
Promoting a reserved kind (§4.0a) or adding a `box.layout` is a versioned amendment with its own conformance
vectors; it is **never** an ad-hoc per-impl extension of the closed vocabulary.

## 5. The two-level registry `[LOCKED model]`

### 5.1 Handler (data → output) — the code layer
Content-addressed, capability-gated, sandboxed code that turns `payload + params` into an `EmbedOutput`.
**Untrusted** — the anti-graveyard rules (§8) apply here. **v1 image/av handlers are trivial pass-throughs**
(no untrusted code). **Handler discovery uses the existing `system/handler/*` manifest mechanism** (S-5) — *not*
a new registry namespace: a handler for `app/embed/image/png` is a `system/handler` entry whose pattern matches
the embed type. (A handler MAY be an entity-native compute graph via `expression_path` — §7.)

### 5.2 Renderer (output → presentation) — the front-end layer
**Trusted, per-front-end** code that draws an `EmbedOutput` onto the substrate's primitives. The registry here is
the **front-end's own** (web's, Godot's GDScript, the terminal's) — this is what makes the *same* output render
everywhere. Renderers are **not** part of the cross-impl format contract; the `EmbedOutput` schema is.

### 5.3 Renderer-capability declaration + rendition selection `[LOCKED — D-RENDERER-CAPS]`
The fallback ladder must not collapse too eagerly (a terminal cannot render `image/png` *at all*; Avalonia can).
A renderer declares a capability set the resolver consults **before** a handler emits a kind the substrate drops:

```cddl
renderer-caps = {                   ; split to match the two-level dispatch (godot) — not one flat list
  variants:      [* tstr],          ; which EmbedOutput variants it draws: "text"/"image"/"box"/"raw"/"fallback"
  box_layouts:   [* tstr],          ; which box layouts it draws: "group"/"columns"/"card"/"figure"
  media_types:   [* tstr],          ; which rendition/image media_types it can present
  has_layout:    bool, has_color: bool, has_inline_active: bool,
  ? max_image_dim: uint,
}
```

**Renderer-caps lifetime (PF-7):** a renderer declares its caps **once at front-end initialization / handler
registration** — they are a **static property of the front-end build**, not per-request and not negotiated mid
-session. The resolver reads the current declaration at resolution time; a front-end that gains a capability
(e.g. an SVG rasterizer is added) re-declares on next init. No per-`Embed` cap handshake.

**Rendition selection (normative):** choose the **highest-ranked `rendition` whose `capability_tags ⊆
(renderer-caps.media_types ∪ renderer-caps.variants ∪ renderer-caps.box_layouts)`**; if none qualifies, render the
`fallback`. Ranking is the authored order of `renditions` (first = most preferred). Renderers MUST agree on this
rule so the same site yields predictable output per substrate. (The set is now precisely typed — godot's flag that
`capability_tags ⊆ renderer-caps` was mathematical but the sets were untyped.)

## 6. The degradation contract — the fallback ladder `[LOCKED — D5]`

The ladder governs an **`Embed`'s rendering** (its handler/output/`raw`/`fallback`). *(PF-6: the base **document
body** — a markdown/HTML page — is the **consuming convention's** concern (e.g. SITE §3, gated by G2 for HTML),
**not** EMBED's; EMBED's ladder is scoped to the embed node, not to host prose.)* For any `Embed`, the renderer
applies a **3-step ladder**:

1. **Try the handler/renderer** (§5) — render the `EmbedOutput` if a handler produced one and a renderer can draw
   its kinds.
2. **Else render the authored `fallback`** — mandatory text/markdown on every `Embed`. *This is the
   `<video>…fallback</video>` / `<img alt>` pattern.* **Fallback is rendered with embed directives DISABLED**
   (depth-1 bound) — an embed directive inside a fallback is **rendered as visible text, not re-expanded and not
   silently stripped** (anti-poisoning, S-8; godot's affirmative-behavior pin).
3. **Else pass `raw` content through *with its format label*** — a substrate that understands the format keeps
   it; one that doesn't **drops it clean** (`RawBlock` model). **Never** dump raw source as visible text.

This is *authored fallback (required) + format-labelled passthrough* — **not** lossy auto-text-extraction and
**not** placeholder-only.

## 7. Capability & compute `[GRADIENT — v1 passive only]`

- **Passive embeds (image/av, no `requires`/`sandbox`)** — v1 floor. Pass-through handler; no untrusted code.
  An inline-payload passive embed renders at the **floor** tier (tree-readable); a pointer-payload passive embed
  needs the **content extension**.
- **Active / interactive embeds (`requires`/`sandbox`)** — `[DEFER build behind gate G1]`. The handler is an
  **entity-native compute graph** (an `expression_path` handler — **requires EXTENSION-COMPUTE on the evaluating
  peer**; `expression_path` is core-stable per V7 §3.7/§6.6) or a **WASM handler** (greenfield host, named
  future, no date). The compute graph bundles inside the host subtree and evaluates on download under an
  **install-audit + prefix-scoped grant** (the purity boundary scoped to the content subtree).
- **Enforcement is the KERNEL capability model.** `requires` is *declared* on the handler manifest; the kernel
  *enforces* it. **No new permission system** (that would be the ambient-authority graveyard mistake).
- **Gate G1 (security, named — never deferred):** before active embeds ship, a baseline audit of install-audit +
  self-issued prefix-scoped grant, with these conditions: audit shell-inspectable (C1); per-front-end audit-prompt
  policy hook (C2); **cross-peer dispatch from bundled compute refuse-by-default** (C3); per-site policy keyed by
  (consuming-peer, site-root-hash) (C4); a distinct install surface for site-scoped compute, not a side door of
  generic handler registration (C5). Owners: arch + consuming L5 teams.

## 8. The anti-graveyard contract `[LOCKED — B2]`

The plugin graveyard (Flash/Java/Silverlight/ActiveX/NPAPI/NaCl) died of seven repeating causes; our substrate
avoids each **by construction**, and a conformant handler **MUST** satisfy:

1. **Content-addressed** — integrity by `content-hash` (the self-describing `(format_code, digest)`, V7 §1.2;
   encoding-agnostic — **not** SHA-256-locked).
2. **Capability-gated, no ambient authority** — kernel caps via declared `requires` (§7).
3. **Sandboxed** — substrate-neutral containment declaration (§3 `sandbox-constraint`); runtime per-substrate.
4. **Portable** — entity-native compute graph or WASM; no per-OS binary.
5. **Declarative spec, not embedded code** — the `Embed` is *data*; the handler is separately resolved (the
   MDX/Flash anti-lesson). *This is why `renderer_hint` as a free code-ish string was removed (§3).*
6. **Mandatory `fallback`** — every `Embed` degrades (§6).
7. **Trust by `content-hash` verify + manifest signature** — SRI / code-signing, entity-native; not a DHT.

## 9. Conformance vectors `[REQUIRED before ratification]`

A FORMAT convention is not validated until vectors exercise it (PRIMER meta-rule). v0.2 ships:
- An `Embed` of each `payload` tag (inline / pointer / child) with a **mandatory `fallback`** — CBOR + expected
  hash.
- An `EmbedOutput` of **each of the five variants** (`text/image/box/raw/fallback`) and each `box.layout` — CBOR +
  expected hash.
- **`raw`-dropped-clean** vector (egui — the spine of cross-substrate interop, previously untested): a `raw`
  output with an unsupported `format` is **omitted from presentation, not rendered as text**.
- **Unknown-`layout`** vector (pinned single behavior — renders `children` as `group`, never breaks).
- **Unknown-`kind`** vector (forward-compat for reserved kinds §4.0a): an `EmbedOutput` carrying an unrecognized
  `kind` (e.g. a future `media`) MUST degrade cleanly — the renderer falls to the authored `fallback`, never
  breaks and never shows the raw value as source text. This is what lets a v0.2 impl coexist with a later peer
  that emits a reserved kind.
- An **encoding-agnostic `content-hash`** vector pair: the *same* `Embed`/`EmbedOutput` carrying a **SHA-256**
  pointer and a **SHA-384** pointer both decode/verify (proves no width lock-in — the v0.2 fix is exercised, not
  just asserted).
- A **payload-tag-ambiguity reject** vector (untagged payload MUST be rejected — G-PIN-2).
- A **fallback-poisoning** vector (embed directive inside a `fallback` rendered as text, not re-expanded — S-8).
- A **non-empty-`requires`/`sandbox` refuse** vector (v0.2 consumer refuses to render — §3).
- A **rendition-selection** vector (given `renderer-caps`, the selected rendition is deterministic — §5.3).
- A **rendition-decline** vector (`V-CAPS-DECLINE`, workbench-go): when **no** rendition's `capability_tags`
  qualify, selection lands on `fallback` — deterministically, not on an arbitrary best-effort kind.

Cross-impl byte-equality on these is the v0.2 lock signal. *Note: paragraphs/links/tables are NOT EMBED vectors —
they are markdown, covered by the CommonMark/GFM conformance suite (§1.1), not by this convention.*

## 10. Open / deferred

- `[DEFER → SITE convention]` the **inline-directive grammar** lives in the consuming convention (the site
  convention), not EMBED; it MUST lower to a `child` payload (§3). v1 embeds append; child-transclusion carries v1.
- `[DEFER → own sub-convention]` **`APP-CONVENTION-EMBED-CHART`** and **`APP-CONVENTION-EMBED-TABLE`** (rich,
  sortable) — genuinely-structured rich types get real schemas of their own; they are **not** `box.layout` kinds
  (§4). Authored when a real use case arrives.
- `[DEFER]` active/interactive embeds build (behind G1, §7); WASM handler host (named future, no date). v0.2 is
  passive-only — consumers refuse non-empty `requires`/`sandbox` (§3).
- `[OPEN]` whether `EmbedOutput` is ever materialized as a stored entity (per-frame render cache) — schema is the
  contract regardless; `app/embed-output/*` prefix reserved (§4).
- `[OPEN]` `box.layout` floor completeness — `group/columns/card/figure` is the v0.2 set; grows via amendment as
  demand surfaces (§4). **Inline text marks are NOT added — prose is markdown (§1.1).**
- `[RESERVED → amendment]` **`media`** (audio/video temporal atom) and **`interactive`** (active surface, behind
  G1) candidate output kinds (§4.0a) — sketched, slot-held, **experimental / not in the v0.2 conformance set.**
  `media` is the stronger case (time is not composable); `interactive` may decompose into `box`+behavior and is
  not designed until G1. Promotion to required is a deliberate amendment with its own vectors. The
  `app/embed-output/{media,interactive}` type prefixes are held against collision.
- `[OPEN]` `renderer_hint` as a `content-hash` renderer-policy pointer — only if a real need surfaces (§3).

## 11. Provenance
- Synthesis / v1 lock: the content-site v1-lock synthesis (`e8a33c5`) §2 (output lock), §4 (pins), §5 (strains), §7 (G1).
- Arch first pass: the L5 semantic-content-site arch-first-pass review (`4351f73`).
- Cross-team passes: egui/Dom, workbench-go (`G-PIN-1..4`, `S-1..11`), godot (`[ASK-ARCH-OUTPUT-SHAPE]`).
- Substrate grounding: `expression_path` core-stable (V7 §3.7/§6.6, EXTENSION-COMPUTE); `system/handler/*`
  dispatch; self-describing `content-hash` `(format_code,digest)` (V7 §1.2/§1.4). Charter: `applications/CHARTER.md`.
