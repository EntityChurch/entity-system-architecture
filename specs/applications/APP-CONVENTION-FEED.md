# APP-CONVENTION-FEED — the entry, the index, the collection and the mirror — v0.1 DRAFT

**Version**: 0.1
**Status**: Draft
**Domain:** `applications/` (fifth member).
**Kind**: normative-spec · **Authority**: binding · **Governed-by**: `guides/GUIDE-APPLICATION-DEVELOPMENT.md` — FORMAT-only (§2.1).
**Depends:** `ENTITY-CORE-PROTOCOL.md` §1.2 (content hash) · §1.4 (paths) · §1.5 (`PeerID`) · §3.5
(`system/signature`, the invariant pointer and its forwardable property) · `EXTENSION-TREE.md` §3.1
(the trie's per-change cost), §3.3a (`published-root`), §9 (canonicalization) ·
`APP-CONVENTION-REFERENCE.md` §2 (the reference atom) · `APP-CONVENTION-EMBED.md` (entry bodies).

> **What this document is.** A vocabulary for *a thing someone posted* — four content shapes and one
> subscription record — so that following people across independent hosts and reading what they
> published is a format two independent implementations can both produce and both read. It adds **no**
> kernel feature, **no** required SDK surface, **no** new authorization path and **no** wire form
> (GUIDE-APPLICATION-DEVELOPMENT §2.2; proposal-first). Everything it needs from below already exists.

---

## 1. Three properties, and everything else is derived from them

An implementer holding these three can re-derive most of this document.

### 1.1 The author is the namespace, and authorship is carried by a detached signature

**[MUST]** An entry's `author` **MUST** equal the peer namespace it is authored under. An entry found
under one peer's namespace claiming a different author is **invalid**, and a conformant reader **MUST**
reject it.

**[MUST]** An entry **MUST** be individually signed. The signature is a **separate** `system/signature`
entity at the core protocol's invariant pointer path — `/{author}/system/signature/{hex(entry_hash)}` —
carrying `target = entry_hash` and `signer = author`.

> **The signature is not part of the entity, and this is the point most likely to be implemented
> wrongly.** An entity is `(type, data)` and its content hash; **no field of it names a signer**, and
> anyone holding the bytes reconstructs an identical entity — which is exactly why bytes cannot
> establish who wrote them. Verification in this substrate is normally **root-anchored**: you walk from
> a publisher's signed root, and inclusion in that tree is what makes the content theirs. **That works
> while you are reading their tree and not at all once an entry travels**, which is the whole of §6.

**Two mechanisms bind an entry to an author. They prove different things. This convention requires the
first and permits the second.**

| | **Detached signature** (required) | **Inclusion proof** (optional) |
|---|---|---|
| Proves | *this author signed these bytes* | *this author had these bytes at this key, under root R* |
| Costs | one small content-addressed entity, per entry | a chain of trie nodes, proportional to tree depth, per entry |
| Travels | yes — content-addressed, dedups, verifiable offline from bytes alone | yes, but anchored to one root |

**The required one is the signature, and the reason is §1.2's reason: the unit of addressing should
also be the unit of verification.** An entry that travels alone should verify alone, in `O(1)` extra
objects — without its author's tree, without their origin, and without a root that may be many
publishes stale. A mirror carrying fifty entries by twelve authors carries fifty signatures, every one
content-addressed and deduped against whatever the reader already holds.

**The inclusion proof is permitted and is not redundant.** It answers a question the signature cannot:
*was this in their published tree, as of sequence N?* That is the anti-omission question, and a mirror
asserting *"and here is proof they published it"* makes a stronger and different claim. It is optional
because it costs depth-many nodes per entry and most readers do not need it.

**What this rule buys:** you do not hold anyone else's key, so you cannot author in their name; you
cannot alter a byte without the signature failing; and **an entry stays verifiable when its author is
offline, when their origin is gone, and when it reaches you from a stranger.** *The only operation
republication permits is carrying what they already chose to publish.*

#### 1.1.1 This is an obligation on the COMPOSER, and it has a permanent boundary

**[MUST]** Whatever authors an entry mints its signature **at authoring time** — not at publish time
and not at mirror time. **A mirror cannot supply it**; a mirror can only carry a signature the author
already minted.

The cost is **per-entry-once, not per-publish**, and the signature entities dedup.

> **Entries authored before an implementation adopted this rule can never be attributed once
> mirrored**, because nobody can retroactively mint a signature they did not make. **That is a
> permanent boundary in the data, not a migration window.** A reader will encounter unattributable
> older entries indefinitely and **MUST** present them as unattributed rather than guessing — see
> §6.1 rule 3.

#### 1.1.2 The scope of this rule — it is about an entry, not about the system

**This rule scopes the entry.** It is the anti-forgery rule for a **singly-authored** object and it is
correct for every shape this convention defines. **It is not a statement that application data has one
signer**, and it MUST NOT be read as one: the substrate says the opposite, binding each signature as a
separate entity in the signer's own namespace, so one entity can carry N signatures from N namespaces,
and K-of-N verification over an arbitrary entity is already available.

**A record two or more parties co-sign** — a co-authored post, a mutual follow read as a *relationship*
rather than as two independent follows, a confirmed invitation, a receipt, an order — **is a different
shape and is not specified here.** It is reserved deliberately: read §1.1 as a system-wide property and
that shape arrives as a contradiction to migrate around instead of as an addition.

### 1.2 One entry is one addressed unit — entries reference, they never embed

**[MUST NOT]** An entry **MUST NOT** contain another entry's bytes. What it replies to, what it quotes,
what it attaches and what it is part of are all carried as **references** (§2.2), never inline.

**This is the granularity rule, and it is what makes deduplication real rather than nominal.** Content
addressing does not deduplicate anything by itself — two peers who never share a store hold two copies
of identical bytes, and that is ordinary replication. What content addressing gives is that **the two
copies have the same name**, so:

- a reader that already holds an entry pays **nothing** to receive it again from a second source, and
  may decline to fetch it at all;
- two mirrors of one conversation, assembled independently by people who never met, hold the **same**
  entries at the **same** hashes, so merging their views is **set union with no reconciliation**;
- and **a reply costs the size of the reply.** Commenting on something does not copy the thing being
  commented on.

**Every one of those fails if an entry inlines what it points at**, because an inlined copy gets a new
hash inside a new parent and is no longer the same object to anybody. **A conformant producer MUST NOT
inline the body of a referenced entry** — not in an entry, and not in a mirror (§6.1). Where a renderer
wants to show quoted text, it resolves the reference and renders what it resolved.

> **The honest scope of the benefit.** Deduplication is real **within a store** and across everything
> that store holds; between two peers with no shared store it is replication, and the win is the shared
> *name*, not shared bytes. That is the useful case anyway — the redundancy that matters is many people
> holding the same conversation, and they each hold it once.

### 1.3 The set only grows

Entries are immutable and content-addressed, and **there is no delete**. A revision is a **new** entry
referencing the old one, and *which one a reader displays* is an interpretation question, never a set
question.

The consequence a reader is built around: **a partial view is never a wrong view, only a short one**,
and every additional source can only lengthen it. This is what makes §4's index an optimization rather
than an authority, what makes §6's mirror unable to lie except by omission, and what makes §7.1's
staleness rule sound.

---

## 2. The vocabulary

Four content shapes plus one subscription record. **The cross-impl contract is the type tag, not the
path.** Cross-peer aggregation is a `type_filter` query over the universal tree with no peer filter, so
the tag *is* the index key — and a tag placed under one application's own prefix makes two
implementations mutually invisible even with a perfect mirror between them.

| Type | Role | Assembled by |
|---|---|---|
| `app/feed/entry` | **one thing someone published** — the atom | its author |
| `app/feed/index-head` | the stream's entry point: which page numbers are in use | its author |
| `app/feed/index-page` | one **key-addressed** page of the author's stream, newest-first within the page | its author |
| `app/feed/collection` | the author's own **bounded, complete, authored-order** set — album, playlist, portfolio | its author |
| `app/feed/mirror` | a **gathered** view of other people's entries, claiming nothing | a reader |
| `app/feed/follow` | a reader's durable subscription to a peer's feed | the reader, privately |

**Four content shapes, and it terminates.** The three collection-shaped types are not subject-matter
categories — they are the three answers to *"who assembled this list, and what does it therefore
claim?"*, and there is no fourth answer. **A new product is a new body type (an embed handler, and that
extension point is open and unbounded) or a new renderer.** It is a new **entity type** only if a
conformant consumer must *behave* differently, and the burden is on whoever proposes one to name the
behaviour. A blog post, a video post, a photo post, a comment and a forum post are all the first row.

