# EXPLORATION — the reader's loop: how a thread reconstructs without a central feed, what the identifier looks like on screen, and what forking a site actually costs

**Status:** Exploration (design record). Not a proposal, not normative.

**The questions, in the operator's words.** *"Someone posts, some other user posts a reply — that reply
lives on the replying person's published state. How you aggregate that as a client into a consolidated
view is a bit more complicated, because we don't have a central feed. You've got to walk the network,
reconstruct it."* · *"When you look at the event stream for ATProto you see DID, PLC, keys. What does
ours look like?"* · *"I'm browsing a site, I can fork a page, publish it at my fork… technically seems
pretty feasible."*

**Why these three together.** They are the three places where *"how does it work in practice"* differs
most from *"how does it work conceptually"*, and all three are about **the client**, which is the part
of the stack the corpus has the least written about.

---

## §0 The result

**1. The thread problem has a zero-infrastructure answer, and it is not a compromise.** Show the
replies written by people you follow. That costs nothing — you already pulled their streams to build
the timeline — and it is what **SSB has done in production for a decade**. Everything else (mirrors, an
author's own reply index, an aggregator) is an **additive coverage upgrade**, not a prerequisite. The
honest framing is not *"we can't see the whole thread"*; it is ***your thread is your network's
thread***, and the alternative systems buy completeness by re-introducing an operator.

**2. All four surveyed systems solve the backlink asymmetry the same three ways, and each way has an
owner.** ActivityPub makes the parent's *server* forward (a W3C MUST, with a named failure mode when it
does not). ATProto answers `getPostThread` from an **AppView that consumed the entire firehose**.
Nostr answers `{"#e": [id]}` from **relays**, over whatever each relay happens to hold. SSB is the only
one with no hub — **and it works by replicating the whole follow neighbourhood**, which is route 1
above with the storage bill attached.

**3. Notification is where our design has a specific, deliberate cost that should be stated rather than
discovered.** `EXTENSION-INBOX` delivery is **grant-scoped by construction** — a `deliver_token` the
recipient minted. So *"a stranger replied to you"* is **not** free: it requires the parent to publish an
open delivery grant, and that grant is exactly the spam route the corpus elsewhere claims we do not
have. **The trade is real and it is the right way round** — but *"replies notify the author"* cannot be
assumed by a client design.

**4. The identifier question has a sourced answer that is better for us than expected.** ATProto's
abstraction over keys is `did:plc`, whose own specification says **"a central directory server collects
and validates operations"** and **"we expect to evolve the system… into something less centralized —
likely a permissioned DID consortium."** Their friendly layer is an operator. **Ours already has the
same three presentation rungs without one** — and the fourth rung, a custodial product that hides keys
entirely, is legitimate and is the operator's own example.

**5. Forking a site is three rungs, and the middle one needs no extension and no DAG.** Measured today:
`entity-browser-rust` has **zero** `system/revision` call sites, `entity-workbench-go` has **12**, and
the site convention **deliberately specifies a tier with "no revision DAG required."** So the operator's
*"most sites aren't in revision"* is correct **and is by design**. Fork-with-provenance costs **one
reference field**, which is the atom that landed last week.

---

## §1 The backlink asymmetry, stated once

**A reply is written by the replier, in the replier's namespace, and the parent author is not
involved.** That is not our peculiarity — it is true of every system where the author owns their own
storage. It produces one question that every social system must answer and that no data model answers
by itself:

> **Holding entry X, how do I find the entries that point at X?**

**References run one way.** The reply names its parent; the parent cannot name its replies, because
they did not exist when it was written. So the reverse edge has to be **built by somebody, somewhere,
after the fact** — and *who builds it* is the entire design question. It is the same shape as the
web's backlink problem, and it is why every search engine is an index and not a protocol.

**What the corpus already settles, and what it does not.**
`EXPLORATION-WHAT-CONVERGES-…` §4.2 establishes that **merging two views of a thread is set union over
signed content-addressed entries** — lossless, order-independent, unforgeable — and §4.3 that a view is
therefore *never wrong, only short*. **Both are about views you already hold.** Neither says how a
reader **obtains** a view, and §4.4's *"what is missing is navigation"* is aimed at topic discovery
rather than at reply discovery. **That gap is what this document is for.**

