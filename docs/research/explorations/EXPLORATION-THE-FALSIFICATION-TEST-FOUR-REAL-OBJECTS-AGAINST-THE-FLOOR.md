# EXPLORATION — the falsification test: four real objects against the taxonomy floor

**Status:** Exploration (design record). Not a proposal, not normative.

**What this is.** `EXPLORATION-THE-L5-CONTENT-TAXONOMY-AND-THE-FLOOR-THAT-STOPS-THE-EXPLOSION` derived
four content shapes from a test applied to *categories people name*. Every input to that derivation was
ours. **This takes four objects nobody designed for us — a real ATProto post, a Nostr kind-1, a Matrix
`m.room.message`, an SSB `post` — and maps each onto the floor, field by field, from the primary
schemas.** It is the only external test the floor has ever had.

**Method, because it is the point.** Every foreign field below comes from the **primary specification
or lexicon**, fetched for this document, never from a comparison page or a summary. That discipline is
not ceremony: the last two corrections that improved this arc both came from reading a primary spec
against our own text, and one of them found a published comparison page wrong about its own system.

---

## §0 The result

**The floor held on the axis it was tested on, and that is a real result rather than a comfortable
one.** All four objects are `app/feed/entry`. Not one demanded a fifth content shape, and the two
places where a foreign object genuinely did not fit — a Matrix *room*, and the profile — are both cases
the taxonomy document **predicted in advance** and left open.

**But the test's most valuable output is not the confirmation.** Mapping the four references turned up
a four-for-four convergence that settles `[P-1]`:

> **Every one of the four systems carries BOTH reference intents — "this exact thing" and "whatever is
> at this place now" — and every one of the four expresses the difference as a DIFFERENT SHAPE OR NAME.
> Not one of them makes it an optional field on a single shape.**

That is the option our own atom does not have, arrived at independently four times, and §4 is the
derivation. **It also partly answers `[OPEN-FEED-7]`** (*where does a profile live?*) from a direction
nobody was looking: in all four systems the profile's distinguishing property is not its content shape
— it is that **it is referenced live** (§5).

---

## §1 The test, and the four objects

**The test, unchanged** (taxonomy §1):

> Two things are different entity types **if and only if** a conformant consumer holding one must
> **behave** differently than if it held the other.

with the inverse guard that matters here: *collapsing two things that do demand different behaviour
into one type with a mode field pushes the distinction into a value, where a consumer must branch on it
anyway and nothing checks that it did.*

**The four objects, and why these four.** ATProto is the closest structural relative; Nostr is the
most minimal thing that works; Matrix is the most operationally serious; SSB is the one whose integrity
model is furthest from ours. **None was designed with us in mind, which is the whole point** — a floor
tested only against categories we named ourselves has been tested against our own imagination.

---

## §2 The mapping, field by field

**Read the empty cells as the finding.** A blank is *the system does not carry this*; the residue —
*the system carries it and we have nowhere to put it* — is §7.

