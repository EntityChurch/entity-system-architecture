# EXPLORATION — the second falsification test: the wiki, the forum, the marketplace and the collaborative document against the floor

**Status:** Exploration (design record). Not a proposal, not normative.

**What this is.** `EXPLORATION-THE-FALSIFICATION-TEST-FOUR-REAL-OBJECTS-AGAINST-THE-FLOOR` mapped four
foreign objects onto the four content shapes and closed with its own limit, stated twice: *"four
microblog-lineage objects… a wiki page, a marketplace listing, a map annotation and a collaborative
document are untested, and `[X-13]` is where a real falsification would come from."* **This is that
test.** Four objects from outside the microblog lineage, each read from a primary schema or source
fetched for this document.

**It is deliberately the harder test.** The first four objects were all *one person publishing one
thing*, which is the case the floor was built for. Every object here is contended in some way, which is
where a floor breaks if it is going to.

---

## §0 The result

**Three of the four absorb. One falsifies, and the falsifier is not a content shape.**

| Object | Verdict |
|---|---|
| **The wiki page** | **absorbs — and it is two different products**, each with its mechanism already specified (§2) |
| **The forum topic** | **absorbs — confirming taxonomy §5.1 against two deployed systems**, and the corpus turns out to hold *both* of the field's answers (§3) |
| **The marketplace order** | **FALSIFIES (§4)** |
| **The collaborative document** | **absorbs at the seam already named**, and the rung-3 cost now has a primary citation instead of an argument (§5) |

**The falsification, stated precisely, because its shape matters more than its existence.** It is not a
fifth *body*. It is that **every shape in the floor has exactly one signing party, this was never
written down as a property, and no microblog object could have surfaced it** — because nothing in a
microblog is ever co-signed.

> `PROPOSAL-APP-CONVENTION-FEED` §1.1: ***"An entry's `author` MUST equal the peer namespace it is
> authored under. An entry found under one peer's namespace claiming a different author is invalid and
> a conformant reader MUST reject it."***
>
> **A two-party record cannot be an entry.** Not "fits awkwardly" — a conformant reader is required to
> reject it. And the three collection shapes do not help: the stream and the authored set are *the
> author's own*, and the gathered view **claims nothing**, which is exactly the claim an agreement has
> to make.

**So there is a fourth provenance cell and the taxonomy's §4.4 termination argument missed it.** That
section asks *"who assembled this list and what does it therefore claim?"*, answers *"three, and it
terminates"*, and the three answers are all **one party**. The fourth answer is **the parties jointly,
and it claims completeness exactly when every named party has signed** — which, unlike a gathered
view's completeness, is **verifiable**, because the party set is named in the record and each party's
signature sits at an invariant path in their own namespace.

**The good news is where the work is.** The substrate is built: V7 already binds signatures as separate
entities at `/{signer}/system/signature/{hex(target_hash)}`, so any entity can carry any number of
signatures from any number of namespaces, and `EXTENSION-QUORUM`'s `verify_k_of_n_signatures` already
validates K-of-N over an arbitrary entity. **What is missing is one L5 shape, not a mechanism** (§4.4).

---

## §1 Method, and what this does not claim to be first at

**Every foreign fact below comes from a primary schema, specification or source file fetched for this
document** — the MediaWiki manual pages, Discourse's annotated `posts` schema, Lemmy's federation
document, the `openbazaar-go` `contracts.proto`, the Automerge binary-format specification. Where a
source was thin, it is said so rather than smoothed over.

**And the honest placement, because the corpus already has verdicts on all four.**
`EXPLORATION-THE-INTERACTION-LANDSCAPE-…` §3 grades wikis, collaborative documents, code forges and
marketplaces **🟡 partial** and gives the determining property for each — *"a shared wiki is contended
and needs merge policy"*, *"listings yes, settlement no"* — and §4.1 already rules that collaborative
editing is not a fifth content shape. **None of that is being re-derived here.** What those verdicts do
not do is check the field-level object against the corpus's own mechanisms, and in three of four cases
that check moves the verdict: **twice because the mechanism turned out to already exist, once because
the object turned out to break a rule nobody had noticed was a rule.**

---