---

## §2 How the four systems actually do it

### §2.1 ActivityPub — the parent's server is the hub, by normative requirement

A reply carries `inReplyTo`, and the spec tells the client to look at it: *"Clients SHOULD look at any
objects attached to the new Activity via the `object`, `target`, `inReplyTo` and/or `tag` fields,
retrieve their `actor` or `attributedTo` properties"* — i.e. **address the reply to the parent's
author.** That gets the reply to one inbox.

**Then §7.1.2 does the real work, and it is a MUST:** when an activity arrives in an inbox, the server
**MUST** forward it to the values of `to`, `cc` and/or `audience` — conditioned on, among other things,
**the `inReplyTo`/`object`/`target`/`tag` values being objects owned by that server.**

> **Read that condition carefully: the obligation attaches to the server that owns the parent.** The
> parent's instance becomes the thread's distribution hub and re-broadcasts replies to the parent's
> followers. **The named failure mode when it does not happen is "ghost replies"** — replies that exist
> but appear disconnected, because the only party positioned to relay them did not.

**Cost:** the parent's server must be up, must be willing, and must not have defederated the replier.
Thread completeness is a property of one instance's connectivity.

### §2.2 ATProto — a global index, and the thread is an API call

`app.bsky.feed.getPostThread` takes `uri`, **`depth` (0–1000, default 6)** and **`parentHeight` (0–1000,
default 80)**, and returns a `threadViewPost` with parent and replies — plus a `threadgate` view.

**Three things that shape is telling you.** *(a)* **The client does not walk anything** — it asks one
endpoint for a subtree of a graph. *(b)* The endpoint can only answer because an **AppView** has
consumed the network's firehose: a Relay *"syncs the repos from PDSes and produces a firehose of change
events"*, and an AppView *"aggregat[es] repo data across the network"* — the glossary's own analogy is
**a search engine**. *(c)* `threadgate` is the author asserting control over *their* thread, which is
the same instinct as §3.3 below, arriving as a lexicon.

**Cost:** somebody runs the AppView, and it must see everything to answer completely. This is the
component people mean when they say Bluesky is federated-with-an-asterisk, and it is a fair description
of the *aggregation* tier specifically.

### §2.3 Nostr — the relay holds the reverse index

A client subscribes with a filter, and tag filters are first class: `{"#e": ["<event id>"]}` returns
events referencing that event. NIP-01 gives exactly this example.

**So Nostr's backlink answer is: ask a relay, and get that relay's coverage.** No relay is required to
be complete and none is; clients hedge across several. **That hedging is the documented centralizing
pressure** — the more clients hedge onto the biggest relay, the more complete the biggest relay becomes
— and the corpus already records that this is what the never-built gossip layer was supposed to
prevent.

### §2.4 SSB — no hub, and the bill is replication

SSB has no relay, no AppView and no inbox forwarding. **A peer replicates the logs of the peers within
its follow neighbourhood** (typically two or three hops), and threads are assembled locally from logs it
already holds. **Backlinks are free because the data is local.**

> **SSB is route 1 (§3.1) taken to its conclusion, and it is the deployed proof that the route
> works socially** — SSB threads *feel* complete to their users, because the people you would ever see
> are in your neighbourhood by construction. **Its cost is the honest one: you store your
> neighbourhood.** That is the trade our pull model deliberately does not force, and it is why our
> route 1 is cheaper *and* shorter-sighted than theirs.

### §2.5 The pattern

| System | Who builds the reverse edge | What it costs | Failure mode |
|---|---|---|---|
| ActivityPub | **the parent's server**, by MUST | the parent must be up and willing | ghost replies |
| ATProto | **an AppView** over the firehose | someone indexes the whole network | the aggregation tier is the centralized one |
| Nostr | **relays**, per-relay coverage | clients hedge | hedging concentrates onto the biggest relay |
| SSB | **nobody — you replicate the neighbourhood** | storage | you cannot see outside the neighbourhood |

**Nobody has a fifth answer.** Either a party builds the index, or you hold the data yourself. **This is
a closed design space, and knowing it is closed is worth more than another survey** — it means our job
is choosing which of the four, not inventing a fifth.

---

## §3 Our four routes, in cost order — and route 1 needs nothing

