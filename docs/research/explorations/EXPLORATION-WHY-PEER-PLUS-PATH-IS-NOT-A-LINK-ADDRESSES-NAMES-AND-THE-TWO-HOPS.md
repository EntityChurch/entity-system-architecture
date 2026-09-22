# EXPLORATION — why `(peer, path)` is not a link: addresses, names, and the two hops

**Status:** Exploration (design record). Not a proposal, not normative.

**Operator-raised, 2026-09-06 (fourth pass), and it is the right question:** *"The entity canonical form
of a peer-to-path — all the things we're talking about exist in the tree in that form, so that's a little
confusing. That tells you that something's in the tree. So why doesn't a peer-plus-path link work? What
does that tell us about the system? It just doesn't tell you enough about how it all fits together, or
to resolve it."*

**This supersedes the framing of the design bake-off, not its scorecard.** That document went looking for
a scheme to mint. **The prior question — what can `(peer, path)` not say, and why — has a cleaner answer,
and the answer is already in the corpus under a different heading.**

---

## §0 The result

**`(peer, path)` is not broken and the confusion is legitimate: it *does* work. The resolution chain
completes.** Given `/{peer}/{path}` a consumer resolves the peer to a transport, calls `tree:get(path)`
to get a hash, calls `content:get(hash)` to get bytes, and renders. **Nothing is missing mechanically.**

**What it cannot do is be a *reference*, and the difference is one sentence:**

> **`(peer, path)` is an ADDRESS — it says where to go and stakes nothing on what you find. A reference
> is a NAME — it says what thing is meant, and must survive being carried to someone who was not there,
> at a time when the publisher may be gone.**