### 2.1 Shared atoms — IMPORTED, not defined here

**`APP-CONVENTION-REFERENCE` §2.1 is the single home for these.** Restated so this document reads
standalone; on any disagreement that document is the authority.

```cddl
content-hash = bstr   ; self-describing (format_code, digest) per V7 §1.2. The leading varint is the
                      ; content_hash_format and THE DIGEST LENGTH FOLLOWS THE CODE. Never fixed-width
                      ; (SPECIFICATION-FORMAT §8.4.5).
peer-id      = tstr   ; V7 §1.5 Base58 peer-id — a TEXT string, not bytes
tree-path    = tstr   ; absolute or peer-relative per V7 §1.4
```

**No fixed-width hash form appears anywhere in this document.**

### 2.2 The two references — pinned and live

**IMPORTED from `APP-CONVENTION-REFERENCE` §2.1**, which is the authority. Reproduced for readability:

```cddl
reference      = pinned-ref          ; "THIS EXACT THING" — the pin, and the default
live-reference = live-ref            ; "WHATEVER IS AT THIS PLACE NOW"
any-reference  = entity-ref          ; only where a site below declares it takes both

pinned-ref = { tag: "pin",  peer: peer-id, hash: content-hash, ? at: anchor, ? via: [* hint] }
live-ref   = { tag: "live", peer: peer-id, path: tree-path, ? seen: content-hash,
                                                            ? at: anchor, ? via: [* hint] }
```

**There are two reference intents, they demand different consumer behaviour, and so there are two
shapes.** A **pin** names *these exact bytes* — self-verifying, satisfiable by anyone holding them, and
its answer can never change. A **live** reference names *whatever is at this address now* — only the
publisher is authoritative for it, and the answer is expected to change.

**Two shapes rather than one shape with an optional hash**, because under an optional hash a reference
arriving without one is indistinguishable between *the author wants the live version*, *the author's
implementation did not populate it*, and *the author only ever had an address* — one intent and two
defects, with nothing to tell them apart.

#### 2.2.1 Each site declares which atom it accepts, and the pinned sites do not widen

| Site | Accepts | Why |
|---|---|---|
| `reply.root`, `reply.parent` | **`reference` only** | the pin is the reason this field exists — a reply must not become as trustworthy as whatever currently answers a location, and a parent must not be editable underneath its replies |
| `prev` | `content-hash` — unchanged | an append-only commitment to a *specific* predecessor is meaningless against a moving target |
| `context` | **either** | *"part of a topic"* is often a maintained index; *"part of this event"* is often a fixed entity |
| `attachments` | **either** | an attached file is usually pinned; an attached *living* document is the case the live shape exists for |

#### 2.2.2 Resolving a live reference — the comparison result is information, not an error

Because `path` is authoritative and `seen` is only an expectation, resolution has more than two
outcomes, and **a reader MUST be able to tell which one it got.**

| # | `(peer, path)` | vs `seen` | Meaning | Reasonable behaviour |
|---|---|---|---|---|
| 1 | resolves | matches, or `seen` absent | you are seeing what the linker saw, or they offered no expectation | render |
| 2 | resolves | **differs** | **the document evolved** — the ordinary case | render current, **and surface that it moved**; the pinned version remains fetchable |
| 3 | **404** | — | the path moved or was unpublished | **fall back to `seen`, fetched from the publisher's declared content origin or any reachable source that has it** — the hash validates the bytes whoever serves them |
| 4 | 404 | `seen` absent or unobtainable | genuinely dangling | the honest failure. Nothing to hide |

**Row 3 is where the second naming layer earns its keep:** a content hash resolves against *any* store,
so **a live reference survives its author unpublishing the path**, with no link database anywhere.

