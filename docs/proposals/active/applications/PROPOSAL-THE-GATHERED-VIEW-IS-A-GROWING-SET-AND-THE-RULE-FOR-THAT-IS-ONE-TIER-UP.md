# PROPOSAL — a gathered view is a growing set, its address is a derive-to-meet value, and both rules already existed one tier up

**Proposes:** `APP-CONVENTION-FEED` v0.3 · `SYSTEM-DATA-EXCHANGE` v0.2
**Status:** DRAFT (2026-09-13) — written to be attacked. Four changes, one argument.
**Tier:** applications + system composition — `APP-CONVENTION-FEED` v0.2 → v0.3, `SYSTEM-DATA-EXCHANGE` v0.1 → v0.2.
**Rests on:** `SPECIFICATION-FORMAT` §8.4.5 / §8.4.6 (the hash-disposition rule) · `EXTENSION-REVISION` §3.1
(`prefix_hash`, the landed instance) · `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4 (the `pages` removal, v0.4.1) ·
`APP-CONVENTION-FEED` §4.2/§4.3 (the bounded head and its rules) · `SYSTEM-DATA-EXCHANGE` §1.2 (the layer split).

> **In one sentence.** The v0.2 fold widened a mirror's `subject` from *one thread* to *any timeline*
> and thereby made a mirror a view of an **unbounded, monotone** set — while leaving its shape a single
> entity holding one flat list, and leaving its address derived by a function this corpus does not
> define. **Both repairs are already written down elsewhere in this corpus**, which is the finding
> worth more than either fix.

---

## §0 Why this is one proposal and not four

Four defects arrived from two implementations in one week. They look independent and they are not: **every one of
them is a rule this corpus already states, in a document the author of §6 had no reason to open.**

| # | The defect | The rule already exists in | Found by |
|---|---|---|---|
| **1** | §6's mirror is an unbounded flat list over a set that only grows | `SITE` §4 (*"paging belongs to `.list` … never to a manifest field"*, v0.4.1) **and** `FEED` §4.3 rule 5, one section above it | an implementation, measured |
| **2** | §6.0.1's live key names `content_hash(absolute-path)` — a one-argument function | `SPECIFICATION-FORMAT` §8.4.6 (derive-to-meet) + `EXTENSION-REVISION` §3.1 (`prefix_hash`, the identical input) | an implementation |
| **3** | §6.0 says the live subject is *"a prefix"* and names none | `REFERENCE` §2.2's live-ref semantics require a **resolvable path**, not a prefix | an implementation |
| **4** | §2.4 declares `? cursor: content-hash`; §4.4 `[MUST]`s `{page, applied}`; our own ruling said *"do not build `{page, applied}`"* | — this one is ours alone, and it is a wrong ruling, not a missing rule | an implementation, and the triangle is real |

⇒ **Three of four are lookup failures, not design failures.** The design was right and was written
without reading three sections that already governed it. §5 proposes the one structural change that
makes the next instance findable.

---

## §1 The mirror is a growing set and has no growth rule — and the corpus states that rule twice

### 1.1 What was measured

The browser implementation, in a 2026-09-13 audit of its own publish path, from a tree published against the real projector:
a 4-pin index page body is **475 bytes**, i.e. **~119 bytes per pin**. §6's mirror is
`entries: [* reference]` in **one entity**, with no head and no pages, and §1.3 makes it **monotone**.

| the author mirrored has | the mirror record is | the author's own leg transfers |
|---:|---|---|
| 5,000 entries | ~0.6 MB in one entity | one page |
| 50,000 entries | **~5.9 MB in one entity** | one page (~4 KB) |

**So §6.2's cost argument inverts.** That section sells the mirror as the cheap leg — *"one signed-root
check instead of 500"* — which is true about **root checks** and silent about **bytes**. At scale the
leg that exists to be cheaper than going to the author directly transfers three orders of magnitude
more than going to the author directly.

> **What is holding it up today is not a bound.** The one shipped gatherer hardcodes a 256-entry gather
> limit, so the largest mirror anyone can currently emit is ~30 KB. **A limit chosen for a different
> reason is not a bound**, and it is in an implementation, not in the format.

### 1.2 The rule exists, twice, and neither statement is where §6's author would look

**`FEED` §4.3 rule 5, one section above §6:**

> **[MUST NOT]** A producer **MUST NOT** emit an unbounded page, and a reader **MUST NOT** assume any
> page size.

**`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4, which removed exactly this field at v0.4.1:**

