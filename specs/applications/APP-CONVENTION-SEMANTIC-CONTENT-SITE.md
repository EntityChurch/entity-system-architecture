# APP-CONVENTION-SEMANTIC-CONTENT-SITE — content sites built on Embed — v0.5.2 DRAFT

**Version**: 0.5.2
**Status**: Draft
**Kind**: normative-spec · **Authority**: binding · **Governed-by**: `guides/GUIDE-APPLICATION-DEVELOPMENT.md` — FORMAT-only (§2.1).
**Consumes:** `APP-CONVENTION-EMBED` (the rich-content node + `EmbedOutput` + the two-level registry + the fallback ladder).

> **What this is.** The convention for a **content site** — a verifiable, transferable, peer-hosted body of pages
> (docs, a blog, a wiki, a product site) — built as a **thin L5 layer over already-shipped subgraph primitives**
> (tree snapshot/extract, content blobs/chunks, the capability gradient, entity-native compute). It invents **no**
> site-specific versioning, transfer, or storage machinery, and **no** kernel feature. It defines the **site
> entity vocabulary** (manifest, page, signed root pin), the **base-format + inline-embed grammar**, and the
> **bundle/closure + trust model** — and it points at EMBED for everything about rich content.

> **The relationship to EMBED, stated once.** EMBED is the **passive content format** (an authored node → a
> handler → a drawable `EmbedOutput`). This convention is where **active code, packaging, and trust** live —
> precisely the things EMBED deliberately does not answer. A site *contains* embeds; an embed never contains a
> site. If a question is "what bytes does this node carry / how does it render," it's EMBED. If it's "how is this
> shipped, made complete, and trusted on arrival," it's here.

---

## 1. The anatomy of a site `[LOCKED — read this first; it answers "where does the compute subgraph go?"]`

A site is **three parts that travel together but are three different kinds of thing.** Conflating them is the
single recurring confusion (compute-vs-embed-vs-asset); separating them is what makes the model click.

| Part | What it is | Trust | Travels as |
|---|---|---|---|
| **1. Document** | `SiteManifest` + `SitePage`s — the base-format bodies (§3) with **embed nodes** inserted | authored data; no code | tree entities in the site subtree |
| **2. Content closure** | the **assets** an embed points at — images, video, blobs. **Passive bytes.** | none — just data | inline tree entities (small) or content-store blobs reached by `content-hash` (large) |
| **3. Compute closure** | the **bundled handlers** active embeds dispatch to — entity-native compute subgraphs (or, future, WASM). **Executable.** | **G1 install-audit + prefix-scoped grant** (§8) | tree entities in the site subtree, *if* the site ships its own handlers |

**The embed is the seam between the three.** An `Embed` (EMBED §3) is **declarative data** in Part 1 that points
*down* into Part 2 (its asset `payload`) and *out* to a handler in Part 3 (by its `media_type` type tag). It is
neither an asset nor a handler. This is the whole resolution of "is an embed a compute subgraph?": **no — an
embed may *reference* a compute subgraph as its handler; the subgraph is the engine, the embed is the
declaration, the asset is the data.**

**Three layers, never collapsed** (the recurring conflation, pinned):

```
  app/embed/{type}      ← INPUT TYPE   (open; a handler.  e.g. app/embed/applet, app/embed/live-chart)
        │  dispatch by media_type (EMBED §5.1)
        ▼
  HANDLER               ← THE TRANSFORM (pass-through | COMPUTE SUBGRAPH | WASM).  ← compute lives HERE
        │  produces
        ▼
  EmbedOutput kind      ← OUTPUT KIND  (closed vocab: text/image/box/raw/fallback; media/interactive reserved)
```

So "what kind of embed uses the compute subgraph — raw? media? interactive?" is a **category error**: those are
*output* kinds (what the handler emits). The compute subgraph is the **handler**, one layer up from the output.
A site's live chart is `app/embed/live-chart` (input type) → a bundled compute subgraph (handler) → a `box`/`raw`
today (output) → drawn by the renderer. Nothing is `interactive` yet — that output kind is **reserved** (EMBED
§4.0a), behind G1.

**Most sites are Parts 1+2 only** (passive: markdown + images + fallback — the v1 floor). A site adds **Part 3**
only when it carries active embeds. The passive floor ships now; Part 3 is G1-gated (§8) and DEFER for v1.

### 1.1 Bundled vs. resolved handlers (the "you transfer the compute subgraph" intuition) `[LOCKED — load-bearing for G1 C3]`
A handler reaches the rendering peer two ways:
- **Bundled** — the site ships the compute subgraph **in its subtree** (Part 3). `tree:extract` carries it for
  free (it's a named tree entity). On arrival it triggers **G1 install-audit** before it may evaluate. This is
  the "package up the compute subgraph" case.
- **Resolved** — the rendering peer **already has** a handler for that `media_type` (like a browser already
  having a video codec). Nothing ships; the peer's installed `system/handler/*` serves it. No G1 event (the peer
  already trusts its own installed handlers).

**Assets vs. compute, the trust asymmetry that makes them feel different:** an asset (Part 2) is passive bytes and
makes **no trust decision** on arrival. A bundled compute subgraph (Part 3) is executable and makes a **G1 trust
decision** on arrival. They travel the same way (the bundle is closure-complete over both); they differ at the
**trust boundary**, not the transport. That is the precise content of "you transfer the compute subgraph, you
don't [just trust] the asset."

## 2. Site identity & the signed root pin `[LOCKED — D1 / G-PIN-3]`

A site's **identity is the root hash of its subtree** — a `tree:snapshot` root (an **EXTENSION-TREE** op; a
core-only peer is a valid reader without it — site *identity* sits at the **+tree-ext** tier, not the floor §7).
Two identical subtrees produce the identical root **given two named agreements** (the caveat the spec states
rather than overclaiming portability — `C-2`):
- **Tree-version: EXTENSION-TREE v4.0.2 is normative for v1.** v4.0 was a hard breaking trie-node-shape change
  (IPLD HAMT rewrite); root hashes are **not comparable across the v4.0 fork.** v1 publishers + readers are all
  v4.0.2+.
- **Chunker agreement:** the canonical `chunk_size` (§6.1) — a blob hash is a function of which chunker ran, so
  the same source bytes under disagreeing chunkers yield different subtree roots.