> **[MUST]** The normative half is **the reader's ability to tell**, not which policy it picks. Strict
> (refuse on mismatch) and lenient (render current) are both legitimate and are the reader's choice.
> What is required is that **a view built from a live reference whose resolved hash differed from
> `seen` MUST make that fact available to the view.** *A view that names its own provenance is
> debuggable; one that does not is indistinguishable from a bug.*

### 2.3 `app/feed/entry`

```cddl
feed-entry = {                       ; type = app/feed/entry
  type: "app/feed/entry",
  data: {
    author:      peer-id,            ; MUST equal the authoring namespace (§1.1)
    created_at:  uint,               ; ms since epoch, author's clock — a DISPLAY HEURISTIC (§2.3.2)
    body:        embed-node,         ; APP-CONVENTION-EMBED — this convention defines no content types
    ? reply:     { root: reference, parent: reference },   ; present => this entry is a reply
    ? context:   any-reference,      ; what this is PART OF — never who it is for
    ? prev:      content-hash,       ; OPT-IN append-only commitment (§2.3.1). NOT navigation.
    ? attachments: [* any-reference] ; referenced, never inlined (§1.2)
  }
}
```

**`body` is an embed node and this convention defines no content types of its own.** A photo post and a
text post are one shape with a different embed inside.

> **`body` is an `embed-node` — `APP-CONVENTION-EMBED` §3.1, the INPUT surface, carried INLINE.** It is
> **not** `EmbedOutput` (EMBED §4): an entry stores what was **authored**, and the handler runs at the
> **reader**. Typing it as the output surface would fix the rendition choice at authoring time for every
> reader forever, leave EMBED §5's handler dispatch nothing to run, and — because §4's vocabulary is
> closed by design — publish a wire format that can never carry a content kind that vocabulary did not
> anticipate.
>
> **No separate `Embed` ENTITY is required for an entry**, and this is the practical consequence: a text
> post is an inline payload, an image post is a pointer payload, and both are one field of one entity.
>
> ```
> text post    { type: "app/embed/text/plain",
>                data: { payload: { tag: "inline", bytes: <utf8> }, fallback: "…" } }
> image post   { type: "app/embed/image/png",
>                data: { payload: { tag: "pointer", hash: <content-hash> },
>                        fallback: "…", renditions: [ … ] } }
> ```
>
> **Only the `child` arm names a separate entity** — that arm is transclusion, and it is optional.

**`reply` carries `root` as well as `parent`, and the second field is load-bearing.** With `parent`
alone, assembling a conversation is a hop-by-hop walk and **one unreachable author truncates everything
below them**. `root` lets any holder of any entry name the whole conversation in one step, which is what
makes a partial view assemblable and a mirror findable.

**A conversation needs no genesis entity: the root entry *is* the conversation, and its hash is the
conversation's identity.**

**`context` is not `reply`.** `reply` says *this answers that*; `context` says *this belongs with that*
— a topic, a collection, an event. They are separate fields because collapsing them makes *"replied
to"* unrenderable.

#### 2.3.1 `prev` is an opt-in append-only commitment, and it is NOT how a reader navigates

> **An entry carrying `prev` is a claim that the author is publishing an append-only sequence** — that
> this entry follows that one and **nothing was removed between them**. An author who uses it is
> **giving up the ability to silently revise**, and a reader may treat a gap in such a chain as
> meaningful.

**It is optional, and making it mandatory would be a defect.** A required backward chain has one
property nobody wants: it makes every old entry **permanently referenced by a newer, still-published
one**. Remove the old entry and the newer one still names its hash, so the removal leaves a residue the
author cannot clear without re-signing the newer entry — which changes *its* hash, which cascades
forward to everything after it. **A chain converts a local edit into a republish of everything since.**
It also truncates: a reader walking backward stops dead at the first hash it cannot fetch, and cannot
tell removal from withholding.

**Two further costs, both structural:**

- **A mandatory chain is a single-writer commitment, so it is a one-device commitment.** One person with
  two devices then has one sequence and two writers, and publishing from the second either forks the
  chain or requires the devices to coordinate on every post. *(This is not a claim that multi-device
  publishing is solved here. It is not — two devices sharing one key and advancing one signed root is a
  race this corpus does not describe. The narrower point stands: making `prev` mandatory would make
  that open problem strictly harder, by adding a second ordering commitment on top of the root
  sequence.)*
- **A chain forces a deletion mechanism.** Events carrying backward pointers cannot be removed without
  breaking the graph, so the skeleton must be kept and the contents emptied — a redaction algorithm and
  a permanently unreclaimable placeholder. **That is §7.3's property lost.**

**So an author who opts into `prev` opts into those costs knowingly.** They gain *"I cannot have quietly
edited this"* and they give up clean removal, not only for the entry they remove but for everything
published after it. **That is the right trade to offer and the wrong one to impose.**

**Navigation is by key (§4), never by chain.**

#### 2.3.2 `created_at` is not an ordering authority

It is the author's own clock, it is unverifiable, and a peer may set it to anything.

**[MUST NOT]** A reader **MUST NOT** rely on `created_at` for correctness, and **MUST NOT** reject an
entry for an implausible timestamp. It is what a renderer shows and sorts by when it has nothing better;
the causal facts are `prev` (within one author) and `reply` (across authors), and both are in the data.

**This is deliberately weaker than it could be.** Any stronger claim requires either a clock nobody can
verify or a coordination step this design does not have, and the cost of getting it wrong — entries
silently dropped as *"too old"* or *"from the future"* — is worse than a display artifact.

### 2.4 `app/feed/follow`

```cddl
feed-follow = {                      ; type = app/feed/follow
  type: "app/feed/follow",
  data: {
    subject:    peer-id,             ; whose feed this follows — a NAMESPACE, not a record
    ? label:    tstr,                ; the follower's own petname; local, never authoritative
    ? via:      tstr,                ; the identifier as typed or scanned, for provenance display
    since:      uint,                ; ms since epoch — when this follow was created
    ? cursor:   content-hash         ; last index page this reader applied (§4.4)
  }
}
```

**This is a distinct type from `app/share/follow`, and the discriminator is the SUBJECT.**
`app/share/follow` follows a **grant** — one titled share record with an audience the publisher
authorized, which means **the publisher knows the follower exists**. `app/feed/follow` follows a
**namespace** — public, pull-only, requiring no grant and no permission, and **the publisher does not
know the follower exists.** Those are different mechanisms with different authorization models.