> *"This avoids the 'download the index' anti-pattern at scale (huge-index → index-of-indexes).*
> **Paging belongs to `.list` — it is inherently incremental — never to a manifest field.**"

**A mirror record is a manifest field by that definition.** SITE sketched `pages: [* page-ref]`, caught
it, removed it, and wrote down why — in a sibling convention that the feed convention's §6 has no
reason to cite. **The rule was landed, correct, and unfindable from where it was needed.**

### 1.3 Why rule 5 does not already cover it, which is the part worth being precise about

Rule 5 governs **an author's index page**. A mirror is not an index page and is not the author's. The
two are related by *the reader's cost*, which is not a relation any gate or grep can see. **This is the
equivalence-collapse shape:** a rule that holds for the issuer's own object
silently fails to cover the same object assembled by a third party, and the tests assert the issuer's
convention.

⇒ ***The v0.2 fold is what created the defect.*** Before it, a mirror's `subject` was **pinned only** —
one entity, a thread. v0.2 widened it to `any-reference` so a mirror could be of a **timeline**, which
is `§1.3`-monotone and unbounded by construction. **The widening moved a bounded object into an
unbounded class and no constraint was re-scoped with it.** **A widening that moves an object into a new class obliges a re-read of every constraint that
keyed on the old one** — and this one was committed by the same fold that widened it.

---

## §2 D27 — the mirror gets the shape §4.2 already defines

**The repair is not a new mechanism. It is the mechanism one section up, applied to the object that
needed it, and every implementation that has a feed index already has this code.**

```cddl
feed-mirror = {                      ; type = app/feed/mirror — at app/feed/mirrors/{coordinate}
  type: "app/feed/mirror",
  data: {
    subject:     any-reference,      ; what this view is OF — §6.0
    current:     uint,               ; the highest mirror page in use
    ? oldest:    uint,               ; the lowest still published (default 0)
    gathered_at: uint,               ; when this gatherer last extended the view
    gathered_by: peer-id
  }
}

feed-mirror-page = {                 ; type = app/feed/mirror-page
  type: "app/feed/mirror-page",      ; at app/feed/mirrors/{coordinate}/{page} — decimal uint
  data: {
    page:        uint,               ; this page's own number — MUST equal its key
    entries:     [* reference],      ; PINNED, always (§6.0) — bounded (§6.3 rule 1)
    updated_at:  uint
  }
}
```

**`entries` leaves the record; `current`/`oldest` replace it.** The mirror record becomes fixed-size —
a subject, two integers and two scalars — whatever the size of the view it heads.

### 2.1 Four rules, and only the last is new thinking

> **[MUST NOT]** A gatherer **MUST NOT** emit an unbounded mirror page, and a reader **MUST NOT**
> assume any page size. *(§4.3 rule 5, verbatim, now stated over the object it needs to cover.)*

> **[MUST]** Mirror pages are **addressed by key, never chained by hash**, and are **never renumbered,
> merged or compacted.** *(§4.3 rules 1 and 3.)*

> **[MUST]** A mirror page's `page` field **MUST** equal its key. *(§4.2's index-page rule.)*

> **[MUST] Mirror pages are filled in GATHER order, and a page is sealed when its successor opens.**
> A gatherer appends what it newly holds; it **MUST NOT** rewrite a sealed page to insert an entry it
> discovered later — that entry goes on the current page.

### 2.2 Why gather order, and not the author's order — the one genuinely new clause

**An author's index is append-mostly**: a new entry goes on the current page and old pages move only on
edit. **A gatherer backfills** — it discovers older entries after newer ones, routinely, because that is
what gathering is. A mirror paged in *the author's* order would rewrite old pages on every gather round,
which is precisely the archive-republishing cost §4.3 rule 1 exists to prevent, transplanted onto the
peer that can least afford it.

**Gather order costs nothing and buys the reader loop.** Three arguments, and the third is the one that
decides it:

1. **Ordering is not the mirror's job.** §6.2 already says a reader *"orders their answers by the
   author's own succession … never by which mirror answered first."* A mirror's page order was never
   going to be rendered.
2. **Monotone append matches §1.3.** A sealed page is immutable, so a reader that has read page 7 never
   has to re-read page 7 — which is what makes a mirror cacheable at all.
3. ⭐ **Gather order is exactly the order a source leg is read in.** §6.2's whole case for the mirror is
   *a reader following 500 authors pays one check instead of 500.* That reader's question is
   **"what do you have that I have not seen?"** — and in gather order that is *read down from `current`
   to your cursor and stop*, which is §4.3 rule 4's `O(new)`. **In the author's order it is not
   expressible at all.** The cheap-leg argument only becomes true under this clause.