## §2 The wiki — and it is two products that share a name

### §2.1 MediaWiki is rung 2, and its mechanism is ours line for line

**Read from the manual rather than from the idea of a wiki**, the mechanism is small and specific:

| MediaWiki | Source | Ours |
|---|---|---|
| a page is `page_id` + `page_latest → rev_id` | `Manual:Database_layout` | a path bound to a version head |
| a revision carries **`rev_parent_id`** — the previous revision | `Manual:Revision_table` | a version entry's `parents` |
| **`rev_sha1`** — *"the SHA-1 text content hash… a nested hash of hashes of `content_sha1` across all slots"* | `Manual:Revision_table` | the trie root; content addressing, arrived at independently |
| an edit submits **`basetimestamp` / `baserevid`** — *"the base revision, used to detect edit conflicts"* — and fails `editconflict` if the head moved | `API:Edit` | **CAS on the head pointer** — `EXTENSION-REVISION` §8.1 |
| *"MediaWiki will automatically merge edits that touch unrelated parts of a page, and will only trigger an edit conflict if multiple users attempt to edit the same lines"* | `Help:Edit_conflict` | **three-way merge, then surface the conflict** — REVISION §5.2, and §2.2's **conflict entity** |

**That is the whole of it.** The largest wiki in the world runs compare-and-swap on a base revision,
attempts a three-way merge, and shows a human the conflict when the merge fails. **`EXTENSION-REVISION`
specifies exactly that** — §8.1 requires CAS+retry or single-writer serialization and declares
implementations that let concurrent head advances overwrite each other **non-conformant**; §5.2 makes
three-way the built-in default; §2.2 makes a conflict a stored entity, so the document never
disappears while the disagreement is open.

> **So the interaction landscape's *"a shared wiki is contended and needs merge policy"* is correct and
> incomplete: the merge policy is specified, and has been since REVISION v3.13.** The wiki is not
> blocked on a data-model question.

**What MediaWiki has that we do not is one server.** Its CAS is against a single authoritative head,
which is what makes "the current article" a well-defined object. Ours is CAS against *a* head, per
peer, with convergence provided by the version DAG (§7.2) rather than by a single writer. **That is the
whole difference, and it is the difference between the two products below.**

### §2.2 Federated Wiki is the rung-0/1 route, and it is deployed

Ward Cunningham's Smallest Federated Wiki answers the same question the other way, and its answer is
ours:

- **every page is owned by one site** — single-writer, no shared object;
- **editing someone else's page means forking it onto your own wiki** — *"the editor has cloned the
  remote page, making it his/her own. Further edits occur in place"*;
- **the journal** — the page's edit history — *"is assumed to be detailed enough to recognize where in
  the journal of the original the fork took place"*;
- **the neighborhood** is the set of sites pulled into view while browsing, so a reader sees several
  sites' versions of the same-named page, with conflicting actions highlighted;
- and the outcome is named honestly by its author: **not consensus, but "a Chorus Of Voices."**

**Both halves of that are already specified here, and neither was written with a wiki in mind:**

| Federated Wiki | Ours |
|---|---|
| fork a page onto my own site | **REVISION §7.3 Cross-Prefix Import** — *"the same version entry, the same DAG, different mount points… this is the git model"* |
| the journal locates the fork point | the version DAG's shared ancestor — §3.3 common-ancestor finding |
| the neighborhood polls sister sites | **REVISION §6.3** — subscribe to the other peer's `system/revision/{H}/head`, fetch, merge locally; *"bidirectional sync… each peer fetches what it doesn't have"* |
| the original author may clone the revisions back | the same import, in the other direction — and because B imported from A, **the graft cost §8.7.3 warns about does not apply**: they already share an ancestor |

### §2.3 So the wiki's real question is a product question, not a model question

**Two products, and they are not variants of each other:**

1. **The chorus wiki** — every page owned, forks visible, no canonical version. **Fully expressible
   today** with `app/site-page` + REVISION + a gathered view. Nothing is missing but a convention and a
   renderer that shows a neighborhood.