> **`label` is a petname: local, chosen by the reader, and never authoritative.** It is the answer to
> *"I cannot read a public key"* that requires no naming authority at all, because the name lives in the
> reader's own tree and is never transmitted as a claim about anyone. `via` records the identifier as it
> was typed or scanned, for provenance display only.

> **A follow record is the reader's private data.** Nothing in this convention publishes it, and nothing
> requires a publisher to learn who follows them. **Publishing a follow list is a separate, voluntary
> act** and is not specified here.

---

## 3. Reply notification is grant-scoped, and this convention does not change that

**A publisher does not learn that they were replied to.** Following is pull-only and requires no grant,
so a reply published in the replier's own namespace produces **no notification anywhere** by itself.

**[MUST NOT]** An implementation **MUST NOT** describe replies as notifying the author unless an
explicit delivery grant exists. Inbound delivery is granted, not ambient: there is no unsolicited
inbox in this system, which is the same property that means there is no spam route — **and *"replies
notify the author"* silently assumes exactly the open delivery grant this architecture declines to
have.**

**The two honest routes**, both ordinary and neither specified here: the author **polls** for entries
whose `reply.root` names one of theirs, or the replier holds a delivery grant the author issued.

---

## 4. The index — a bounded head plus key-addressed pages

### 4.1 What it is for

The primary social operation is *"what did they post that I have not seen?"* Verifying a **known**
region of a published tree costs the tree's depth — a handful of nodes. But **discovering what is new
under a prefix costs the whole tree**, because the trie is keyed by hash bits rather than by path
locality: keys that look adjacent live in unrelated leaves, and all of them must be read to find them.

**So the index is not a convenience added after the fact.** It converts the primary operation from the
expensive enumeration into a cheap known-key read, and it is therefore a prerequisite of the reader
loop rather than a later optimization.

### 4.2 The shape

```cddl
feed-index-head = {                  ; type = app/feed/index-head — at /{peer}/app/feed/index
  type: "app/feed/index-head",
  data: {
    current:    uint,                ; the highest page number in use
    ? oldest:   uint,                ; the lowest page number still published (default 0)
    updated_at: uint
  }
}

feed-index-page = {                  ; type = app/feed/index-page
  type: "app/feed/index-page",       ; at /{peer}/app/feed/index/{page} — page is a decimal uint
  data: {
    page:       uint,                ; this page's own number — MUST equal its key
    entries:    [* reference],       ; newest first within the page
    updated_at: uint
  }
}
```

**Two pinned paths:** the head at `/{peer}/app/feed/index` and pages at `/{peer}/app/feed/index/{page}`.
A reader holding no reference has to start somewhere, and a reader resuming has to jump straight to
where it left off.

### 4.3 The rules

1. **[MUST]** Pages are **addressed by key, never chained by hash.** A hash back-chain makes page *N*'s
   identity depend on page *N−1*, so editing one old page forces rewriting every page after it — the
   archive gets republished to remove one entry. With key addressing, rewriting page 12 changes page
   12's binding and nothing else: **`O(tree depth)`, the same cost as posting.**
2. **A reader jumps.** Holding a cursor at page 12, it fetches page 12 directly rather than walking the
   head backward to reach it.
3. **[MUST NOT]** Pages are **never renumbered, never merged and never compacted.** A page that loses
   an entry is a page with fewer entries. **Renumbering would invalidate every reader's cursor**, which
   is the one thing a publisher must not be able to do by accident.
4. **A reader fetches the head, reads down from `current` to its cursor, and stops.** Cost is
   **`O(new)`**, not `O(all)`.
5. **[MUST NOT]** A producer **MUST NOT** emit an unbounded page, and a reader **MUST NOT** assume any
   page size. **A fixed number is deliberately not specified** — the right value depends on entry size
   and publishing cadence, which vary by orders of magnitude between publishers. **What is normative is
   the shape, not the arithmetic.**
6. **[MUST NOT]** The index is an optimization and **MUST NOT** be treated as the authority. A reader
   that cannot fetch it falls back to enumerating the prefix — slower, same answer. **An entry absent
   from the index is still a valid entry**, and a reader encountering one by reference **MUST NOT**
   reject it for being unlisted.

> **Rule 6 is a floor obligation (`SPECIFICATION-FORMAT` §8.6) and it closes a hole:** were the index authoritative, it
> would be a place for a publisher to lie by omission about their own posts, and a reader would have no
> way to tell an omission from an absence.

### 4.4 The cursor

**The cursor is `{page, applied}`** — the page number the reader reached, and the newest entry hash it
took from that page.

**[MUST]** If `applied` no longer resolves, the reader **resumes from `page`**. This is why the cursor
carries a number and not only a hash: an author may remove the very entry a reader was holding as its
position, and **a cursor that cannot survive that is a cursor that breaks on edit.**

### 4.5 What this settles about ordering, and what it does not

**This document adds one semantic ordering contract and nothing more: `entries` within an index page is
newest-first, as ordered by the publisher.** The order is **authored**, not derived — it is whatever
sequence the publisher wrote, and a reader renders it in that sequence.

It is not a claim that the tree is sorted, it does not make `created_at` authoritative, and it does not
touch the content-site convention's determinism floor, which continues to govern raw enumeration views.

**A reader merging entries from several publishers has no cross-publisher order in the data**, and this
document deliberately does not invent one. Sorting a merged view is a presentation choice (§9.3).

---

## 5. The collection — the author's own, bounded, and complete

An album, a playlist, a portfolio and a curated reading list are one shape, and that shape differs from
§4's index on three axes at once — which is what makes it a type rather than a label.

| | the index (§4) | the collection |
|---|---|---|
| Bounded? | no — it grows forever | **yes; a finite thing with edges** |
| Complete? | never claims to be | **yes — this *is* the album** |
| Reader holds | a cursor, and stops at it | nothing; it fetches the set |
| Order means | recency | **the author's arrangement, which is content** |
| *"Show me all of it"* | not a sane operation | **the primary operation** |

