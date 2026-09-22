# PROPOSAL — the entry body is an inline `Embed` node, the term named the wrong surface, and the site asset gets declared

**Status:** IMPLEMENTED 2026-09-10 — all four rulings folded the same session. `R1` into
`APP-CONVENTION-EMBED` §3.1 (`embed-node` defined) · `R1b` into `APP-CONVENTION-FEED` §2.3 (which
surface, and the two worked shapes) · `R2` into `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4
(`app/site-asset` declared, with the pointer `[MUST]`) · `R3`/`R4` are sequencing rulings and land in
the design register (`AT-35`, `AT-36`) rather than in spec text.
**Target:**
`specs/applications/APP-CONVENTION-EMBED.md` §3 (**R1** — define `embed-node`) ·
`specs/applications/APP-CONVENTION-FEED.md` §2.3 (**R1b** — the citation resolves, and which surface it names) ·
`specs/applications/APP-CONVENTION-SEMANTIC-CONTENT-SITE.md` §4 + §6.1 (**R2** — declare `app/site-asset`)
**Provenance:** the application tier's stage-1 sequencing review, which asked what an entry body is
when no implementation has an `Embed` entity. The answer is that the question rests on a term this
corpus never defined.

---

## §0 The finding, stated first because it inverts the ask

**The question raised was:** `APP-CONVENTION-FEED` §2.3 declares `body: embed-node` and imports
`APP-CONVENTION-EMBED`; **no implementation has an `Embed` entity node**; therefore stage 1's joint
fixture cannot pin an entry's bytes, and stage 1 may have to wait on the embed convention.

**Measured against the corpus, three things are true and the conclusion does not follow:**

1. ⭐ **`embed-node` is defined nowhere.** It occurs **exactly once** in all of `specs/` and
   `guides/` — in `APP-CONVENTION-FEED` §2.3's own CDDL, as the type of a normative field. The embed
   convention defines `embed-data` (§3) and the `Embed` entity (§3); it defines no production of that
   name. **A normative field is typed by a production that does not exist.**
2. ⚠ **The only descriptive use of the term in the corpus points at the WRONG ONE of the embed
   convention's two surfaces.** A design exploration describes an entry's body as *"an `embed-node`
   tree (four leaves plus one `box` container)"* — and *four leaves plus one `box` container* is
   **`EmbedOutput`**, the §4 **output** contract, not the §3 **input** entity.
3. ✅ **Under the correct surface, nothing blocks stage 1.** A feed entry's body needs no separate
   `Embed` entity, because the node is **inline**.

**So the ask resolves to a one-production addition, and the sync point is unblocked today.**

---

## §1 `R1` — define `embed-node` as the inline `(type, data)` form of an `Embed`

### 1.1 Why the output surface is the wrong answer, and it is not a close call

`APP-CONVENTION-EMBED` §2 draws the split in a diagram and labels both ends:

```
INPUT  (authored, content-addressed, travels in the subtree)
  Embed entity   type = app/embed/{media_type}    data = §3 CDDL
         │  handler (media_type → output)
         ▼
OUTPUT  (structured value; renderer-neutral; the cross-substrate contract)
  EmbedOutput    Text | Image | Box{kind} | Raw{format} | Fallback   (§4)