2. **The canonical wiki** — one article the world edits toward. **Also expressible**, and it requires
   somebody's namespace to be the one everybody merges into, plus a **propose-back path** so a
   non-owner's edit can reach the owner. `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4 already names that
   arc — *"the propose-back/edit arc"* — as `[GRADIENT][DEFER build]`, so the gap is recorded; what it
   is missing is that the propose-back is not a new mechanism either. **It is B's fork, plus a
   reference telling A where to look**, and A merging is REVISION §6.3 run once.

**The honest limit, which is social rather than technical.** The canonical wiki's hard problem is not
merging text; it is *who decides* — edit wars, protection, vandalism reversion. That is the authority
question, and `EXPLORATION-THE-PUBLIC-SOCIAL-STACK` §5 already gives the only answer available here:
**exclusion, not removal**, and *"choosing whose mirrors to read IS choosing whose moderation to
accept."* A canonical wiki in this system is canonical **to whoever chose to read that namespace**, and
that is a real product with a real limit rather than a failure.

---

## §3 The forum — the corpus holds both of the field's answers

**Taxonomy §5.1 ruled that a forum post is an entry with `reply` + `context`, a thread is a gathered
view, and the forum needs no new entity type.** It called this *"the strongest result in the document…
worth stating carefully, because it sounds too convenient."* Tested against two deployed systems, it
holds — and each system contributes one thing the ruling did not have.

### §3.1 Discourse — the one thing we cannot reproduce, and do not need

From the annotated schema of the `posts` table: a post carries **`topic_id`**, **`post_number`**,
**`reply_to_post_number`**, `raw`, `cooked`, `version`, `deleted_at`, plus roughly twenty denormalized
counters (`like_count`, `reply_count`, `reads`, `incoming_link_count`, `quote_count`, …).

**`post_number` is a per-topic sequential counter**, which is why a Discourse URL is
`/t/slug/:topic_id/:post_number`. **It is a shared monotonic counter across many writers — a uniqueness
invariant spanning writers, which is the ladder's rung 4, and it requires the server that assigns it.**
We cannot produce one and should not try. **We also do not need it**, because it is doing addressing
work that a content hash does better: `post_number` is a name for a post, and we already have one that
does not require an assigner.

**The counters are the second half of the same observation.** Every one of them is a global count, and
`EXPLORATION-THE-INTERACTION-LANDSCAPE-…` §5.2 already has the disposition: **not closed, convergent —
a count over your coverage, which can only grow as you read more mirrors.** Discourse stores them
because it *has* the whole set; the honest version here is a number with a coverage figure attached.
**Neither of these is a data-model finding, which is the point: the entire delta between a forum post
and our entry is addressing and counters, and both were already ruled.**

### §3.2 Lemmy — the field's answer to "a subject nobody owns" is to give it an owner

Lemmy's federation document: a **Community is an ActivityPub `Group`**, a user is a `Person`, a post is
a `Page`, a comment is a `Note`. A user sends `Create/Page` **to the community**, and *"if the Community
receives any Post or Comment related activity… it will forward this to its followers"* — wrapped in
`Announce`. **The community is an actor with a key, hosted on one instance, and it is the distribution
authority for its own content.**

**That is a real answer with a real cost**, and it is the cost this project exists to avoid: the
community lives on an instance, so the instance's admin is the moderator, the outage, and the
deletion. **The taxonomy's answer — a topic named by the content hash of its subject, with the thread
as a union — has no owner and therefore none of those failure modes**, and pays for it in navigation:
you can compute the topic identifier without coordinating, and you still have to find somebody who is
in it.

**What was not noticed until this test: the corpus has Lemmy's answer too, and in a better shape.**
`EXTENSION-GROUP` v1.4 defines a group as *"an identity that represents multiple individuals
collectively"* — its own quorum, controller and agents, with membership entities, five governance
patterns, and `attest_acting_on_behalf` for a member speaking as the group. **A forum community can be
a group peer that publishes a curated stream, moderated by K-of-N rather than by one instance admin.**

> **And the two answers are not competitors — they compose, which is the actual result of this
> section.** The **union over the topic hash is the substrate** (nobody owns it, nobody can delete it,
> anyone can compute it). **A group's published stream is a curated view over that union** — which is
> exactly `EXPLORATION-THE-PUBLIC-SOCIAL-STACK` §5.3's *"curation and moderation are the same act"*,
> arriving with an owner who is a K-of-N quorum instead of a hosting company. A reader who trusts the
> group reads its stream; a reader who does not walks the union. **Nothing about the first prevents the
> second**, which is precisely what fails on Lemmy, where defederation removes the content.

