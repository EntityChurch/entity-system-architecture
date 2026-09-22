# EXPLORATION — the durable reference, the anchor, and the three-way read

**Status:** Exploration (design record). Not a proposal, not normative.

**Origin: the operator's own design sketch, 2026-09-04.** Recorded here because a decision made in
conversation is not durable until it is on disk, and because working it out **corrected a finding this
corpus published the day before.**

> `hash {peer -> tree -> (Target_hash)#AnchorPoint}` — CAS-Reads?
>
> *"The subdocument addressing — I guess that was anchor points, essentially, in the HTML form. We can
> work with something there. The problem is an offset and length… if I'm known to a hashed reference.
> But I could hash a tree redirect — because say I want to say 'hey, check out this document', but
> it's a document I continuously update, and then I drop an anchor in there. I may change other parts
> of the document, so the hash is evolving. **The hash of the link stays the same — it's this peer,
> this tree path.** … So this kind of tuple tells me where to go — peer, tree — and then **what to
> expect when you get there.** So it's kind of CAS on this global distributed peer-to-peer that may
> change. You can still 404 on that, or the peer-to-tree path changed. Well, they published at this
> target hash — you can go and try to find it, or you can just take the latest and update your link.
> And then maybe the anchor point does or doesn't exist. So there's a lot of different ways that can
> break."*

---

## §0 The result, and it corrects yesterday's finding

**`EXPLORATION-THE-HYPERTEXT-LINEAGE-…` §6.3 published: *"we do not have span addressing; the fix is
`{reference, offset, length}` over immutable bytes."*** That was right about the mechanism and **wrong
about the problem**, and the operator's sketch is why.

**Offset-and-length only works against a hash you have pinned.** It is exact, permanent — and it can
only ever point at a **frozen snapshot.** The use case that actually matters is the opposite one:
*"check out this document, at this spot, and I will keep updating the document."* **An offset into a
document that evolves is not stale, it is silently wrong** — it will resolve, to different words.

**The correct decomposition has three parts with three different stabilities:**

| Part | What it is | Who maintains it | Stability |
|---|---|---|---|
| `(peer, path)` | **the durable name** — where to go | the **publisher** | survives every edit |
| `target_hash` | **the expectation** — what you should find | the **linker**, at link time | frozen by construction |
| `#anchor` | **the spot inside** | the **author of the document** | survives edits *if the author preserves it*, and that is the right party to depend on |

**And the second result, which is the one with a cost:** our reference atom today is
`{peer, hash, path?}` with **`hash` required and `path` an explicit non-address** — *"the hash is the
claim; the locator is a convenience… a reader MUST NOT treat a failure to fetch at `path` as evidence
the entry does not exist."* **That atom cannot express the operator's use case at all.** It is a
snapshot pin, and the living-document reference has no shape. §3.

---

## §1 Why an anchor beats an offset — and it is the whole Xanadu lesson, inverted

**Xanadu's hardest problem was making a sub-document address survive an edit.** They solved it with
tumblers, enfilades, and the I-to-V / V-to-I transforms: **a machine that maintains the mapping** from
a permanent address to a current position, so a link into character 101 still lands on the same words
after 67 characters were inserted above it. It is elegant, it is the bulk of forty years of
engineering, and **the full system never shipped.**

**HTML made the opposite choice and it is still working thirty years later.** `<a name>` / `id="…"` —
**the author declares the anchor, and the author preserves it across edits.** No machine maintains
anything. If the author restructures and drops the anchor, the link breaks visibly, and that is
acceptable.

**Hypothes.is is the third point and it shows what the middle costs.** Because they anchor into
documents whose authors never cooperated, they need `TextQuoteSelector` (the quote plus ~32 characters
of prefix and suffix) **and** `TextPositionSelector` **and** a fuzzy last-resort match through
diff-match-patch/Bitap — with **confidence scores** and an **orphan** state for annotations that no
longer land. Positions drift even without edits, because a page can inject different advertising text
between two loads.

> **The rule the three of them establish together: a sub-document address is maintained by exactly one
> party, and the only party who can maintain it correctly is the one who edits the document.**
> Xanadu tried to make it the system's job and it was too hard. Hypothes.is makes it the reader's job
> and pays in heuristics and orphans. HTML makes it the author's job and it works.