| Our `app/feed/entry` | ATProto `app.bsky.feed.post` | Nostr kind 1 | Matrix `m.room.message` | SSB `post` |
|---|---|---|---|---|
| `author` | **absent from the record** — supplied by the repo DID | `pubkey` | `sender` | `author` (envelope) |
| `created_at` | `createdAt` — ISO datetime **string**, required | `created_at` — unix **seconds** | `origin_server_ts` — ms, **the server's clock** | `timestamp` — ms |
| `body` (embed-node) | `text` (≤3000 bytes / 300 graphemes) + `facets` + `embed` union of six | `content` — a bare string | `content.msgtype` (`m.text`/`m.image`/…) + body | `content.text` |
| `reply.root` | `reply.root` — a `strongRef` | `e` tag, marker `"root"` | `m.relates_to` with `rel_type: m.thread` | `content.root` |
| `reply.parent` | `reply.parent` — a `strongRef` | `e` tag, marker `"reply"` | `m.relates_to.m.in_reply_to.event_id` | `content.branch` — **MAY be an array** |
| `context` | — | — | **`room_id`** | — |
| `prev` (opt-in) | — *(the repo commit chain sits a layer below the record)* | — | *(the room DAG's `prev_events` sit a layer below the message)* | **`previous` + `sequence` — MANDATORY** |
| `attachments` | `embed.images` / `.video` / `.external` | *(URLs inline in `content`)* | `content.url` on `m.image`/`m.file`/… | `content.mentions` (blob links) |

**Four things in that table are worth stating out loud.**

**(a) ATProto does not carry the author in the record, and we do — correctly.** Their record key is a
`tid` inside a repo whose DID *is* the author, so the field would be redundant. **Ours is redundant too,
right up until the entry travels**, which is the entire subject of `FEED` §4. An entry in a mirror has
no namespace context to supply an author from. **So this is not a divergence to fix; it is the mirror
model showing up as a field**, and the external comparison is what makes that visible.

**(b) Three of the four put the causal chain a layer below the message, and SSB puts it in the
envelope and makes it mandatory.** That is exactly the split `FEED` §2.3.2 reasoned to on its own —
`prev` as an opt-in commitment rather than navigation — and SSB is the deployed system that took the
other branch. **This is the second half of `[P-2]` arriving with data attached** (§8.1).

**(c) `reply.root` + `reply.parent` is not our invention and not ATProto's alone — it is four-for-four.**
Nostr marks the same two roles on `e` tags, Matrix separates thread-root from in-reply-to, SSB names
them `root` and `branch`. **A conversation needing exactly two upward pointers is about as settled as a
design gets**, and `FEED` §2.3's argument for `root` (one unreachable author must not truncate the
thread) is the reason all four found it.

**(d) SSB's `branch` may be an array and our `parent` is one reference.** SSB threads fork and merge; a
reply may acknowledge several branches at once. **Ours cannot express a merge.** Small, real, and the
honest note is that no one has asked for it — recorded in §7 rather than proposed.

---

## §3 Result 1 — the floor holds, and here is precisely what that is worth

**All four objects are `app/feed/entry`.** A blog-length ATProto post, a 40-character Nostr note, a
Matrix image message and an SSB threaded reply differ in body, in length, in transport and in integrity
model — and **a conformant consumer holding any of them does the same things**: check the signature,
render the body, follow `reply` upward if present.

**What makes this stronger than the original derivation.** The taxonomy document collapsed six of ten
*named categories* — blog, vlog, microblog, photo post, comment, forum post — by arguing that the naming
was the only difference. **That argument is only as good as the imagination that produced the ten
names.** Here the inputs are schemas written by four teams with different goals, and they still land in
one shape. **A floor that survives objects it was not designed against is a floor rather than a
tautology.**

**And the honest limit, stated before anyone else states it: four objects is four objects.** All four
are *microblog-lineage* systems. This test says nothing about a wiki, a code forge, a marketplace
listing or a map annotation — and `[X-13]` (games, marketplaces, collaborative editors) is still
unstarted and is where a genuine falsification is most likely to come from. **This result narrows the
risk; it does not close it.**

---

## §4 Result 2 — four for four on two reference intents, always as two shapes

**This is the finding that changes something.** `EXPLORATION-THE-DURABLE-REFERENCE-…` §3 established
that our atom expresses *"this exact thing"* and cannot express *"whatever is at this place now"*, and
§6.1 left three options open. **Mapping the four systems' references answers it.**

| System | "this exact thing" | "whatever is at this place now" | How the two are told apart |
|---|---|---|---|
| **ATProto** | `strongRef` — `{uri, cid}` | a bare AT-URI (**"weak ref"**) | **two shapes**; the style guide names which to use when |
| **Nostr** | **`e` tag** — `[id, relay, marker, pubkey]`, `id` is a hash of the event | **`a` tag** — `["a", "<kind>:<pubkey>:<d-tag>", relay]`, addressable, relays keep only the latest | **two different tag names** |
| **Matrix** | `event_id` — a timeline event, immutable | **room state keyed by `(type, state_key)`** — the current value resolves; also `#alias:server`, reassignable | **two different addressing mechanisms** |
| **SSB** | `%<hash>.sha256` — a message, immutable | `@<pubkey>.ed25519` — a feed, an ever-growing sequence | **two different sigils** |

**Four independent designs, four times both intents, four times a distinct name or shape. Zero times a
mode field.** That is not a coincidence and it is not fashion — §4.1 is why.