> **The cost, named rather than discovered.** A reader who wants *the author's newest 50* from a mirror
> must read the mirror and sort, because gather order is not post order. That is the right trade: the
> mirror exists to be **a source**, and a reader who wants the author's own order has the author's own
> index, which is one signed-root check away and is authoritative for it.

### 2.3 What this costs an implementation

A small mirror is now two entities rather than one — one extra bind, one extra fetch. That is what the
author's index already costs for a one-post feed, and it is the price of the shape being uniform.
**Both seats that will implement this already implement `feed-index-head` + `feed-index-page`.**

---

## §3 D28 — the live key is `prefix_hash`, and this corpus has defined it since `REVISION`

### 3.1 The defect

**§6.0.1 today:**

| subject | key |
|---|---|
| **live** | `hex(content_hash(absolute-path))` — the path absolute, `/{peer}/{path}`, canonical UTF-8 |

`content_hash` in this corpus is the hash over the ECF encoding of an entity's `{type, data}`
(`ENTITY-CORE-PROTOCOL` §1.2). **A path is not an entity, so there is no `type` to supply**, and the
clause names a one-argument function that does not exist. An implementation reports two surviving
readings producing different bytes, and shipped one of them — pinned to a literal computed with
`hashlib` rather than with the function under test, which is the correct discipline for a guess.

⭐ **This is not an ordinary ambiguity, because the key IS `FEED-12`'s comparand.** That vector asks two
gatherers of one author to derive the same address. **A seat cannot reach it by reading the text.**

### 3.2 The answer is landed, classed, pinned, and implemented at the core tier

**`SPECIFICATION-FORMAT` §8.4.6 defines the class and names the instances:**

> **Derive-to-meet — pinned to the ECFv1-SHA-256 floor (`0x00`).** You **compute** the hash from an
> agreed input in order to construct a path, key, or token that **another party must independently
> compute and match**. … *(`SIGNALING` §3.1 rendezvous key; `REVISION` §3.1 `prefix_hash`; `{peer_id_hex}`
> in `ROLE`; `{contact_id_hex}` in `IDENTITY`; and the `system/peer` identity entity itself.)*
>
> **A specification introducing a hash MUST state which disposition it is.**

**§6.0.1 introduces a derived hash and states no disposition.** It is in breach of a landed MUST in the
corpus's own authoring standard — which is the cheap, mechanical statement of what went wrong.

**And `EXTENSION-REVISION` §3.1 already defines the function, over the identical input:**

```
prefix_hash(prefix) = hex(content_hash(type="system/tree/path", data=prefix))
```

with the disposition stated, the floor pinned, the 66-character width derived from the format rather
than asserted, and the reason given: *"both sides compute `{H}` independently from the same prefix
string, so a home-format derivation would have two conformant peers construct different paths for the
same prefix and never meet, with nothing failing loudly."* **That is `FEED-12`'s failure mode, written
out three specs away, before this convention existed.**

### 3.3 The change

> **[MUST]** A mirror's coordinate is **derive-to-meet** (`SPECIFICATION-FORMAT` §8.4.6) and is pinned
> to the ECFv1-SHA-256 floor.
>
> | subject | coordinate |
> |---|---|
> | **pinned** | `hex(hash)` — the referenced entity's own content hash, hold-and-fetch, used verbatim |
> | **live** | `prefix_hash(path)` = `hex(content_hash(type="system/tree/path", data=path))`, `path` absolute — `EXTENSION-REVISION` §3.1 |

**The implementation's wrapper convention was right and their digest input is wrong.** They
independently derived the 66-hex `00`-prefixed shape — which is the floor-format spelling — and then
hashed raw UTF-8 where the corpus hashes the ECF encoding of a typed entity. **One function moves**, as
their packet predicted, and their pinned literal is re-pinned with it.

> **On their argument for the bare digest — *"the ECF reading needs a type tag that would itself have to
> be agreed"* — the premise is false and could not have been checked from where they stood.** The tag
> exists, is `system/tree/path`, is landed in `REVISION` and `QUERY`, and is implemented by three core
> peers. **It is not a new vocabulary item and is not a `spec vocab` finding.** *(The standing rule: before ruling a
> cross-implementation semantic, search every tier for an implementation that already has it. The party
> that could not run that search is not the party that owed it.)*

---

## §4 D29 — the live subject names `app/feed/index`, and *"a prefix"* is the wrong word