```cddl
feed-collection = {                  ; type = app/feed/collection
  type: "app/feed/collection",
  data: {
    title:       tstr,               ; human-facing; NOT an identifier
    members:     [* reference],      ; ORDERED — the order is authored and is part of the content
    ? note:      tstr,
    ? cover:     reference,          ; one member, for presentation
    created_at:  uint,
    updated_at:  uint
  }
}
```

**[MUST]** A renderer **MUST** present `members` in the order given. **A collection's order is not
derived** from the tree, from names or from timestamps — it is a field, written by the author, and **a
collection re-sorted is a different collection.**

**A collection holds references, never bodies** (§1.2). A photo appears in three albums by being
referenced three times; the bytes exist once.

**Membership is not exclusive and is not authority.** An entry may be in any number of collections, in
its author's tree and in other people's, and being in a collection says nothing about who may read the
entry — that is the grant layer's question and this convention does not touch it.

**Why this is one type and not five.** Album, playlist, portfolio, pinned set and reading list demand
identical consumer behaviour — fetch the set, render in the given order — so by the taxonomy's own test
they are one shape with different bodies inside. `title` is what distinguishes them to a person, and a
renderer is free to present a collection of image entries as a grid and one of audio entries as a
queue. **That is presentation, and it is per-front-end by the tier standard (§2.2).**

> **Media needs nothing from this convention, and the gallery case is the cheapest thing here.** An
> image post is an entry whose body is an embed with a **pointer payload** — a content hash into the
> store — and the embed convention already specifies **renditions** so a consumer can select a size or
> format. Large immutable content-addressed blobs served from a static origin behind a cache is exactly
> what that infrastructure was built for: cacheable forever because the hash *is* the name, deduped
> across every collection and mirror that carries them, and **the publisher does no work per viewer.**

---

## 6. The mirror — assembly as a by-product of participation

```cddl
feed-mirror = {                      ; type = app/feed/mirror
  type: "app/feed/mirror",
  data: {
    subject:     reference,          ; the root entry this view is of
    entries:     [* reference],      ; what this mirror holds — republished, UNMODIFIED
    gathered_at: uint,
    gathered_by: peer-id             ; the key that assembled it
  }
}
```

### 6.1 Four rules, each closing a specific hole

1. **[MUST]** **Republish the original bytes.** Not a re-encoding, not a re-normalization, not a
   re-serialization through a local model. **A re-encoded entry no longer verifies against its author's
   signature**, which silently turns a mirror from evidence into hearsay. Combined with §1.2, a
   mirror's `entries` is a list of **references** and the bytes it carries are stored
   content-addressed — so a reader who already holds an entry from one mirror pays nothing to accept it
   from a second.
2. **[MUST NOT]** **A mirror cannot claim completeness, and the shape gives it no way to.** There is no
   `complete` field and there will not be one. The verifiable property is narrower and more useful:
   **a mirror can omit but never substitute.**
3. **[MUST]** **A mirror is not authorship, and it carries what proves that.** The gatherer signs the
   *mirror record*. **Each republished entry travels with its author's detached signature entity**
   (§1.1) — without it a mirror carries integrity but not authorship, and a reader can confirm the
   bytes match the hash while having no way to learn who wrote them. **Attribution follows
   `entry.author`, verified against that signature, always; a renderer that attributes a mirrored entry
   to the gatherer is non-conformant**, and one holding an entry whose signature is absent **MUST**
   present it as unattributed rather than attributing it to anyone. A mirror **MAY** additionally carry
   an inclusion proof where it wants to assert that the author *published* the entry as of a named root
   — a stronger and different claim than authorship.
4. **Unknown fields survive**, automatically, because bytes are republished rather than re-serialized.
   An implementation mirroring an entry that uses a field it has never heard of cannot strip it.

> **Rule 3 answers *"is republishing someone else's post legitimate?"* structurally rather than as a
> norm.** You do not hold their key; you cannot author in their name; you cannot alter a byte without
> the signature failing. All you can do is carry what they already chose to publish — which is what
> publishing has always meant, and which this format makes **verifiable** rather than merely customary.

> **A hazard for whoever implements or tests rule 1, in any language.** If an implementation's entity
> equality compares **content hash only** — the right default nearly everywhere — **it is the wrong
> granularity here and hides this entire class.** A byte-fidelity check compares bytes. Likewise a
> round-trip through bytes your own encoder produced proves nothing: the input must be **deliberately
> non-canonical** — legal input your encoder would never emit.

### 6.2 What the mirror is for

A conversation spans publishers, and no single publisher holds all of it. **A mirror is one reader's
answer to *"here is what I gathered"*, published so the next reader does not have to gather it again**
— and because merging two mirrors is set union over signed content-addressed entries (§1.2, §1.3), two
people who gathered disjoint halves independently produce the whole with no coordination and no
conflict.

**Completeness is unattainable without a gatekeeper and this document does not pretend otherwise.**
What is attainable is that **a view is never wrong, only short**, and that shortness is both visible
(compare two mirrors) and repairable (read one more).

---

## 7. Freshness, editing and withdrawal

### 7.1 Two classes of polled data, and only one of them can be *wrong*

| | **Monotone** data | **Authoritative** data |
|---|---|---|
| Examples | an entry, an index page, a mirror | a name binding, a transport record, a revocation |
| A stale copy is | **short** — missing what is newer | **wrong** — pointing somewhere that no longer holds |
| Recheck cadence is | a **reader preference** | a **correctness parameter**, bounded by the issuer |

- **[MUST NOT]** A reader **MUST NOT** surface feed staleness as an error. *"Nothing new since you last
  looked"* and *"we have not looked recently"* are different sentences and both are ordinary. A reader
  that checks a followed peer once a day is not misconfigured; it is a reader that checks once a day.
- **[MUST NOT]** A reader **MUST NOT** extend an authoritative record's lifetime because a fetch
  failed. The degradation is to **unresolvable**, never to *stale but usable*.

> **One honest hole, not closed here.** A quiet publisher and a withholding origin are **byte-identical
> at the consumer**. A follow view therefore cannot truthfully say *"no new posts"* on the strength of
> one origin; the true statement is *"nothing new at the origin we asked."* Distinguishing the two
> requires a second source — which is exactly what a mirror provides, and which is why §6 is on the
> critical path for honesty rather than only for convenience.