```

**A feed entry is authored, content-addressed and travels in the subtree** — that is the input row,
word for word.

**Typing a stored entry's body as `EmbedOutput` would break three landed mechanisms at once**, and it
is worth naming them because each is a `[LOCKED]` section:

| Mechanism | What storing output does to it |
|---|---|
| §5's **two-level registry** | The handler layer has nothing left to dispatch — the author already ran it. `{media_type → handler}` becomes dead. |
| §5.3's **rendition selection** | The reader's substrate capabilities decide which rendition to fetch. If the author emitted output, **the author chose for every reader, once, forever.** |
| §6's **degradation ladder** | Degradation is defined over an output produced *for this substrate*. There is nothing left to degrade differently. |

And it breaks §4.0's extensibility story outright: *"a new embed type = a new `app/embed/{type}` + a
handler that lowers your type into the vocabulary."* **If the body is output, a reader never sees the
type**, so the open extension point is unreachable from the one vocabulary that most needs it.

> **The narrower and more useful way to say it: `EmbedOutput` is a CLOSED vocabulary by design (§4.0),
> and a published wire format typed by it can never carry a content kind the vocabulary did not
> anticipate.** The input surface is open on purpose. **A convention that publishes the closed one
> has published the wrong half of the model** — and the tier's own ordering principle is that
> whatever publishes first becomes the baseline.

### 1.2 Why an entity reference is also wrong

The remaining alternative — `body` names a separate `Embed` **entity** — is refused by the embed
convention's own sentence about this exact field:

> *An image or video **entry** (`APP-CONVENTION-FEED` §2.3) is an entry whose `body` is an embed
> carrying a **`pointer-payload`** — a content hash into the store.*

**"an entry whose body IS an embed carrying a payload"** is the inline reading. And
`APP-CONVENTION-FEED` §2.3's own prose agrees: *"A photo post and a text post are one shape with a
different embed inside."* **Inside, not beside.**

The structural argument is the same one: requiring a second addressable entity for every text post
doubles the entity count of the tier's most common object to carry a field that is never separately
referenced, never separately fetched, and never separately signed.

### 1.3 The edit — `APP-CONVENTION-EMBED` §3, after the `embed-data` CDDL

> **`embed-node` — the inline form.** An `Embed` is ordinarily an addressable entity. **Where a
> consuming convention carries an embed *inside* another entity's field rather than beside it, it
> carries an `embed-node`: the same `(type, data)` pair, inline and not separately addressed.**
>
> ```cddl
> embed-node = {
>   type: tstr,          ; "app/embed/" .cat media-type — the SAME dispatch key as the entity's
>   data: embed-data,    ; §3, unchanged
> }
> ```
>
> **The dispatch key is carried, not dropped.** §3 puts the media type in the type tag rather than in
> `data` precisely so there is one source of truth; an inline node keeps that property, which a bare
> `embed-data` would lose. **A node and an entity differ in addressing only** — a handler that
> accepts one accepts the other, and an implementation MUST NOT make the behaviour depend on which
> form the embed arrived in.
>
> **When to use which.** Inline where the embed *is* the field's value and is never referenced
> independently (a feed entry's `body`). A separate entity where it is referenced by path or hash
> from more than one place, or transcluded — which is what `child-payload` names, and is why the
> child arm is a reference and the other two are not.
>
> **The inline bound applies unchanged.** `inline-payload`'s `.size (1..16384)` ceiling is a property
> of the payload, not of the addressing, so an inline node carrying large bytes uses `pointer` for
> exactly the reasons §3's note already gives.

### 1.4 `R1b` — `APP-CONVENTION-FEED` §2.3

The CDDL line needs no change once the production exists. **Add the surface note**, because the
ambiguity that produced this proposal is a reader's, not a writer's:

> **`body` is an `embed-node` — the INPUT surface (`APP-CONVENTION-EMBED` §3), inline.** It is **not**
> `EmbedOutput` (§4): an entry stores what was authored, and the handler runs at the reader. **No
> separate `Embed` entity is required for an entry** — a text post is an inline payload, an image post
> is a pointer payload, and the two are the same shape.

### 1.5 What this unblocks, concretely

A stage-1 joint fixture can pin entry bytes **today**, with no dependency on any implementation
shipping an `Embed` entity:

```
text post   { type: "app/embed/text/plain",
              data: { payload: { tag: "inline", bytes: <utf8> }, fallback: "…" } }

image post  { type: "app/embed/image/png",
              data: { payload: { tag: "pointer", hash: <content-hash> },
                      fallback: "…", renditions: [ … ] } }