**The forum's remaining problem is unchanged and is still the right one:** discovery and navigation —
finding anyone at all who is in a topic. `EXPLORATION-WHAT-CONVERGES-…` §4.4 already says *"what is
actually missing is navigation, not aggregation."* This test found nothing to add to that, and it did
find that a group-published topic index is one of the two obvious ways to answer it.

---

## §4 The marketplace — and the empty cell in our own 2×2

**This is the falsification.** It is not the listing, and it is not settlement.

> **Scope, corrected on operator challenge and worth reading before the section.**
> `[operator, 2026-09-05: "what does this even mean as a marketplace… it's not like we're building the
> commerce platform or building a currency exchange… I post a collection of stuff and if you want to
> contact me."]`
> **That reading of a marketplace is right, and it is what §4.1 concludes: a listing is an entry, a
> shopfront is a collection, and contact is out of band.** No commerce direction is proposed here and
> none should be read into this section.
> **The marketplace order was chosen as a TEST OBJECT, not as a product** — it is the most heavily
> co-signed everyday record that has a published schema to read, which makes it the sharpest available
> probe of a floor built entirely from singly-authored objects. **The finding is about co-signing, not
> about commerce**, and its real consumers are the ones the operator named in the same message:
> **vouching, conversations and threads, groups, and co-authorship** (§4.5). If the word *marketplace*
> is doing any work in this document beyond *"here is a schema with two signers in it"*, that is a
> defect in the framing rather than a direction.

### §4.1 The decomposition, so the finding is not confused with the parts that were already ruled

| Part of a marketplace | Verdict | Where |
|---|---|---|
| **the listing** | **an entry** — schema.org's `Offer` is `itemOffered` + `price` + `priceCurrency` + `availability` + `seller`, which is a body with fields | taxonomy §1 corollary 1 |
| **the catalog / search** | **closed** — nothing here produces global search, and it has no owner | interaction landscape §3, §5 |
| **inventory, "3 left", auctions** | **genuinely closed** — a contended scarce resource needs an arbiter | interaction landscape §5.1 |
| **money, escrow release** | **an arbiter, and it can be a K-of-N one** — `GUIDE-MULTISIG` §4.2 already works the buyer/seller/agent case | in corpus |
| **reputation** | **rides on `system/attestation`** — `SYSTEM-IDENTITY-COMPOSITION` names reputation as a consumer of the same signed-edge primitive, with `properties.kind = "reputation"` | in corpus, unbuilt at L5 |
| **the order** | **has no shape** | **§4.2** |

### §4.2 What an order actually is, read from the protobuf

`openbazaar-go`'s `pb/protos/contracts.proto` — the only fully specified decentralized marketplace this
ecosystem has to read — makes the order **one growing document with per-section signatures**:

```protobuf
message RicardianContract {
    repeated Listing vendorListings                    = 1;
    Order buyerOrder                                   = 2;
    OrderConfirmation vendorOrderConfirmation          = 3;
    repeated OrderFulfillment vendorOrderFulfillment   = 4;
    OrderCompletion buyerOrderCompletion               = 5;
    Dispute dispute                                    = 6;
    DisputeResolution disputeResolution                = 7;
    Refund refund                                      = 9;
    repeated Signature signatures                      = 10;
}
message Signature {
    Section section      = 1;      // LISTING | ORDER | ORDER_CONFIRMATION | ORDER_FULFILLMENT
    bytes signatureBytes = 2;      // | ORDER_COMPLETION | DISPUTE | DISPUTE_RESOLUTION | REFUND
}
```

**Three properties, and each one breaks a different thing:**

1. **Alternating authorship in one record.** The vendor signs `LISTING`, the buyer signs `ORDER`, the
   vendor signs `ORDER_CONFIRMATION`. **Section-scoped signatures exist precisely because no single
   party can sign the whole thing.**
