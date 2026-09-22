# EXPLORATION — the L5 content taxonomy: what is genuinely a different type, and the floor that stops the explosion

**Status:** Exploration (design record). Not a proposal, not normative.

**The question it answers:** *a feed, a blog, a vlog, a photo gallery, stories, a forum, a chat, a
site. How many entity types is that — and how do we know we picked the right number?*

**Why it exists.** The risk in an application vocabulary is not getting one shape wrong; it is the
**taxonomic explosion** — a type per product, each one somebody's reasonable reading of a category,
none of them wrong, and interoperability quietly lost between them. The opposite failure is just as
real: a vocabulary so generic that a consumer holding an entity cannot tell what to do with it. **This
document is an attempt to find the floor between those two, using a test rather than a preference.**

**Rests on:** `EXPLORATION-THE-RESOLUTION-CHAIN-…` §7 (the record models, read from primary
specifications) · `EXPLORATION-THE-PUBLIC-SOCIAL-STACK-THE-COMPLETE-MAP` §4 (the four shapes) ·
`APP-CONVENTION-EMBED` (which already owns everything about *media*) ·
`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4.2 (which already ruled on collections once).

---

## §1 The test, stated before it is applied

Everything below turns on one question, and the value of the document is mostly in having a question
at all rather than an argument per case.

> **Two things are different entity types if and only if a conformant consumer holding one must
> BEHAVE differently than if it held the other.**
>
> Not if they are called different things. Not if they look different on a screen. Not if a product
> manager would put them in different tabs.

**Three corollaries fall straight out, and each kills a category of would-be type.**

1. **Different body, same type.** If the only difference is what is inside the content payload, the
   dispatch that handles that payload absorbs it. A vlog is not a type.
2. **Different presentation, same type.** If the only difference is how a renderer chooses to lay it
   out, it is a renderer's decision and the charter already forbids it being in the format.
3. **Different discovery, same type.** If the only difference is *how you came to be holding it*, the
   difference is in the route, not in the object. **This one is doing the most work below**, and it
   is the least obvious.

**And the inverse, which is the guard against over-unifying.** If two things demand different consumer
behaviour — a different fetch strategy, a different completeness assumption, a different attribution
rule — **collapsing them into one type with a mode field pushes the distinction into a value**, where
a consumer must branch on it anyway and nothing checks that it did. That is the "generic Thing with a
kind string" failure, and it is worse than one extra type, because a wrong mode value is invisible
while a wrong type tag at least fails a filter.

---

## §2 The landscape, and what it actually shows

Two decentralized systems took opposite routes on exactly this question, and both results are useful.

**The large-vocabulary route.** ActivityStreams 2.0 ships a broad object vocabulary — `Note`,
`Article`, `Image`, `Video`, `Audio`, `Page`, `Event`, `Place`, `Profile`, `Tombstone`, plus
`Collection` / `OrderedCollection` / their page forms — and a comparably broad activity vocabulary
(`Create`, `Update`, `Delete`, `Follow`, `Like`, `Announce`, `Add`, `Remove`, `Block`, `Undo`, …).
**The vocabulary is expressive and the deployed interoperability is a small subset of it.** In
practice the fediverse converged on a handful of shapes and implementations vary in what they accept
beyond them.

**The reading to take, and it is the operator's fear stated as an observed outcome:** a large
vocabulary does not produce large interoperability. It produces **a de-facto small vocabulary plus a
long tail that different implementations handle differently** — which is exactly the taxonomic
explosion, arriving as fragmentation rather than as a design argument anybody lost.

**The small-vocabulary route.** ATProto keeps few record types under reverse-domain namespaced
identifiers and carries an explicit compatibility contract — *all old data must remain valid under the
updated schema, and new data valid under the old*; new fields optional, no renames, breaking change
takes a new name. **The vocabulary is smaller and the rules for growing it are written down.**

**Both halves matter and we currently have neither.** The floor below is the first half. The second
half is already proposed as a charter discipline.

**One prior-art borrowing worth naming: `Tombstone`.** ActivityStreams has an object type meaning
*something was here and was deleted*. **We do not need it and structurally cannot use it** (§6), and
knowing why is more instructive than adopting it.

---

## §3 Applying the test to the eight things people name

| What people call it | Different consumer behaviour? | Verdict |
|---|---|---|
| **blog post** | no — walk, fetch, render | **entry** |
| **vlog / video post** | no — the body is a video embed, dispatch handles it | **entry** |
| **microblog / status** | no — shorter body | **entry** |
| **photo post** | no — the body is an image embed with a pointer payload | **entry** |
| **comment / reply** | no — an entry that carries `reply` | **entry** |
| **forum post** | **no** — see §4, this is the big one | **entry** |
| **chat message** | the *message* no; the **conversation** yes | **entry + conversation machinery** (§5) |
| **photo gallery / album** | **yes** — bounded and complete vs unbounded and cursored | **a second collection shape** (§4.3) |
| **story** | no — an entry the author asks you to stop showing at T | **entry + a hint field** (§6.2) |
| **site page** | **yes** — navigable, not published-at-a-time | **already owned by the site convention** |

**Six of the ten collapse into one type.** That is the result, and the rest of the document is why the
four that do not are the ones that do not.

---

## §4 The three collection provenances — the real distinction, and it is not subject matter

The entry is easy. **The interesting question is what a *collection* of entries is**, and this is
where the taxonomy either explodes or does not.

The instinct is to type collections by **subject** — a feed, an album, a timeline, a thread, a
playlist. That is the explosion, and it never terminates, because subject matter is unbounded.

**Type them by provenance instead — who assembled it and what it therefore claims — and there are
exactly three, because there are exactly three answers to "where did this list come from?"**

### §4.1 The stream — the author's own, unbounded, cursored

The author's continuing output. **Unbounded by nature**, so it cannot be one object: it is a mutable
head page with an immutable back-chain, and a reader walks back only as far as its cursor.

**Consumer obligations:** hold a cursor · walk `prev` · stop · never assume the head is the whole
thing. **Claims:** nothing about completeness; an entry absent from it is still a valid entry.

*Covers: blog, microblog, vlog, photo feed, "my posts."*

### §4.2 The gathered view — assembled by a reader, about other people's entries

A reader's answer to *"here is what I found."* Spans authors. **Cannot claim completeness and the
shape gives it no way to.**

**Consumer obligations:** attribute every entry to its own author, never to the gatherer · treat
omission as expected · merge with other views by union. **Claims:** only that these entries exist and
verify.

*Covers: a forum thread view, a topic timeline, an aggregator, a "what I saw today" digest.*

### §4.3 The authored set — the author's own, bounded, and COMPLETE

**This is the shape the corpus does not have, and the photo gallery is what surfaces it.**

An album is not a short feed. The difference is not size and not subject:

| | stream | authored set |
|---|---|---|
| Bounded? | no — grows forever | **yes — it is a finite thing with edges** |
| Complete? | never claims to be | **yes — this IS the album; there is no page 2 hiding elsewhere** |
| Reader holds | a cursor | nothing; it fetches the set |
| Order means | recency | **the author's arrangement**, which is content |
| "Show me all of it" | not a sane operation | **the primary operation** |

**A conformant consumer must behave differently**, on three axes at once, so by §1's test this is a
type. **And it is one type, not five** — album, playlist, portfolio, curated reading list and
"pinned" are the same shape with different bodies inside.

**Why the ordering point is not cosmetic.** In a stream, order is derivable (recency) and the tree
carries no semantic order anyway, which is why the site convention pinned only a determinism floor.
**In an authored set the order is authored — it is part of what the author made.** A gallery
re-sorted is a different gallery. So the set carries its order as data, and that is a real difference
from everything the site convention ruled about.

### §4.4 The result

**Three collection types, and it terminates**, because "who assembled this and what does it claim" has
three answers and not thirty. Two of the three are already in the feed proposal (the index page and
the mirror). **The gallery adds exactly one.**

---

## §5 The forum, and the chat — the two cases where the answer is surprising

### §5.1 A forum post is not a type, and the forum needs no new entity at all

**This is the strongest result in the document and it is worth stating carefully, because it sounds
too convenient.**

A forum post is: authored by one person, in reply to something, in a topic, published into the
author's own namespace. **That is an entry with `reply` and `context` — both of which already exist.**

A forum *thread* is: entries by many authors, related by `reply`, assembled from partial views.
**That is a gathered view (§4.2), which already exists.**

So what actually distinguishes a forum from a comment section under someone's blog post? **Only how
you arrived**: at a blog you followed a person, at a forum you followed a subject. By §1 corollary 3,
**different discovery is not a different type.**

**What the forum genuinely needs is not an entity type — it is a way to name a subject that nobody
owns.** A topic that is a content hash gives exactly that: hash the subject and everyone who cares
about the same subject computes the same identifier without coordinating. It goes in `context`, which
is already a reference field. **The remaining forum problem is discovery and navigation — finding
anyone at all who is in a topic — and no entity type solves it.**

**The honest caveat:** this says the *data model* needs nothing new. It does not say the forum is
easy; the aggregation is solved and the navigation is not, which is where that work already stood.

### §5.2 A chat message is an entry; a conversation is not a collection

Chat is the case where over-unifying is genuinely tempting and partly wrong.

**The message is an entry.** Same author rule, same body, same reply structure. Minting
`app/chat/message` beside `app/feed/entry` would create two types that a consumer handles identically,
which is corollary 1 with a different label.

**The conversation is not any of §4's three collections**, and this is the real distinction:

| | a gathered view | a conversation |
|---|---|---|
| Membership | none — anyone may be in it | **a roster, and it evolves** |
| Delivery | pull; the author may be offline forever | **expected**, and its absence is a failure |
| Who may read | anyone | **the membership**, usually cryptographically |
| Completeness | not claimed | **missing a message is a bug, not a short view** |

Those are four different consumer obligations, so the conversation earns its own machinery — which is
what the existing chat proposal is about. **The recommendation is narrower than that proposal
currently assumes: reuse the entry as the message atom, and let the chat convention contribute the
conversation identity, the roster log and the delivery model.**

**Why this matters beyond tidiness.** If the message and the entry are one type, then a message can be
quoted into a feed, a feed entry can be replied to in a chat, and a forum post can be forwarded into a
conversation — **without a conversion, because there is nothing to convert.** If they are two types,
every one of those crossings needs a mapping, and mappings are where vocabularies fork. **This is a
decision to take to both application seats rather than to rule from here**, and it is the single most
consequential open question in the L5 tier.

---

## §6 The two things people expect that this model answers differently

### §6.1 Deletion — withdrawal is unpublication, and it is not erasure

**The operator's position, and it is the right one:** you own what you published, you may edit it or
take it down, and someone who already saved it keeps it. Both halves are legitimate — the right to
revise and the right to keep a record — and the tension between them is not an artefact of this design.

**What the model does:**

- **Take down** — remove the binding and republish. New readers do not receive it. **The author's
  origin stops serving it, which is what "delete" means on the open web.**
- **Edit** — publish a new entry referencing the old. Which one a reader displays is the reader's
  choice, and both remain addressable by anyone who holds them.
- **Never publish it in the clear** — the only mechanism that actually withholds, and it is the
  encrypted/membership path, which is later work.

**What the model cannot do, stated plainly because a user should be told the truth:** there is **no
global takedown, ever.** A holder of the bytes keeps them, and no protocol operation reaches into
somebody else's store. This is the exact counterpart of *no platform*: there is nobody to appeal to,
in either direction.

**Why we cannot use a `Tombstone`.** A tombstone is a *request to a cooperating replica* to display an
absence. It works where a small set of servers agree to honour it. Here the holder set is unbounded and
uncoordinated, and a signed "please forget this" is unenforceable by construction. **Publishing a
tombstone would be a promise the architecture cannot keep**, which is worse than the honest statement.
**An honest UI says "removed from your site; people who already have it still have it" — and that
sentence should appear at the moment of deletion, not in a help page.**

### §6.2 Stories, and why an expiry is a hint and not a mechanism

A story is an entry the author would like you to stop showing after a time. **`expires_at` is a
display hint and MUST be documented as unenforceable** — a well-behaved reader honours it, and nothing
prevents a reader from not. That is the same honesty as §6.1 and the same one sentence.

It is a field, not a type, and it is probably not v1.

---

## §7 The floor, assembled

**Four entity shapes, and the products people name are combinations of them.**

| | Shape | What varies inside it |
|---|---|---|
| **1** | **the entry** — a thing someone published | the **body** (an embed — text, image, video, anything), and optionally `reply` / `context` |
| **2** | **the stream** — the author's own, unbounded, cursored | nothing; it is structural |
| **3** | **the authored set** — the author's own, bounded, complete, ordered as authored | what is in it |
| **4** | **the gathered view** — assembled by a reader, claims nothing | what the gatherer found |

Plus **the follow**, which is a subscription rather than content, and **the reference**, which is how
all four point at things.

**Everything in §3 maps onto it:**

- blog · vlog · microblog · photo feed → **1 + 2**
- gallery · album · playlist · portfolio → **1 + 3**
- forum thread · timeline · aggregator digest → **1 + 4**
- comment · reply → **1**, with `reply`
- chat → **1** + the conversation machinery (§5.2)
- site → the site convention's own pages and nav, which stay where they are
- story → **1**, with an expiry hint

**And the growth rule, which is the part that actually prevents the explosion:** a new product is a new
**body type** (an embed handler — that extension point is already open and unbounded) or a new
**renderer**. **It is a new entity type only if it fails §1's test**, and the burden is on the proposer
to name the consumer behaviour that differs.

---

## §8 What this document does not settle

1. **Whether the chat message and the feed entry unify.** §5.2 recommends it and does not rule it.
   **This needs both application seats**, and it is the highest-consequence open question here.
2. **Whether the authored set is one type or two.** An ordered album and an unordered bag may differ
   enough to matter; the argument above says order is content, which implies one type with the order
   always present. Untested against a real gallery.
3. **Where a profile lives.** Nobody asked, and every surveyed system has one. It is not obviously any
   of the four shapes and it is the most likely fifth.
4. **Whether `context` is doing too much.** It carries "part of a topic", "part of an album" and
   "part of an event" with one field, and if consumers must branch on which, it fails §1's own inverse
   test and should be split.
5. **The landscape here is two systems deep.** The record-model axis was read from primary
   specifications for two of them; the products this document reasons about — the ones with the photo
   galleries — are **not specified anywhere and were reasoned about from their observable behaviour.**
   That is a weaker input than the rest of the corpus's prior-art work and it should be treated as such.
