# PROPOSAL — `APP-CONVENTION-FEED` — the entry, the reference, the index and the mirror

**Status:** DRAFT (2026-09-04)
**Tier:** applications — an `APP-CONVENTION-*` member. **No protocol change, no extension change, no
new kernel surface.**
**Target:** `specs/applications/APP-CONVENTION-FEED.md` (new; the fourth `applications/` member) + a
`CHARTER.md` Members row + one new charter discipline (§8).

> **What this proposes, in one sentence.** A vocabulary for *a thing someone posted* — four entity
> types and one reference atom — so that following people across independent hosts and reading what
> they published is a format two independent implementations can both produce and both read.

**Why now, and it is not a judgement call.** `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4.2 cut the
ordered page collection on good reasoning, pinned a determinism floor, and deferred the semantic
layer **"to a named feed/index extension authored when a concrete feed use case arrives."** *This is
that document, and the use case is the one named in §7.* The deferral has expired on its own stated
terms; nothing in §4.2's reasoning is reopened and §3 below satisfies it rather than overturning it.

**Rests on:** `EXPLORATION-THE-L5-CONTENT-TAXONOMY-AND-THE-FLOOR-THAT-STOPS-THE-EXPLOSION` (**why
these four shapes and not eleven** — the test, the landscape, and where a gallery, a forum, a story
and a chat each land) · `EXPLORATION-THE-FALSIFICATION-TEST-FOUR-REAL-OBJECTS-AGAINST-THE-FLOOR`
(**the floor tested against four objects nobody designed for us** — the floor holds, §13 item 5; and
the four-for-four derivation behind §2.2's second atom) ·
`EXPLORATION-THE-DURABLE-REFERENCE-THE-ANCHOR-AND-THE-THREE-WAY-READ` (the two reference intents and
the three-way read that §2.2.4 states) · `EXPLORATION-THE-RESOLUTION-CHAIN-FROM-A-SOCIAL-IDENTIFIER-TO-AN-ENTITY` (the chain
audit that found this link empty) · `EXPLORATION-THE-PUBLIC-SOCIAL-STACK-THE-COMPLETE-MAP` §4 (where
these four shapes were worked out) · `EXPLORATION-WHAT-CONVERGES-WHAT-CANNOT-AND-THE-ORDER-TO-BUILD-IT`
(why this is stage 1 and why the forum is not) · `APP-CONVENTION-EMBED` (the body) ·
`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4.2 (the deferral and the floor).

**Explicitly NOT in scope, and each is named so the document is not read as complete:** the follow
set and the reader cursor (`PROPOSAL-FOLLOW-THE-PATTERN-THE-SET-AND-THE-TWO-ROUTES` owns those, and
§7 below states the dependency) · ranking and any notion of a timeline algorithm (§6.3) · private,
encrypted or membership'd conversation (§9.2) · the forum's topic discovery (§9.3) · any bridge to
another social protocol (§9.4).

---

## §1 Three properties, and everything else is derived from them

Stated first because every rule below is a consequence of one of them, and an implementer who holds
these three can re-derive most of the document.

### §1.1 The author is the namespace — and authorship is carried by a detached signature

**An entry's `author` MUST equal the peer namespace it is authored under.** An entry found under one
peer's namespace claiming a different author is **invalid** and a conformant reader MUST reject it.

**An entry MUST be individually signed.** The signature is a separate `system/signature` entity at the
core protocol's invariant pointer path — `/{author}/system/signature/{hex(entry_hash)}` — carrying
`target = entry_hash` and `signer = author`. This is not a new mechanism: it is the core's existing
one, and §3.5 of the core protocol names the property this convention needs in its own words —
***"when Carol asks Alice for Bob's data, Alice can include Bob's signatures. Carol can verify Bob's
authority without contacting Bob."*** That is republication, specified two layers down, before anyone
was thinking about feeds.

> **This paragraph replaces an error, and the error is worth recording because it was load-bearing.**
> An earlier draft said an entry was *"signed by author"* as though the signature were part of the
> entity. **It is not.** An entity is `(type, data)` and its content hash; **no field of it names a
> signer**, and anyone holding the bytes reconstructs an identical entity — which is exactly why bytes
> cannot establish who wrote them. Verification in this substrate is normally **root-anchored**: you
> walk from a publisher's signed root, and inclusion in that tree is what makes the content theirs.
> **That works perfectly while you are reading their tree and not at all once an entry travels**, which
> is the whole of §4. **So the mirror model needed a mechanism it did not name, and the corpus already
> had it.** Found by an implementation seat measuring §13 item 1 rather than reviewing it.

**Two mechanisms exist for binding an entry to an author, they prove different things, and this
convention requires the first and permits the second.**

| | **Detached signature** (required) | **Inclusion proof** (optional) |
|---|---|---|
| Proves | *this author signed these bytes* | *this author had these bytes at this key, under root R* |
| Costs | **one small content-addressed entity, per entry** | a chain of trie nodes, **proportional to tree depth, per entry** |
| Travels | yes — content-addressed, dedups, verifiable offline from bytes alone | yes, but anchored to one root |
| Needs | the author to have signed the entry | nothing beyond the published root |

**The required one is the signature, and the argument is the same one that produced §1.2: the unit of
addressing should also be the unit of verification.** An entry that travels alone should verify alone,
in `O(1)` extra objects, without its author's tree, without their origin, and without a root that may
be many publishes stale. **A mirror carrying fifty entries by twelve authors carries fifty signatures,
every one content-addressed and deduped against whatever the reader already holds.**

**The inclusion proof is permitted and is not redundant**, because it answers a question the signature
cannot: *was this in their published tree, as of sequence N?* That is the anti-omission question, and a
mirror asserting *"and here is proof they published it"* is making a stronger and different claim. It
is optional because it costs depth-many nodes per entry and most readers do not need it.

**What this rule buys, stated as the property the rest of the document leans on:** you do not hold
anyone else's key, so you cannot author in their name; you cannot alter a byte without the signature
failing; and **an entry stays verifiable when its author is offline, when their origin is gone, and
when it reaches you from a stranger.** *The only operation republication permits is carrying what they
already chose to publish.*