```

**The transclusion arm (`child`) is the only one that needs a separate entity, and it is optional** —
so a fixture may include it as a case rather than a prerequisite.

---

## §2 `R2` — `app/site-asset` is declared, and declaring it closes a chunker bypass

`app/site-asset` occurs **zero** times in `APP-CONVENTION-SEMANTIC-CONTENT-SITE`. One implementation
ships it; the convention does not mention it. **Declare it**, for three reasons that are measured
rather than aesthetic:

1. **It is why the reproducible-publish fixture carries two roots** — one comparable, one not. An
   undeclared type cannot be compared across implementations, so the asset row is excluded from the
   only cross-implementation evidence this tier has.
2. ⚠ **The shipped shape inlines raw bytes at any size, which BYPASSES §6.1's chunker `[MUST]`.**
   §6.1 requires v1 publishers to use the canonical 1 MiB FastCDC default so that *"same image → same
   site root"* holds. **Bytes that never enter the content store never meet that rule** — so the
   MUST is not failing, it is unreachable on this path, which is worse because no check can see it.
3. **The other implementation has already paid for this.** The embed convention's §3 note records an
   inline-failure history above 10 MB at the second seat, and sets the inline ceiling *"because inline
   bytes inflate trie nodes and `.list`."*

**The edit — a new type in §4, deliberately reusing the embed payload union rather than minting a
second one:**

> **`app/site-asset`** — a named, site-local binary asset (an image, a font, a stylesheet), addressed
> by its tree binding relative to the site root so that a directory-relative `ref` (§3.2) resolves to
> it.
>
> ```cddl
> site-asset = {
>   type: "app/site-asset",
>   data: {
>     media_type: tstr,            ; IANA type — an asset has no dispatch tag of its own
>     payload:    embed-payload,   ; APP-CONVENTION-EMBED §3 — inline (bounded) or pointer
>   }
> }
> ```
>
> **[MUST]** An asset whose bytes exceed `inline-payload`'s ceiling **MUST** use a `pointer` payload,
> placing the bytes in the content store, **where §6.1's canonical chunking governs them.** An
> implementation **MUST NOT** inline unbounded bytes in a site asset: it defeats §6.1's
> reproducible-publish guarantee for that asset silently, and it is the failure the embed convention
> §3's inline ceiling exists to prevent.

**Why not withdraw it instead.** A site asset needs a **name** — `assets/figures/x.svg` — because
§3.2's directory-relative `ref` resolves against the tree, and a bare content-store blob has no name.
The type is what the naming binding points at. **The alternative would be to make every asset an
`Embed` entity**, which is defensible and is the second implementation's reserved direction; it is not
proposed here because it would make the two implementations diverge further today in order to converge
later, and the payload union above is compatible with taking that step afterwards.

---

## §3 `R3` — the three draft proposals: build against none of them yet

The sequencing question asked whether to build phase 2's refresh loop against
`PROPOSAL-FOLLOW-THE-PATTERN` §5 while it is DRAFT.

**The reading offered was that phases 0–3 need none of the three and phase 4 needs all three. That is
correct, and it is the answer.** Stated as a ruling so it does not have to be re-derived:

| Phase | Needs a draft? | Why |
|---|---|---|
| 0 — measurement, close the divergence | **No** | Landed vocabularies only |
| 1 — the carrier seam | **No** | Refactor of shipped code; no vocabulary |
| 2 — publish and follow feeds | **No**, with one carve-out below | The feed convention is landed |
| 3 — the reader loop | **No** | Forward traversal and reply-indexing are client-side; no protocol |
| 4 — the gatherer | **All three** | The walk object, the coordinate, and the pattern's loop |

**The carve-out is phase 2's refresh loop, and the ruling is: build the loop, do not build the
vocabulary.** Cadence, backoff, coalescing and sequence-number comparison are **client behaviour over
an already-landed signed root** — the published-root entity carries `root_hash`, `seq` and
`predecessor`, and every input the loop needs is in landed text. **What is DRAFT is the description of
that loop, not the substrate it runs on**, so an implementation that reads the landed root and polls it
is not building against a draft; it is building the thing the draft describes. **If the draft moves, a
`[SHOULD]` about cadence moves — not a wire format.**

**What must not be built: the walk object, the coordinate, and any gatherer vocabulary.** One of the
three has its **tier explicitly undecided**, and a vocabulary whose tier is open cannot have its
namespace chosen. That is phase 4, and phase 4 waits.

---

## §4 `R4` — the follow list's tier was already reframed, and the binary is not live

The question *"is the follow list an app convention or an extension?"* is recorded in a planning
document's open-question list **struck through**, with the reframe beside it: **the follow *list* is
application data; the follow *pattern* is a mechanism already present in the substrate by two routes.**
The decision-relevant question became the promotion test — *is there a second non-social consumer of
the identical loop?* — **and the cursor half of that was discharged on three shipped consumers**, which
is why the feed convention carries `cursor` provisionally and flagged.

**So: `app/feed/follow` is an application convention and its tier is not open.** It is application
data — a reader's own list, in the reader's own namespace, which nobody else validates. **Nothing is
owed before phase 2a**, and a conformance vector for it is an ordinary convention vector.

> **The reason this is worth a ruling rather than a pointer:** the struck-through row and the reframe
> sit in the same list, so a reader arriving at the list sees a live-looking question. **A reframed
> question left in an open-questions list reads as open**, which is the same failure as a deferral
> whose trigger has fired. The row should be moved out of the list rather than struck within it.

---

## §5 What this proposal does not touch

- **No test vectors, fixtures or bytes are authored here.** §1.5's shapes are the *cases* a fixture
  must discriminate, written so the fixture's author can produce them; the bytes and the run are the
  implementations'.
- **`EmbedOutput` is unchanged**, and nothing here reopens §4.0's closed vocabulary. The correction is
  about which surface a *different* convention cites.
- **The second implementation's reserved `assets/{name}` direction is not foreclosed** — §2's payload
  union is compatible with promoting assets to embed entities later.
- **The gatherer's bootstrapping question is not answered here.** It is narrowed in the design record
  rather than closed, and narrowing is not solving.