A static publisher serves the verifiable root beside the manifest as a **signed `site-root` pin** — the
"commit-wraps-tree" shape without a revision DAG:

```cddl
site-root-pin = {                            ; type = app/site-root  (an ECF entity)
  root:     content-hash,                    ; the subtree root — self-describing (format_code, digest), V7 §1.2
  seq:      uint,                            ; monotonic; advance = resign with seq+1; gaps tolerated
  site_id:  tstr,
  ? passthrough_of: peer-id,                 ; G2 republish marker (F-3): set when a front-end re-publishes a site
                                             ; it did NOT author → names the origin peer. Rides the SIGNED pin so
                                             ; it cannot be stripped; downstream applies origin-trust policy (§8).
}
peer-id      = tstr                          ; IMPORTED from APP-CONVENTION-REFERENCE §2.1 — a PeerID in the
                                             ; canonical Base58 form of V7 §1.5. A TEXT string, not bytes.
                                             ; (This read `bstr`, citing V7 §1.2 — the CONTENT HASH section.
                                             ;  A peer id is spelled, not hashed; see REFERENCE §2.1.1.)
```

**Signing contract (the highest-priority invisible-failure surface — `G-PIN-3`):**
- The pin is **signed by the publisher's identity-entity** (V7 hash-addressed identity, materialized from the
  publisher's Ed25519 keypair — **not** a raw public key).
- **Signature target = the canonical ECF encoding of the pin entity's `(type, data)`** — the *same* canonical
  bytes the content hash is computed over (V7 §1.3 / ENTITY-CBOR-ENCODING §4.1). **One signing surface**; no
  ad-hoc `root_bytes || seq_varint` concatenation.
- The signature travels as the standard `signature` ref; anyone holding the publisher's identity verifies it.
- `seq` is monotonic; verifiers take the **highest valid `seq`**. Jumps allowed, gaps tolerated (simple-publisher
  tier — no revision DAG required).
