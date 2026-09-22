# PROPOSAL — a site asset names bytes, so the `child` arm is not its; and "no grant" names the audience model, not the policy table

**Status:** IMPLEMENTED 2026-09-10 — both rulings folded the same session. `R1`/`R1b` into
`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4 + §9 · `R2`/`R2b` into `APP-CONVENTION-SHARE` §2.5 + §8 ·
`R3` into `guides/GUIDE-APPLICATION-DEVELOPMENT.md` §3 (the tier-wide row; third instance).
**Target:**
`specs/applications/APP-CONVENTION-SEMANTIC-CONTENT-SITE.md` §4 + §9 (**R1**, **R1b**) ·
`specs/applications/APP-CONVENTION-SHARE.md` §2.5 + §8 (**R2**, **R2b**) ·
`guides/GUIDE-APPLICATION-DEVELOPMENT.md` §3 (**R3**)
**Provenance:** two asks filed by the application tier's integration seat while implementing the
`app/site-asset` pointer `[MUST]` and the audience-less publication type. Both were filed as *"nothing
is blocked, we would rather match you than be independently reasonable"*, and both are answerable from
this corpus without a new mechanism.

---

## §0 Both asks, and why neither needed a new decision

Each ask reports a document disagreeing with itself, and in each case the corpus already carries the
sentence that settles it. **Neither ruling adds a rule; both make a normative half agree with an
intent the same document already states.**

| Ask | The disagreement | Settled by |
|---|---|---|
| **A-30** | `site-asset.payload` is typed `embed-payload` (**three** arms) beside a comment reading *"inline (bounded) or pointer"* (**two**) | `APP-CONVENTION-EMBED` §3's own reason for having no `data.media_type` |
| **A-31** | *"No `audience` and no grant"* against the next sentence, *"authorization at fetch is none required, pull-only"* | `SHARE-9`, and `ENTITY-CORE-PROTOCOL` §4.4 / §6.2 |

---

## §1 `R1` — `child` is not a site-asset payload, and the field narrows to say so

**RULED: no.** A conformant publisher **MUST NOT** emit an `app/site-asset` carrying a `child`
payload. The field narrows to `inline-payload / pointer-payload`; the comment was right and the type
was loose.

Four arguments, strongest first. **The first is decisive and is internal to the embed convention.**

1. ⭐ **`media_type` beside a `child` ref is the second source of truth EMBED §3 removed by hand.**
   `child-payload.ref` names a sibling `Embed`, and an `Embed`'s dispatch key **is its type tag** —
   `app/embed/{media_type}`. EMBED §3 says so in its first sentence and states the reason: *"there is
   **no** `data.media_type` field (it would be a redundant second source of truth — dropped per
   cross-team S-4)."* A `site-asset` carries `media_type` in `data`. **So a `child`-payload site asset
   is exactly the two-media-types-able-to-disagree shape that S-4 deleted, reintroduced one convention
   over, with no rule anywhere saying which wins.**
2. **The type exists to NAME BYTES, and an entity is already named.** §4's own preamble: *"§3.2's
   directory-relative `ref` resolves against the TREE, and a bare content-store blob has no name; the
   naming binding points at this."* A `child` ref names an **entity**, which has a tree path already.
   The arm would buy one extra hop and no expressiveness. *(This is the filing seat's own inference,
   offered rather than asserted. It is correct, and argument 1 is why it is correct.)*
3. **`child` is the only arm that may cross a peer boundary; a site asset is site-local.** EMBED §3:
   *"the **ONLY** payload that may cross a peer boundary, so it carries the authority term."* §4
   declares the type *"a **NAMED, site-local** binary asset."* Worse, §6 makes the `.entsite` bundle
   *"one transferable, **closure-complete** artifact"* — a cross-peer `child` ref inside an asset makes
   closure uncloseable, and it would surface at bundle time rather than at authoring time.
4. **It is a third path around §6.1's chunker.** The §4 `[MUST]` routes over-ceiling bytes into the
   content store *"where §6.1's canonical chunking governs them"*, because bytes that never enter the
   store make the reproducible-publish row **unreachable rather than failing**. A `child` arm reaches
   the store only through another entity, so the asset's own row is incomparable again by a second
   route.

> **This costs the site convention nothing, and checking that was the point — a tightening that closes
> no hole is not conservative, it is just narrower.** **Sites keep `child` exactly where it belongs.**
> EMBED §3's *Embedding modes*
> (a) is child-entity transclusion — *"the **v1** mode"* — and it lands on a `site-page`, whose
> `::embed` directive **MUST** lower to a `child` payload on a sibling `Embed`. §9 already names that
> vector (*"an `::embed` directive ⇔ child-`Embed` lowering vector, round-trip lossless"*). **A page
> transcludes; an asset names bytes.** Narrowing the asset removes no authoring capability and leaves
> the only landed `child` consumer untouched.

### `R1b` — the refusal is `invalid-for-type`, and it is NOT the untagged refusal

**This is a correction to the filing seat's stated plan, and it is the reason this ruling was worth
sending rather than filing.** Their reader splits payload outcomes three ways and they proposed that a
"no" answer *"moves `child` from the second row to the third and nothing else changes"* — from
*unsupported arm (our gap)* to *`NoPayload` (theirs, per EMBED §3's untagged MUST)*.

**That collapses two distinct facts and mis-attributes one of them.** EMBED §3's refusal is
*"decoders MUST reject an **untagged/ambiguous** payload"* — a shape with no discriminator. A `child`
payload on a site asset is **correctly tagged, well-formed, and unambiguous**; it is invalid *for this
type*. Three outcomes that must stay apart:

| the reader sees | outcome | whose defect |
|---|---|---|
| a tag it has not built | unsupported arm | **the reader's gap** |
| **`tag: "child"` on an `app/site-asset`** | **invalid for type** | **the publisher's — a schema violation** |
| no tag at all | untagged refusal (EMBED §3) | the publisher's — a malformed payload |

**Folding row 2 into row 3 reports a deliberate schema violation as a malformed byte**, which is the
same conflation the filing seat itself refused when they kept `InvalidForType` apart from *malformed*
for `SHARE-8`. It is the same distinction, one convention over.

---

## §2 `R2` — "no grant" is a statement about the AUDIENCE MODEL. The filing seat's reading is confirmed

**RULED: confirmed.** *"No grant"* means **no member is enumerated and no per-member token is
minted** — it is the audience axis and nothing else, consistent with the type being *"`share-record`
minus `audience`, and nothing else differs."* **It is not an instruction to author no authorization
for the read.**

**The deciding argument is ours and the filing seat found it:** `SHARE-9` requires a consumer
presenting **no token** to be **not refused for lacking one**. Under the literal reading the fetch is
refused for there being no authority at all — so the vector is unpassable, and a vector this
convention names cannot be unpassable by this convention's own text.

**The ask arrived supported by a read of a reference implementation's connection-grant defaults. That
read was re-verified independently before any of it was moved into this ruling — at a later commit
than the one cited — and it holds: the connection-time floor covers the type and handler namespaces
only, and reaches no published target.** *(The pinned observation is in the internal record; a
specification does not carry it, and the argument below does not need it.)*

⭐ **Because the mechanism is not an implementation detail at all — it is core-spec, and that is what
makes the ruling portable.** `ENTITY-CORE-PROTOCOL` states all of it normatively:
§4.4 — *"the initial grant scope delivered to A is the **union** of the SHOULD floor above and the
matched policy entry's grants… implementations without a policy table populated for peer A deliver
only the SHOULD floor"*; §6.2 — resolution is *"exact-match-or-`default`-fallback."* **So this is the
core's defined path, and SHARE names it rather than inventing one** (the convention adds *"no new
authorization path"*, §0).

### `R2b` — the hazard the packet reached for and did not close

The filing seat identified the ceiling/floor collision as *why the sentence reads two ways*. **It is
more than that: it is a live side effect of the fix they shipped, and nothing warns anyone.**

`ENTITY-CORE-PROTOCOL` §6.2 makes the **same `default` entry** a **union term** at §4.4
authenticate-response and a **per-peer ceiling** at `system/capability:request`, and says both are
intended. §6.2 also says that with no entry, *"pure-attenuation flow … works without a policy entry by
skipping the policy ceiling — step 3 only enforces bounds that exist."*

**Therefore: creating a `default` entry where a deployment had none converts *no request-time ceiling*
into *this request-time ceiling*, for every peer without an entry of its own.** A publisher who
authors an entry carrying only the publication's read grants has silently narrowed what every unlisted
peer may `request` — which is the regression shape the core's own text is written around. The fix is
one sentence at authoring time (write the entry as a **union** with what the deployment already
intends to allow), and it is invisible without the warning.

**SHARE states the obligation and points at the core for the mechanism** (`SPECIFICATION-FORMAT`
§10.3 — restate nothing corpus-wide; where you restate, name the authority). A new vector,
**`SHARE-10`**, makes the side effect observable, since the convention names cases and the
implementations run them.

---

## §3 `R3` — the carve-out shape has its third instance, and it is now tier-wide

The filing seat's `SHARE-8` note predicted its own trigger: *"Two conventions, one trap — if a third
appears it is probably a tier-wide note rather than three local ones."*

**A third appeared in the same packet pair, and it is `R1`.** The shape:

> **A general rule you correctly satisfy has an exception carved out by a narrower type — and
> following the general rule silently launders the one shape the exception exists to exclude.**

| # | the general rule satisfied | the carve-out | what silence costs |
|---|---|---|---|
| 1 | `GUIDE-ENTITY-WORKBENCH-APP` §5.4 rule 3 | — | non-conformant for two months |
| 2 | V7 §2.6 MUST-ignore unknown fields | `SHARE-8` — a publication carrying `audience` is invalid | the excluded shape enters as an ordinary row |
| 3 | **`embed-payload` admits three arms** | **`R1` — a site asset admits two** | **a decoder reusing the shared union accepts `child` and reports nothing** |

**Instance 3 is the one that generalizes the rule past "unknown fields":** the laundering vector here
is not MUST-ignore at all, it is **an imported type being wider than the importing field**. That is a
new shape of the same failure, so the note belongs at the tier and not in a third convention.

**Enforcement point:** the row lands in `GUIDE-APPLICATION-DEVELOPMENT` §3's binding table — the
document an author of a *sixth* convention actually reads — with the same reasoning §3's note already
gives for the two rows added 2026-09-09: *"an author reads this document and their own draft, not a
peer's §7.5."* Each carve-out keeps its own refusal and its own named check; §3 carries the pointer.

---

## §4 What is NOT ruled here

- **Whether the shared payload union is one type in an implementation's binary is not ours** — that is
  an implementation-structure question and the filing seat has already sized it correctly. `R1` decides
  only that `child` is legal in the union and refused at the site layer, which is the arrangement they
  proposed.
- **`app/share/follow`'s `strategy` vocabulary stays NOT LOCKED** (SHARE §5) — untouched.
- **Nothing here is a build-state claim about any implementation.** The reference read noted in §2 is
  pinned and dated in the internal record, and is evidence for the core-spec citation — never a
  conformance assertion about anyone.

## §5 Version disposition

Both are implementation findings against text that contradicted itself, folded in place with a rev
bump on each, because both move a normative half: `APP-CONVENTION-SEMANTIC-CONTENT-SITE` **0.5 → 0.5.1**,
`APP-CONVENTION-SHARE` **0.2 → 0.2.1**. The roadmaps and `GUIDE-APPLICATION-DEVELOPMENT` §5's members
table are copies of those headers and are swept in the same session (`spec roster`).