### 4.1 The defect

§6.0 calls the live subject *"**a prefix** one peer owns — an author's feed"* and names none.
`FEED-12`'s own row says *"a live reference to an author's **feed prefix**."* **Two gatherers agreeing
on the hash function still derive different keys if they disagree about the string.**

### 4.2 *"A prefix"* is not merely under-specified — it is incompatible with the atom

**A live reference's `path` is resolvable, and the whole of `REFERENCE` §2.2.2 depends on it.** That
table has four rows — *resolves and matches*, *resolves and differs*, *404 with `seen`*, *404 without* —
and a `seen` field that is *"an expectation"* about what is **at** the path. **A prefix resolves to
nothing**, so a live-ref naming one has no row in that table, and its `seen` field is meaningless.

⇒ **`app/feed/` is not a legal live-ref target.** The subject must be a path that resolves.

### 4.3 The change — and it is the only candidate that survives

> **[MUST]** The live subject of a **timeline** mirror is a live reference to
> **`/{peer}/app/feed/index`** — the author's index head (§4.2).

Three properties, and no other candidate has all three:

1. **The convention pins it by hand** (§4.2, *"two pinned paths"*), so a second seat computes it from
   the peer id alone. An entry prefix is implementation-chosen — §2's *"the cross-impl contract is the
   type tag, not the path"* — so a key derived from one **can never be computed by another seat**,
   which is fatal for `FEED-12` specifically.
2. **It resolves**, to the author's own entry point, so §2.2.2 applies and `seen` means something.
   ⭐ *A reader that decides to leave the mirror and go to the author has already fetched the thing it
   needs* — the subject is not merely a name, it is the fallback.
3. **It does not move when the author posts.** The head's *hash* changes on every post; its **path** does
   not.

> **This is the repair §6.0's own `[MUST NOT]` implies, not a contradiction of it.** That clause forbids
> a subject that is *"a pin to a value that moves"* — the head's **hash**, a witness masquerading as an
> identity, and underivable besides. A **live reference to the head's path** has neither defect.
> An implementation made this argument and it is adopted verbatim.

**§6.0's table wording changes from *"a prefix one peer owns"* to *"a resolvable path one peer owns —
for a timeline, the author's index head."***

---

## §5 D30 — the cursor triangle, which is ours, and the ruling was the wrong corner

### 5.1 The three statements, all live at `5026073`

| where | what it says |
|---|---|
| `FEED` §2.4 CDDL | `? cursor: content-hash` — a bare hash on the **published follow record** |
| `FEED` §4.4 | `[MUST]` *"The cursor is `{page, applied}` … a cursor that cannot survive that is a cursor that breaks on edit"* |
| arch, `ROUTING-2026-09-11-g` §143 | *"`A-36` **SUPERSEDED** … the position leaves the declaration entirely. **Do not build `{page, applied}`.**"* |

§2.4's declared field **cannot express** §4.4's `[MUST]`. An implementation built §2.4 because
it is the declared one, read the ruling's last sentence as *do not build a two-part cursor at all*, and
**built no cursor of any kind** — so §4.3 rule 4's `O(new)` is unimplemented, and every poll of a
followed author re-reads the newest 50 entries forever (measured: ~103 signed-root re-fetches per poll,
per author).

### 5.2 The ruling's sentence is the defect

*"Do not build `{page, applied}`"* was written about **reshaping the declared wire field**, and it does
not say so. Read alone — which is how a ruling is read — it forbids the mechanism. **The harmful reading
is the natural one**, and it cost a seat its entire reader loop.

⇒ **Arch owns this one outright.** It is not a spec ambiguity that a seat mis-resolved; it is a ruling
that said less than it meant, in the direction that removes a `MUST`.

### 5.3 The change — the triangle gets a consistent corner

1. **§2.4: `? cursor: content-hash` is REMOVED** from `app/feed/follow`. The position is **local reader
   state** and is not published. *(This is what `[OPEN-FEED-1]` resolved; the field's removal is the
   half that never landed.)*
2. **§4.4 stands, and gains one sentence:** *"The cursor is **local reader state**. Nothing in this
   convention publishes it."* `{page, applied}` and the resume-from-`page` `[MUST]` are unchanged and
   are the reason rule 4 is implementable.
3. **§12 `F-1` is closed** — it exists to flag that §2.4 carries the field pending a better home. The
   field is gone; a general reader-loop mechanism may still arrive and now has nothing to displace.
4. **`FEED-R14` is unchanged.**

---