**The operator's sketch chooses the author, which is the choice that shipped.** And in our substrate it
is stronger than HTML's, because the anchor is not the only thing carried — the `target_hash` beside it
means **a reader can always tell whether the anchor it is looking at is the one the linker saw.** HTML
has no way to know that. **That is a genuinely new capability and it comes from having two naming
layers.**

> **But "an HTML-style label" is the wrong first answer here, and §5 is corrected accordingly.**
> `[operator, 2026-09-04: "what an anchor is is a bit confusing, because theoretically that's a
> higher-level L5 application interpretation. Entities are entities, they're hashed fields. Field
> referencing kind of works — if that's what it is, a reference and then a specific field, that should
> be somewhat stable. But an anchor that's not a field… entity types understand fields, so that's
> native to our interpretation."]`
>
> **HTML needed to invent `id=` because HTML has no type system.** We have one, and **a field is
> already a named, typed, stable sub-part of an entity.** So the native anchor is **a field path**, and
> the L5 label is a *second, higher* rung that only earns its place where a field path cannot reach.
> **§5 is the ladder.**

---

## §2 `strongRef` — the same design, deployed, with a published rationale

**Two of the three parts are not novel and that is good news.** ATProto ships
`com.atproto.repo.strongRef`:

```json
{
  "$type": "com.atproto.repo.strongRef",
  "uri": "at://did:plc:44ybard66vv44zksje25o7dz/app.bsky.feed.post/3jvz2442yt32g",
  "cid": "bafyreigbtj4x7ip5legnfznufuopl4sg4knzc2cof6duas4b3q2fy6swua"
}
```

**Their rationale is the operator's sentence in different words:** the AT-URI is *where to find it*,
the CID is *what you should find*. Neither works alone — *"content-addressed hashes aren't resolvable
by themselves in atproto; there's no global DHT you can throw a bare CID at"*, and a bare URI is *"a
live pointer with no integrity guarantee."*

**The named attack is worth carrying because it is the reason this is not merely tidy: the rug-pull.**
Someone posts something benign, you quote it or report it, they then `putRecord` over the same rkey
with different content — *"your quote now endorses, or your moderation report now describes, content
that was never there."* **The pinned hash makes the reference non-repudiable**, which is why their
moderation reports carry a `strongRef` rather than a URI: the moderator must adjudicate *the exact
bytes that were reported.*

**And the third part — the anchor — is a slot they reserved and never filled.** The AT-URI spec:
fragments are *"supported in general syntax, not currently used"*, with **JSON Path syntax reserved
for future fragment support.**

> **So the operator independently derived a three-part reference of which two parts are deployed prior
> art with a published rationale, and the third is a reserved-and-empty slot in that same prior art.**
> That is about as good a signal as a design gets.

**One divergence in emphasis, and it matters (§3).** ATProto treats the **URI as the address of record
and the CID as a version marker**. We do the reverse. Their own guidance names both intents:
> *"Use a strongRef when the **specific version** matters — replies, quotes, reports, likes,
> signatures. Use a bare AT-URI (a **weak ref**) when you genuinely want to track the live record…
> If you're modeling something that legitimately mutates, a weak ref plus a resolution step is often
> the better fit than a strongRef you have to keep rewriting."*

---

## §3 The gap: two reference intents, one shape

**Our atom** (`PROPOSAL-APP-CONVENTION-FEED` §2.2):

```cddl
reference = {
  peer: peer-id,                     ; WHO published it
  hash: content-hash,                ; WHAT it is — the assertion
  ? path: tree-path                  ; WHERE they put it — a hint, and OPTIONAL
}
```

with the prose: *"The hash is the claim; the locator is a convenience… `path` is optional and a reader
MUST NOT treat a failure to fetch at `path` as evidence the entry does not exist; it is a starting
point, not an address of record."*

**That is a strongRef, and a good one.** For its derivation case it is exactly right: §2.2 reached it
from the reply problem — *"one lineage references a reply by location alone, which makes a reply
exactly as trustworthy as whatever currently answers that location and lets a parent be edited
underneath its replies."* **Correct, and the rug-pull is the same attack ATProto names.**

**The defect is that the atom was derived from the reply case and then adopted as the universal
shape.** Two intents exist:

| Intent | Example | What must be authoritative | Expressible today? |
|---|---|---|---|
| **"this exact thing"** | a reply, a quote, an attestation, a moderation subject, a signature target | **the hash** | ✓ — this is the atom |
| **"whatever is at this place now"** | *"check out this document"* — a living page, a profile, a maintained index, a long-lived list | **the path** | **✗** |

**With `hash` required and `path` declared not-an-address, the second intent has no shape.** A
publisher who links to their own continuously-updated document either pins a hash that goes stale the
next time they edit, or re-writes every referring entry on every edit — which is precisely the
back-chain cascade `FEED` §2.3.2 removed `prev` to avoid, arriving through a different door.

**This is not L25** (a tightening that closes no hole) — the tightening closes a real hole, for
replies. **It is narrower and more ordinary: a rule derived from one case, correct there, generalized
to a domain containing a second case with the opposite requirement.** The tell is that §2.2's
justification talks exclusively about replies and parents.

---

## §4 The three-way read — the state machine, and the *comparison result is information*

**This is what the operator's "CAS-Reads?" is asking, and the pun is exact: it is Content-Addressed
Storage *and* compare-and-swap.** You go to `(peer, path)`, you compare what you got against
`target_hash`, and **the comparison outcome is a thing the reader should be told, not something the
system should paper over.**

| # | `(peer, path)` | hash comparison | anchor | Meaning | Reasonable behaviour |
|---|---|---|---|---|---|
| 1 | resolves | **matches** | present | **exact** — you are seeing what the linker saw | render, jump to anchor |
| 2 | resolves | **differs** | present in current | **the document evolved** | render current, **say so**, offer the pinned version |
| 3 | resolves | **differs** | absent in current, present in pinned | **the author moved or removed the anchor** | strongest available signal — render current at top, offer pinned-with-anchor |
| 4 | resolves | **differs** | absent in both | evolved, and the anchor is gone or never existed | render current, note the anchor did not resolve |
| 5 | **404** | — | — | path moved or was unpublished | **fall back to `target_hash`, fetched from the publisher's declared content origin or any reachable source that has it** — the hash validates the bytes whoever serves them. *(This read "fetched from anywhere — the author, a mirror, a cache, a stranger". There is no operation answering* who has this hash?*, so "a stranger" was unreachable; what exists is the transport-profile → content-origin composition, which is a real normative path and still the property the comparison below claims.)* |
| 6 | 404 | pinned hash **also unobtainable** | — | genuinely dangling | the honest `404`. Nothing to do and nothing to hide |
| 7 | resolves | matches | **anchor absent** | the linker's anchor was already wrong | render, note it |

**Three things fall out of that table and each is a design position we do not currently hold.**

**(a) Row 5 is where the second naming layer earns its keep, and it is the row nobody else has.**
ATProto cannot do row 5 — *"there's no global DHT you can throw a bare CID at"*, so a moved record is
simply gone. **We can**, because a content hash resolves against **any** store: our own, a mirror's, a
cache's. **A reference that survives its author unpublishing it is a property of having content
addressing under the naming layer, and it is exactly Hyper-G's celebrated "restore the document and
the links come back" — obtained without a link database.**

**(b) Row 2 versus row 3 is the strict/lenient policy split, and it should be the reader's, declared.**
ATProto has this as *guidance* (strict for moderation and attestation, lenient for embeds) and Bluesky
leans lenient while keeping the CID as an audit trail. **We have no statement at all.** And our own
capstone §5 already says what the answer should be: *"a view declares what produced it — which
ranking, which mirror set, which exclusions. A view that names its own provenance is debuggable; one
that does not is indistinguishable from a bug."* **A reference that resolved to different bytes than
the linker saw is exactly that class of fact.**

**(c) The failure modes the operator listed are all in the table and none of them is silent.** *"So
there's a lot of different ways that can break"* — yes, and **every one of them breaks visibly**,
which is the property Xanadu tried to eliminate (make the change invisible; the link follows the text)
and Hyper-G tried to prevent (integrity enforced; deletion propagates). **We make it visible. That is
the honest third option** and it is the only one of the three that does not require either a machine
maintaining a mapping or a database owning the link space.

---

## §5 The granularity ladder — and the native rung is a field path, not a label

**Rewritten 2026-09-04 on the operator's correction.** The first version went straight to an
HTML-style label on an embed node. **That skipped the rung our substrate already has.**