### §3.1 Route 1 — replies by people you follow. **Free, and it is the v1 answer**

**You already have them.** Building the timeline means pulling each followed peer's stream; a reply is
an ordinary entry in that stream carrying `reply.parent` and `reply.root`. **Indexing them by parent as
they arrive is a client-side dictionary**, and no protocol, extension, grant or aggregator is involved.

**What the reader sees:** under any post, the replies from people they follow. **What they do not see:**
replies from strangers.

**Why this is a feature and not an apology.** *"Your thread is your network's thread"* is a coherent
product statement, it is what SSB users experience, and it is **the honest default in a system with no
gatekeeper** — the alternative is not *complete*, it is *somebody else's idea of complete*. And by
`EXPLORATION-WHAT-CONVERGES-…` §4.3 the view can only ever grow: **route 1 is a floor, never a ceiling.**

**One property makes route 1 strictly better here than in SSB:** we do not have to replicate the
neighbourhood to get it. A followed peer's stream is pulled from a static origin, and the reply arrives
as part of a fetch we were doing anyway.

### §3.2 Route 2 — mirrors. **Specified already; the coverage upgrade**

`FEED` §4 is exactly this: a reader publishes what they gathered, `entries` are republished **original
bytes** with each author's detached signature, and *a mirror can omit but never substitute*. **Reading a
mirror is how a reader gets replies from people they do not follow**, and merging mirrors is set union.

**So route 2 is route 1 plus other people's route 1** — and it needs no new mechanism, only somebody
choosing to publish what they gathered. **This is the aggregator tier, and its verifiability is the
thing no surveyed system has cleanly**: an ATProto AppView or a Nostr relay can silently omit too, but
you cannot check them; a mirror hands you signed bytes you can verify without trusting the gatherer.

### §3.3 Route 3 — the author publishes their own reply index. **The comment section, and it has a prerequisite**

For *"comments under my blog post"*, the author is the natural gatherer: they publish a mirror of the
replies to their own entry, and any reader of their site gets the thread in one fetch. **This is
ActivityPub's answer and ATProto's `threadgate` instinct, without either's operator** — the author
curates their own comment section, which the corpus already frames correctly as *curation and
moderation are the same act*.