- **Site placement (v0.5).** A site is a **free subgraph** at any publisher-chosen tree path. The pin entity
  (`signed-pointer` to the manifest) lives at a publisher-chosen location alongside the site subgraph it
  pins; the manifest names the placement (via the site subgraph's root path) so a reader can find it. The
  site's capability scope is the site subgraph's own root, not a slot carved out of someone else's
  namespace. **The prior v0.4.2 `{publisher_peer_id}/content/sites/{site_id}/_root` placement is dropped**
  — `system/content/*` is the CONTENT-extension namespace per `EXTENSION-CONTENT §6.4.2` (the leaf is always
  `{hex(H)}`), and an L5 application subgraph does not belong there. The URL projection (see §11) is the
  legacy-web surface, not a tree-storage rule.

A cross-impl vector ships a fixture pin signed by a known identity with expected signature bytes, so
that **at least two independent L5 implementations** verify **byte-identically** before circulation
closes. **Independent means built over different cores** — two front ends linking the same core cannot
disagree about it and do not constitute two.

## 3. The base format & the inline-embed grammar `[LOCKED — EMBED pushed this down to here]`

### 3.1 Base format
A `SitePage` has a **base format** (EMBED §1.1 — the base-format choice is the *consuming convention's*, i.e.
**ours**). v1:
- **`markdown`** (CommonMark/GFM) — the **recommended, universal-reach** base; renders on every substrate, public
  conformance suite. The v1 default.
- **`html`** — permitted at the **web tier behind gate G2** (sanitization, §8); priced by reach (non-DOM
  substrates fall to the ladder). A publisher's choice, not the default.

Prose (paragraphs, links, emphasis, lists, GFM tables, headings, code) is **base content**, rendered by each
front-end's own base renderer (web→DOM, Godot→BBCode/`RichTextLabel`, terminal→text). It is **not** an embed and
**not** `EmbedOutput` (EMBED §1.1, §4). The site convention adds **no** prose vocabulary — that's the base
format's standard.

> **One exception, and it is a scoping rather than an addition: a LINK in a page body has an
> entity-native meaning.** It is a reference string and it resolves by `APP-CONVENTION-REFERENCE` §3.4
> — directory-relative to the current page, root-absolute on a leading `/`, `site:` for a same-peer
> cross-site target, `entity+ref://` when fully qualified. **The base format supplies the syntax; this
> system supplies what the string denotes**, because a link is the one piece of prose that names
> something outside the document and the base format's standard has nothing to say about an entity
> tree. Everything else in the sentence above stands: no other prose vocabulary is added.

### 3.2 The inline-embed directive (this convention owns it; EMBED §10 deferred it here)
Embeds enter a page body two ways:
- **Child transclusion** (the v1-sufficient mode, EMBED §3): the page references a sibling `Embed` entity by
  path/hash. Always available.
- **Inline directive** — sugar inside a markdown body. **It MUST lower to a `child` payload** (EMBED §10
  round-trip pin: "edit in entity-browser-rust, view in workbench" requires the directive and the child entity be *the same
  thing*). The directive is **not a parallel format**; it is a serialization of a child reference.

v1 directive grammar (leaf form, CommonMark-directive style):

```
::embed[fallback text]{ref="<path-or-content-hash>"}
```

- `ref` resolves to a sibling `Embed` entity (the `child` payload target). The bracketed text seeds the rendered
  fallback if the embed cannot render *and* the entity is unreachable.
- **`ref` is a reference string and resolves by `APP-CONVENTION-REFERENCE` §3.4**, identically to a link in a
  page body (§3.1): a leading `/` is root-absolute **within the current site**; `site:` is the same-peer
  cross-site form; `entity+ref://` is fully qualified; **anything else is directory-relative to the current
  page.** A string parsing as none of these is treated as leaving the system and is refused, not guessed at.
- **A content-addressed embed target is spelled as a reference, never as a bare digest.** Use
  `APP-CONVENTION-REFERENCE` §3.1's pinned form (`entity+ref://{peer}/?hash={hash}`) when the target is on
  another peer, or a **`pointer` payload** (`APP-CONVENTION-EMBED` §2) when it is a blob in the resolving peer's
  own content store. **A bare hex digest in a `ref` position is not a distinguishable form and MUST NOT be
  emitted** — it is indistinguishable from a directory-relative reference to a page of that name.
- *(`F-1`'s former path-vs-content-hash rule is **WITHDRAWN**. It disambiguated the untagged
  `(path / content-hash)` union that `APP-CONVENTION-EMBED` §2 retired in favour of a tagged reference atom
  (`APP-CONVENTION-REFERENCE` §2.1 `REF-R1` — there is no untagged atom); with no untagged union at the wire
  layer there is no ambiguity for the string form to inherit. Every form above is syntactically decidable,
  which is the property `F-1` existed to provide. Its leading-`/` arm additionally read as a V7 §1.4 **absolute
  entity path**, which addresses the whole tree rather than the site subgraph and is an off-subgraph
  reference — `APP-CONVENTION-REFERENCE` §3.4's root-absolute-within-the-site reading is the correct one and
  is confined.)*
- A renderer that doesn't understand the directive renders the bracketed text (degradation by construction).
- **Lowering is normative and lossless:** `::embed[…]{ref=X}` ⇔ a `child`-payload `Embed` pointing at `X` (with
  `X` resolved per `APP-CONVENTION-REFERENCE` §3.4). A conformant editor round-trips the two without divergence.
- v1 embeds **append / block-level only** (no mid-paragraph interleave); a container/inline-flow directive is a
  named post-v1 extension of *this* grammar, not EMBED's.

## 4. Site entities `[LOCKED shape / CDDL]`

**Type-tag namespace (`F-8`, confirmed):** the `app/site-*` tags are **final** — consistent with `app/embed/*`
and the `applications/` domain. (Impls shipping `content/site/*` migrate; bounded, tracked impl-side, not a spec
change.)

```cddl
; --- SiteManifest: the site's COVER — identity + ONE optional human menu. No collections (§4.2) ---
site-manifest = {                            ; type = app/site-manifest
  site_id:    tstr,
  title:      tstr,
  ? nav:      [* nav-node],                  ; OPTIONAL human navigation menu — a curated tree of pointers; NOT the
                                             ; discovery index, NOT exhaustive (§4.2). Holds no content.
  ? params:   { * tstr => any },             ; open attribute bag; string keys only (EMBED §3 discipline)
}
nav-node  = { label: tstr, ? target: link-ref, ? children: [* nav-node] }   ; tree; cycle rule §4.1
link-ref  = tstr                             ; F-2: optional (section headers have none). A reference string in
                                             ; the form APP-CONVENTION-REFERENCE §3 defines: relative
                                             ; ("./about"), same-peer scheme ("site:labs/intro"), or absolute
                                             ; ("entity+ref://{peer}/…"). Resolution, the base-URI rule and the
                                             ; refusal behaviour are that document's §3.1/§3.3/§3.4 — NORMATIVE,
                                             ; not per-renderer. NOT narrowed to absolute-only (that would
                                             ; regress real authoring).

; --- SitePage: one page ----------------------------------------------------------------
site-page = {                                ; type = app/site-page
  format:     "markdown" / "html",           ; base format (§3.1); markdown is the recommended default
  body:       tstr,                           ; the base-format body (may carry ::embed directives, §3.2)
  ? frontmatter: { ? title: tstr, * tstr => any },   ; title-only is conformant; MAY derive title from first H1
  ? embeds:   [* (path / content-hash)],     ; sibling Embed entities this page transcludes (child mode)
}

; --- a NAMED, site-local binary asset (image, font, stylesheet). It exists because §3.2's
;     directory-relative `ref` resolves against the TREE, and a bare content-store blob has
;     no name; the naming binding points at this.
site-asset = {                               ; type = app/site-asset
  type: "app/site-asset",
  data: {
    media_type: tstr,                        ; IANA type — an asset has no dispatch tag of its own
    payload:    inline-payload / pointer-payload,   ; APP-CONVENTION-EMBED §3's arms, NARROWED — see below.
                                             ; NOT a second payload union; the embed ones, reused.
                                             ; `child-payload` is NOT admitted here (the [MUST NOT] below).
  }
}
```

> **Where a link or an asset reference RESOLVES — the authority is `APP-CONVENTION-REFERENCE` §3.4, and
> this position is not special `[v0.5.2]`.**
>
> **[MUST]** A link or asset reference appearing as a **bare string** in a page body — including in an
> embed directive's asset position — is resolved per `APP-CONVENTION-REFERENCE` §3.4: **a leading `/` is
> root-absolute within the current site**; anything else is **directory-relative to the current page**.
> **This convention states no rule of its own about it**, and an implementation that refuses the
> root-absolute form refuses the form §3.4's own paragraph **recommends producers emit**.
>
> *Stated because two implementations diverged here for a week — one resolving the form, one refusing
> it, so the same published page rendered a figure in one reader and a gap in the other — while each
> searched for a rule scoped to the asset position.* **The rule is scoped to the FORM, one convention
> over.** An absolute `entity+ref://` URI is a different case with a different base (§3.1: a
> peer-absolute tree path), and that difference is by design; whether the asset position admits the
> atom at all is not settled here.

**`app/site-asset` — declared, and the `[MUST]` closes a bypass rather than adding a rule.**

> **[MUST]** An asset whose bytes exceed `inline-payload`'s `.size (1..16384)` ceiling **MUST** use a
> **`pointer`** payload, placing the bytes in the content store **where §6.1's canonical chunking governs
> them.** An implementation **MUST NOT** inline unbounded bytes in a site asset.
>
> **Why this is a bypass and not merely a size preference.** §6.1 makes *"same image → same site root"*
> depend on every v1 publisher chunking at the canonical 1 MiB FastCDC default. **Bytes that never enter
> the content store never meet that rule** — so on an all-inline asset path the MUST is not failing, it is
> *unreachable*, and no reproducible-publish check can observe it. That is why an inline-at-any-size asset
> makes its row incomparable across implementations rather than merely large. `APP-CONVENTION-EMBED` §3's
> inline ceiling exists for the adjacent measured reason (inline bytes inflate trie nodes and `.list`).

**The payload field is NARROWER than the union it imports, and the narrowing is normative.**

> **[MUST NOT]** An `app/site-asset` **MUST NOT** carry a `child-payload`. Its `payload` admits
> `inline-payload` and `pointer-payload` only. A reader that decodes one **MUST** refuse the entity as
> **invalid for its type**, and **MUST NOT** report it as a malformed or untagged payload.
>
> **Why the arm is excluded — an asset NAMES BYTES, and `child` names an entity.** `child-payload.ref`
> resolves to a sibling `Embed`, and an `Embed`'s **dispatch key is its type tag**
> (`app/embed/{media_type}`). `APP-CONVENTION-EMBED` §3 carries **no `data.media_type` field**, for the
> stated reason that it *"would be a redundant second source of truth."* A site asset carries
> `media_type` beside its payload. **So an asset with a `child` payload holds two media types that can
> disagree, with nothing to say which wins** — the exact redundancy the embed convention removed by
> hand, reintroduced one convention over. The naming argument is the same one this type exists for
> (above): a bare blob has no name and needs this binding; an entity in the tree already has one.
>
> **`child` is also the only arm that may cross a peer boundary** (EMBED §3 — it is why that arm alone
> carries the authority term), and this type is declared **site-local**. §6's `.entsite` bundle is
> *closure-complete*; a cross-peer reference inside an asset makes closure uncloseable, and it would
> surface at bundle time rather than at authoring time.
>
> **This narrows the asset and nothing else — sites keep `child` where it belongs.** A `site-page`'s
> `::embed` directive **MUST** lower to a `child` payload on a sibling `Embed` (§3.2; EMBED §3's v1
> embedding mode), and §9 names the round-trip vector for it. **A page transcludes; an asset names
> bytes.** No authoring capability is removed.
>
> **The refusal is its own outcome, and collapsing it loses the attribution.** EMBED §3's *"decoders
> MUST reject an untagged/ambiguous payload"* governs a payload with **no discriminator**. A `child`
> payload here is correctly tagged, well-formed and unambiguous — it is invalid *for this type*. Three
> outcomes stay distinct: **a tag the reader has not built** is the reader's gap; **`child` on an
> asset** is the publisher's schema violation; **no tag at all** is a malformed payload.

**Merge policy (`S-9`):** v1 SiteManifest merge is **last-write-wins on the whole manifest** (simple-publisher
posture). Field-level named strategies (nav = ordered-set union; root = conflict-error) are **deferred** with the
propose-back/edit arc (itself `[GRADIENT][DEFER build]`).

### 4.1 Navigation safety `[v1 renderer contract — promoted from post-v1; DoS surface]`
`nav` is a tree and MAY contain authored cycles or pathological depth. A renderer walking nav **MUST**: maintain
a **visited-set** (cycle detection), enforce a **max depth (recommend 32)**, and on either limit **stop cleanly**
(render what it has; never infinite-loop / stack-overflow). One-line contract; v1-blocking.

**The visited-set obligation is on the WALK, and an implementation whose decoded `nav` representation cannot
express a cycle discharges it structurally.** A representation in which children are owned values decoded from a
serialization carrying no back-reference has no cycle to detect, and a visited-set there guards a state the
representation forbids. **What MUST hold in every case is the outcome** — no unbounded recursion and no stack
exhaustion on any authored input. An implementation claiming the structural discharge states the property of its
representation that provides it.

**A manifest a decoder REFUSES is a different outcome from a walk that stops at the depth limit, and the two MUST
be distinguishable.** *"Stop cleanly (render what it has)"* binds the walk. Where a manifest cannot be decoded at
all — including because its authored nesting exceeds a decoder's own nesting limit — the renderer **MUST**
surface a refusal distinct from a successfully-decoded empty or untitled manifest. **Substituting a default
manifest is not conformant**, because it reports a fact about the site (that it has no title, no identity) that
is not true, and it reports it silently.

**Implementations SHOULD NOT rely on a dependency's nesting limit to provide this section's bound.** Where the
no-unbounded-recursion property holds only because a decoder happens to stop first, a change in that dependency
moves it with nothing to announce it; a check for this rule should fail on such a change rather than track the
number.

### 4.2 Discovery, ordering & why the manifest holds no page-collection `[v0.4.2 — discovery+floor LOCKED; semantic feeds OPEN/researching]`
**Discovery is lazy, one-level-at-a-time `.list` over the site namespace** — the scalable v1 default. A renderer
walks the tree namespace level by level (like a filesystem) and **never requires downloading a full index to
render the first page.** This avoids the "download the index" anti-pattern at scale (huge-index →
index-of-indexes, our SCALE landscape; entity-browser-rust `discovery.rs`, `cd283a5`). **Paging belongs to `.list`** — it is
inherently incremental — never to a manifest field.

**The manifest carries identity + one optional human menu (`nav`) — and NO page-collection field.** v0.4 sketched
an optional `pages: [* page-ref]` ordered list; **v0.4.1 removes it.** Honest accounting: `pages` was arch's
addition in the v0.3 assembly — **not** in any impl, design doc, or the three-team v1 lock — and on merit it does
not earn a slot:
- **Redundant with `nav`** — a flat ordered list of page pointers is just a `nav` with no nesting; `nav` already
  expresses ordered pointers.
- **Reintroduces the anti-pattern this section forbids** — a flat *exhaustive* page list at scale is exactly the
  "download the index" shape `.list` exists to avoid. It "works" only for small sites, which `nav` or an
  index-page-with-links already covers.
- **Its one unique use — a body-free canonical sequence for feeds / prev-next / "3 of 12"** — is better served by
  **deriving order from each page's own frontmatter** (single source of truth: a page's `date`/`order`, read via
  `.list` + frontmatter for a small site), or, when a real RSS/feed-at-scale use case arrives, a **named post-v1
  feed/index extension** — not a core field built ahead of the use case.