### §4.1 The derivation, which does not rest on any of them

**Our own taxonomy test settles this without citing a single foreign system, and it should be read
first**, because a ruling that leans on what four peers happen to do is a ruling that expires when they
change (and this corpus has been burned by exactly that).

**The two intents demand different consumer behaviour, on three axes:**

| | "this exact thing" | "whatever is at this place now" |
|---|---|---|
| **What is authoritative** | the hash | the path |
| **Fetch strategy** | **anywhere** — author, mirror, cache, stranger; the hash validates the bytes regardless of source | **the named peer at the named path**; nobody else can answer for what is *current* there |
| **Hash mismatch means** | **the reference is unsatisfied** — refuse | **the document evolved** — expected, and the reader is told |
| **A 404 at the path means** | *nothing* — `path` is a hint (`FEED` §2.2 says so in a MUST NOT) | **the reference is broken**, modulo falling back to the last-seen hash |

**A consumer must branch on these, and there is nothing for it to branch on.** By the taxonomy's own
inverse guard, that is the definition of a distinction that must be carried in the shape rather than in
a value. **And the failure mode of getting it wrong is silent in exactly the way the guard predicts:**
under option (a) — *make `hash` optional* — a reference arriving without a hash is indistinguishable
between *"the author wants the live version"*, *"the author's implementation did not populate it"* and
*"the author only ever had a URL."* One of those is an intent and two are bugs, and **the reader cannot
tell, so the rug-pull guarantee `FEED` §2.2 exists to provide degrades to a convention.**

> **So option (a) is eliminated on our own reasoning, and option (c) — declare the living-document case
> out of scope — is eliminated by the three-way read table**
> (`EXPLORATION-THE-DURABLE-REFERENCE-…` §4), which shows the live reference has *better* defined
> behaviour than the pinned one in one respect: **row 5, where the path 404s and the last-seen hash is
> still fetchable from anywhere.** That is a capability none of the four systems above has, and
> declaring the case out of scope discards it.
>
> **Option (b) — a distinct shape — is the answer.** The four systems are **corroboration, cited last
> and verified in their primary specs this session** (§10); if all four had done the opposite, the
> derivation above would be unchanged.

### §4.2 What the shape should be — the inversion is the whole content

The two atoms are the **same three terms with the required and the optional swapped**, which is a good
sign that the distinction is real and minimal:

```cddl
reference = {                  ; "THIS EXACT THING" — unchanged, and still the default
  peer: peer-id,
  hash: content-hash,          ; THE claim
  ? path: tree-path            ; a hint; a 404 here proves nothing
}

live-reference = {             ; "WHATEVER IS AT THIS PLACE NOW"
  peer: peer-id,
  path: tree-path,             ; THE address of record
  ? seen: content-hash         ; what the linker saw at link time — an expectation, NOT a requirement
}
```

**The discriminator is the field name, not a mode value**, which is the point: a decoder holding
`hash` has a pin and a decoder holding `seen` has an expectation, and neither can be mistaken for the
other. **`seen` is also the honest name.** `hash` asserts *this is what it is*; `seen` asserts only
*this is what was there when I linked*, which is exactly the weaker claim a live reference makes and
exactly the input the three-way read needs.

**And the rule that keeps this from weakening anything, which is where §6.1(a)'s worry actually lands:
each site declares which atom it accepts.** `reply.root` and `reply.parent` take `reference` **only** —
the rug-pull argument is undiminished, because the site that needed it never gains the weak form.
`context` and `attachments` may reasonably take either. **A site that accepts both is the only place the
distinction has to be read at runtime, and there it is a field-name check.**

**This is written up as a proposal delta, not landed here.** `[P-1]`.

---

## §5 Result 3 — the profile is the first consumer of intent 2, and that is most of `[OPEN-FEED-7]`

`FEED` `[OPEN-FEED-7]` reads: *"Where does a profile live? Nobody asked for one, every surveyed system
has one, and it is not obviously any of the four shapes. **The most likely fifth type.**"*

**Checked against the four primary schemas, the profile is four-for-four a *live-addressed* object, and
in three of the four that is stated normatively:**