### 7.2 Retroactive edit costs what posting costs

Verification is **root-anchored**: a reader fetches the publisher's signed root, verifies one signature,
and walks the content-addressed trie from it. **There is no per-entry chain to break**, and after §2.3.1
there is no required backward pointer anywhere in this vocabulary.

Removing an entry means **removing a binding and republishing the root**. Changing one binding creates
new nodes only along the hash path from root to the affected leaf — `O(log_K N)` — and every unchanged
sub-node is shared by hash reference. **So removing a months-old entry costs what publishing a new one
costs.** There is no archive rewrite, no re-signing of later entries and no cascade — **provided nothing
newer points backward at it**, which is exactly why §2.3.1 made `prev` opt-in and §4.3 rule 1 made pages
key-addressed. **Both of those exist to protect this property.**

### 7.3 The removal leaves no trace in the current tree

The trie is canonicalized so that *the same binding set produces the same root regardless of insertion
or **deletion** history.* Read against this use case: **a tree from which an entry was removed is
byte-identical to a tree that never contained it.** Not similar — identical.

**There is no tombstone, no gap, no null, no *"entry deleted"* marker, and no way for a reader to detect
from the current root that anything was ever there.** That is not a policy this convention adopted; it
is a property of the data structure.

### 7.4 What an author cannot do, stated precisely

**You cannot un-publish the past, and the mechanism that prevents it is one you already publish.**
`system/peer/published-root` carries `predecessor` — the prior root's content hash — so the sequence of
roots is itself a chain, signed by you, and anyone who kept an old root kept a signed commitment to what
you were serving then.

| | Your **current** tree | The **root chain** |
|---|---|---|
| Is | what you claim to publish, now | what you signed at each publish |
| You may | **curate it freely — add, amend, remove, with no residue** | **not rewrite it**; old roots are out, signed |

**So curation is effective and honest at the same time.** You control your presentation; you cannot
forge your past; nobody can force you to keep serving something. The only party who can show that you
changed something is a party who was already holding the old version — which is the situation on every
publishing medium that has ever existed.

### 7.5 The UI obligation, which is where this bites

| The act | What happens | What does not happen |
|---|---|---|
| **Edit** | publish a new entry referencing the old; readers may show either, and a conformant renderer **SHOULD** show the newest it holds | the old one does not stop existing for anyone holding it |
| **Take down** | remove the binding, republish the root; new readers do not receive it, your origin stops serving it, and the tree carries no trace it was there | nobody's held copy is affected, and anyone who kept an older signed root can still show that what you served changed |

**[MUST NOT]** Removal from a tree is **unpublication, not erasure**, and a conformant application
**MUST NOT** present removal as deletion. There is **no global takedown**, no protocol operation reaches
into another peer's store, and none will. *"Removed from your site — people who already have it still
have it"* is the honest sentence, and it belongs at the moment of the action rather than in a help page.

> **Why there is no tombstone, and why an expiry hint is a display preference.** A published *"please
> forget this"* marker works where a bounded set of cooperating servers agrees to honour it. Here the
> holder set is unbounded and uncoordinated, so such a marker would be **a promise the architecture
> cannot keep** — worse than the honest statement, because a person would believe it. A well-behaved
> reader **MAY** honour an expiry hint; nothing compels one to; and it **MUST** be documented as a
> display preference rather than a control.

> **The amendment approach is the wrong mechanism, and the reason is sharp: a retraction only means
> something while the thing it retracts is still published.** A marker pointing at content you removed
> is a dangling reference; a marker pointing at content you kept means **you are still publishing the
> thing you are disowning, forever, in order to disown it.** Unbinding needs no marker. A retraction
> *entry* remains available to an author who wants the record to show a correction — that is an ordinary
> entry saying so, and it is a choice rather than a mechanism.

### 7.6 The light-client property

*"I do not want your whole history; I want to be current and to know it is really yours."* That is **one
signature and a bounded read**:

1. Fetch `system/peer/published-root`, verify one signature, check `seq` has not gone backward.
2. Read the index head — a known key, tree-depth cost.
3. Read down to your cursor and stop.

**Everything read under that root is committed to by that one signature**, because the trie is
hash-linked from the root. There is no chain to sync, no history to replay and no genesis to reach.

**One limit, already normative and not a defect to close:** a negative answer is scoped to the root that
produced it. **No signature can attest that a publisher showed you everything they hold.** That is the
same withholding hole §7.1 records, and it is a property of publication rather than of this design.

---

## 8. Syndication emission

**A publisher MAY emit a standard syndication feed (RSS or Atom) from a published root**, and doing so
makes the tree legible to every reader on the internet at essentially zero design cost — the format is
external and frozen, and nothing in this convention needs to change to accommodate it.

**It is a projection, never an authority.** The entities are the record; a syndication document is a
rendering of them for consumers outside this system, exactly as an HTML page is. A conformant
implementation **MUST NOT** treat an emitted feed as a source of truth for its own reads.

---

## 9. What this convention deliberately does not decide

### 9.1 Selection

Which entries a reader shows is **per-reader by design and MUST NOT converge.** Filtering, muting and
blocking are choices, and a system in which selection converged would be one with a central moderator.
Because a mirror can omit but never substitute, **choosing what to republish is itself moderation**, and
choosing whose mirrors to read is choosing whose moderation to accept. **No field in this document
expresses an exclusion, and none should: an exclusion is the absence of a reference.**

### 9.2 Ranking

Not specified. Specifying it would be the centralizing move — it makes the algorithm something someone
must own. The recorded direction: **a ranking is publishable, subscribable data like anything else**,
several may coexist, and where auditability is wanted the ranking function can itself be
content-addressed, so a behaviour change is a different hash rather than a silent one. **None of that is
needed here**, and chronological and reply order — what a reader actually does most of the time —
converge for free.

### 9.3 Cross-publisher merged order

A reader assembling a timeline from many publishers is making a presentation decision with no authority
in the data (§2.3.2). **[MUST]** This document requires only that **a view declare what produced it** —
which sources, which mirrors, which exclusions — when it presents itself as more than one publisher's
feed.

### 9.4 Addressing — *who this entry is for*