**Four rungs, increasing precision and decreasing stability:**

| # | Rung | What it addresses | Stable under | Native? |
|---|---|---|---|---|
| **0** | **the entity** — `{peer, hash, path?}` | the whole thing | everything | ✓ **this is today's atom** |
| **1** | **a field path** — `+ field` | a named, typed sub-part | **any edit that does not remove or rename that field** | ✓ **the type system already names it** |
| **2** | **a named node inside a body** | a spot in an authored structure | edits that preserve the label | ✗ **L5 — the convention must define a name slot** |
| **3** | **a byte offset + length** | an exact span | **nothing** — only a pinned hash | ✓ mechanically (ECF is deterministic), ✗ semantically |

### §5.1 Rung 1 is the one to look at, and it is nearly free

**A field name is a stable identifier that survives a content change**, which is exactly the property
a byte offset lacks and the property an anchor needs. If I reference `title` and you rewrite `body`,
**my reference still means what it meant** — even though the entity's hash moved.

**That is the whole point and it is worth stating sharply:** *the hash changes, the field path does
not.* So a field-path anchor **degrades correctly through §4's three-way read** — on a hash mismatch
you can still ask *does this field still exist, and is it still this type?*, which is a far better
question than *are bytes 400–460 still the same sentence?*

**And it is not a new mechanism.** Field paths already exist and are already load-bearing —
`compute/lookup` walks them, `system/capability` scopes over paths, the type system validates by
field. **A field-scoped reference is composing two things the corpus already has.**

### §5.2 Rung 3 works, and only against a pin

**The operator is right that it is mechanically well-defined:** *"theoretically, in entity CBOR you do
have a deterministic byte sequence, so you could drop right in there at an offset, assuming it's a
stable reference."* ECF is canonical, so byte *N* of an entity is a fact, not an implementation detail.

**And right about the limit:** *"but that target hash may change — I change my blog, I update it, I
make an edit. If I'm not using some sort of application-aware anchoring within the data model, these
byte offsets may not make sense."*

> **So rung 3 is exact-and-frozen and rung 1 is approximate-and-durable, and they are for different
> jobs.** A byte span is right for *"quote exactly these words as they stood"* — an attestation, a
> moderation subject, a citation in an archive. A field path is right for *"look at the abstract of my
> paper"*, where the paper keeps changing and the abstract keeps being the abstract.

### §5.3 Rung 2 is real but it is an application concern, and that is why it felt confusing

**The operator's instinct is right and there is a structural reason for it.** An entry's `body` is an
`embed-node` tree (`APP-CONVENTION-EMBED`: four leaves plus one `box` container). **A field path can
reach a node's *position* — `body.children[3]` — but a position is an index, and an index breaks on
insertion.** To be stable, a node inside a body needs **a name**, and nothing in EMBED gives one.

**So rung 2 requires the convention to mint a name slot on embed nodes** — an application-level
decision, not a core one, which is exactly why *"an anchor that's not a field"* reads as an L5
interpretation. **It is coherent; it is just not free, and it is not the first thing to build.**

**If it is ever built, three things it would inherit for free:** EMBED's *unknown → drop clean, show
`fallback`* discipline covers renderers without anchor support · it mints **no new entity type** (a
consumer's required behaviour does not change, which is the taxonomy floor's own test) · and it is
**author-maintained by construction**, because the node is authored — §1's rule satisfied by where the
field sits.

### §5.4 What this ladder actually recommends

**Nothing is proposed here.** But the ordering is now clear enough to state:

- **Rung 1 (field path) is the cheap, native, high-value rung** and it is the one worth a real look,
  because it composes existing machinery and needs no new application vocabulary.
- **Rung 3 (byte span) is worth having only where the reference is already pinned**, and there it is
  exact and correct. It is not an alternative to rung 1; it answers a different question.
- **Rung 2 (named node) waits for a consumer**, per L26 — and its precondition is a name slot in
  EMBED, which is a convention change, not a core one.
- **All three sit *beside* §3's unresolved question**, which is more urgent: the atom cannot yet
  express *"whatever is at this place now"* at **any** granularity, and that is the gap under review.

---

## §6 What this leaves open