| System | The profile | The mechanism |
|---|---|---|
| **Nostr** | **kind 0** | NIP-01 lists kind 0 among **replaceable** events: *"only the latest event MUST be stored by relays, older versions MAY be discarded"* |
| **ATProto** | `app.bsky.actor.profile` | record key is **`"literal:self"`** — one per repo, a fixed key, updated in place |
| **Matrix** | `m.room.member` displayname/avatar | a **state event**, keyed `(type, state_key)`; the current value resolves |
| **SSB** | `about` messages | latest-wins by convention over an append-only log |

> **So the profile's distinguishing property is not a content shape at all. It is that it is referenced
> live.** Every one of these systems has a body-shaped bag of fields — display name, description,
> avatar, banner — that would map onto an entry without complaint. **What none of them does is pin it.**

**That reframes `[OPEN-FEED-7]` and probably shrinks it.** If the live reference lands (§4), a profile
may need **no fifth type**: a well-known path in the author's namespace, an entry-shaped body, and
everyone referencing it with a `live-reference`. The thing that made it look like a fifth type was that
our *only* reference shape pinned a hash, so a profile modelled as an entry would have had every
reference to it go stale on the first edit — **which is the missing intent presenting as a missing
type.**

**Two honest caveats.** *(1)* This is a hypothesis with one supporting mechanism, not a ruling — the
profile also raises discovery questions (how does a stranger find it without being told the path?)
that a reference shape does not touch. *(2)* `[OPEN-FEED-7]` says *nobody asked for one*, and **L26 cuts
toward letting a seat discover the shape by building it.** The contribution here is narrower and safer
than a design: **the profile is now known to be blocked on `[P-1]` rather than on a taxonomy question**,
so it should not be reasoned about as a fifth type until the reference question is settled.

---

## §6 Result 4 — Matrix's room reproduces §5.2's prediction from a real object

**The one place a foreign object genuinely did not fit is the Matrix *room*, and the taxonomy document
called it in advance.**

A room is none of the three collection provenances: not an author's own stream (it is multi-writer),
not a reader's gathered view (it is authoritative, not a claim-nothing assembly), not an author's
bounded complete set. **Taxonomy §5.2 predicted exactly this** — *"the conversation is not any of §4's
three collections"* — and listed four obligations that would distinguish one. **A Matrix room satisfies
all four:**

| §5.2's predicted obligation | Matrix |
|---|---|
| a roster, and it evolves | `m.room.member` state events |
| delivery is **expected**; its absence is a failure | the sync loop; a gap is an error condition, not a short view |
| who may read is **the membership** | join rules, history visibility, and E2EE key sharing |
| missing a message is **a bug**, not a short view | the room DAG's `prev_events`; a server backfills gaps |

**A prediction made from our own reasoning, satisfied point-for-point by a system that was not
consulted, is the strongest form this kind of check takes.** It does not prove the conversation needs
its own machinery — but it does mean `[OPEN-FEED-6]` (*does a chat message unify with
`app/feed/entry`?*) is being asked at the right seam: **the message unifies, the container does not**,
and that is now supported by an external object rather than by argument alone.

---

## §7 The residue — what the four carry and we have nowhere to put

**Ranked by how many of the four carry it, then by whether it changes consumer behaviour.** None of
these is a fifth *type*; all are fields or bridge concerns, so **none of them falsifies the floor** —
which is precisely why they are easy to miss when only the floor is being tested.