> **The prerequisite is the finding: the author has to learn the reply exists, and that is not free
> here.** `EXTENSION-INBOX` delivers to an inbox **against a `deliver_token` the recipient minted** —
> §1.1's flow is a *caller* granting delivery for a result it asked for. **There is no unsolicited
> inbox**, by construction.
>
> **So "notify the author of a reply" requires the author to publish an open delivery grant** — and an
> open delivery route is precisely the spam surface the corpus claims we lack (*"spam needs a route,
> and every route is opt-in"*). **Both halves of that are true and they are in tension**, and a client
> design that assumes reply notifications is assuming an open route.
>
> **The cheap alternative that keeps the property:** the author **polls** for replies from the people
> they follow (route 1, run by the author), and anyone else's reply reaches them only if they read a
> mirror that carried it. **Notification degrades to discovery, and no grant is issued.** Where an
> author *does* want an open comment inbox, that is an explicit, revocable grant with a stated spam
> cost — which is a much better shape than a protocol-wide push.

### §3.4 Route 4 — an aggregator's reverse index. **The primitive exists and nobody has pointed it at this**

`EXTENSION-QUERY` §2.2 already specifies a **path link index** — `referenced_path → set of {source_path,
source_type, field_name}`, with *"Backlinks: what entities link to this path?"* as a named use case
(SHOULD, Level 1) — and §2.1's reverse hash index does the same for content hashes, at **MUST**.

**So an aggregator is: sync a lot of namespaces, and answer backlink queries out of an index the query
extension already defines.** That is a Nostr relay's `#e` filter and an AppView's thread endpoint, built
from a primitive that is already in the corpus for GC and impact analysis.

**And it inherits route 2's honesty property**: the aggregator returns signed entries, so it can omit but
never substitute, and a reader can check any subset against the authors' own trees.

### §3.5 The ordering, and what it means for the release

| Route | Needs | Coverage | Build |
|---|---|---|---|
| **1 — follows** | **nothing** | your follow graph | **v1** |
| **2 — mirrors** | someone to publish one (`FEED` §4, specified) | + whoever they read | v1, additive |
| **3 — author's index** | route 1 run by the author **or** an open delivery grant | the author's own comment section | v1 for the blog case |
| **4 — aggregator** | a peer syncing widely + QUERY indexes | broad | later, and it is the differentiator |

**Routes 1–3 are all buildable with what is specified today.** Route 4 is the one that needs a
proposal, and it is the one worth taking time over, because it is where the surveyed systems all
centralized.

---

## §4 The day-one loop, concretely

**What the operator asked for — *"first is first: follow a list of people, client starts, pulls the
latest, sorts chronologically"*** — written as the actual sequence, with what is specified and what is
not.

| # | Step | Mechanism | State |
|---|---|---|---|
| 1 | read my follow list | `app/feed/follow` entries in my own tree | **specified** (`FEED` §2.4) |
| 2 | for each followed peer: resolve name → peer | REGISTRY | **live, three-way green** |
| 3 | resolve peer → origin | published-root / transport set | **`http-poll` deployed**; bare-id case is `[OPEN-FEED-5]` |
| 4 | fetch their signed root, compare `seq` | `EXTENSION-NETWORK` §6.5.6 | **specified + deployed** |
| 5 | if it moved, walk the index head back to my cursor | `app/feed/index-head` → `index-page` chain | **specified** (`FEED` §3) |
| 6 | fetch new entries; verify each detached signature | `FEED` §1.1 | **specified**; the composer-signs obligation is new work |
| 7 | index replies by `reply.parent` as they land | client-side | **route 1 — nothing to specify** |
| 8 | sort by `created_at`, render | `FEED` §2.3.1 — the author's own unverifiable clock | **specified, with the honesty caveat** |
| 9 | **poll cadence, backoff, coalescing, what to do with a root I cannot fetch** | — | **NOT SPECIFIED — the gap** |

**Step 9 is the one thing in the day-one loop nobody has written**, and it was already named as roadmap
**P2**: *"RSS's entire operational character lives here and we have written none of it."* **That
assessment is still accurate.** Everything else in the loop has a document.

**Two things step 9 has to answer that RSS answers badly and we can answer well:**

- **A root you cannot fetch is ambiguous** — quiet publisher, dead origin, or withholding — and that
  indistinguishability is `[OPEN-FEED-4]`, unowned across the corpus.
- **`FEED` §5 already splits polled data into two freshness classes and says only one of them can be
  *wrong*.** The cadence design should fall out of that split rather than being invented beside it.

---

## §5 The identifier on screen

### §5.1 What ATProto actually does, from the source

`did:plc` is *"a self-authenticating DID method"* — the identifier is the hash of the genesis operation,
so it is cryptographically bound to its own history. **And its own README states the rest plainly:**

> **"A central directory server collects and validates operations, and maintains a transparent log of
> operations for each DID."**
>
> **"We expect to evolve the system (in a backwards-compatible manner) into something less
> centralized — likely a permissioned DID consortium."**

**So the friendly layer over ATProto's keys is an operator, acknowledged as such, with a stated
intention to become a consortium rather than to disappear.** Their alternative, `did:web`, moves the
trust to DNS and *"does not provide a mechanism for migration or recovering from loss of control of the
domain name."*

**That is the honest comparison, and it is not a criticism** — it is the same trade every system makes:
**a memorable, rotatable, recoverable identifier requires somebody to answer "what does this name mean
now."** The only question is who.

### §5.2 Our four rungs, and only one of them needs an operator

| Rung | What the user sees | Who must exist | What it survives |
|---|---|---|---|
| **0 — the key** | the public key / peer id | **nobody** | everything; it is the identity |
| **1 — a petname** | a local label the reader chose | **nobody** — `app/feed/follow.label` is *"local, never authoritative"* | everything, and it cannot be spoofed because it is not published |
| **2 — a registry name** | `alice.example` | a registry (which is a static file at a domain) | as long as the domain and the registry entry live |
| **3 — a custodial product** | an email login; the key is never shown | **a company** | as long as that company does |

**Rung 1 is underrated and is nearly free.** Petnames solve the *"I can't read a public key"* problem
**without any naming authority at all**, because the label lives in the reader's own tree. Every reader
gets their own nickname for you; nobody can take it; there is nothing to squat. **`FEED` §2.4 already
carries the field** — `label`, plus `via` for *"the identifier as typed or scanned, for provenance
display."* That pair is a complete petname design and nothing in the corpus points at it as one.

**Rung 3 is legitimate and is the operator's own example** — *"normal email signup, I give them a public
key ID, I store their private key, as far as they're concerned it's a standard website."* **Nothing in
the protocol forbids it and the data stays interoperable**, which is the whole point: a custodial
product's users publish the same entries as everyone else, and if the product dies the *data* is still
signed, still addressable, still mirrored. **That is strictly better than a platform, even when it looks
identical to one.** The honest disclosure is who holds the key.

### §5.3 The recommendation, and it follows the corpus's existing posture

**Do not hide the key; make it rarely necessary.** The reader's surface is rung 1 by default and rung 2
where a name exists; the key is what everything resolves *to*, is always inspectable, and is what
survives when rungs 2 and 3 fail. **The difference from ATProto is not that we lack an abstraction — it
is that our abstraction layers are optional and theirs is load-bearing.** A `did:plc` identifier is
useless without the directory; a peer id is not.

---

## §6 Copying a site and republishing it — three rungs, and versioning is the COPIER's choice

> **This section was written with a conflation in it and is corrected.** `[operator, 2026-09-05]`
> *"When I say fork a site I was speaking somewhat loosely… I'm not saying all sites are revisioned.
> As a user I can revision anything in my tree — I could copy their whole tree, start a revision on it,
> and modify it. The fork is also just a general term for: copy over this site, I want to change one
> page."*
>
> **Two errors, and the second one made the expensive rung look more expensive than it is.**
> *(a)* **"Fork" here means copy-and-republish**, not the git operation, and the primary act carries no
> ancestry requirement at all. *(b)* The original table said rung 3 needs *"the site versioned at
> publish time"* — **wrong, and it inverts who decides.** Versioning is applied to **a prefix in your
> own tree**, and `EXTENSION-REVISION` §8.7.2's own scope table lists **"Single foreign namespace
> `/{peerD}/` — tracks local view of one peer's data"** as a recommended scope. **So the copier
> versions their own copy, unilaterally, and the publisher is not involved and need not agree.**

**What is true about the measurement, restated without the inference:** `entity-browser-rust` has **one**
file referencing `system/revision` and **zero** revision call sites, `entity-workbench-go` has 26 files
and 12 call sites, and the site convention specifies a *"commit-wraps-tree shape without a revision
DAG"* tier stating **"no revision DAG required."** **That says browsing and publishing sites does not
route through REVISION** — which was the operator's point — **and says nothing about whether a user may
version a copy, because that is a different actor doing a different thing.**

| Rung | What you do | What it costs | What you get | What you lose |
|---|---|---|---|---|
| **1 — copy** | walk the closure (`tree:extract` + `content:ensure_closure`), change the page, republish under my root | **nothing new** — the operator's own sequence, and it works today | a working site of my own | no recorded ancestry; nobody can tell where it came from |
| **2 — copy + provenance** | rung 1, plus the changed page carries **a pinned `reference` to the source page's exact hash** and **a `live-reference` to the source path** | **one field** — both atoms landed at `FEED` §2.2 | attribution, *"this came from X at version Y"*, **and a reader can see the original has moved on since** | no merge |
| **3 — version my copy** | version the imported prefix in **my own** tree (REVISION §8.7.2's foreign-namespace scope; §7.3 if importing an existing DAG) | **my decision alone** — no cooperation from the publisher | full local ancestry of my edits, and **re-importing their later state and merging is §8.7.3's one-time graft** | the UI to drive it, and a graft cost on the first merge if they never versioned |

> **Rung 2 is almost certainly the v1 answer, and it is a nice result: the reference-atom pair that
> landed last week for a completely different reason — the rug-pull guarantee — turns out to be exactly
> the shape this provenance needs.** *What it came from* is a pin; *where it lives* is a live reference;
> and a reader holds both without either being wrong when the source changes.

**What is genuinely unanswered, and the operator named it:** licensing, and how much UI this takes.
**Neither is a protocol question**, and rung 2's honesty is what makes the licensing conversation
possible at all — a copy that records what it copied is attributable, and one that does not is
indistinguishable from plagiarism.

---

## §7 What this puts on the docket

| Item | Kind | Note |
|---|---|---|
| **The refresh loop** (day-one step 9) | **proposal-shaped, unblocked** | roadmap P2, still unwritten, and it is the only unspecified step in the whole first-run sequence |
| **Route 4 — the aggregator's backlink index** | **proposal-shaped** | the differentiator, and where every surveyed system centralized. `EXTENSION-QUERY` §2.2 is the primitive |
| **Reply notification is grant-scoped, and that should be said in `FEED`** | **wording** | a client design that assumes push notification is assuming an open spam route |
| **Petnames are already a complete design nobody has named** | **wording / guide** | `FEED` §2.4's `label` + `via`; the answer to *"I can't read a public key"* that needs no authority |
| **Fork-with-provenance at rung 2** | **convention-shaped** | one reference field on a site page; no REVISION dependency |
| **`[OPEN-FEED-4]`** — unfetchable root is ambiguous | **open, unowned** | the refresh loop forces it |

---

## §8 What this does not settle

1. **Route 4 is sketched, not designed.** Who runs an aggregator, what it claims, how a reader picks
   one, and how two aggregators reconcile are all open — and the reconciliation half is GOSSIP, which
   the roadmap already sequences after the aggregator contract.
2. **No cost model.** Nothing here says what route 1 costs a client following a thousand people, and
   `EXPLORATION-WHAT-CONVERGES-…` §6 is the document that started that and should be extended rather
   than duplicated.
3. **The ATProto AppView description is assembled from the glossary and one lexicon**, not from an
   AppView implementation. The claim *"an AppView consumes the firehose and answers thread queries"* is
   well supported by both; anything finer would need their source.
4. **§3.3's spam argument is structural, not measured.** No open delivery grant has been deployed and
   observed being abused.
5. **The petname claim is about the schema, not about a UI.** No client has been observed rendering a
   petname, and *"nobody can squat it"* is a property of the field's locality, not evidence that users
   find it usable.
6. **Rung 2 forking has no consumer yet.** It is derived from two atoms that exist; nobody has asked
   for it, and L26 says let the seat that builds it shape it.

---

## §9 Sources

**Primary, fetched for this document:**
[W3C — ActivityPub](https://www.w3.org/TR/activitypub/) (§7.1.2 forwarding from inbox; `inReplyTo`
addressing guidance) ·
[ATProto — glossary](https://atproto.com/guides/glossary) (PDS, Relay/firehose, AppView, DID, handle) ·
[ATProto — `app.bsky.feed.getPostThread` lexicon](https://raw.githubusercontent.com/bluesky-social/atproto/main/lexicons/app/bsky/feed/getPostThread.json)
(`depth`, `parentHeight`, `threadgate`) ·
[ATProto — DID specification](https://atproto.com/specs/did) (`did:plc` and `did:web` as the two blessed
methods; did:web's migration limitation) ·
[`did-method-plc` — README](https://github.com/did-method-plc/did-method-plc) (**the central directory
server; the stated intent to become a permissioned consortium**) ·
[Nostr — NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) (REQ filters; the `#e`
example) · [Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/)
(follow-graph replication — *carried from the first falsification test's read, not re-fetched*).

**Measured this session:** `entity-browser-rust` `d30f906` — 1 file referencing `system/revision`, 0
revision call sites · `entity-workbench-go` `64605f4` — 26 files, 12 call sites.

**In-corpus, opened for this document:** `EXTENSION-QUERY` §2.1–§2.3 (**the backlink primitive**) ·
`EXTENSION-INBOX` §1, §1.1 (**the grant-scoped delivery finding**) · `EXTENSION-REVISION` §6.3, §7.3 ·
`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §1.1, §6 (the no-DAG tier; the closure sequence) ·
`PROPOSAL-APP-CONVENTION-FEED` §1.1, §2.2, §2.3, §2.4, §3, §4, §5 · `EXPLORATION-WHAT-CONVERGES-…` §4,
§5, §6 · `EXPLORATION-THE-PUBLIC-SOCIAL-STACK-…` §5.3 · `ROADMAP-SOCIAL-TIER-CLOSEOUT` §1, §3 (P2).