**No field here says who an entry is addressed to, and that is deliberate.** `context` says what an
entry is *part of* and never who it is *for*; `reply` names what it answers, which is an entry rather
than an audience.

**The reason to decline it is that it is a different concern with a different owner.** An addressing
list is a **delivery instruction**: it decides who gets told, which is an inbox question and a
capability question, and it drags in consent, rate limits and the whole surface by which unsolicited
delivery becomes abuse. **Putting it in the content vocabulary would place a delivery mechanism in a
format that has no delivery.** A mention that is merely *rendered* — a link to a peer inside a body —
already works and needs nothing from this document.

**Not an oversight, and not permanently closed.** If an implementation builds notification and finds it
needs a structured addressee list, that is a proposal against the inbox surface.

### 9.5 Not private conversation, not the forum, not a bridge

Everything here is public-broadcast. Confidential entries can ride the same rails — nothing the format
reads lives inside the body — but **membership, key distribution and rotation are the hard part and are
not addressed.** Following *people* is solved: they told you where they are. Following an *idea* —
arriving at a conversation with no contact already in it — needs a topic identifier and a discovery walk
this document does not specify; **the aggregation half is solved by §6, the navigation half is open.**
Reading *into* this network from another social protocol is a real extension per protocol, not a shim.

---

## 10. Floor (SPECIFICATION-FORMAT §8.6)

**A peer with no revision extension, no subscription engine, no group extension and no always-on
presence is a fully valid participant.** A conformant minimal peer:

- publishes `app/feed/entry` entities into its own namespace, plus an index head and page;
- puts that tree on any static host, and **is followable** — a static content-addressed tree with a
  signed root is a fully participating publisher, and **there is no daemon requirement to be read**;
- reads others by fetching roots and walking indexes, holding a cursor locally;
- and **publishes nothing at all if it is only a reader.** The read-only follower needs no origin, no
  name, no inbound reachability and no published root. **That is the cheapest participant in the
  system.**

Capability adds: a mirror is optional, a revision chain is optional, a live connection is optional, an
inclusion proof is optional.

---

## 11. Conformance

**Requirement id prefix:** `FEED`. **Requirement ids are `FEED-Rn`; vector ids are `FEED-n`** and are
carried unchanged from the design record so existing citations resolve.

### 11.1 Requirements

| id | Requirement | Level | § |
|---|---|---|---|
| `FEED-R1` | Reject an entry whose `author` differs from the namespace it was found under | MUST | §1.1 |
| `FEED-R2` | Sign each entry as a detached `system/signature` entity at `/{author}/system/signature/{hex(entry_hash)}` | MUST | §1.1 |
| `FEED-R3` | Mint that signature at authoring time; a mirror may carry one but never supply one | MUST | §1.1.1 |
| `FEED-R4` | Present an entry whose signature is absent as **unattributed**, rather than attributing it | MUST | §1.1.1, §6.1 r3 |
| `FEED-R5` | Inline another entry's bytes, in an entry or in a mirror | MUST NOT | §1.2 |
| `FEED-R6` | Accept only a pinned reference at `reply.root` and `reply.parent` | MUST | §2.2.1 |
| `FEED-R7` | Make available to the view the fact that a live reference resolved to something other than `seen` | MUST | §2.2.2 |
| `FEED-R8` | Rely on `created_at` for correctness, or reject an entry for an implausible timestamp | MUST NOT | §2.3.2 |
| `FEED-R9` | Describe replies as notifying the author absent an explicit delivery grant | MUST NOT | §3 |
| `FEED-R10` | Address index pages by key; never chain them by hash | MUST | §4.3 r1 |
| `FEED-R11` | Renumber, merge or compact index pages | MUST NOT | §4.3 r3 |
| `FEED-R12` | Emit an unbounded index page, or assume any page size when reading one | MUST NOT | §4.3 r5 |
| `FEED-R13` | Treat the index as authoritative, or reject an entry for being unlisted | MUST NOT | §4.3 r6 |
| `FEED-R14` | Resume from `page` when a cursor's `applied` hash no longer resolves | MUST | §4.4 |
| `FEED-R15` | Present `collection.members` in the authored order | MUST | §5 |
| `FEED-R16` | Republish mirrored entries as original bytes — never re-encoded, re-normalized or re-serialized | MUST | §6.1 r1 |
| `FEED-R17` | Assert or imply completeness of a mirror | MUST NOT | §6.1 r2 |
| `FEED-R18` | Attribute a mirrored entry to the gatherer rather than to `entry.author` | MUST NOT | §6.1 r3 |
| `FEED-R19` | Surface feed staleness as an error | MUST NOT | §7.1 |
| `FEED-R20` | Extend an authoritative record's lifetime because a fetch failed | MUST NOT | §7.1 |
| `FEED-R21` | Present removal as deletion | MUST NOT | §7.5 |
| `FEED-R22` | Show the newest revision it holds, where an entry has been revised | SHOULD | §7.5 |
| `FEED-R23` | Declare what produced a view — sources, mirrors, exclusions — when it presents more than one publisher's feed | MUST | §9.3 |
| `FEED-R24` | Treat an emitted syndication document as a source of truth for its own reads | MUST NOT | §8 |

**Ids are allocated once and never reused** (`SPECIFICATION-FORMAT` §8.5a).

### 11.2 Required checks — what an implementation must discriminate

**These are the cases this convention names; the fixtures, the bytes and the run are the
implementations' and the conformance oracle's** (GUIDE-EXTENSION-DEVELOPMENT §7, `GUIDE-CONFORMANCE` §5.1a). Class:
application-tier format checks — example entities compared byte-for-byte across independent
implementations. **The convention is authored; it is validated when these have been exercised.**