| # | Thing | Carried by | Changes consumer behaviour? | Status |
|---|---|---|---|---|
| **R-a** | **Mentions / addressing** — *who this is for* | **3 of 4** — ATProto `facets` (mention), Nostr `p` tags, SSB `content.mentions` | **yes** — it is the input to notification and delivery | **A deliberate absence.** `FEED` §2.3 says `context` is *"never who it is for"* — correctly, since addressing is a different concern. **But the absence is implicit, and an absence that is never written down is indistinguishable from an oversight.** Same shape as `[C-3]` |
| **R-b** | **Foreign integrity proof** | **4 of 4** — every one has a signature or CID we cannot reproduce | **yes**, for any bridge that claims fidelity | **Belongs to the bridge track** (`[X-4]`), not to `FEED`. An `entry` has no slot to carry a foreign proof, and §8.2 shows why SSB makes this sharp |
| **R-c** | **Content warning / self-label** | 1 of 4 (ATProto `labels`) + Mastodon's CW, which `[X-3]` will confirm | **yes** — a conformant consumer hides it behind an interstitial | **A real gap, and the clearest thing in this table to pass our own behaviour test.** Small and unproposed |
| **R-d** | **Quote / cite** — *render that entry inside this one* | 2 of 4 — ATProto `embed.record`, Nostr `q` tag | **yes** — fetch and render inline, and it is neither a reply nor a file | **Expressible twice and assigned neither.** `attachments` is a reference list; the body is an open handler layer (`app/embed/{type}` is unbounded, so an entity-ref handler is legal). **Two candidate homes and no rule** |
| **R-e** | **Language** | 1 of 4 (ATProto `langs`) | no — a display and filtering hint | Minor. Recorded, not pursued |
| **R-f** | **Thread merge** — a reply acknowledging several parents | 1 of 4 (SSB `branch` may be an array) | marginally — rendering a merge | Recorded. Nobody has asked; **L26 says wait for a consumer** |

**The ordering matters more than any single row.** R-a and R-b are the two that a bridge hits on day
one, and neither is a `FEED` question: one is a scope statement `FEED` should make explicit, the other
is the bridge track's central unanswered question. **R-c is the only row that is straightforwardly a
missing field in a proposal under review**, and it is the one to raise with the two seats.

---

## §8 Two structural asymmetries the mapping exposed

### §8.1 SSB hashes the signature into the identity, so an SSB ingress bridge must carry bytes

**Our model:** an entity is `(type, data)`, its content hash covers that, and **the signature is a
separate entity** (`FEED` §1.1) — detached, so anyone holding the bytes reconstructs an identical
entity and the hash is stable whether or not you have the signature.

**SSB's model:** the message ID is `%<hash>.sha256` where the hash is taken over the **complete
formatted message including the signature**, under a strict canonical JSON form (two-space indent,
fixed key order).

> **Consequence, and it is concrete: an SSB message's identity is not derivable from our translation of
> it.** No amount of faithful field mapping reproduces `%<hash>`, because the hash covers a signature
> over a byte form we do not emit. **So an ingress-client bridge from SSB that wants to preserve
> identity MUST carry the original canonical bytes as a payload, not merely translate the fields** —
> and a bridge that translates without carrying has silently downgraded from *carried integrity* to
> *re-authored integrity*, which is the distinction `EXPLORATION-THE-BRIDGE-…` §2 names as *"where
> honesty lives."*

**What is actually new here is narrower than it first looked, and the difference is worth recording.**
`EXPLORATION-THE-BRIDGE-…` §5.2 **already states both facts** — *"message id = sha256 **including** the
signature"* and *"a bridge extension implementing their canonical JSON can verify their signatures on
carried bytes, so ingress can be **carried**, not merely re-authored."* **So the bridge document is not
missing this; it establishes that carrying is possible.**

**The addition is the negative half, which neither document had drawn: that translation alone is not
merely lower-fidelity, it is identity-destroying and irrecoverably so.** *Carrying is possible* and
*not carrying loses the identity permanently* are different sentences, and only the first was written.
A bridge author reading §5.2 could reasonably conclude that carried bytes are an optimization for
signature verification; **they are the only way the message keeps its name.** That is the sentence to
add to §5.2, and it is a sharpening of an existing finding rather than a new one.

### §8.2 Matrix redacts and we unpublish, and the reason is the DAG

**Matrix's redaction algorithm strips content fields but preserves the event skeleton and its place in
the room DAG.** It has to: `prev_events` means a removed event leaves a hole that breaks the graph, so
the event must stay and be emptied. **We unpublish, and `FEED` §5b.3 claims the removal leaves no trace
in the current tree.**

**Both are correct, and the difference is caused by exactly one thing: Matrix has a chain and our tree
does not.** That is the same argument `FEED` §2.3.2 made when it demoted `prev` from navigation to an
opt-in commitment — *a chain converts a local edit into a republish of everything since* — and here is
a deployed system paying that cost in the form of an entire redaction algorithm and a tombstoned event
that can never be reclaimed.