**Ordering — one small pinned floor; the semantic layer is open research (L5, not locked).** First the fact that
governs it: the tree carries **no** semantic order. Under EXTENSION-TREE v4.0.2 `.list` enumerates in **hash-bit
order, random w.r.t. names** (§enumeration: *"callers needing lex-sorted output MUST sort at output"*). So order
is a **renderer presentation choice, not a tree property** — we are *not* giving tree location semantic meaning.

- **Determinism floor (the one pinned cross-impl rule):** a renderer presenting raw `.list` as a browse / sitemap
  / auto-sidebar view **MUST sort by page-name segment, byte-wise lexicographic ascending.** This is the *only*
  ordering contract v1 needs — it stops two impls scrambling the same site differently (entity-browser-rust already does this via
  a name-keyed `BTreeMap`). It is a **presentation rule, not a claim the tree is sorted.**
- **Naming-convention bridge (advice, NOT required):** name pages `YYYY-MM-DD-post` (ISO date) or `001-intro`
  (zero-pad) and the pinned name-sort *coincides* with chronological / sequential — free, self-documenting, no
  body read. Follow it and name-order carries meaning; ignore it and you still get deterministic name-order.
- **Frontmatter sort is optional local-renderer flavor, NOT the contract** — a renderer MAY offer "sort my view by
  `frontmatter.date`," but frontmatter is optional, so it can't be the cross-impl guarantee (that would re-open
  the same non-determinism `pages` was cut for). Don't let it be load-bearing.