## §6 D31 — the promotion, which is the finding rather than the fix

**Everything above repairs one convention. The class will recur, and §1.2 is the evidence: this corpus
has now written the bounded-collection rule twice, in two app conventions, neither citing the other,
and a third convention made the mistake anyway.**

**`SYSTEM-DATA-EXCHANGE` exists for exactly this** — it was created one week ago to hold the four
republication rules *"because a second convention shipping a gathered view would otherwise inherit the
closure property without inheriting the rules that make it true."* **The growth rule is the sibling of
that sentence and it fails the same way**: a convention inherits *"a gatherer's output is consumable by
another gatherer"* and does not inherit *"and it must still be readable when it is large."*

> **Proposed `SYSTEM-DATA-EXCHANGE` §2.5 — the growth rule.**
>
> **[MUST]** A republication format whose membership **grows with participation** MUST NOT carry its
> members as an unbounded collection in a single entity. It MUST be a **bounded head plus
> key-addressed pages**, pages MUST NOT be renumbered, merged or compacted, and a reader MUST NOT
> assume any page size.
>
> **The test, and it is one question:** *does this collection grow with how many parties participate, or
> with something the format's author controls?* If the first, it is unbounded and it is paged.
>
> **Why this is at this tier.** The cost is the **reader's**, and it is invisible from the format that
> causes it: a flat list is correct, verifiable, cheap to write, and passes every check at the size its
> author tested. It becomes a defect only in someone else's fetch, at a size no fixture has.

### 6.1 The census, published with the surface it ranges over

Every collection field in `specs/` that could be an instance, **including the ones that pass**:

| type · field | grows with | verdict |
|---|---|---|
| `app/feed/mirror` · `entries` | **how much a gatherer gathered** | ⛔ **INSTANCE — unbounded, monotone (§1.3), live.** Repaired by §2 |
| `app/site-manifest` · `pages` | how many pages a site has | ✅ **caught and REMOVED at SITE v0.4.1**, with the rule stated |
| `app/feed/index-page` · `entries` | — | ✅ bounded by §4.3 rule 5 |
| `system/encrypted` · `wrapped_keys` | group membership | ✅ **declared bounded** — ENCRYPTION §8.1 scopes group mode to *"static groups only"*, and sends churn to a future extension |
| `system/peer/transport-set` · `profiles` | a peer's own transports | ✅ bounded by the publisher, not by participation |
| `system/registry/binding` · `transports` | a target's own addresses | ✅ same |
| `system/role/exclude-result` · `revoked_token_hashes` | one sweep's result | ✅ bounded by the operation |
| `app/site-manifest` · `nav` | author curation | ✅ authored, and SITE §4 says so explicitly |

**One live instance, one previously caught, six correctly bounded.** *(Surface: every `array_of` /
`[* …]` field in `specs/`, 8 candidates after excluding operation parameters and results.
`grep -rn "array_of\|\[\* " specs/`.)*

### 6.2 The enforcement point, because a rule without one does not count

**`SPECIFICATION-FORMAT` §8.4.6 already carries the model:** *"A specification introducing a hash MUST
state which disposition it is … this converts 'audit indefinitely for new instances' into a checkable
property."* **The same move applies here, and it is why §6's rule is phrased as a question rather than a
prohibition:** a spec introducing a collection states which side of the test it is on. That is
greppable, and it is a `spec` analyzer of the same shape as `declare` — proposed, not built, and named
as owed rather than claimed.

---

## §7 What this proposal does NOT do

- **It does not touch §6.1's four republication rules or §6.2's purpose.** The mirror's *meaning* is
  unchanged; only its shape and its address move.
- **It does not specify gatherer discovery.** §6.0.1 makes a mirror's address derivable once you know
  **whose** mirror you want, and nothing in §6 names the gatherers. That is open, it is known, and
  `DESIGN-REGISTER` `F-16a` argues it is narrower than it reads — a gatherer's output is an ordinary
  entry in an ordinary stream, so forward traversal finds it. **Not folded, not closed.**
- **It does not measure the 250×.** Neither seat has walked 500 follows. §6.2's one-check-per-gatherer
  derivation is **derived, not measured**, and §2.2's argument for gather order is likewise derived.
- **It does not rule on a mirror page's size.** §4.3 rule 5's *"a fixed number is deliberately not
  specified"* carries over verbatim and for the same reason.
- **It is not a cross-impl run.** `FEED-12` has one seat, one derivation and a literal vector; it needs a
  second gatherer, which is what §3 unblocks.