2. **It is a state machine, and validity is a property of the sequence.** An `ORDER_FULFILLMENT`
   without a preceding `ORDER_CONFIRMATION` is not a short view; it is invalid.
3. **A third party — the moderator — may sign too**, and the escrow is 2-of-3. The arbiter is scoped to
   the *money*, not to the record.

### §4.3 The derivation, from our own test, before any of that is cited

**The taxonomy's own §1 test:** *two things are different entity types if and only if a conformant
consumer holding one must behave differently.* Hold an order beside an entry:

| | an entry | an order |
|---|---|---|
| **Signers** | exactly one, and it **MUST** equal the namespace (`FEED` §1.1) | two or more, in different namespaces |
| **To validate** | verify one signature over the bytes | verify **each party's signature over its own section**, and that the sections are in a legal order |
| **Completeness** | not a question — an entry is whole | **the question** — an unsigned counterpart means *no agreement*, not *a short view* |
| **Where it lives** | the author's namespace, by rule | **neither party's namespace is privileged**; both hold it |
| **What a missing half means** | nothing | **the record does not yet exist** |

**Four different behaviours, so by our own test this is a shape.** And the inverse guard applies with
unusual force: collapsing it into an entry with a `signatures` field would push *"is this agreed?"* into
a value that nothing checks — and the failure mode is a consumer displaying a one-sided order as a
completed sale.

**Then the 2×2 that was always there.** The taxonomy types collections by *who assembled this and what
does it therefore claim*, and answers **three**:

| | **claims completeness** | **claims nothing** |
|---|---|---|
| **one party** | the **authored set** (§4.3) | the **stream** (§4.1) — unbounded, cursored |
| **many parties** | ← **empty** | the **gathered view** (§4.2) |