- **Arbitrary curated order = `nav`** (authored).

**Semantic feeds were explicitly OPEN, and the deferral has been discharged on its own stated terms.**
"Newest-first," prev/next, RSS — an order that *declares its meaning* and must agree cross-impl — was the one
place a real contract was genuinely owed. v1 deliberately did **not** guess a field or format ahead of the use
case; it pinned only the determinism floor above and **deferred the semantic layer to a named feed/index
convention, to be authored when a concrete feed use case arrived.**

> **That convention is `APP-CONVENTION-FEED`, and its §4 is the answer.** A bounded head plus
> **key-addressed** pages, with exactly one semantic ordering contract — *`entries` within an index page
> is newest-first, as ordered by the publisher* — and the order is **authored, not derived.**
>
> **The determinism floor above is UNCHANGED and continues to govern raw enumeration views.** The feed
> convention satisfies this deferral rather than reopening it: it adds a semantic layer beside the floor,
> it does not make `created_at` authoritative, and it makes no claim that the tree is sorted. A renderer
> presenting a raw `.list` view still sorts by name segment, byte-wise, exactly as pinned here.

Paging of any ordered view is `.list` `limit`/`offset`.

**The principle (so it never creeps back):** the manifest is a **cover, not a collection store.** Every "do we
also need an ordered list / a set / a keyed map / paging of pages?" is answered **no** — collections of pages live
in **content** (an index page lists links — human) or in **`.list`/query** (enumerate + client-sort — machine),
never as proliferating manifest fields.

**The complete v1 set:** `manifest` (identity + optional `nav`) · `page` (content) · `nav` (menu) · `.list`
(discovery). Human navigation = `nav` + index-pages-with-links; **ordering floor = lexicographic-by-name** (a
renderer presentation rule; `.list` is hash-ordered); **semantic feeds = open, named post-v1 extension.**

## 5. Embeds in a site — pointer to EMBED `[reconciled to EMBED v0.2.2]`

Everything about an embed's bytes, handlers, output shape, fallback ladder, and reserved kinds is **EMBED's
contract**, not restated here. This convention only fixes the **site-level** facts:

- An embed is an `app/embed/{media_type}` ECF entity (EMBED §3). The **type tag is the dispatch key**; there is
  no `data.media_type` (`S-4`).
- All asset/handler references are the self-describing **`content-hash`** `(format_code, digest)` (V7 §1.2) —
  **never** a fixed-width / SHA-256-locked form. *(This convention inherits the corpus encoding-agnostic rule (`SPECIFICATION-FORMAT` §8.4.5);
  the old `hex33` is gone everywhere — EMBED v0.2 fix.)*
- A handler's output is an **`EmbedOutput`** — the closed dispatch vocabulary (EMBED §4): `text`/`image`/`box`/
  `raw`/`fallback`, with `media`/`interactive` **reserved** (EMBED §4.0a). A site renderer **MUST** degrade an
  unknown output `kind` via the fallback ladder (forward-compat for reserved kinds).
- **Renderer-capability + rendition selection** is EMBED §5.3 (capability-tagged renditions; deterministic
  selection; `⊆ renderer-caps` → else `fallback`). A site resolver consults `renderer-caps` **before** a handler
  emits a kind the substrate would drop.
- **v1 is passive-only at the format layer.** A v1 consumer **MUST refuse to render** any `Embed` carrying a
  non-empty `requires`/`sandbox` (EMBED §3) — active embeds are Part 3 / G1-gated (§8), DEFER for v1.

## 6. The bundle — one transferable, closure-complete artifact `[LOCKED — A1/A2 blessed; A3 rejected]`

A site transfers as a **closure-complete subgraph**. The closure rulings (arch first-pass §1, verified against
shipped ops — *nothing is invented*):

| Case | Mechanism | Verdict |
|---|---|---|
| Small asset (≤ ~16 KiB, §6.1) | **inline tree entity** → travels free in `tree:extract` (**A1**) | **BLESSED** — recommended for icons/thumbnails/SVG. This *is* "put it in the tree." |
| Large blob asset (full bundle) | `tree:extract` + **`content:ensure_closure`** (**A2** — both ship) | **BLESSED — the v1 pattern.** Sequencing of existing ops, not a new primitive. |
| Large blob asset (lazy browse) | on-demand `content:get` via namespace; cache under `{publisher}/…` (PRIMER §3) | **BLESSED** — don't eager-bundle the browse path. |
| Bundled compute handler (Part 3) | ships as a tree entity; `tree:extract` carries it; **G1** on arrival (§8) | **BLESSED** (DEFER build, G1-gated). |
| Bind a blob's *chunks* at tree paths | project chunk refs into the trie | **NOT BLESSED** — layering inversion (chunk structure is a content-store detail, not a namespace fact; chunk hashes are non-canonical chunker artifacts; cardinality blowup). Use A2. |
| One-shot "bundle-with-closure" core op (**A3**) | new wrapper op | **REJECTED for v1** — two honest sequenced calls beat a tidy new primitive. |