**The corpus already draws this distinction, precisely, and uses it for something else.**
`GUIDE-RESOLUTION` §5 splits every resolution target into **self-verifying** (*"content-hash → bytes: the
bytes hash-match from any source… the hash replaces trust"*) and **authority-scoped** (*"name, path:
value depends on who owns it… requires the consumer's own trust policy"*). **It uses that split to decide
where a resolver lives — server-side extension vs client-side SDK. Nobody has applied it to references.**
Applied there it settles everything at once:

| | authority-scoped — `(peer, path)` | self-verifying — `hash` |
|---|---|---|
| **Who can answer** | **only the owner** | **anyone** |
| Survives the publisher vanishing | ✗ | **✓** |
| Third party can verify | ✗ — you get what they serve today | **✓** |
| What a routing hint buys | **a route to the one party who can answer** | **a list of candidates** |
| What it is good for | *"whatever is current here"* | *"this exact thing"* |

**And the second result, which is the reduction the operator has been asking for.** `GUIDE-RESOLUTION`
§1's **(P1)**: *"Getting an entity is always two hops. `tree: path → hash`, then `content: hash →
bytes`."* **The two reference intents are literally the two hops:**

- **`live-reference` = "do hop 1 yourself, every time."** It names hop 1's *input*. Hop 1 is
  authority-scoped by construction, because it is a walk of *their* tree — so only they can answer, and
  it dies with them.
- **`reference` (pin) = "I already did hop 1; here is the answer."** It names hop 2's *input*. Hop 2 is
  self-verifying, so **anyone** can answer, and it outlives its publisher.

> **That is why the two shapes cannot be collapsed, stated better than `FEED` §2.2.3 states it: they are
> not two flavours of one thing, they are two different rungs of the resolution ladder.** And it is why
> `path` is only a *hint* in a pinned reference — it is a **cached record of where hop 1 was performed
> once**, useful for finding a source, never authoritative, because hop 1 has already been discharged.

**The third result: `entity://` being the EXECUTE dispatch URI is not a naming collision to work around.
It is correct, and it is the same distinction a third time.**

---

## §1 What `(peer, path)` cannot say — four things, in ascending order of surprise

**Each is a thing a link needs and an address does not.**

**1 · Integrity.** No pin, so you get whatever is served now. This is the rug-pull. **But note this is not
a defect of `(peer, path)`** — for a *maintained document* it is exactly right, which is why
`live-reference` exists and keeps the path authoritative. **An address is the correct shape for the live
intent.** It only fails when the intent was "this exact thing."

**2 · Survivability.** When the publisher unpublishes the path or disappears, the reference is dead and
there is nothing to fall back to. **A pinned hash survives**, because hop 2 is answerable by anyone.

**3 · Routability when the peer is unresolvable.** `EXTENSION-NETWORK` §6.5.4 puts peer→transport
discovery out of v1 scope and defers it to REGISTRY; §6.5.1a makes self-publication a **SHOULD** and says
*"consumers MUST NOT assume the self-published path exists."* So a bare peer-id with no registry binding
has no route. **This is the `via` gap, and it is the one Matrix states in the same words** (*"room IDs are
not routable on their own as there is no reliable domain to send requests to"*).

**4 · Type — and this is the one nobody here has raised.** A path is one opaque string. **Following it
tells you what it is only after you have fetched it.** So a client cannot route to a renderer, decide
whether to prefetch, or decline a 400 MB video, without first paying for it.

> **Both mature systems carry the type in the reference and we carry it nowhere.** Nostr's `naddr`
> coordinate is **`kind:pubkey:d-tag`** — the kind *is part of the address*. ATProto's AT-URI is
> `at://authority/COLLECTION/rkey`, where the collection is the NSID, i.e. **the record type**. §4 works
> out whether we should copy this; the answer is *yes, but advisory*, and the reason is interesting.

---

## §2 Address versus name, and why `entity://` is right where it is

**The `entity://` scheme is the EXECUTE dispatch URI** — `entity://{peer_id}/{handler-path}`, with
`ENTITY-SYSTEM-REFERENCE` §228 stripping the prefix to get a handler path. The bake-off treated that as
an obstacle: *the obvious scheme is taken, pick another.* **That was the wrong reading. It is taken by
the thing it should belong to.**

| | **dispatch address** (`entity://`) | **reference** |
|---|---|---|
| Mood | **imperative** — *do this, there* | **declarative** — *this is the thing I mean* |
| Tense | **present** — the target is live, now | **durable** — must hold later, elsewhere |
| Who authenticates | the **session** (V7 §4.2, IDENTIFY) | the **artifact** — a hash, a signature |
| Third party present? | **no** — it is a two-party act | **yes, by assumption** — it will be carried |
| Fails when | the peer is down | it should not fail when the peer is down |

**An operation target legitimately does not need a hash**: you are talking to the peer live, the session
is authenticated, and there is no third party to convince. **A reference is the opposite object on every
row.** Putting both under one scheme would give one word two meanings — the exact thing
`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §11 forbids (*"each word in the system carries one meaning"*).

> **So the finding sharpens: `SITE` §4's `link-ref` naming `"entity://…"` as a nav link's absolute form
> is not a sloppy comment, it is a category error with a diagnosis.** It types a **hyperlink** as an
> **operation target**. A renderer that follows it literally would be dispatching an EXECUTE at every
> click.

**And it explains the confusion in the operator's own question.** *"All the things we're talking about
exist in the tree in that form"* — **true, and that is what makes the tree address universal as a
LOCATION.** Universal is not the same as sufficient. Content hashes live in the tree too; the tree is
where both hops happen. **The tree address names a place in someone's filing system; a reference names a
thing.** Both are canonical, and they answer different questions.

**One more reason a path is a weak name, which is structural rather than a gap:** a tree path is the
publisher's own layout, and **nothing obliges them to keep it.** `SITE` §2 makes a site *"a free subgraph
at any publisher-chosen tree path"*, and §11 makes the URL form *"a publish-time projection, not a
tree-storage rule."* **A path is a private filing system exposed publicly** — correct for an address,
weak for a name, and there is no rule anywhere making one stable.

---

## §3 The reduction: a reference is an entry point into the chain

**`GUIDE-RESOLUTION` §3 already has the table this whole line of work has been circling.** It is titled
*"What do I have → where I enter the ladder"*:

```
  a name         alice        top:    name → peer_id      a registry (Layer A)
  a peer_id      z6Mk…        middle: peer_id → transport reachability (Layer B)
  a transport    wss://…      lower:  connect directly    nothing — dial it
  a content-hash b3…          bottom: hash → bytes        substitute / any source
  a URL          name@auth/path  top, composed            the whole chain
```

> **A reference is a declaration of which rung you are entering at, plus optional hints for the rungs
> below.** That is the primitive form, and it is not new machinery — it is the corpus's own model, read
> as a data structure instead of as a walkthrough.

**Five entry points, and our two atoms cover two of them.** A link to `alice@entity-church` (a name) is a
perfectly good link; so is a link to a bare peer; so is a bare hash. **All are legitimate references and
none has a shape.**

**Nostr's NIP-19 is exactly this table, encoded.** Five forms, and they are five entry points:

| NIP-19 | Entry point | Ours |
|---|---|---|
| `npub` | a key | a peer-id |
| `nprofile` | a key **+ relay hints** | a peer-id + `via` — **no shape** |
| `note` | an event id (self-verifying) | a content-hash |
| `nevent` | event id **+ hints + author + kind** | `reference` — **minus hints and kind** |
| `naddr` | **`kind:pubkey:d-tag`** — the *live* coordinate | `live-reference` — **minus kind** |

**And their tag vocabulary carries the same split**: an **`e` tag** references *"a specific event by its
ID… one particular version"*; an **`a` tag** references *"whichever version is currently latest."*
**`e` is our pin, `a` is our live-reference, derived independently, deployed, and load-bearing.**

**This is the strongest external corroboration this design has** — not that the two intents exist (four
systems already showed that) but that **the two intents plus the entry-point table plus routing hints
compose into one URI vocabulary**, which is precisely the thing under design here.

---

## §4 Should a reference carry the type?

**Both mature systems do. The case for and against is genuinely balanced and the resolution is a split.**

**For.** A client must decide *how to handle a link before dereferencing it* — which renderer, whether to
prefetch, whether to fetch at all. Without a type, every link is opaque until paid for. This is the
`nav`-walking case (`SITE` §4.1 already worries about renderer cost), the prefetch case, and the
hostile-payload case.

**Against, and it is specific to us.** In Nostr and ATProto **the type is part of the address** —
`kind:pubkey:d` and `at://authority/collection/rkey` — so it *cannot* lie: a different kind is a
different address. **Ours would be an annotation beside a path, so it can disagree with the entity it
names.** For a pinned reference that is harmless (fetch, check, discard on mismatch — the same discipline
Mechanism A already applies to bytes). **For a `live-reference` it can go silently wrong**: the publisher
replaces the document at that path with a different type, the link still resolves, and the annotation is
now a lie that a client may have already routed on.

> **So: carry it, and carry it as advisory — same class as `via`, never as an authority.** *A reader that
> ignores the type hint and dispatches on the fetched entity's actual `type` MUST reach the same answer.*
> That sentence makes it a pure optimization and makes the lying case a performance bug rather than a
> correctness one. **It also means the type hint is exactly as droppable as a routing hint, which is the
> uniformity worth having: one rule covers both.**

**What it is not.** It is **not** the relationship — *reply* vs *quote* vs *attachment* vs *see-also*.
That is carried by **which field the reference sits in** (`reply.parent`, `context`, `attachments`), and
that is already right: Nostr needs marked `e` tags (`"reply"`, `"root"`, `"mention"`) because their tags
are a flat list, and **we get it from structure for free.** No change wanted.

---

## §5 What this changes

**Nothing on the scorecard. Design C still wins and the staging still holds.** What changes is the
*derivation*, and three things follow that the bake-off did not have:

1. **The scheme question is smaller than it looked.** The bake-off treated `entity://` as taken and
   listed three replacement candidates. **The right framing is that a reference scheme and a dispatch
   scheme are different objects that should look different**, so a sibling name is not a workaround, it
   is the correct outcome. The naming call is still the operator's; the *anxiety* about it was misplaced.
2. **A `type` hint joins `via` as an advisory term**, under one shared rule (*a reader ignoring all
   advisory terms MUST reach the same answer, more slowly*). This is a small addition to Design C's atom
   and it is the term both deployed systems have and we do not.
3. **The atom should be able to express more than two entry points.** Today's two cover hash and path. A
   name (`alice@entity-church`) and a bare peer are legitimate link targets with no shape, and **the
   tagged encoding chosen in Design C is what makes adding them later cheap** — which retroactively
   strengthens the tag-vs-field-name argument that was the weakest part of that scorecard.

**And one thing it explicitly does not change: no new docket item.** This is derivation for **D-37**, and
the enforcement point from the last three sessions applies to this one too — *a session that produces a
better argument for an item already on the board has not produced a new item.*

---

## §6 The honest state, since the operator said *"not quite there yet"*

**Agreed, and here is what "there" would require.** Three things are now settled enough to write down and
two are not:

| | State |
|---|---|
| **Why `(peer, path)` is insufficient** | **settled** — §0–§2, and it is derived from `GUIDE-RESOLUTION` §5's own split |
| **What a reference is, primitively** | **settled** — an entry point into §3's ladder plus advisory hints |
| **Which design** | **settled** — C, staged with D-27 |
| **What the string scheme is called** | **open — operator's call**, and it is the only blocked item |
| **Whether the atom covers all five entry points or two** | **open, and it is the next real design question.** Two is what we have; five is what Nostr shipped; the taxonomy floor's test (*must a conformant consumer behave differently?*) has not been run on the other three |

**The last row is the one to take next**, and it is cheap: run the floor test on `name`, `peer` and
`transport` as reference targets. If a consumer must behave differently holding each, they earn shapes;
if not, they are strings that resolve into the two we have.

---

## §7 Sources

**In-corpus, opened this session or the immediately preceding ones and cited by section:**
`GUIDE-RESOLUTION` §1 (P1/P2), §3 (the entry-point table), §5 (**self-verifying vs authority-scoped — the
load-bearing section**) · `ENTITY-SYSTEM-REFERENCE` §228 and the `entity://` dispatch examples ·
`EXTENSION-NETWORK` §6.5.1a, §6.5.4 · `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §2, §4, §4.1, §11 ·
`PROPOSAL-APP-CONVENTION-FEED` §2.2–§2.2.4.

**External, opened this session:**
[NIP-01 — replaceable and addressable events, `e` vs `a` tags](https://github.com/nostr-protocol/nips/blob/master/01.md)
(*"an `a` tag references… whichever version is currently latest"*; `kind:pubkey:d-tag` coordinates) ·
carried from the predecessors: [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) ·
[ATProto AT-URI](https://atproto.com/specs/at-uri-scheme) ·
[Matrix appendices](https://spec.matrix.org/latest/appendices/).

## §8 What is unread

- **`entity://` still has no located normative home.** Third document to say so. §2's account of what it
  means rests on `ENTITY-SYSTEM-REFERENCE` §228 plus consistent usage in five documents. **Before a
  sibling scheme is minted, find the authority or establish there is none** — and if there is none, that
  is a finding about the corpus, not about this design.
- **The taxonomy-floor test on the three uncovered entry points** (§6's last row) — named, not run.
- **Both app-tier trees.** Fourth document to name this. Still the cheapest unexecuted check on the
  board, and it now bears on §4 as well: **a renderer may already be branching on something to decide
  how to handle a link**, and whatever it branches on is the de facto type hint.
- **NIP-19's `naddr` TLV encoding in detail.** §3's table row is from NIP-01's coordinate semantics plus
  NIP-19's TLV list; the exact `naddr` field layout was not re-read for this document.