**The empty cell is the agreement**, and the reason four microblog objects could not find it is that
**an ATProto post, a Nostr note, a Matrix message and an SSB post are all singly-authored** — the first
falsification test was structurally incapable of reaching this, in the same way the taxonomy's own ten
named categories were structurally incapable of testing its floor. *(OpenBazaar is corroboration, cited
after the derivation and verified in its own `.proto` this session, per L18. If it had done the
opposite, §4.3's table would be unchanged.)*

### §4.4 What is already built, which is most of it

**The mechanism is complete at the substrate and nobody has assembled it:**

- **A signature is a separate entity in the signer's own namespace**, at the core protocol's invariant
  pointer path `/{signer_peer_id}/system/signature/{hex(target_hash)}` (`EXTENSION-ATTESTATION` §4.0,
  quoting V7). **So one entity can carry N signatures from N namespaces with no new type**, and each
  party keeps custody of their own assent.
- **Completeness is therefore checkable**, which is the property the gathered view lacks: the record
  names its parties, and for each party you look at one invariant path. *A gathered view cannot tell
  you what it is missing; an agreement can.*
- **`EXTENSION-QUORUM`'s `verify_k_of_n_signatures` already validates K-of-N over any entity**, and
  `system/quorum` is *"a special node in the signed graph"* whose signature is K of N. A 2-of-2 buyer
  and seller, or a 2-of-3 with a moderator, is the existing primitive.
- **`EXTENSION-ATTESTATION` is the signed edge**, with an **open** `properties.kind` vocabulary and
  reputation named as a future consumer.

**And what is genuinely absent, stated as a negative I searched for rather than assumed.**
`EXTENSION-TRANSACTION` is the document that would plausibly hold this, and it does not: §1.1 is
**multi-path atomic writes on one peer**, and §1.2 says so outright — *"Distributed transactions. This
extension is local — one peer. Cluster transactions (multi-peer coordination) compose on top… See §8."*
`APP-CONVENTION-SHARE`'s three tags and `FEED`'s six are all single-author. **There is no two-party
record anywhere in `specs/`, and multi-peer coordination is named as a future extension.**

### §4.5 The cell is not a commerce concern, which is why it matters now rather than at v3

**Instances of the empty cell, none of which is a marketplace:**

| Instance | Why the union is not enough |
|---|---|
| **a co-authored post** | every major platform ships this; two authors, one object, and `FEED` §1.1 forbids it outright |
| **mutual follow as a relationship** | two independent follows are a union and work fine — **but *"we are mutuals"* is only true if both signed**, and that is the gate a private audience is built on |
| **an RSVP that the host confirmed** | interaction landscape §4.2 correctly makes the RSVP a union; **a confirmed seat is a different claim** and needs the host's half |
| **a receipt** | *"I paid"* is not the same object as *"you paid"* |
| **a dispute resolution** | three parties, and the outcome is only binding if the right ones signed |

**The mutual-follow row is the one to weigh for the release**, because it is the difference between
following and a relationship, and a private audience is expressed on top of it.

---

## §5 The collaborative document — no falsification, and the cost is now cited

`EXPLORATION-THE-DATA-MODEL-LADDER-…` §8.1(c) argued that a fine-grained text CRDT on this framework
must bring its own operation log, because our version DAG is a **state** DAG — endpoint trie roots,
with operations derived by diffing — while Eg-walker replays **operations**. That argument now has a
measured neighbour.

**The Automerge binary-format specification, read this session:** *"Automerge stores the full history of
changes to the document: this is a large amount of data but in practice it is very repetitive and
amenable to compression."* A **change** is an actor id, a sequence number, the hashes of its
dependencies, and its operations. A **change chunk** *"does not include any dependent changes, so you
can only apply the change to a document that already contains those dependent changes"* — and the
format's dependency graph means **history cannot be discarded.**

**Three things follow, and none of them is a falsification:**

1. **The ladder's rung-3 price is confirmed by the field's leading implementation** — *metadata that
   grows with history; the object stops being a value you can hash and be done with.*
2. **Our §5.4 choice is a different design point, not an inferior one.** Eg-walker's principle — the
   CRDT is a computational artifact during merge, not a storage format — is what lets a document stay a
   content-addressed value at rest. **The price we pay instead is §8.1(c): the operation history is not
   recoverable from a diff**, so delete-and-retype is indistinguishable from a no-op.
3. **The seam is publication, and it was already ruled.** Interaction landscape §4.1: *"you collaborate
   in an operation log and you publish an entry."* Automerge is what the left-hand side looks like when
   somebody builds it, and its op log is **exactly the thing that does not need to be our data model**.

**One consequence for the bridge track:** an Automerge document cannot be carried into our model
faithfully by translating its current value — the value is a projection of the op log, and the op log is
the document. **Same shape as the SSB finding** (falsification test §8.1): translation is
identity-destroying, so an ingress bridge must carry the original bytes as a payload.

---

## §6 What changes on the board

| # | Item | Change |
|---|---|---|
| **X-13** | **DONE for three of the four named classes.** Games remain untested — and the interaction landscape §2 already covers them at verdict level, so the marginal value is low | |
| **NEW — the fourth cell** | **The agreement: a bounded, multi-party record whose completeness is verifiable.** Falsifies the four-shape floor; substrate complete; no L5 shape. **The first genuinely new content shape this arc has produced** | **open, and it is a proposal-shaped item, not a research item** |
| **Taxonomy §4.4** | *"three collection provenances and it terminates"* — **the termination argument is wrong**, and the correction is a 2×2 with one empty cell rather than a new list | a correction to fold into the taxonomy record |
| **`FEED` §1.1** | The one-signer rule is **normative and correct for feeds**, and is now known to be **the floor's load-bearing constraint** rather than a feed detail. Worth saying so where the floor is stated | wording |
| **Wiki** | Upgrades from *"🟡 needs merge policy"* to **two products, both with in-corpus mechanisms**; the only build gap is the propose-back arc, already named `[DEFER]` in the site convention | evidence for the site convention's own deferral |
| **Forum** | Taxonomy §5.1 **survives two deployed systems**, and `EXTENSION-GROUP` is now known to be the corpus's version of the Group-actor answer | no new work |
| **Collaborative document** | No change to the ruling; the rung-3 cost is now primary-sourced, and the bridge consequence in §5 is new | one sentence to the bridge document |

---

## §7 What this does not settle

1. **The agreement is derived, not designed.** Nothing here says what its tag is, whether the terms
   entity is authored by one party and counter-signed or assembled from halves, how a party discovers
   the counterpart's signature without being told, or what a *repudiated* agreement looks like. **Those
   are proposal questions and the seats building commerce or private audiences should be in the room.**
2. **Three of the four are single objects from single systems.** MediaWiki is not every wiki; Discourse
   is not every forum. The federated-wiki source is a project wiki and a search summary rather than a
   specification — **the weakest sourcing in this document, and it is load-bearing for §2.2.**
3. **Games are untested** and remain the one named class from `[X-13]` with no field-level read.
4. **§4.5's instances are asserted, not tested.** The co-authored post and the mutual-follow gate are
   the two most likely to matter and neither was checked against a deployed system's schema.
5. **Nothing here was executed.** Same limit as the first falsification test: no bytes were
   round-tripped, and a structural mapping is weaker than a run.
6. **The wiki's authority problem is restated, not solved.** §2.3 gives it the corpus's existing answer
   and that answer is *exclusion*, which is a real limit and not a mechanism.

---

## §8 Sources

**Primary, fetched for this document:**
[MediaWiki — `Manual:Database_layout`](https://www.mediawiki.org/wiki/Manual:Database_layout) ·
[MediaWiki — `Manual:Revision_table`](https://www.mediawiki.org/wiki/Manual:Revision_table)
(`rev_parent_id`, `rev_sha1`) ·
[MediaWiki — `Help:Edit_conflict`](https://www.mediawiki.org/wiki/Help:Edit_conflict) (automatic merge
of non-overlapping edits) ·
[MediaWiki — `API:Edit`](https://www.mediawiki.org/wiki/API:Edit) (`basetimestamp`, `baserevid`,
`editconflict`) ·
[MediaWiki — `Transclusion`](https://www.mediawiki.org/wiki/Transclusion) (render-time resolution) ·
[Discourse — `app/models/post.rb`](https://raw.githubusercontent.com/discourse/discourse/main/app/models/post.rb)
(annotated `posts` schema) ·
[Lemmy — federation](https://join-lemmy.org/docs/contributors/05-federation.html) (`Group`, `Person`,
`Page`, `Note`, `Announce`) ·
[schema.org — `Offer`](https://schema.org/Offer) ·
[`openbazaar-go` — `pb/protos/contracts.proto`](https://raw.githubusercontent.com/OpenBazaar/openbazaar-go/master/pb/protos/contracts.proto)
(`RicardianContract`, `Signature.Section`) ·
[Automerge — binary format specification](https://automerge.org/automerge-binary-format-spec/) ·
[Smallest Federated Wiki — project wiki](https://github.com/WardCunningham/Smallest-Federated-Wiki/wiki)
(fork, journal, neighborhood — **weakest source; see §7.2**)

**In-corpus, opened for this document:** `EXTENSION-REVISION` §5.2, §6.3, §7.2, §7.3, §8.1, §8.7 ·
`EXTENSION-ATTESTATION` §3.1, §4.0 (the invariant signature path) · `EXTENSION-QUORUM` §1, §2 ·
`EXTENSION-GROUP` §1, §1.1 · `EXTENSION-TRANSACTION` §1.1, §1.2 (**the negative**) ·
`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §3.2, §4, §4.2, §6 · `APP-CONVENTION-EMBED` §3 ·
`PROPOSAL-APP-CONVENTION-FEED` §1.1 (**the one-signer MUST**), §1.2, §1.3 ·
`EXPLORATION-THE-L5-CONTENT-TAXONOMY-…` §1, §4, §5.1, §7 (**§4.4 is what §4.3 corrects**) ·
`EXPLORATION-THE-FALSIFICATION-TEST-…` §3, §9.1 (**the limit this document discharges**) ·
`EXPLORATION-THE-DATA-MODEL-LADDER-…` §3, §4, §8.1 ·
`EXPLORATION-THE-INTERACTION-LANDSCAPE-…` §3, §4.1, §4.2, §5.1, §5.2 ·
`EXPLORATION-THE-PUBLIC-SOCIAL-STACK-…` §5, §5.3 · `EXPLORATION-WHAT-CONVERGES-…` §4.4 ·
`SYSTEM-IDENTITY-COMPOSITION` §1 · `GUIDE-MULTISIG` §4.2.