1. ~~**The atom needs a second intent, or an explicit refusal.**~~ **DECIDED — option (b), a distinct
   shape.** The three options were *(a)* make `hash` optional, so `{peer, path}` alone is a live
   pointer; *(b)* a distinct shape for a live reference; *(c)* declare the living-document case out of
   scope.
   **(a) is eliminated on our own reasoning** — the taxonomy's inverse guard says collapsing two
   behaviours into one shape pushes the distinction into a value nothing checks, and here the failure is
   silent in exactly that way: a reference arriving without a hash is indistinguishable between *"the
   author wants live"*, *"the implementation did not populate it"* and *"the author only had a URL."*
   One intent, two bugs, and the rug-pull guarantee degrades to a convention.
   **(c) is eliminated by §4's own table** — row 5, where the path 404s and the last-seen hash is still
   fetchable from anywhere, is a capability none of the surveyed systems has, and declaring the case out
   of scope discards it.
   **Corroboration, verified in four primary specs and cited last:** ATProto (`strongRef` vs a bare
   AT-URI), Nostr (`e` tag vs `a` tag), Matrix (`event_id` vs state keyed by `(type, state_key)`) and
   SSB (`%message` vs `@feed`) **all carry both intents and all four express the difference as a
   distinct shape or name — none as an optional field.** If all four had done the opposite the
   derivation above would be unchanged. `EXPLORATION-THE-FALSIFICATION-TEST-…` §4.
   **Landed** as `PROPOSAL-APP-CONVENTION-FEED` §2.2.2's `live-reference`, with the site rule (§2.2.3)
   that keeps `reply` pinned — so the rug-pull guarantee is undiminished at the site that needed it.
2. ~~**The read policy is unstated.**~~ **LANDED.** §4. Strict/lenient is a reader choice; *that the
   reader must be able to tell* is a format obligation, and it is the half that needed a MUST. It is now
   `PROPOSAL-APP-CONVENTION-FEED` §2.2.4 — a view built from a `live-reference` whose resolved hash
   differed from `seen` MUST make that fact available to the view, which is the capstone's
   *a view names its own provenance* rule applied to references.
3. **The anchor granularity** — §5's ladder. **Rung 1, a field path, is the native and cheap one** and
   the only rung worth examining before a consumer exists; rung 3 (byte span) is exact but only
   against a pin; rung 2 (a named node in a body) needs EMBED to mint a name slot and is an
   application decision. **None of the three is as urgent as (1) above.**
4. **Nobody has asked for any of this.** Recorded honestly: no seat has hit it, and **L26 cuts toward
   letting the seats discover by building** rather than arch minting a field ahead of a consumer. **The
   one thing that argues for moving sooner is §3** — the atom is under review *right now*, and a
   missing intent is cheaper to fix before two implementations ship against it than after.

## §7 Sources

[ATProto — `com.atproto.repo.strongRef`](https://atproto.blue/en/latest/atproto/atproto_client.models.com.atproto.repo.strong_ref.html) ·
[ATProto — AT-URI scheme](https://atproto.com/specs/at-uri-scheme) ·
[ATProto — Lexicon Style Guide (strongRef guidance)](https://atproto.com/guides/lexicon-style-guide) ·
[*Why are there 3 different ways to embed a CID in a record?* — atproto discussion #2190](https://github.com/bluesky-social/atproto/discussions/2190) ·
[*Record References: Authoritative vs Unauthoritative Patterns*](https://blog.smokesignal.events/posts/3lvbownlrme2a-atprotocol-record-references-authoritative-vs-unauthoritative-patterns) ·
[Hypothes.is — Fuzzy Anchoring](https://web.hypothes.is/blog/fuzzy-anchoring/) ·
[W3C Web Annotation Vocabulary](https://www.w3.org/TR/annotation-vocab/) ·
[Nelson — *Xanalogical Structure*](https://xanadu.com.au/ted/XUsurvey/xuDation.html)

**In-corpus:** `PROPOSAL-APP-CONVENTION-FEED` §2.2 (the atom), §2.3.2 (why `prev` is opt-in — the
same cascade argument arrives here through a different door) · `APP-CONVENTION-EMBED` §4–§6 (where an
anchor would live) · `EXPLORATION-THE-HYPERTEXT-LINEAGE-…` §6.3 (**the finding this document
corrects**) · capstone §5 (*a view declares what produced it*).