**The blessed bundle helper (SDK/L5, not a core op).** "One self-contained artifact, no second dereference" is
met at the **envelope layer**: the envelope's `included` map carries the content-store entities (assets +
bundled-compute), produced by `tree:extract` + `content:ensure_closure` + the A4 completeness check. Bless a
standard **closure-complete-bundle helper** that sequences those and emits **one transferable artifact** — the
**`.entsite`** single-file envelope (CBOR shape pinned in §6.0). *Rationale: if completeness is load-bearing
enough to add the A4 ingest guardrail, producing a complete bundle should not be an un-blessed two-step every L5
app re-implements.*

**Until A4 lands** (the EXTENSION-CONTENT ingest-completeness erratum — proposal-first, co-authored entity-browser-rust+arch),
publishers MUST use the **transactional wrapper**: extract into a **staging** sub-namespace → `ensure_closure` →
**rename live** (never ship the silent-success-now / 404-later path).

### 6.0 The `.entsite` bundle — pinned CBOR shape `[LOCKED — C-1; convergent CRITICAL, wb-go + godot]`
Both teams flagged: blessing the helper while leaving the bytes `[OPEN]` = three impls, three divergent bundles.
Pinned now. **`.entsite` is NOT a new envelope format** — it is a naming + packaging convention over V7's existing
`MaterializedEnvelope` (proposal-first: improvise no protocol), with three site-specific constraints:

```cddl
entsite = {                                  ; a single-file serialization; canonical CBOR (V7 §1.3 / ECF §4.1)
  v:        uint,                            ; entsite format version = 1
  envelope: materialized-envelope,           ; V7 system/envelope/v1 — UNMODIFIED
  ? pin:    site-root-pin,                   ; the SIGNED site-root pin (§2) travels for offline verification
}
materialized-envelope = {                    ; V7 shape, restated for completeness — not redefined
  envelope_hash: content-hash,               ; merkle over root + included (V7)
  root:          content-hash,               ; MUST equal pin.root when pin present (site identity)
  included:      { * content-hash => entity },
}
```

**The three site-specific constraints (what makes an envelope a valid `.entsite`):**
1. **`root` is a site subtree root** (a `tree:snapshot` root, §2) — and **MUST equal `pin.root`** when a `pin` is
   carried, binding the signed identity to the bundled bytes.
2. **`included` is closure-complete** over the site: every `SitePage`, every transcluded `Embed`, every asset
   blob **and its chunk closure**, and every bundled compute entity (Part 3) reachable from `root`. (This is the
   A2 output; the A4 check verifies it before the bundle is sealed.)
3. **Canonical CBOR only** (V7 §1.3) — so the same site produces a byte-identical `.entsite` across impls (the
   cross-impl portability the helper exists for; a `.entsite` conformance vector pins it).

Encoding-agnostic throughout: every hash is the self-describing `content-hash` (no width lock). A consumer
verifies `envelope_hash`, then (if `pin`) the publisher signature over `pin`, then `pin.root == envelope.root`.

### 6.1 Canonical chunking & reproducible publish `[LOCKED — G-PIN-4]`
"Same image → same site root" silently breaks if publishers chunk differently (no chunking-independent content
hash — EXTENSION-CONTENT §2.1/§2.3). **v1 publishers MUST use the canonical default `chunk_size` = 1 MiB FastCDC
average** (min/avg/max = 256 KiB / 1 MiB / 2 MiB — the shipped FastCDC params). A **reproducible-publish**
conformance test ingests one fixture under two publishers and asserts an identical site root.

## 7. Tiers & the floor `[LOCKED — capability adds, never assumed]`

| Tier | Reads | Notes |
|---|---|---|
| **Floor** (core only) | markdown bodies + **inline** passive embeds (≤16 KiB) + every embed's authored `fallback` | no handler tier → **no active content reaches the floor by construction**; the base markdown renderer is on the safe side of the ladder (`S-11`). A floor peer is a *valid* reader, not a degraded one. |
| **+content ext** | large blob assets (pointer payloads), renditions | `content:get` / `ensure_closure`. |
| **+tree ext** | site **identity** (verifiable root hash), bundle extract | `tree:snapshot`/`tree:extract` are EXTENSION-TREE, not core. |
| **+compute** | **active embeds** (Part 3) | requires **EXTENSION-COMPUTE on the evaluating peer**; G1-gated; DEFER v1. Active-compute embeds **require** COMPUTE — state the dependency precisely (not "compute embeds ship" unqualified). |

**Cap scope (v0.5).** The site subgraph's own root path is blessed as a **first-class capability scope** —
the site subgraph *is* the cap surface, derived structurally from the publisher-chosen placement (it does
not need to be carried in the manifest). Cross-peer caching under `/{other_id}/…` is PRIMER §3 (the cache
*is* the tree, partitioned by peer), not a new feature. The prior v0.4.2 wording naming
`{publisher_peer_id}/content/sites/{site_id}/` as the scope is dropped per the v0.5 placement erratum above
(§2 placement): a site is a free subgraph, and the scope is wherever the publisher put it.

**The scope of a site's CONTENT and the scope of VERIFYING it are different sets `[v0.5.2]`.** The site
subgraph is the capability scope of the site's **bytes**. The evidence that makes those bytes verifiable
— the publisher's `system/peer/published-root` (`EXTENSION-TREE` §3.3a) and the `system/signature/`
location of its root (V7 §5.2) — sits **outside the subgraph, in the publisher's namespace, by design**:
a signed root that lived inside the subtree it commits to could not commit to itself.

> **[MUST]** A grant intended to make a site **verifiable** covers those two locations **in addition to**
> the site subgraph. A grant covering the subgraph alone yields a **readable, unverifiable** site.
>
> **[MUST NOT]** An implementation **MUST NOT** report an authorization failure at those locations as an
> **absent published root**. The reader observes *"this publisher has published no root"* — a **false
> statement about the publisher, manufactured by the reader's own grant** — and it points a diagnosis at
> the wrong machine.

**The scope sentence in §2 is unchanged and is not being widened** — the scope of the content really is
the subgraph. This is an adjacent obligation, and it is worth stating because **the narrow grant is the
one a careful implementer writes**: it is what the clause says, and least-privilege is the instinct.
*The trap is baited with good practice.*

## 8. Security `[gates named with owners — security is never deferred, even when the build is]`