> **Which sharpens the `prev` rule usefully: an author who opts into `prev` is opting into Matrix's
> problem.** They gain *"I cannot have quietly edited this"* and they lose clean removal, permanently,
> for everything after the entry they remove. **That trade is currently stated as a general principle;
> it now has a system to point at.** `[P-2]`.

---

## §9 What this does not settle

1. **Four microblog-lineage objects.** §3's limit, restated because it is the one that matters: a wiki
   page, a marketplace listing, a map annotation and a collaborative document are untested, and
   `[X-13]` is where a real falsification would come from.
   > **DISCHARGED 2026-09-05, and this limit was right.**
   > `EXPLORATION-THE-SECOND-FALSIFICATION-THE-WIKI-THE-FORUM-THE-MARKETPLACE-AND-THE-COLLABORATIVE-DOCUMENT`
   > ran three of the four named classes from primary sources. **The wiki, the forum and the
   > collaborative document absorb; the marketplace order falsifies** — and the falsifier is the axis
   > this test structurally could not vary, because **all four objects here are singly authored.** The
   > floor's shapes each have exactly one signing party; an order has two or more, and `FEED` §1.1
   > requires a conformant reader to *reject* an entry whose author is not its namespace. Games remain
   > untested.
2. **The mapping is structural, not executed.** Nothing was actually round-tripped. **An executed
   mapping — take real bytes, produce an entity, produce the foreign object back, compare — is a
   different and stronger test**, and it is `[X-2]`'s neighbour rather than this document's claim.
3. **R-c (content warning) is unproposed.** It passes our own behaviour test and has no home. Named,
   not designed.
4. **§5's profile hypothesis has one mechanism and no consumer.** It reframes `[OPEN-FEED-7]`; it does
   not close it.
5. **Nostr has no author-side stream object at all** — a user's feed is a *filter query over relays*,
   not a published index. Our `index-head`/`index-page` exist because a reader pulls from the author's
   own tree. **Not a falsification — a scoping difference — but it means Nostr offers no evidence
   either way about the index shape**, and it should not be cited as if it did.

---

## §10 Sources

**Primary, fetched for this document:**
[ATProto — `app.bsky.feed.post` lexicon](https://raw.githubusercontent.com/bluesky-social/atproto/main/lexicons/app/bsky/feed/post.json) ·
[ATProto — `app.bsky.actor.profile` lexicon](https://raw.githubusercontent.com/bluesky-social/atproto/main/lexicons/app/bsky/actor/profile.json) ·
[Nostr — NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) (event structure, id computation, replaceable/addressable kinds, `e`/`a`/`p` tags) ·
[Nostr — NIP-10](https://github.com/nostr-protocol/nips/blob/master/10.md) (`root`/`reply` markers, the deprecated positional scheme, `q`) ·
[Matrix — Client-Server API](https://spec.matrix.org/latest/client-server-api/) (`m.room.message`, `msgtype`, `m.relates_to`, `m.in_reply_to`, `m.thread`, state events, room aliases) ·
[Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/) (message fields, canonical form, message ID over the signed message, `root`/`branch`/`mentions`)

**In-corpus:** `EXPLORATION-THE-L5-CONTENT-TAXONOMY-AND-THE-FLOOR-THAT-STOPS-THE-EXPLOSION` §1 (the
test), §3 (the ten), §4 (the three provenances), §5.2 (**the conversation prediction §6 confirms**) ·
`PROPOSAL-APP-CONVENTION-FEED` §1.1, §2.2 (**the atom §4 amends**), §2.3, §2.3.2, §5b.3, §12 ·
`EXPLORATION-THE-DURABLE-REFERENCE-THE-ANCHOR-AND-THE-THREE-WAY-READ` §3, §4 (the three-way read),
§6.1 (**the three options §4 closes**) · `EXPLORATION-THE-BRIDGE-…` §2 (the integrity disposition
axis, which §8.1 gives its first concrete mechanism), §5.2 (SSB) · `APP-CONVENTION-EMBED` §4.0 (the
closed output vocabulary and the open handler layer — why R-d is expressible and unassigned).