| id | Vector | Drives | What fails without it |
|---|---|---|---|
| `FEED-1` | `app/feed/entry`, text body, no reply — canonical ECF + expected hash | — | the entry encoding is not known to be byte-stable cross-impl |
| `FEED-2` | entry with `reply {root, parent}` and a `prev` | `FEED-R6` | the reference atom and the threading fields |
| `FEED-3` | entry whose `author` ≠ its namespace — **rejected** | `FEED-R1` | the forgery gate |
| `FEED-4` | `app/feed/index-page` and its predecessor, after the head moves | `FEED-R10`, `FEED-R11` | a superseded page is not known to stay byte-identical |
| `FEED-5` | mirror carrying two entries by two authors, **each with its author's detached signature** | `FEED-R2`, `FEED-R16` | republished bytes are not known to still verify against *their* signatures |
| `FEED-6` | an entry carrying an **unknown field**, mirrored | `FEED-R16` | round-trip byte identity |
| `FEED-7` | `app/feed/follow` | — | the follow record encoding |
| `FEED-8` | `app/feed/collection`, three members and a `cover` | `FEED-R15` | authored order is not known to be preserved |
| `FEED-9` | a mirrored entry **with its signature stripped** | `FEED-R4` | **integrity without authorship** — invisible without this vector |
| `FEED-10` | publish N entries across 3 pages, remove one from the oldest, republish: **(a)** the new root is byte-identical to a tree built without it · **(b)** the changed-node count is `O(log_K N)`, not proportional to the archive · **(c)** the other pages' bindings are untouched | `FEED-R10`, §7.2, §7.3 | **the one that fails loudest if the design drifts back toward a chain** |
| `FEED-11` | a reader holding cursor `{page: 1, applied: H}` where **H was since deleted** | `FEED-R14` | a cursor that breaks on edit |

**`FEED-3`, `FEED-5`, `FEED-6` and `FEED-9` are the load-bearing four.** Three fail if an implementation
re-serializes instead of republishing — the failure that turns the model from evidence into hearsay —
and the fourth fails if it treats *the hash matched* as *the author wrote this*.

> **Two notes for whoever authors these, in any language.** The `FEED-6` fixture must be **deliberately
> non-canonical** — legal input the local encoder would never emit — or the test proves only that an
> encoder round-trips its own output, which is a tautology; it wants an anti-vacuity guard asserting
> that the fixture really is non-canonical and that normalizing it moves the hash. **And if an
> implementation's entity equality compares content hash only** — the right default nearly everywhere —
> **it is the wrong granularity here and hides the whole class.** Compare bytes.

### 11.3 Types installed

```
app/feed/entry
app/feed/index-head
app/feed/index-page
app/feed/collection
app/feed/mirror
app/feed/follow
```

---

## 12. Open items

**Carried explicitly, because an open item that disappears into a landed document becomes invisible
exactly when it stops being true.** None of these blocks an implementation of §§1–11; each names what
would resolve it.

| # | Item | Resolves when |
|---|---|---|
| **F-1** | **The cursor's permanent home.** §2.4's `cursor` field is carried here so a v1 reader is implementable from this document alone. **A general reader-loop mechanism with non-social consumers is the better home**, and if one lands, this field is removed in favour of it. Flagged so nobody builds a second permanent home for it | a general reader-loop mechanism is specified |
| **F-2** | **Whether `app/feed/follow` and `app/share/follow` should eventually unify.** Recorded, not ruled — see §2.4 and the share convention's own note. Unifying requires making its `record` field optional, which changes what an absent field means in a landed schema | implementations converge |
| **F-3** | **Retention.** Republication makes storage grow monotonically and nothing reclaims it. §6's mirror is what makes this load-bearing rather than tidy-up: a thread mirror grows forever and no rule anywhere says what a reader keeps or for how long | a retention policy exists |
| **F-4** | **The quiet-publisher / withholding-origin indistinguishability** (§7.1). Byte-identical at the consumer, and unowned across the whole corpus. §6 narrows it — a second source detects divergence — but does not close it | a second-source comparison is specified |
| **F-5** | **Whether a chat message unifies with `app/feed/entry`.** The shapes match on most fields, **but a conversation carries a policy** — open, invited, closed — and a consumer holding a message from a closed conversation MUST NOT republish it. **The policy lives on the conversation, so a message lifted into a feed carries no *do-not-mirror* bit** — the §6 failure mode one layer up, with the disposition **missing rather than wrong**. Unifying would put a republish-forbidding message into the one type the mirror exists to republish. **SPECIFICATION-FORMAT §8.9 is the general rule this produced**; what remains is the specific call | both application-tier implementations weigh in |
| **F-6** | **Where a profile lives.** Nobody asked for one and every comparable system has one. Probably **not a fifth type**: a profile is distinguished not by its content shape but by being **live-addressed — never pinned**. With §2.2's two atoms landed, the likely answer is a well-known path plus a live reference | someone needs a display name |
| **F-7** | **A self-applied content warning has no home.** It passes the taxonomy's own test — a conformant consumer holding one must **behave** differently (render behind an interstitial), not merely lay it out differently. **Its shape is already decided by SPECIFICATION-FORMAT §8.9**: it is a property governing what a consumer may do with an entity, so it lives on the **entry**, never on a collection, mirror or index. What is open is whether it is a field at all, versus a reader-side list | an implementation needs it |
| **F-8** | **`collection`'s authored order rests on weak evidence.** §5 distinguishes the collection partly on authored order, **and the one real bounded-complete set in the field renders in *derived* order** — so the property the type leans on is not the one its only consumer exercises. The suggestion on the table is an optional order field on one type rather than two types. *Bounded* and *complete* are unaffected and still hold | a second bounded-complete consumer exists, or the optional-order shape is proposed |
| **F-9** | **A quote — *render that entry inside this one* — is expressible two ways and assigned neither.** It is not a reply (it asserts no answer) and not an ordinary attachment (it is an entry, not a file). `attachments` is a reference list; the body's handler layer is open, so an entity-reference embed type is legal. **Two candidate homes, no rule, and picking one is cheap** | a renderer needs to show a quote |
| **F-10** | **The multi-party record — reserved at §1.1.2, not designed.** Every shape here has exactly one signing party. A co-signed record is bounded, multi-party and claims completeness **verifiably**, since it names its parties and each signature sits at the core's invariant pointer in that party's namespace. **The substrate is complete and no shape exists.** Instances: a co-authored post, a mutual follow read as a relationship, a confirmed invitation, a receipt, an order | an implementation needs a co-signed object |
| **F-11** | **Is `context` doing too much?** It carries *part of a topic*, *part of an album* and *part of an event* in one field. **If consumers must branch on which, it fails the taxonomy's own test and should be split** | a second consumer of `context` exists |