- **G1 — active-compute embed audit** (Part 3; gates the active-compute milestone, **not** v1 passive). Before
  active/interactive embeds ship, a baseline audit of **install-audit + self-issued prefix-scoped grant**, with
  five conditions (workbench-go, accepted): **C1** audit shell-inspectable (structured read/write/handler paths,
  peer-qualified, grant scope printable — today `ImpureOperations` is opaque, surface it); **C2** per-front-end
  audit-prompt policy hook (modal / interactive / refuse-by-default; no forced global style); **C3** **cross-peer
  dispatch from bundled compute = refuse-by-default**, even if a cap could cover it (`S-10`); **C4** per-site
  policy keyed by **(consuming-peer-id, site-root-hash)** (trusting a site under peer A ≠ under peer B); **C5**
  `expression_path` install is a **distinct SDK surface** (`InstallSiteScopedCompute`), not a side door of
  generic handler registration. Grant is **read-only-within-site** scoped; auto-install-without-review is **never**
  the default. Owners: arch + entity-browser-rust.
- **G2 — web HTML sanitization** (gates `format:"html"` / raw-HTML embeds on the **web** front-end). Audit the
  sanitizer (allowlist completeness; text-escaping ≠ sanitization). Non-DOM substrates (Godot/terminal) sidestep
  this by having no DOM — a property of the ladder. **Republish clause:** a front-end re-publishing a site it did
  not author attaches a **`passthrough_of: {origin_peer_id}` marker** so downstream renderers apply origin-trust
  policy (closes the HTML-laundering hop). Owners: entity-browser-rust (web renderer) + arch review.

Neither gate blocks the **v1 passive floor** (markdown + passive media + fallback — no active code, no raw HTML by
default).

## 9. Required checks — what an implementation must discriminate `[REQUIRED before ratification — exercised JOINTLY with EMBED's]`

A FORMAT convention is not validated until conformance checks exercise it (PRIMER meta-rule). **This section
names the cases; the fixtures, the bytes and the run are the implementations' and the conformance oracle's**
(GUIDE-EXTENSION-DEVELOPMENT §7, `GUIDE-CONFORMANCE` §5.1a). The cases this convention names:
- A **`SiteManifest`** CBOR + expected hash; a **`SitePage`** round-trip (markdown body with a `::embed`
  directive → lowered `child` `Embed` → re-serialized, byte-identical).
- A **signed `site-root` pin** that verifies **cross-impl** (`G-PIN-3`) — fixture signed by a known identity, with
  expected signature bytes (byte-identical across at least two independent L5 implementations).
- A **reproducible-publish** test (`G-PIN-4`) — one fixture, two publishers, an identical **`tree:snapshot` root
  over the site subgraph** (`EXTENSION-TREE` §3.2). **Not the published-root head**, which carries `published_at`
  and is not comparable across runs or across implementations — a check posed on the head fails one hundred
  percent of the time and, across two implementations, fails wearing a real divergence's clothes.
- An **`::embed` directive ⇔ child-`Embed` lowering** vector (round-trip lossless, §3.2) — including a `ref` of
  **each resolvable form** (`APP-CONVENTION-REFERENCE` §3.4: root-absolute, `site:`, fully-qualified, and
  directory-relative) **plus one unparseable string**, to pin that resolution *and refusal* agree across
  implementations.
- A **nav-cycle / max-depth** vector (`F-5`): a fixture of **authored depth 40** (> the max 32) with a cycle;
  assert the renderer stops at depth 32 and on the cycle without looping — pinned depth so "stop cleanly" is not
  vacuously conformant.
- A **`.entsite` bundle** vector (`C-1`): a closure-complete bundle of a small site, canonical-CBOR, byte-equal
  cross-impl, with `pin.root == envelope.root` and a verifying signature (§6.0).