> **What this costs, corrected — it is a PUBLISHER change, and the table above reads as if it were a
> mirror-carriage cost.** `[entity-browser-rust` `62c6d62`, from their implementation.]`
> **Nothing signs individual entities today: a publish signs exactly one thing, the root.** So the
> detached per-entry signature is a **new obligation on the composer**, and a mirror cannot supply it —
> *a mirror can only carry a signature the author already minted.* Three consequences worth stating
> rather than discovering:
>
> 1. **The composer signs.** Whatever authors an entry mints its signature at authoring time, not at
>    publish time and not at mirror time.
> 2. **The cost is per-entry-once, not per-publish**, and the entities **dedup** — which makes it much
>    cheaper than the table's *"per entry"* framing suggests to a reader who assumes republishing.
> 3. **Entries published before this rule can never be attributed once mirrored**, because nobody can
>    retroactively mint a signature they did not make. **That is a permanent boundary in the data, not
>    a migration window**, and a reader will encounter unattributable old entries forever. It should be
>    stated in the spec rather than found.
>
> **The machinery is proven** — the registry already mints and verifies detached signatures at the same
> invariant-pointer path — so this is an obligation to schedule, not a mechanism to design.

> **The scope of this MUST, stated so it is not generalized into a law about the system.**
> `[added 2026-09-05 — `EXPLORATION-THE-SECOND-FALSIFICATION-…` §4]`
> **This rule scopes the entry.** It is the anti-forgery rule for a singly-authored object and it is
> correct for every shape this convention defines. **It is not a statement that application data has
> one signer**, and it must not be read as one — the substrate says the opposite: V7 binds each
> signature as a **separate entity in the signer's own namespace** at
> `/{signer}/system/signature/{hex(target_hash)}`, so one entity can carry N signatures from N
> namespaces, and `EXTENSION-QUORUM`'s `verify_k_of_n_signatures` already validates K-of-N over an
> arbitrary entity.
>
> **Why this sentence is here rather than in a later revision.** A record that two or more parties
> co-sign — a co-authored post, a mutual follow read as a *relationship* rather than as two
> independent follows, an RSVP the host confirmed, a receipt, an order — **is a different shape, and
> it is not specified anywhere yet.** If §1.1 is read as a system-wide property, that shape arrives as
> a contradiction to be migrated around instead of an addition. **It is reserved here, deliberately
> not designed here**, and designing it wants the seats building commerce or private audiences in the
> room. **`[OPEN-FEED-12]`**

### §1.2 One entry is one addressed unit — entries reference, they never embed

**An entry MUST NOT contain another entry's bytes.** What it replies to, what it quotes, what it
attaches and what it is part of are all carried as **references** (§2.2), never inline.

**This is the granularity rule, and it is what makes deduplication real rather than nominal.** Content
addressing does not deduplicate anything by itself — two peers who never share a store hold two copies
of identical bytes, and that is ordinary replication. What content addressing gives is that **the two
copies have the same name**, so:

- a reader that already holds an entry pays **nothing** to receive it again from a second source, and
  can decline to fetch it at all;
- two mirrors of the same conversation, assembled independently by people who never met, hold the
  **same** entries at the **same** hashes, so merging their views is set union with no reconciliation;
- and a reply costs the size of the reply. **Commenting on something does not copy the thing being
  commented on.**

**Every one of those properties fails if an entry inlines what it points at**, because an inlined copy
gets a new hash inside a new parent and is no longer the same object to anybody. So the anti-pattern
is named outright: **a conformant producer MUST NOT inline the body of a referenced entry** — not in
an entry, and not in a mirror (§4.1). Where a renderer wants to show quoted text, it resolves the
reference and renders what it resolved.

**The honest scope of the benefit, so nobody oversells it.** Deduplication is real **within a store**
and across everything that store holds; between two peers with no shared store it is replication, and
the win is the shared *name*, not shared bytes. That is the useful case anyway: the redundancy that
matters is many people holding the same conversation, and they each hold it once.

### §1.3 The set only grows

Entries are immutable and content-addressed, and there is no delete. A revision is a **new** entry
referencing the old one, and *which one a reader displays* is an interpretation question, never a set
question.

The consequence a reader must be built around: **a partial view is never a wrong view, only a short
one**, and every additional source can only lengthen it. This is what makes §3's index an optimization
rather than an authority, what makes §4's mirror unable to lie except by omission, and what makes §6's
staleness rule sound.

---

## §2 The vocabulary

Four type tags plus one shared atom. **The cross-impl contract is the type tag, not the path** — the
same rule `APP-CONVENTION-SHARE` §2 states and for the same reason: cross-peer aggregation is a
`type_filter` query over the universal tree with no peer filter, so the tag *is* the index key, and a
tag under one application's own prefix makes two implementations mutually invisible even with a
perfect mirror between them.

| Type | Role | Provenance |
|---|---|---|
| `app/feed/entry` | **one thing someone published** — the atom | its author |
| `app/feed/index-head` | the stream's entry point: which page numbers are in use | its author |
| `app/feed/index-page` | one **key-addressed** page of the author's stream, newest-first within the page | its author |
| `app/feed/collection` | the author's own **bounded, complete, authored-order** set — album, playlist, portfolio | its author |
| `app/feed/mirror` | a **gathered** view of other people's entries, claiming nothing | a reader |
| `app/feed/follow` | a reader's durable subscription to a peer's feed | the reader, privately |

**Four content shapes and it terminates, which is the point.** The three collection types are not
subject-matter categories — they are the three answers to *"who assembled this list and what does it
therefore claim?"*, and there is no fourth answer. **A new product is a new body type (an embed
handler, and that extension point is open and unbounded) or a new renderer.** It is a new entity type
only if a conformant consumer must *behave* differently, and the burden is on whoever proposes one to
name the behaviour. `EXPLORATION-THE-L5-CONTENT-TAXONOMY-AND-THE-FLOOR-THAT-STOPS-THE-EXPLOSION` is
the derivation, including why a blog, a vlog, a photo post, a comment and a forum post are all the
first row.

### §2.1 Shared atoms — **IMPORTED, not defined here**

**`APP-CONVENTION-REFERENCE` §2.1 is the single home for these.** Restated so this document reads
standalone; on any disagreement that document is the authority.

```cddl
; Self-describing (format_code, digest) per V7 §1.2/§1.4 — the leading varint is the
; content_hash_format and THE DIGEST LENGTH FOLLOWS THE CODE. Never fixed-width (charter #6).
content-hash = bstr

peer-id      = tstr                  ; V7 §1.5 Base58 peer-id — a TEXT string, not bytes
tree-path    = tstr                  ; absolute or peer-relative per V7 §1.4
```

**No fixed-width hash form appears anywhere in this document.**

### §2.2 The two references — pinned and live, and they are two shapes on purpose

**There are two reference intents and they demand different consumer behaviour, so there are two
atoms.** An earlier draft had only the first, derived it from the reply case, and generalized it to a
domain that contains a second case with the opposite requirement.

**RE-CUT: these are now the tier's shared atom, imported rather than defined here.**
`APP-CONVENTION-REFERENCE` §2.1 generalizes exactly these two shapes for the whole application tier;
this document was where they were first derived, and it is no longer where they live.

```cddl
; IMPORTED — APP-CONVENTION-REFERENCE §2.1. Reproduced for readability; that document is the authority.
reference      = pinned-ref          ; "THIS EXACT THING" — the pin, and the default
live-reference = live-ref            ; "WHATEVER IS AT THIS PLACE NOW"
any-reference  = entity-ref          ; only where a site declares it takes both (§2.2.3)

pinned-ref = { tag: "pin",  peer: peer-id, hash: content-hash, ? at: anchor, ? via: [* hint] }
live-ref   = { tag: "live", peer: peer-id, path: tree-path, ? seen: content-hash,
                                                            ? at: anchor, ? via: [* hint] }
```

**Two changes from the shape this document first derived, and both are stated rather than absorbed:**

1. **The discriminator is a `tag`, not the field name.** §2.2.3's argument is preserved in full and is
   satisfied by a stronger mechanism — see the restatement at the end of that section.
2. **`? path` on the pinned form becomes a `via` hint** — `{ tag: "path", value: … }`. **It was always
   a hint** (this document's own words: *"a starting point, not an address of record"*), and it shared
   a field name with the term that is *authoritative* on the live shape. **One field, one job:** `path`
   is authoritative only where it is the identity term. The obligation not to read a `404` at that
   location as absence is unchanged and is now normative at `APP-CONVENTION-REFERENCE` §2.3.

#### §2.2.1 `reference` — the pin

**The hash is the claim; the locator is a convenience.** A consumer holding the bytes fetches nothing.
One that does not may obtain them **from the publisher's declared content origin, or from any source
that has them and that the reader can already reach** — because the hash validates them regardless of
source. The locator is a hint and a reader MUST NOT treat a failure to fetch at it as evidence the
entry does not exist; it is a starting point, not an address of record.

> **Scoped deliberately, and the earlier wording is the reason.** This read *"from anybody — the
> author, a mirror, a cache, a stranger"*, which **names a mechanism the substrate does not have**:
> there is no operation answering *who has this hash?*, and bare-hash substitution is explicitly out
> of scope in the extension that would own it. **What DOES exist is the composition** — a reader
> holding a publisher peer-id resolves that peer's transport profile, reads its content-origin prefix,
> builds a URL, fetches and hash-verifies, with no preconfigured source. **That is a normative path
> to the bytes and it is still ahead of the pull-based systems this is compared to**; it just is not
> *anywhere*. Two narrow cases remain genuinely unanswered and are named rather than papered over: a
> peer-id that resolves to no transport (which is what a hint slot is for), and a publisher who is
> simply gone (the only case that would need content routing, an axis measured and declined).

**This is derived from a split in the deployed field rather than invented.** One lineage references a
reply by **location alone**, which makes a reply exactly as trustworthy as whatever currently answers
that location and lets a parent be edited underneath its replies. The other carries a content hash
beside the locator for precisely that reason. We get the second for free, and the `peer` term is added
because that is the unit the naming layer resolves.

#### §2.2.2 `live-reference` — the maintained document

**The use case is ordinary and the pin cannot express it:** *"read this document, at this place, and I
will keep updating it."* A publisher linking to their own maintained page under `reference` either pins
a hash that goes stale on the next edit, or rewrites every referring entry on every edit — **which is
precisely the cascade §2.3.2 removed `prev` to avoid, arriving through a different door.**

**The two atoms are the same three terms with required and optional swapped**, and that inversion is
the entire content of the distinction:

| | `reference` | `live-reference` |
|---|---|---|
| **Authoritative** | the hash | the path |
| **Fetch strategy** | **anywhere** — the hash validates the bytes regardless of source | **the named peer at the named path** — nobody else can answer for what is *current* there |
| **A hash mismatch means** | the reference is **unsatisfied** — refuse it | the document **evolved** — expected, and the reader is told (§2.2.4) |
| **A 404 at `path` means** | **nothing** — `path` is a hint | the reference is **broken**, modulo `seen` (§2.2.4 row 5) |

**`seen` is named for what it claims.** `hash` asserts *this is what it is*; `seen` asserts only *this
is what was there when I linked*. It is optional, and an author who omits it is saying they have no
expectation to offer.

#### §2.2.3 The discriminator, and each site declares what it accepts

**A reference carries `hash` or it carries `seen`; an atom carrying both is invalid and a conformant
reader MUST reject it.** The distinction is therefore never a mode value a consumer might fail to
branch on — **which is the whole reason there are two shapes rather than one shape with `hash` made
optional.** Under an optional `hash`, a reference arriving without one is indistinguishable between
*"the author wants the live version"*, *"the author's implementation did not populate it"* and *"the
author only ever had a URL"*: one intent and two bugs, with no way for a reader to tell them apart.

> **RESTATED against the tag, and nothing above is retracted.** The requirement this section sets is
> that **a reader can always tell**, and the argument it makes is against an **optional field** — not
> against a tag. A tag satisfies that requirement more directly than a field-presence rule does: the
> discriminator becomes a **value the reader reads** rather than an **inference the reader computes**,
> it is detectable at the first field rather than only by a reader that implemented the presence rule,
> and a third intent later becomes a third tag instead of a third presence rule interacting with the
> first two. **Two shapes, not one with an optional hash — that conclusion stands unchanged.** What
> changes is only how a reader reaches the right one, and the tier's other two conventions already
> discriminate this way.
**That would degrade the guarantee §2.2.1 exists to provide into a convention.**

**Each site that takes a reference declares which atom it accepts, and the pinned sites do not widen:**

| Site | Accepts | Why |
|---|---|---|
| `reply.root`, `reply.parent` | **`reference` only** | the rug-pull argument in §2.2.1 is the reason this field exists; it is undiminished, because the site that needed the pin never gains the weak form |
| `prev` | `content-hash` — **unchanged** | an append-only commitment to a *specific* predecessor is meaningless against a moving target |
| `context` | **either** | *"part of a topic"* is often a maintained index; *"part of this event"* is often a fixed entity |
| `attachments` | **either** | an attached file is usually pinned; an attached *living* document is the case this atom exists for |

#### §2.2.4 Resolving a `live-reference` — the comparison result is information, not an error

Because `path` is authoritative and `seen` is only an expectation, resolution has more than two
outcomes, and **a reader MUST be able to tell which one it got.**

| # | `(peer, path)` | vs `seen` | Meaning | Reasonable behaviour |
|---|---|---|---|---|
| 1 | resolves | matches, or `seen` absent | you are seeing what the linker saw, or they offered no expectation | render |
| 2 | resolves | **differs** | **the document evolved** — the ordinary case | render current, **and surface that it moved**; the pinned version remains fetchable |
| 3 | **404** | — | the path moved or was unpublished | **fall back to `seen`, fetched from the publisher's declared content origin or any reachable source that has it** (§2.2.1) — the hash validates the bytes whoever serves them |
| 4 | 404 | `seen` absent or unobtainable | genuinely dangling | the honest failure. Nothing to hide |

**Row 3 is where the second naming layer earns its keep and it is the row the surveyed field does not
have.** A content hash resolves against *any* store, so **a live reference survives its author
unpublishing the path** — the property Hyper-G obtained with a server-owned link database, obtained
here without one.

**The normative half is the reader's ability to tell, not which policy it picks.** Strict (refuse on
mismatch) and lenient (render current) are both legitimate and are the *reader's* choice; what this
convention requires is that **a view built from a `live-reference` whose resolved hash differed from
`seen` MUST make that fact available to the view.** This is the capstone rule applied to references —
*a view that names its own provenance is debuggable; one that does not is indistinguishable from a
bug.*

### §2.3 `app/feed/entry`

```cddl
feed-entry = {                       ; type = app/feed/entry
  type: "app/feed/entry",
  data: {
    author:      peer-id,            ; MUST equal the authoring namespace (§1.1)
    created_at:  uint,               ; ms since epoch, author's clock — a DISPLAY HEURISTIC (§2.3.1)
    body:        embed-node,         ; APP-CONVENTION-EMBED — this convention defines no content types
    ? reply:     { root: reference, parent: reference },   ; present ⇒ this entry is a reply
    ? context:   any-reference,      ; what this is PART OF — never who it is for. Either atom (§2.2.3)
    ? prev:      content-hash,       ; OPT-IN append-only commitment — see §2.3.2. NOT navigation.
    ? attachments: [* any-reference] ; referenced, never inlined (§1.2). Either atom (§2.2.3)
  }
}
```

**`body` is an embed node and this convention defines no content types of its own.** A photo post and
a text post are one shape with a different embed inside — which is why `APP-CONVENTION-EMBED` was
factored as foundational, and this is the payoff.

**`reply` carries `root` as well as `parent`, and the second field is load-bearing.** With `parent`
alone, assembling a conversation is a hop-by-hop walk and **one unreachable author truncates
everything below them**. `root` lets any holder of any entry name the whole conversation in one step,
which is what makes a partial view assemblable and a mirror findable. The mail lineage reached the same
conclusion by carrying the entire ancestry chain; `root` + `parent` is its bounded form.

**A conversation needs no genesis entity: the root entry *is* the conversation, and its hash is the
conversation's identity.**

**`context` is not `reply`.** `reply` says *this answers that*; `context` says *this belongs with
that* — a topic, a collection, an event. They are separate fields because collapsing them makes
"replied to" unrenderable.

#### §2.3.2 `prev` is an opt-in append-only commitment, and it is NOT how a reader navigates

**An earlier draft made `prev` a per-author causal chain and that was a defect.** A chain of hashes
running backward through an author's history has one property nobody wanted: **it makes every old
entry permanently referenced by a newer, still-published one.** Remove the old entry and the newer one
still names its hash — so the removal leaves a residue the author cannot clear without re-signing the
newer entry, which changes *its* hash, which cascades forward to every entry after it. **A chain
converts a local edit into a republish of everything since.** It also truncates: a reader walking
backward stops dead at the first hash it cannot fetch, and cannot tell removal from withholding.

**So `prev` is optional, it is not navigation, and its meaning is narrow and useful:**

> **An entry carrying `prev` is a claim that the author is publishing an append-only sequence** —
> that this entry follows that one and nothing was removed between them. **An author who uses it is
> giving up the ability to silently revise**, and a reader may treat a gap in such a chain as
> meaningful.

**That is a real use case and it should be available**: a changelog, a public record, an attested
log, anything where *"I cannot have quietly edited this"* is the point. **It is the wrong default**,
because most publishing is not that, and imposing it would take away the curation right §5b
establishes for everyone in order to serve the minority who want to surrender it.

**Navigation is by key (§3), never by chain.**

##### The derivation has a second half, and two deployed systems pay for it

The argument above is about **the author's own history** — a chain makes curation cost a republish. Two
systems that took the mandatory-chain branch show two further costs, and both are structural rather
than incidental.

**A mandatory chain is a single-writer commitment, so it is a one-device commitment.** The surveyed
append-only-log lineage puts `previous` and `sequence` in the message envelope and requires them. **One
human with a phone and a laptop then has one sequence and two writers**, and publishing from the second
device either forks the log or requires the devices to coordinate on every post. The deployed answers to
this are all unsatisfying — a single designated device, or last-write-wins over a clock nobody can
verify. **Our rule survives this cleanly precisely because `prev` is opt-in**: a multi-device author
simply does not use it, and pays nothing.

> **This is not a claim that we have solved multi-device publishing.** We have not — two devices sharing
> one key and advancing one signed root is a race this corpus does not describe, and that gap is real
> and open. **What §2.3.2 establishes is narrower and still worth stating: making `prev` mandatory would
> have made that open problem strictly harder**, by adding a second ordering commitment on top of the
> root sequence.

**And a chain forces a deletion mechanism, which is where the second cost shows up.** A system whose
events carry backward pointers cannot remove an event without breaking the graph, so it must keep the
skeleton and empty the contents — an entire redaction algorithm, and a permanently unreclaimable
placeholder in place of what was removed. **That is §5b.3's property lost**: our removal leaves no trace
in the current tree, and it can do so only because the integrity anchor is the root and not a chain.

> **So an author who opts into `prev` is opting into that system's problem, knowingly.** They gain *"I
> cannot have quietly edited this"* and they give up clean removal — not just for the entry they remove,
> but for everything published after it. **That is the right trade to offer and the wrong one to
> impose.**

#### §2.3.1 `created_at` is not an ordering authority

It is the author's own clock, it is unverifiable, and a peer may set it to anything. **A reader MUST
NOT rely on it for correctness** and MUST NOT reject an entry for an implausible timestamp. It is
what a renderer shows and sorts by when it has nothing better; the causal facts are `prev` (within one
author) and `reply` (across authors), and both are in the data.

**This is deliberately weaker than it could be**, and the reason is worth stating: any stronger claim
requires either a clock nobody can verify or a coordination step this design does not have, and the
cost of getting it wrong — entries silently dropped as "too old" or "from the future" — is worse than
a display artifact.

### §2.4 `app/feed/follow`

```cddl
feed-follow = {                      ; type = app/feed/follow
  type: "app/feed/follow",
  data: {
    subject:    peer-id,             ; whose feed this follows — a NAMESPACE, not a record
    ? label:    tstr,                ; the follower's own petname; local, never authoritative
    ? via:      tstr,                ; the identifier as typed/scanned, for provenance display
    since:      uint,                ; ms since epoch — when this follow was created
    ? cursor:   content-hash         ; last index page this reader applied — see the note below
  }
}
```

**Why this is a distinct type from `app/share/follow`, stated because two `follow` tags in one
vocabulary is exactly the kind of thing that confuses an implementer.** The discriminator is the
**subject**: `app/share/follow` follows a **grant** — one titled share record, with an audience the
publisher authorized, which means the publisher knows the follower exists. `app/feed/follow` follows a
**namespace** — public, pull-only, requiring no grant and no permission, and **the publisher does not
know the follower exists.** Those are different mechanisms with different authorization models, and
overloading one tag would require making its `record` field optional, which changes what an absent
field means in an already-landed schema. **If the implementing peers converge on unifying them, that
is recorded rather than ruled here** — the same stance `APP-CONVENTION-SHARE` §5 takes on its own
follow surface.

**`cursor` is carried here provisionally and its home is an open question that this document does not
own.** `PROPOSAL-FOLLOW-THE-PATTERN-THE-SET-AND-THE-TWO-ROUTES` argues the cursor is a *general*
reader-loop concern with consumers outside anything social, and if it lands there this field is
removed in favour of it. It is written here so a v1 feed reader is implementable from this document
alone, and it is flagged so nobody builds a second permanent home for it. **`[OPEN-FEED-1]`**

---

## §3 The index — a bounded head page with an immutable back-chain

### §3.1 The problem it solves, measured

The primary social operation is *"what did they post that I have not seen?"* On a real published tree
of ~1,000 bindings the trie is 52 nodes at max depth 2. Verifying a **known** region costs the tree's
depth — a handful of nodes, ~11 KB, even while the publisher rewrites most of the estate. But
**discovering what is new under a prefix costs the whole tree**, because the structure is keyed by
hash bits rather than by path locality: three keys under one site live in three different leaf nodes,
and all 51 must be read to find them.

So the index is not a convenience layer added after the fact. **It is the thing that converts the
primary operation from the expensive enumeration to the cheap known-key read**, and it is therefore a
prerequisite of the reader loop rather than a later optimization.

### §3.2 The shape

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
  type: "app/feed/index-page",       ; at /{peer}/app/feed/index/{page}   — page is a decimal uint
  data: {
    page:       uint,                ; this page's own number — MUST equal its key
    entries:    [* reference],       ; newest first within the page
    updated_at: uint
  }
}
```

**Two pinned paths: the head at `/{peer}/app/feed/index` and pages at
`/{peer}/app/feed/index/{page}`.** A reader that holds no reference has to start somewhere, and a
reader resuming has to be able to jump straight to where it left off.

### §3.3 The rules, and why each one is here

1. **Pages are addressed by key, never chained by hash.** This is the load-bearing change and §5b is
   why. A hash back-chain makes page *N*'s identity depend on page *N−1*, so **editing one old page
   forces rewriting every page after it** — the archive gets republished to remove one entry. With
   key addressing, rewriting page 12 changes page 12's binding and nothing else: **O(tree depth), the
   same cost as posting.**
2. **A reader jumps.** Holding a cursor at page 12, it fetches page 12 directly rather than walking
   the head backward to reach it. **The chain form had no random access; this does.**
3. **Pages are never renumbered, never merged, and never compacted.** A page that loses an entry is a
   page with fewer entries. **Renumbering would invalidate every reader's cursor**, which is the one
   thing a publisher must not be able to do by accident.
4. **A reader fetches the head, reads down from `current` to its cursor, and stops.** Cost is
   **O(new)**, not O(all). A page it already holds is skipped when the tree says it has not moved —
   which is a known-key check costing tree depth, not a refetch.
5. **The page is bounded and this document does not pin the bound.** A producer MUST NOT emit an
   unbounded page; a reader MUST NOT assume any page size. A fixed number is deliberately not
   specified — the right value depends on entry size and publishing cadence, which vary by orders of
   magnitude between publishers, and a MUST naming a number would be a threshold nobody could state
   the unit of. **What is normative is the shape, not the arithmetic.**
6. **The index is an optimization and MUST NOT be the authority.** A reader that cannot fetch it falls
   back to enumerating the prefix — slower, same answer. **An entry absent from the index is still a
   valid entry**, and a reader that encounters one by reference MUST NOT reject it for being unlisted.

**The cursor is `{page, applied}`** — the page number the reader reached and the newest entry hash it
took from it. **If `applied` no longer resolves, the reader resumes from `page`**, which is why the
cursor carries a number and not only a hash: an author may have removed the very entry a reader was
holding as its position, and a cursor that cannot survive that is a cursor that breaks on edit.

**Rule 4 is a charter-#4 floor obligation and it also closes a hole**: if the index were authoritative,
it would be a place for a publisher to lie by omission about their own posts, and a reader would have
no way to tell an omission from an absence.

### §3.4 What this does and does not settle about ordering

`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4.2 pinned exactly one cross-impl rule — a renderer presenting
a raw `.list` view sorts by name segment, byte-wise — and left the *semantic* layer open, correctly
observing that the tree carries no semantic order and that enumeration is hash-bit ordered.

**This document adds one semantic ordering contract and nothing more: `entries` within an index page is
newest-first, as ordered by the publisher.** The order is *authored*, not derived: it is whatever
sequence the publisher wrote, and a reader renders it in that sequence. It is not a claim that the tree
is sorted, it does not make `created_at` authoritative, and it does not touch §4.2's determinism floor,
which continues to govern raw enumeration views.

**A reader merging entries from several publishers has no cross-publisher order in the data**, and this
document deliberately does not invent one. Sorting a merged view is a presentation choice (§6.3).

---

## §3a The collection — the author's own, bounded, and complete

**The gallery is what surfaced this, and it is not a short feed.** An album, a playlist, a portfolio
and a curated reading list are one shape, and that shape differs from §3's index on three axes at
once — which is what makes it a type rather than a label.

| | the index (§3) | the collection |
|---|---|---|
| Bounded? | no — it grows forever | **yes; it is a finite thing with edges** |
| Complete? | never claims to be | **yes — this *is* the album** |
| Reader holds | a cursor, and stops at it | nothing; it fetches the set |
| Order means | recency | **the author's arrangement, which is content** |
| "Show me all of it" | not a sane operation | **the primary operation** |

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

**Order is authored here and derived in §3, and that is the substantive difference.** The site
convention correctly observed that the tree carries no semantic order and pinned only a determinism
floor for raw enumeration. **This does not reopen that**: a collection's order is not derived from the
tree, from names, or from timestamps — **it is a field, written by the author, and a collection
re-sorted is a different collection.** A renderer MUST present `members` in the order given.

**A collection holds references, never bodies** (§1.2). A photo appears in three albums by being
referenced three times; the bytes exist once.

**Membership is not exclusive and is not authority.** An entry may be in any number of collections, in
its author's tree and in other people's, and being in a collection says nothing about who may read the
entry — that is the grant layer's question and this convention does not touch it.

**Why this is one type and not five.** Album, playlist, portfolio, pinned set and reading list demand
identical consumer behaviour — fetch the set, render in the given order — so by the taxonomy's test
they are one shape with different bodies inside. `title` is what distinguishes them to a human, and a
renderer is free to present a collection of image entries as a grid and a collection of audio entries
as a queue. **That is presentation, and it is per-front-end by charter.**

### §3a.1 Media, and why the gallery case is the cheapest thing here

**Nothing about photos or video needs anything from this convention**, and that is worth saying
because it is the use case most likely to be assumed hard. `APP-CONVENTION-EMBED` already carries the
whole of it: an image post is an entry whose body is an embed with a **pointer payload** — a content
hash into the store, which is what every real image and video already uses — and the embed convention
already specifies **renditions** so a consumer can select a size or format.

**The economics land better here than anywhere else in the design.** Large immutable
content-addressed blobs served from a static origin behind a CDN is precisely what that
infrastructure was built for: they are cacheable forever because the hash *is* the name, they dedup
across every collection and every mirror that carries them, and **the publisher does no work per
viewer.** A photo library is the case this architecture is best at, not a stretch of it.

---

## §4 The mirror — assembly as a by-product of participation

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

### §4.1 Four rules, each closing a specific hole

1. **Republish the original bytes.** Not a re-encoding, not a re-normalization, not a re-serialization
   through a local model. **A re-encoded entry no longer verifies against its author's signature**,
   which silently turns a mirror from evidence into hearsay. Byte preservation on forward is already
   this ecosystem's rule for cross-peer traffic; here it is the property the whole model rests on.
   Combined with §1.2, this also means **a mirror's `entries` is a list of references and the bytes it
   carries are stored content-addressed** — so a reader who already holds an entry from one mirror pays
   nothing to accept it from a second.

   > **MEASURED, and it holds structurally rather than by luck.** This rule was filed as the
   > highest-risk claim in the proposal and testable in an afternoon, and it was tested that way: a
   > foreign entity driven through wire decode → a real content store put/get → the exact call the
   > root projector emits with, comparing bytes and hash at the far end. **Identical; validation still
   > passes.** What makes it a measurement and not a tautology is the fixture — **deliberately
   > non-canonical CBOR** (an integer in non-minimal `uint8` form where the local encoder emits the
   > short form), legal input that our own encoder would never produce, with an anti-vacuity guard
   > asserting that the fixture really is non-canonical and that normalizing it *does* move the hash.
   > Neutered to normalize on write, the assertion goes red. **Round-tripping bytes your own encoder
   > produced proves nothing**, and that is the trap this test was built to avoid.
   > **One hazard recorded for whoever writes the next test in this area, in any language:** if an
   > implementation's entity equality compares **content hash only** — which is the right default
   > nearly everywhere — it is exactly the wrong granularity here and **would have hidden this
   > entirely.** A byte-fidelity test compares bytes.
2. **A mirror cannot claim completeness, and the shape gives it no way to.** There is no `complete`
   field and there will not be one. The verifiable property is narrower and more useful: **a mirror can
   omit but never substitute.**
3. **A mirror is not authorship, and it carries what proves that.** The gatherer signs the *mirror
   record*. **Each republished entry travels with its author's detached `system/signature` entity**
   (§1.1) — without it a mirror carries integrity but not authorship, and a reader can confirm the
   bytes match the hash while having no way to learn who wrote them. Attribution follows
   `entry.author`, verified against that signature, always; a renderer that attributes a mirrored
   entry to the gatherer is non-conformant. **A mirror MAY additionally carry an inclusion proof**
   (§1.1) where it wants to assert that the author *published* the entry, as of a named root — a
   stronger and different claim than authorship.
4. **Unknown fields survive**, automatically, because bytes are republished rather than re-serialized.
   An implementation mirroring an entry that uses a field it has never heard of cannot strip it.

**Rule 3 is the answer to "is republishing someone else's post legitimate?", and the answer is
structural rather than a norm.** You do not hold their key; you cannot author in their name; you cannot
alter a byte without the signature failing. All you can do is carry what they already chose to publish
— which is what publishing has always meant, and which this format makes verifiable rather than merely
customary.

### §4.2 What the mirror is for, in two sentences

A conversation spans publishers, and no single publisher holds all of it. **A mirror is one reader's
answer to "here is what I gathered", published so the next reader does not have to gather it again** —
and because merging two mirrors is set union over signed content-addressed entries (§1.2, §1.3), two
people who gathered disjoint halves independently produce the whole with no coordination and no
conflict.

**Completeness is unattainable without a gatekeeper and this document does not pretend otherwise.**
What is attainable is that a view is never wrong, only short, and that shortness is both visible
(compare two mirrors) and repairable (read one more).

---

## §5 Freshness — two classes of polled data, and only one of them can be *wrong*

**This is the rule that keeps a reader from being built with one cadence for everything**, and it falls
straight out of §1.3.

| | **Monotone** data | **Authoritative** data |
|---|---|---|
| Examples | an entry, an index page, a mirror | a name binding, a transport/endpoint record, a revocation |
| A stale copy is | **short** — missing what is newer | **wrong** — pointing somewhere that no longer holds |
| Recheck cadence is | a **reader preference** | a **correctness parameter**, bounded by the issuer |
| Governed by | nothing normative; whatever the reader wants | the record's own lifetime, and the resolver's ceiling |

**Concretely:** a reader that checks a followed peer once a day is not misconfigured, it is a reader
that checks once a day, and no conformance statement anywhere may call it wrong. A resolver that
serves a binding past its stated lifetime **is** wrong, because it can route to a peer the issuer has
since repointed.

Two normative consequences:

- **A reader MUST NOT surface feed staleness as an error.** "Nothing new since you last looked" and
  "we have not looked recently" are different sentences and both are ordinary.
- **A reader MUST NOT extend an authoritative record's lifetime because a fetch failed.** The
  degradation is to *unresolvable*, never to *stale but usable*.

**One honest hole, carried from the chain audit and not closed here.** A quiet publisher and a
withholding origin are **byte-identical at the consumer**. A follow UI therefore cannot truthfully say
*"no new posts"* on the strength of one origin — the true statement is *"nothing new at the origin we
asked"*. Distinguishing the two requires a second source, which is exactly what a mirror provides and
which is why §4 is on the critical path for honesty rather than only for convenience.

---

## §5b Retroactive edit — what it costs, and what it leaves behind

**The question this section answers:** *I want to delete or amend something I published three months
ago, deep in the feed. Does that break the cryptographic chain? Do I have to republish my whole
history? And can a reader still verify they are reading my real feed without pulling all of it?*

**Three answers, and all three are better than the naive expectation.**

### §5b.1 Nothing breaks, because the integrity anchor is the root and not a chain

Verification in this substrate is **root-anchored**: a reader fetches the publisher's signed root,
verifies one signature, and walks the content-addressed trie from it. **There is no per-entry chain to
break**, and after §2.3.2 there is no required backward pointer anywhere in this vocabulary.

Removing an entry means **removing a binding and republishing the root**. The new root is signed, its
`seq` increases, and it commits to a tree that does not contain the entry. **A reader gets a complete,
internally consistent, fully verifiable feed** — one that simply does not include the thing that was
removed.

### §5b.2 It costs the same as posting, not the same as republishing

**`EXTENSION-TREE` §3.1:** changing one binding creates new nodes **along the hash path from root to
the affected leaf — `O(log_K N)` per change, K=32** — and every unchanged sub-node is shared by hash
reference. On the measured live tree that is a handful of nodes.

**So deleting a three-month-old entry costs what publishing a new one costs.** There is no archive
rewrite, no re-signing of later entries, and no cascade — **provided nothing newer points backward at
it**, which is exactly why §2.3.2 made `prev` opt-in and §3.3 rule 1 made pages key-addressed. **Both
of those changes exist to protect this property.**

### §5b.3 The removal leaves no trace in the current tree — and this is the strong result

**`EXTENSION-TREE` §9:** the trie is canonicalized so that *"the same binding set produces the same
root regardless of insertion or **deletion** history."*

Read that against this use case and it says something worth stating plainly: **a tree from which an
entry was removed is byte-identical to a tree that never contained it.** Not similar — identical.

**There is no tombstone, no gap, no null, no "entry deleted" marker, and no way for a reader to
detect from the current root that anything was ever there.** That is not a policy this convention
adopted; it is a property of the data structure, and it happens to be exactly the psychological
closure a person wants when they take something down: **your current published state carries no scar.**

### §5b.4 What you cannot do, stated precisely

**You cannot un-publish the past, and the mechanism that prevents it is one you already publish.**
`system/peer/published-root` carries **`predecessor`** — the prior root's content hash — so the
sequence of roots is itself a chain, signed by you, and anyone who kept an old root kept a signed
commitment to what you were serving then.

**This is the right place for the honest half of the tension, and it resolves it rather than fudging
it:**

| | Your **current** tree | The **root chain** |
|---|---|---|
| Is | what you claim to publish, now | what you signed at each publish |
| You may | **curate it freely — add, amend, remove, with no residue** | **not rewrite it**; old roots are out, signed |
| A reader with only the new root sees | your presentation as you intend it | — |
| A reader who kept an old root can | — | prove that what you served changed |

**So curation is effective and honest at the same time**, which is the outcome to aim for. **You
control your presentation; you cannot forge your past; nobody can force you to keep serving
something.** The only party who can show that you changed something is a party who was already
holding the old version — which is precisely the situation on every publishing medium that has ever
existed, and the one thing that no protocol can or should alter.

### §5b.5 Why the amendment approach is the wrong mechanism

**The instinct — publish a marker saying *"ignore the entry from June"* — is worse than it looks, and
the reason is the one the operator identified: a retraction only means something while the thing it
retracts is still published.** A marker pointing at content you have removed is a dangling reference;
a marker pointing at content you have kept means **you are still publishing the thing you are
disowning, forever, in order to disown it.**

**Unbinding needs no marker.** It is cheaper, it is complete, and it does not require the author to
keep serving what they wanted gone. **A retraction entry remains available to an author who
specifically wants the record to show a correction** — that is an ordinary entry saying so, and it is
a choice rather than a mechanism.

### §5b.6 The light-client property, since it is the same question from the other side

*"I do not want your whole history; I want to be current and I want to know it is really yours."*

**That is one signature and a bounded read**, and it needs no history at all:

1. Fetch `system/peer/published-root`, **verify one signature**, check `seq` has not gone backward.
2. Read the index head — a known key, tree-depth cost.
3. Read down to your cursor and stop.

**Everything read under that root is committed to by that one signature**, because the trie is
hash-linked from the root. **There is no chain to sync, no history to replay, and no genesis to
reach** — the signed root is the checkpoint, and every publish issues a new one.

**One limit, already normative and not a defect to close:** `EXTENSION-TREE` §3.3a — *"a negative
answer is scoped to the root that produced it… signing what you published cannot prove you published
everything you hold."* A reader learns what is in this root; **no signature can attest that a
publisher showed them everything.** That is the same withholding hole §5 records, and it is a property
of publication rather than of this design.

---

## §5a Editing and withdrawal — what an author can actually do

**A person publishing must be told the truth about this, at the moment they act, and the truth is not
what a platform trained them to expect.**

| The act | What happens | What does not happen |
|---|---|---|
| **Edit** | publish a new entry referencing the old; readers may show either, and a conformant renderer SHOULD show the newest it holds | the old one does not stop existing for anyone holding it |
| **Take down** | remove the binding, republish the root; **new readers do not receive it, your origin stops serving it, and the tree carries no trace it was there** (§5b.3) | nobody's held copy is affected, and anyone who kept an older signed root can still show that what you served changed (§5b.4) |
| **Never publish it in the clear** | the only mechanism that actually withholds | — *and it is later work* |

**Both halves of this are legitimate and the tension is not an artefact of this design.** An author
may revise or retract what they said; a reader may keep what they were given. Every publishing medium
that ever existed has had exactly this tension, and the difference here is only that **there is no
intermediary who can pretend to resolve it.**

**Stated as the normative floor:** removal from a tree is **unpublication, not erasure.** There is
**no global takedown**, no protocol operation reaches into another peer's store, and none will. This
is the precise counterpart of *no platform* — nobody to appeal to, in either direction.

**And it constrains the UI, which is where this actually bites.** A conformant application MUST NOT
present removal as deletion. *"Removed from your site — people who already have it still have it"* is
the honest sentence, and it belongs at the moment of the action rather than in a help page.

**Why there is no tombstone.** A published *"please forget this"* marker works where a bounded set of
cooperating servers agrees to honour it. Here the holder set is unbounded and uncoordinated, so such a
marker would be **a promise the architecture cannot keep** — which is worse than the honest statement,
because a user would believe it. **The same reasoning governs any expiry hint** on an entry: a
well-behaved reader may honour it, nothing compels one to, and it MUST be documented as a display
preference rather than a control.

---

## §6 What this convention deliberately does not decide

### §6.1 Selection

Which entries a reader shows is **per-reader by design and MUST NOT converge.** Filtering, muting and
blocking are choices, and a system in which selection converged would be one with a central moderator.
Because a mirror can omit but never substitute, **choosing what to republish is itself moderation**,
and choosing whose mirrors to read is choosing whose moderation to accept. No field in this document
expresses an exclusion, and none should: an exclusion is the absence of a reference.

### §6.2 Ranking

Not specified, and specifying it would be the centralizing move — it makes the algorithm something
someone must own. The design direction, recorded rather than proposed: **a ranking is publishable,
subscribable data like anything else**, several may coexist, and where auditability is actually wanted
the ranking function can itself be content-addressed, so a behaviour change is a different hash rather
than a silent one. **None of that is needed for this document, and chronological and reply order —
which is what a reader actually does most of the time — converge for free.**

### §6.3 Cross-publisher merged order

A reader assembling a timeline from many publishers is making a presentation decision with no
authority in the data (§2.3.1). This document requires only that **a view declare what produced it** —
which sources, which mirrors, which exclusions — when it presents itself as more than one publisher's
feed. A view that names its own provenance is debuggable; one that does not is indistinguishable from
a bug, and that is the difference between two clients disagreeing and the system appearing broken.

### §6.4 Addressing — *who this entry is for*

**No field in this document says who an entry is addressed to, and that is deliberate.** `context`
says what an entry is *part of* and never who it is *for* (§2.3); `reply` names what it answers, which
is an entry rather than an audience.

**This is worth stating rather than leaving as an absence, because the surveyed field carries it and a
reader of this document would reasonably expect it.** Three of the four systems mapped in the
falsification exploration carry an explicit mention or addressing list, and in all three it is the
input to **notification and delivery** — not to rendering.

**The reason to decline it here is that it is a different concern with a different owner.** An
addressing list is a delivery instruction: it decides who gets told, which is an inbox question and a
capability question, and it drags in consent, rate limits and the whole surface by which unsolicited
delivery becomes abuse. **Putting it in the content vocabulary would place a delivery mechanism in a
format that has no delivery.** A mention that is merely *rendered* — a link to a peer inside a body —
already works today and needs nothing from this document.

**So: not an oversight, and not permanently closed.** If a seat builds notification and finds it needs
a structured addressee list, that is a proposal against the inbox surface, and this section is the
statement of what was decided and why rather than a claim that the question is settled.

---

## §7 The chain, walked end to end — *"I add someone. What happens?"*

The document is easier to review against a concrete sequence than against a schema, and this is the
sequence. **Marked by what exists**: ✅ built and deployed · 🟡 designed, unbuilt · 🔴 missing.

| # | Step | What it reads / writes | State |
|---|---|---|---|
| 1 | A person supplies an identifier — a name, a key, a link, a scanned code | — | ✅ |
| 2 | Resolve it to a **peer id** | a registry binding, a well-known document, a petname, or the key itself | ✅ names · 🟡 the web-native backend |
| 3 | Obtain an **origin** for that peer | the binding's glue transports, or a cached transport profile | ✅ *with a name* · 🔴 **with only a bare id** |
| 4 | Write an `app/feed/follow` into **my own** tree | mine, signed, durable across devices and reinstalls | 🔴 **this document** |
| 5 | Fetch their **signed root** at the origin | `system/peer/published-root` — `root_hash`, `seq`, `prefix` | ✅ deployed |
| 6 | Read `/{peer}/app/feed/index` and walk back to my cursor | §3 | 🔴 **this document** |
| 7 | Fetch the entries I do not hold; verify each against the author's key | §1.1, §1.2 | ✅ substrate |
| 8 | Render — body via the embed vocabulary | `APP-CONVENTION-EMBED` | ✅ |
| 9 | Store the cursor; next wake, refetch **only the root** and compare `seq` | unchanged ⇒ stop | 🟡 the reader loop |

**Two things this table is for.** First, the shape of the remaining work: **steps 1–3 and 5–8 are
substrate that exists**, and what is missing is steps 4 and 6 — *this proposal* — plus the loop in
step 9 and one hole in step 3. Second, the cost model: **step 9 is the steady state**, and it is one
small fetch per followed peer with no publisher-side work at all. **The publisher does no work per
follower; a million followers and one cost the same.**

**The hole in step 3 is real and is not this document's to close.** With a name you get transports for
free; with only a bare peer id there is currently nowhere to look. Every id-first system in the
deployed field solves this the same way — *the endpoint set is published by the id holder, signed by
the id holder, fetched by id* — and `PROPOSAL-PEER-TRANSPORT-SET` is that work. **A feed reader can
ship without it** (follows acquired by name or by link carry an origin), which is why this is a
dependency to name rather than a blocker.

---

## §8 The compatibility contract — proposed as a charter discipline, not a clause in this document

**The vocabulary is the thing outsiders copy, and once anything is published against it a change stops
being a decision and becomes a migration.** This ecosystem already holds the wire half of that
discipline — unknown fields are MUST-ignore, and the locked core is never renumbered — and holds
**nothing equivalent for the application-tier type vocabulary.**

That absence is not hypothetical. Two independent implementations of one landed convention currently
emit **different type tags for the same object**, and neither side can see it: a type-filtered query
using the wrong tag returns a correct, complete, **empty** answer. **A naming divergence has no
discovery path except somebody reading both implementations** — it never surfaces as a byte mismatch,
because the two sides never hold each other's data at all.

**Proposed as `CHARTER.md` discipline #7**, because it governs every convention rather than this one:

> **7. States a compatibility contract, and keeps it.** A convention's published vocabulary is a
> commitment. **All data valid under a previous version of the vocabulary MUST remain valid under the
> current one, and data produced under the current one MUST remain valid under the previous.**
> Concretely: **new fields are optional; a field's type never changes; a field is never renamed; a
> tag is never repurposed; and a breaking change takes a new type tag.** A convention that must break
> compatibility mints a new name and leaves the old one meaning what it meant.

**And the enforcement point, because a discipline without one is theatre:** the type tags a convention
pins are greppable, and a convention's vector set (charter #5) is the artifact that fails when a shape
changes. **Concretely: every tag this document pins ships with a vector (§10), so a rename or a type
change fails a byte comparison rather than passing silently as an empty query.**

---

## §9 The floor, and the four things this is not

### §9.1 The floor (charter #4)

**A peer with no revision extension, no subscription engine, no group extension and no always-on
presence is a fully valid participant.** Concretely, a conformant minimal peer:

- publishes `app/feed/entry` entities into its own namespace and a `app/feed/index-page` head;
- puts that tree on any static host, and **is followable** — a static content-addressed tree with a
  signed root is a fully participating publisher, and there is no daemon requirement to be read;
- reads others by fetching roots and walking indexes, holding a cursor locally;
- and **publishes nothing at all if it is only a reader** — the read-only follower needs no origin, no
  name, no inbound reachability and no published root. **That is the cheapest participant in the
  system and it should be stated rather than implied.**

Capability adds: a mirror is optional, a revision chain is optional, a live connection is optional.

### §9.2 Not private conversation

Everything here is public-broadcast. Confidential entries can ride the same rails — nothing the format
reads lives inside the body — but **membership, key distribution and rotation are the hard part and are
not addressed.** Stated because the surveyed field's characteristic failure is exactly this: shipping
public-first and retrofitting privacy onto an architecture that assumed publication.

### §9.3 Not the forum

Following *people* is a solved route: they told you where they are. Following an *idea* — arriving at a
conversation with no contact who is already in it — is a different problem, and it needs a topic
identifier and a discovery walk that this document does not specify. **The aggregation half is solved
by §4; the navigation half is open.**

### §9.4 Not a bridge

Emitting a syndication feed from a published root is nearly free once this exists and makes the network
legible to every reader on the internet, but it is a separate item. Reading *into* the network from
another social protocol is a real extension per protocol, not a shim, and is sized separately.

---

## §10 Conformance vectors — REQUIRED before ratification

**Charter #5, and this convention will not be ratified without them.** The evidence for insisting: the
one convention that shipped without vectors diverged between two implementations within six days, and
the divergence was undetectable from either side.

| ID | Vector | What it pins |
|---|---|---|
| FEED-1 | `app/feed/entry`, text body, no reply — canonical ECF + expected hash | the entry encoding is byte-stable cross-impl |
| FEED-2 | `app/feed/entry` with `reply {root, parent}` and a `prev` | the reference atom and the threading fields |
| FEED-3 | `app/feed/entry` whose `author` ≠ its namespace | **MUST be rejected** — §1.1, the forgery gate |
| FEED-4 | `app/feed/index-page` with `prev`, plus its immutable predecessor | the back-chain, and that a superseded page is byte-identical after the head moves |
| FEED-5 | `app/feed/mirror` carrying two entries by two different authors, **each with its author's detached signature** | republished bytes still verify against **their** signatures — §1.1 and §4.1 rules 1 and 3 |
| FEED-6 | An entry carrying an **unknown field**, mirrored | round-trips byte-identical — §4.1 rule 4 |
| FEED-7 | `app/feed/follow` | the follow record encoding |
| FEED-8 | `app/feed/collection` with three members and a `cover` | the set encoding, and that `members` order is preserved as authored — §3a |
| FEED-9 | A mirrored entry **with its signature stripped** | **MUST NOT** be presented as attributed — §1.1. Integrity without authorship is the failure mode the signature exists to prevent, and it is invisible without this vector |
| FEED-10 | **The retroactive-delete case**: publish N entries across 3 pages, remove one from the oldest page, republish | **(a)** the new root is byte-identical to a tree built without that entry (§5b.3, the deletion-history-independence property) · **(b)** the changed-node count is `O(log_K N)`, **not proportional to the archive** (§5b.2) · **(c)** the other two pages' bindings are untouched |
| FEED-11 | A reader holding cursor `{page: 1, applied: H}` where **H was since deleted** | resumes from page 1 without error and without re-delivering the whole stream — §3.3 |

**FEED-10 is the one that fails loudest if the design drifts back toward a chain.** If anyone
reintroduces a hash back-chain between pages or makes `prev` mandatory, (b) and (c) both go red and
the failure names itself.

**FEED-3, FEED-5, FEED-6 and FEED-9 are the load-bearing four.** Three of them fail if an
implementation re-serializes instead of republishing — the failure that turns the model from evidence
into hearsay — and the fourth fails if it treats *the hash matched* as *the author wrote this*.

**Two notes for whoever authors these, in any language.** The fixture for FEED-6 must be
**deliberately non-canonical** — legal input the local encoder would never emit — or the test proves
only that an encoder round-trips its own output, which is a tautology; and it wants an anti-vacuity
guard asserting the fixture really is non-canonical and that normalizing it moves the hash. **And if
an implementation's entity equality compares content hash only** — the right default nearly
everywhere — **it is the wrong granularity here and hides the whole class.** Compare bytes.

> **FEED-5's rule 1 half is already measured and PASSED** in one implementation, through wire decode →
> a real content store → the projection call, with both guards above in place and the test falsified
> in both directions. **The remaining risk in this row is the signature half**, which is new text and
> unmeasured.

---

## §11 The normative delta

| # | File | Section | Change |
|---|---|---|---|
| D1 | `specs/applications/APP-CONVENTION-FEED.md` | whole file | **NEW** — §§1–10 above, authored as the spec |
| D2 | `specs/applications/CHARTER.md` | Members table | add the `APP-CONVENTION-FEED` row |
| D3 | `specs/applications/CHARTER.md` | charter discipline list | add **#7**, the compatibility contract (§8) |
| D4 | `specs/applications/APP-CONVENTION-SEMANTIC-CONTENT-SITE.md` | §4.2 | replace *"defers the semantic layer to a named feed/index extension authored when a concrete feed use case arrives"* with a **citation to `APP-CONVENTION-FEED` §3**, and state that §4.2's determinism floor is unchanged and continues to govern raw enumeration views |
| D5 | `specs/applications/APP-CONVENTION-SHARE.md` | §2 | add one sentence distinguishing `app/share/follow` (follows a **grant**) from `app/feed/follow` (follows a **namespace**), pointing at `APP-CONVENTION-FEED` §2.4 |
| D6 | `ROADMAP-APPLICATIONS.md` | members | list the new convention |
| D7 | `specs/applications/APP-CONVENTION-EMBED.md` | §3 or §5 | one pointer noting that an image or video **entry** is an entry whose body is an embed with a pointer payload, and that renditions are the selection mechanism — so nobody builds a second media path for feeds (§3a.1) |
| D9 | `specs/applications/CHARTER.md` | the discipline list, beside #7 and #8 | **the disposition rule, and it is `entity-browser-rust`'s** *(`62c6d62`)*: **a property governing what a consumer may DO with an entity — may it be republished, must it be warned over, may it be cached — MUST live on the entity, never on a collection, conversation, index or mirror that contains it. Containers do not travel; entities do.** Derived from a live case (a closed-conversation message lifted into a feed carries no *do-not-mirror* bit) and it generalizes past that case immediately — it settles `[OPEN-FEED-9]`'s shape and it is the structural statement of FEED-9's *integrity without authorship* failure. **It belongs in the domain rather than in one member**, because every `APP-CONVENTION-*` that mints a container will face it |
| D8 | `specs/applications/CHARTER.md` | the discipline list, beside #7 | **the growth rule**: a new product is a new body type or a new renderer; it is a new *entity type* only if a conformant consumer must behave differently, and a proposal minting one **names that behaviour**. This is the anti-explosion gate and it belongs in the domain, not in one member |

**D4 and D5 are the reason this proposal enumerates its homes rather than editing one file.** A rule
lives in every document that states it, and two landed conventions already state something this one
changes the meaning of: the content-site's deferral now has an answer, and the share convention's
`follow` tag now has a sibling it must be distinguishable from. Neither is optional and neither is
cosmetic — a reader who finds `app/share/follow` first and never sees D5 will use the wrong tag.

---

## §12 Open items — these do NOT fold with the proposal

Carried explicitly, because an open item that folds into an implemented proposal becomes invisible
exactly when it stops being true.

| ID | Item | Owner | Resolves when |
|---|---|---|---|
| **`[OPEN-FEED-1]`** | The cursor's permanent home — this document's `follow.cursor` field, or a general reader-loop mechanism with non-social consumers | the follow proposal | that proposal rules; **if it lands generally, §2.4's `cursor` is removed** |
| **`[OPEN-FEED-2]`** | Whether `app/feed/follow` and `app/share/follow` should eventually unify | the implementing peers; recorded here, not ruled | peers converge |
| **`[OPEN-FEED-3]`** | Retention — republication makes storage grow monotonically and nothing reclaims it. **The mirror (§4) is what makes this load-bearing rather than tidy-up** | unowned | a policy exists |
| **`[OPEN-FEED-4]`** | The quiet-publisher / withholding-origin indistinguishability (§5) | unowned across the corpus | a second-source comparison is specified |
| **`[OPEN-FEED-5]`** | Bare-id → origin (§7 step 3) | `PROPOSAL-PEER-TRANSPORT-SET` | that proposal lands |
| **`[OPEN-FEED-6]`** | **Does a chat message unify with `app/feed/entry`? — LEANING NO, on arch's own test, argued by the seat that builds both.** `entity-browser-rust`: the shapes match on 4 of 6 fields, **but the conversation carries a policy — `closed`/`invite`/`open` — and a consumer holding a message from a closed conversation MUST NOT republish it.** The policy lives on the **conversation**, so **a chat message lifted into a feed carries no bit saying *do not mirror me***. That is FEED-9's failure mode one layer up: bytes that verify, in a tool that will republish them, with the **disposition missing rather than wrong.** **The general rule, and it is theirs: *a property governing what a consumer may do with an entity must live on the entity, not on its container — containers do not travel, entities do.*** Unifying would put a republish-forbidding message into the one type the mirror is built to republish | **both application seats.** One has now ruled with a measurement-grade argument; **`entity-workbench-go` has not weighed in and the item says two seats** | workbench-go responds. **Arch is not closing this on one seat** |
| **`[OPEN-FEED-7]`** | **Where does a profile live?** Nobody asked for one and every surveyed system has one. **Reframed, and it is probably no longer a fifth type:** checked against four primary schemas, the profile is four-for-four a **live-addressed** object — its distinguishing property is not its content shape but that **it is never pinned**. What made it look like a fifth type was that our only reference shape pinned a hash, so a profile modelled as an entry would have had every reference to it go stale on the first edit — **a missing intent presenting as a missing type.** With §2.2.2 landed, the likely answer is a well-known path plus a `live-reference` | unowned | someone needs a display name. **No longer blocked on a taxonomy question** |
| **`[OPEN-FEED-9]`** | **A self-applied content warning has no home**, and it is the clearest thing in the residue to pass the taxonomy's own test — a conformant consumer holding one must **behave** differently (render behind an interstitial) rather than merely lay it out differently. One of the four surveyed systems carries it as a first-class record field. **Its *shape* is already decided by `[OPEN-FEED-6]`'s rule, which was written for a different question and answers this one:** a content warning is exactly *a property governing what a consumer may do with an entity*, so **it lives on the entry — never on a collection, a mirror or an index that contains it.** What remains open is whether it is a field at all versus a reader-side list | unowned — **raise with both application seats**, since it is a field in a document under their review | a seat needs it, or the ActivityPub read confirms a second carrier |
| **`[OPEN-FEED-11]`** | **`collection`'s authored order is weak evidence for the shape.** §3a distinguishes the collection partly on **authored order**, and FEED-8 asserts `members` order is preserved — but `entity-browser-rust` reports that **their one real bounded-complete set renders in *derived* order**, so the property the type leans on is not the one the only implementation exercises. Their suggestion: **an optional order field on one type rather than two types.** *Bounded* and *complete* are unaffected and still hold | `entity-browser-rust`, raised `62c6d62` | a second bounded-complete consumer exists, or the seat proposes the optional-order shape |
| **`[OPEN-FEED-10]`** | **A quote — *render that entry inside this one* — is expressible two ways and assigned neither.** It is neither a reply (it asserts no answer) nor an ordinary attachment (it is an entry, not a file). `attachments` is a reference list; the body's handler layer is open, so an entity-ref embed type is legal. **Two candidate homes, no rule, and picking one is cheap** | unowned | a seat renders a quote |
| **`[OPEN-FEED-12]`** | **The multi-party record — reserved at §1.1, not designed.** Every shape this convention defines has exactly one signing party, and §1.1's MUST makes that normative for the entry. **A co-signed record is a fourth provenance cell** (`EXPLORATION-THE-SECOND-FALSIFICATION-…` §4): bounded, multi-party, and claiming completeness — **verifiably**, since the record names its parties and each signature sits at the core's invariant pointer path in that party's namespace. **The substrate is complete and no L5 shape exists**; `EXTENSION-TRANSACTION` §1.2 is the checked negative (*"this extension is local — one peer"*). Instances: a co-authored post, a mutual follow read as a relationship, a confirmed RSVP, a receipt, an order | unowned — **wants the seats building commerce or a private audience** | a seat needs a co-signed object. **The release-window obligation is only the §1.1 scoping note, which is landed** |
| **`[OPEN-FEED-8]`** | Is `context` doing too much? It carries *part of a topic*, *part of an album* and *part of an event* in one field. **If consumers must branch on which, it fails the taxonomy's own test and should be split** | unowned | a second consumer of `context` exists |

---

## §13 What would make this proposal wrong

Stated because a proposal that cannot say what would refute it has not been stress-tested.

1. ~~**If republishing bytes verbatim is impractical**, the mirror model needs rework.~~ **TESTED AND
   IT HOLDS** — see §4.1 rule 1. Measured through wire decode, a real content store, and the
   projection call, against a deliberately non-canonical fixture, falsified in both directions. **No
   rework needed, and the rule stands verbatim.**
   **What the same test found instead is the more valuable half, and it is now §1.1:** entities carry
   **no** embedded signature, so `{peer, hash}` alone gave a mirror **integrity without authorship**.
   The fix is one content-addressed object per entry rather than a redesign. **The lesson to carry:
   the highest-risk claim was fine and the defect was in the sentence next to it** — a claim that
   survives its own test can still be resting on a false neighbour, and only building it finds that.
2. **If the index's known key collides with how a real publisher lays out a tree**, §3.2's pinned path
   is wrong and the head needs to be discovered rather than known.
2a. **If a key-addressed page set turns out to cost more to maintain than a chain** — because a
   publisher must now decide when to open a new page number and that decision has no obviously right
   answer — then §3.3's trade is wrong in practice even though it is right on paper. **FEED-10 tests
   the property; nothing yet tests the ergonomics**, and the ergonomics are what an author lives with.
3. **If `root` + `parent` proves insufficient to assemble a real conversation** — because a reader
   holding a middle entry cannot find siblings without an ancestry walk after all — then §2.3's bounded
   form is too bounded, and the mail lineage's full chain was carrying something we removed.
4. **If two implementations produce different bytes for the same authored entry**, the vocabulary is
   underspecified somewhere and §10's vectors are the instrument that says where.
5. ~~**If a real object from a comparable system needs a fifth content shape**, the four-shape floor is
   wrong and the taxonomy explodes after all.~~ **TESTED, AND IT HOLDS — with a caveat that is part of
   the result.** Four objects designed by four other teams — an ATProto post, a Nostr kind-1, a Matrix
   `m.room.message` and an SSB `post` — were mapped field by field onto the floor from their primary
   schemas. **All four are `app/feed/entry`. None demanded a fifth shape.** Two things did not fit, and
   **both were predicted in advance by the taxonomy document**: the Matrix *room* is none of the three
   collection provenances (it satisfies all four of the obligations §5.2 listed for a conversation), and
   the profile is `[OPEN-FEED-7]`.
   **The caveat, stated because the result is worth less without it: all four are microblog-lineage
   systems.** A wiki, a marketplace listing, a map annotation and a collaborative document are untested,
   and that is where a genuine falsification would come from.
   **What the test found instead is the more valuable half, and it is the same shape as item 1's
   lesson:** the floor survived and **the atom next to it did not.** All four systems carry *both*
   reference intents and all four express the difference as a distinct shape or name — never as an
   optional field — which is the derivation behind §2.2. **A claim that survives its own test can still
   be resting on a false neighbour.**