- A **passive-only refuse** vector (v1 consumer refuses a non-empty `requires`/`sandbox` embed — §5).
- An **asset payload discrimination** vector (§4's `[MUST NOT]`) — one `app/site-asset` per outcome,
  asserting the three stay **distinct and distinctly attributed**: an `inline` and a `pointer` asset
  **accepted**; a **correctly tagged `child`** asset **refused as invalid for its type** (the
  publisher's defect); an **untagged** payload refused as malformed (EMBED §3). **A run that reports
  the `child` case as malformed, or as an unimplemented arm, fails this vector** — the whole point is
  that the three outcomes are not one.
- *(EMBED ships the `EmbedOutput`/payload/`raw`-drop-clean/unknown-`kind`/agility-pair/rendition vectors; this
  convention does not duplicate them — it cites EMBED §9.)*

Cross-impl byte-equality on these + EMBED's is the **joint v1 lock signal.**

## 10. Open / deferred
- `[PROPOSAL-FIRST]` **A4** — EXTENSION-CONTENT ingest-completeness guardrail (loud-reject on absent chunk
  closure, or explicit `partial:true` ingest mode). Co-authored entity-browser-rust+arch; this convention is input, not the edit.
- `[DEFER → G1 build]` active/interactive embeds (Part 3); WASM handler host (named future, no date). v1 is
  passive (Parts 1+2).
- `[DEFER, named]` field-level manifest merge (nav union / root conflict-error) with the propose-back/edit arc;
  FTS + subtree-scoped query (D2); content-dereference registry-arm (D3, the content-network endgame);
  broken-link / dead-peer fail-mode contract; handler upgrade/re-resolution semantics; rename/re-home
  forwarding-pin (`site-redirect`); within-peer tier-upgrade migration (forward-only); compute quota/revocation
  per site; offline-first "snapshot from {t}"; audit-diff on site update; inline-flow (`::embed` mid-paragraph)
  directive extension (§3.2).
- *(`.entsite` envelope CBOR — **now pinned**, §6.0; left the deferred list this pass, `C-1`.)*

## 11. URL projection prefix `[v0.5 — registers sites at the §6.5.6 demux]`

Sites are tree subgraphs. To make them addressable on the legacy web (and the static no-JS surface for
permalinking / SSG-style consumption), the SITE convention claims a **reserved first-segment URL literal**
at the `EXTENSION-NETWORK §6.5.6` demux layer:

```
{base}/sites/{peer_id}/{site_id}/…
```

`sites` is the SITE convention's registered reserved word per **`EXTENSION-NETWORK §6.5.6 G4` (Amendment 9
— reserved-word table extensibility hook + length-floor rule)**: NETWORK exposes the mechanism; the SITE
convention owns the entry. The literal `sites` is five characters, comfortably below the Ed25519 peer-id
minimum length — it satisfies the length-floor rule and cannot collide with a parseable peer-id at the
demux.

**Projection semantics.**
- The URL form is a **publish-time projection**, not a tree-storage rule. A site lives wherever the
  publisher chose to put its subgraph (§2 placement); the projection maps `(peer_id, site_id)` to that
  subgraph at publish time.
- The publisher's manifest names the site subgraph's tree root explicitly, so a reader (or a static
  exporter) can find the site without scanning. Discovery mechanism (tree-walk, query, curated list of
  manifest refs) is **implementer's choice** — this convention does not prescribe one.
- The static surface is for permalinking, no-JS readers, and the SSG use case. Real verification (cap
  checks, signature verification beyond what static metadata can claim) happens in a live entity-aware
  peer.

**Word-overloading rule.** Each word in the system carries one meaning. `content` is the CONTENT
extension's namespace word per `EXTENSION-CONTENT §6.4` — the SITE convention does NOT squat on it. The
v0.4.2 `content/sites/{site_id}/_root` placement is the layer violation v0.5 corrects (§2 placement).

**Other L5 conventions.** The SITE convention registers `sites` only. Other L5 conventions (a future
repos / spaces / etc.) register their own prefixes in their own specs under the same NETWORK §6.5.6
extensibility hook; this convention does not preemptively claim words on their behalf.

**Adoption posture.** The prefix is the SITE convention's claim, not a universal mandate. Adoption by
other parties is social convergence (cf. `GUIDE-APPLICATION-DEVELOPMENT` framing — L5 is convention-not-conformance for
non-format axes); the convention is here to be analyzed and adopted on merit, not joined.

## 12. Provenance
- Arch first pass: the L5 semantic-content-site arch-first-pass review (`4351f73`) — closure
  rulings (§1), `[ASK-ARCH]` point rulings (§2), gates (§4), domain (§5).
- Cross-team v1 lock: the content-site v1-lock synthesis (`e8a33c5`) — the output lock (§2,
  now relocated into EMBED), format pins (§4), strain dispositions (§5), G1 conditions (§7).
- Consumed convention: `applications/APP-CONVENTION-EMBED.md` v0.2.2 (`6f29909`) — node, `EmbedOutput`, registry,
  ladder, reserved kinds. Charter: `guides/GUIDE-APPLICATION-DEVELOPMENT.md`.
- Substrate grounding: `tree:snapshot`/`tree:extract` (EXTENSION-TREE v4.0.2 §3/§6); `content:ensure_closure`
  (SDK-EXTENSION-OPERATIONS §11 Amendment A); `expression_path` core-stable (V7 §3.7/§6.6, EXTENSION-COMPUTE
  v3.14); self-describing `content-hash` `(format_code,digest)` (V7 §1.2/§1.4).

---

## Document History

**v0.5.2:** two clauses, neither widening a rule, both naming an authority that already binds. §4 states
that a bare-string link or asset reference in a page body resolves per `APP-CONVENTION-REFERENCE` §3.4 —
**a leading `/` is root-absolute within the current site** — because **the asset position is not special
and this convention has no rule of its own about it**, and two implementations diverged for a week over a
form §3.4's own paragraph recommends producers emit. §7 adds the verification scope: the site subgraph is
the capability scope of the site's **bytes**, and the evidence that makes those bytes verifiable — the
publisher's `system/peer/published-root` and the `system/signature/` location of its root — sits **outside
the subgraph by design**, so a grant covering the subgraph alone yields a readable, unverifiable site and
reports it as *"this publisher has published no root"*, which is a false statement about the publisher
manufactured by the reader's own grant.

**v0.5.1:** `app/site-asset`'s `payload` is **narrowed to `inline-payload / pointer-payload`** and the
`child` arm is refused as **invalid for its type** (§4), with a §9 vector asserting the three payload
outcomes stay distinctly attributed. The field imported a union wider than the type admits; an asset
names bytes, and a `child` ref names an entity whose dispatch tag would be a second `media_type`. Per
the site-asset-child-arm-and-publication-grant proposal.

**v0.5:** `{publisher_peer_id}/content/sites/{site_id}/_root` placement (a layer violation: `system/content/*` is the
CONTENT-extension namespace for capability-scoping the content-hash address space, where the leaf is always
`{hex(H)}` per `EXTENSION-CONTENT §6.4.2`; an L5 application subgraph has no business there). Sites are now
free subgraphs at publisher-chosen tree paths; the site's capability scope is its own subgraph root. Adds a
**new §11 URL projection prefix** that registers `sites` as the SITE convention's reserved first-segment
literal at the `EXTENSION-NETWORK §6.5.6` demux layer (per Amendment 9's reserved-word extensibility hook
and length-floor rule). Word-overloading rule: each word in the system carries one meaning; `content` is
the CONTENT extension's, `sites` is the SITE convention's. Per
the network-reserved-word-extensibility-and-site-prefix proposal.

**v0.4.2:** joint cross-team round folded per
the joint embed/site convergence synthesis (spine locked three ways; `.entsite` pinned,
`.list` discovery folded, nav/directive/tree-version fixes). **v0.4.1: `pages` field REMOVED** (arch scaffolding;
redundant with `nav`; reintroduced the index anti-pattern — §4.2). **v0.4.2: ordering tightened (entity-browser-rust)** — pin one
determinism floor (lexicographic-by-name presentation rule); frontmatter is optional local flavor not the
contract; **semantic feeds flagged OPEN / still-researching, deferred to a named post-v1 extension** (L5 — guidance
now, lock later). v1 primitives: `manifest`/`page`/`nav`/`.list`. Reconciled to **APP-CONVENTION-EMBED v0.2.3**.
Supersedes the scattered design set (entity-browser-rust `SPEC-SEMANTIC-CONTENT-SITE` Rev 0.1/0.2 + the v1-lock synthesis).
**Next: exercise the §9 checks jointly with `APP-CONVENTION-EMBED`'s → ratify** (no further team cycle on the
contracts — the cross-impl exchange is the lock; folds recirculate as a confirm).
**Domain:** `applications/` (second member, first consumer of EMBED).
