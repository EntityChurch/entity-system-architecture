# EXPLORATION — the two-level index: `coordinate → who`, and why a gatherer's output must be the same kind of thing as a source's

**Internal working document.** Design derivation. **Not a proposal, nothing normative, nothing folded.**

**The operator's question, and it is deliberately not about forums:** *"Is there some distributed data
structure that helps with this? How do you do a decentralized AppView? What do you agree upon in the
network, and how do you do a distributed index that lets you navigate it?"* — with the standing
constraint that **the pattern must be the deliverable, not the application**: *"revisions, repos,
forges, chat, forum, marketplace, messaging, encyclopedia — it doesn't matter. It's the pattern of the
information flow that we're standardizing."*

---

## §1 The general problem, stated without mentioning replies

**Strip the application away and every one of those examples is the same object:**

> **N independent writers, each owning only their own namespace, contribute to a subject nobody owns.
> A reader holding the subject must find the contributions.**

| Application | The subject | The contributions |
|---|---|---|
| Forum / social | a thread's root entry | replies |
| Forge | a repository | patches, issues, reviews |
| Wiki / encyclopedia | a page | edits |
| Marketplace | a listing | offers, bids |
| Chat / messaging | a room | messages |
| Revisions | a document | versions |

**Three things are needed and only the third is unsolved:**

1. **A coordinate** — a name for the subject that **any party can compute without coordinating.** A
   content hash does this (the thread's root entry, the listing, the page's canonical name). *We have
   this; it is what content addressing is for.*
2. **Contributions that name the coordinate** — signed entries in the contributor's own namespace.
   *We have this.*
3. **The reverse edge: coordinate → contributions.** *This is the whole problem, in every application,
   and it is one problem rather than six.*

**This is why the forum work kept bottoming out in "navigation."** Navigation is not a social-tier
concern that leaked; **it is the general problem wearing one application's clothes.**

## §2 The move: the reverse edge is TWO indexes, and everyone builds the expensive one

**This is the finding.** `coordinate → contributions` decomposes, and the decomposition is not
cosmetic:

| | What it maps | Size | Trust needed | Who can answer |
|---|---|---|---|---|
| **A. `coordinate → WHO`** | subject → the set of peer-ids that contributed | **~32 bytes per participant** | **none** | anyone |
| **B. `WHO → WHAT`** | a peer → their entries | their whole stream | **none** (signed, content-addressed) | the peer, or any mirror |

**Index B already exists and is not a new mechanism.** It is the fetch a client performs to build a
timeline: resolve the peer, get their signed root, walk the stream. **We built it and it is deployed.**

**So the only thing missing in the entire system is index A**, and index A is *small*.

**Every surveyed system builds a `coordinate → CONTENT` index instead**, which is A and B fused:

- An ATProto **AppView** answers `getPostThread` because it **consumed the whole firehose**. The
  glossary's own analogy is *a search engine*.
- A Nostr **relay** answers `{"#e":[id]}` over the events it happens to hold.
- An ActivityPub **instance** holds the thread because replies were forwarded into it.

**Fusing them is what forces the firehose**, and the firehose is what makes that tier the centralized
one — the operator's own observation about the Bluesky stream (*"who has time to read all that"*) is a
statement about the cost of the fused index, not about the network.

### §2.1 Why splitting them changes the trust model, not just the size

**Index A cannot lie in the direction that matters.** If a participant index says *"Bob contributed to
C"*, you go fetch Bob's entry and verify its signature and that it names C. **A forged entry fails; a
fabricated participant costs you one wasted fetch.**

> **So index A can OMIT but cannot FORGE** — which is the property `FEED` §4.2 already ratifies for
> mirrors (*omit but never substitute*), arriving on a much smaller object.

> **⚠ Correction to this document's first draft.** It said *"roughly three orders of magnitude
> smaller"* and that is **wrong**. Per edge it is a peer-id (~32 bytes) against a stored entry
> (~300–1000 bytes) — **roughly 10–30×, not 1000×.** For a thread with 50 repliers a participant set is
> ~1.7 KB against ~25 KB of entries.
>
> **The size win is real and it is not the structural one. The structural win is SELECTIVITY.** An
> AppView must ingest **everything**, because it cannot know which post will be queried. **A gatherer
> covers only the coordinates it chose** — one community, one repo, one topic — and is complete-ish
> there while holding nothing else. ***That is the difference that matters, and it exists only because
> index A is separable: you cannot selectively fuse.*** A fused index is all-or-nothing by
> construction; a split one is scoped by choice.

**And omission is repairable by union.** A participant set is a **set of peer-ids**, so merging two
partial indexes is set union — associative, commutative, idempotent, no ordering, no conflict. **Two
partial answers compose into a better one with no coordination and no trust in either source.**

**That is the "distributed data structure" the question was reaching for**, and it is not exotic: *the
thing you replicate is a grow-only set of identities, and the content stays where it was written.*

### §2.1a The base case needs NO reverse edge at all, and this is the thing not to lose

`[operator: if Alice follows me, everything I do is something Alice wants to know about — so she can
simply come and look at what I am doing. Three requests to their trees.]`

**A reply is an entry in the replier's own stream.** So if Alice follows Bob, Bob's reply arrives in
the fetch Alice was already doing. **No index, no gatherer, no coordinate lookup — the reverse edge is
not consulted because it is not needed.**

**And the graph is discoverable FORWARD, which is the part the "reverse edge" framing obscures.** Every
entry carries its own outbound edges: my reply *names* Bob's post. So Alice, pulling only my stream,
**learns that Bob's post exists and can fetch it.** She discovers Bob without following Bob and without
anyone indexing anything.

> **Forward traversal from your follow set is a walk over the content graph, and it is free.** It
> discovers new peers, new subjects and context, one hop at a time. **This is `EXPLORATION-WHAT-
> CONVERGES` §4.4's already-ruled conclusion** — *traversal must be lazy: one level at a time, on
> demand, budgeted* — and it is the operator's *"lazy uncovering… the client brings in the context,
> maybe the first chunk of other threads."*

**So the reverse index answers exactly ONE query, and it should be scoped that narrowly:**

> **Given a coordinate, find contributors I do NOT already follow.**

**That is a far smaller claim than "how do threads work."** Threads work by following people. The
gatherer buys **reach past your own graph** — and nothing else.

### §2.2 The precedent, and it is the one that actually scaled

**BitTorrent already made this exact split.** BEP 5's DHT stores, per infohash, **peer contact
information — 6 bytes** — and, in the spec's own terms, *"never stores actual torrent files, file
content, or metadata… it exclusively maintains peer locations."*

**A tracker is `coordinate → who`.** It was small enough to run on a hobbyist's server and it is why
the content layer never had to centralize. **The operator's instinct — *"maybe there's a tracker who
goes along with the registry"* — is this, and it is the right instinct.**

**Measured in our corpus: we have never used the idea.** `get_peers` **0** · `convergent index` **0** ·
`tracker` 19 hits, **all of them meaning "proposal tracker."** Our indexes are `name → peer`
(REGISTRY), `peer → next hop` (ROUTE/RELAY's DHT mentions, all routing), and `path → referrer`
(`EXTENSION-QUERY` §2.2, **local to one store**). **`coordinate → who` is absent from the corpus.**

#### §2.2a Converting the design — the mapping, and the one place it breaks

| BitTorrent | Ours | Note |
|---|---|---|
| infohash | **coordinate** (content hash of the subject) | both computable by anyone, no coordination |
| peer list | **participant set** | both grow-only sets of identities |
| `get_peers` | fetch a participant set from a gatherer | **ours is a document fetch, not an RPC** (§3) |
| `announce_peer` | **contributor tells a gatherer** — see §5a | theirs is mandatory, ours is an optimization over polling |
| the token | **not needed** | the token stops UDP source-address spoofing for reflection; we are pull-based over HTTP and have no amplification vector |
| tracker / DHT | **gatherer** | we take the tracker; §3 says why not the DHT |

**Where the analogy breaks, and it is load-bearing:**

> **BitTorrent's peer set is a REDUNDANCY set. Ours is a COMPOSITION set.**

In BitTorrent every peer serves **the same bytes**, so peers are interchangeable and **a partial peer
list costs you nothing** — you still get the identical file, just from fewer sources. In ours **each
participant wrote something different**, so **a partial participant list means you literally get less
content.**

**That is why coverage is a real cost for us (§6) and is not one for them**, and it is why *"a tracker
is enough"* transfers as a mechanism but not as a completeness argument. **It also means our
participant set has an ordering/importance dimension theirs never needed** — fifty repliers are not
fungible, and which fifty you got matters.

## §3 The constraint that decides the shape: our peers are static

**A live DHT is not available to us, and our own corpus already measured why.**

`EXPLORATION-THE-OPERATING-MODELS-AND-THE-ALWAYS-ON-TIER` §2: *every system that survived has an
always-on tier that is not the user's device.* §3.6 records **IPFS/the DHT as "the ghost-town failure,
quantified."**

**And structurally: a DHT lookup requires live nodes to answer. Alice's peer is a static HTTP
backend.** She publishes and closes her laptop. She cannot answer a `get_peers` and cannot store
someone else's announce.

> **So index A must be PUBLISHED, not looked up live.** A participant set is *an entity in some peer's
> namespace*, fetched exactly like everything else — not an RPC against a live overlay. And it is the
> operator's *"registry service that wakes up, polls, and republishes its view."*

### §3.1 The above derivation was over-fitted, and the real reason is better

`[operator: "right now we're doing the HTTP backends because it's easy and makes sense, but it could be
live, it could be other mechanisms."]` **Correct, and it invalidates the argument as written.** The
static origin is **today's deployment choice, not a property of the system** — so a conclusion resting
on it expires the moment the always-on tier exists, which is a tier this corpus is actively designing.

**The conclusion survives on a stronger footing: an artifact is not a service.**

| | published walk | live lookup |
|---|---|---|
| after the publisher goes away | **still there** | gone |
| verifiable offline | **yes** | no |
| re-servable by a third party | **yes, by anyone** | no |
| merges with another partial answer | **union** | you get one answer |
| same type as everything else (§4) | **yes** | **no — it is an API** |

**Every one of those is a property of the artifact, not of the transport.** So even with every peer
always-on, **you would still want the published form**, because it is the only form that composes with
the rest of the system — and the last row is decisive, since a live endpoint reintroduces exactly the
type break §4 identifies as the cause of centralization.

> **A live overlay is therefore an ACCELERATOR over the same artifact, never a replacement for it.** A
> DHT among always-on peers would be a cache of published walks with a faster lookup path — additive,
> optional, and unable to say anything a walk could not.

**This is the corpus's own posture generalized: content-addressed signed artifacts are primary and
transports are interchangeable.** The static-origin argument was a *symptom* of that principle, mistaken
for the principle.

## §4 The closure property — the part that makes it federate instead of centralize

**A gatherer is just a peer that polls a set of sources and publishes what it found.** The question is
what shape its output takes, and there is exactly one answer that composes:

> **A gatherer's output MUST be the same kind of object as a source's output — signed entries in its
> own namespace.**

**Because then a gatherer is gatherable.** You subscribe to a gatherer the same way you subscribe to a
person; a gatherer can gather other gatherers; and a client merges gatherer output and source output
with the same code, because they are the same type. **That is the operator's *"you add people's
registries and you get the activity"*, and it works *because* of the type identity, not alongside it.**

**Contrast, and this is why every surveyed system's aggregation tier is special:** an AppView's output
is an **API**, a relay's output is a **query response**, an instance's output is **its own database**.
None of those is the same kind of thing as a user's post, **so none of them can be aggregated by the
next layer** — which is precisely why there is one AppView and not a mesh of them. ***They did not
centralize because of a policy choice; they centralized because their aggregator's output was a
different type from its input.***

**A gatherer is therefore not a privileged role.** It is a peer that chose to do work. Anyone can run
one, over any set of sources, with any policy. A client that trusts none of them can run its own over
the peers it follows — **which is route 1, i.e. the degenerate gatherer where the source set is your
follow list.** *The same mechanism spans "no infrastructure at all" to "a well-resourced public index",
with no discontinuity and no protocol change.*

### §4.1 There is no gatherer. Gathering is an ACT, and the noun should go.

`[operator: "we keep saying The Gatherer, but every peer publishes stuff, shares it — so they're all
gatherers. They all have publish capability."]` **This is a naming correction with a design consequence
and it should be taken.**

**A noun for the role invites a tier, and a tier is the thing §4 just proved unnecessary.** Every
sentence above that says *"a gatherer publishes…"* is more accurately *"a peer publishes a walk"* —
and the moment it is written that way, **the aggregation tier disappears from the vocabulary as well as
from the architecture.**

**The corpus already ruled the property that makes this exact.** `FEED` §4.1: *"A mirror is not
authorship, and it carries what proves that… attribution follows `entry.author`… a renderer that
attributes a mirrored entry to the gatherer is non-conformant."*

> **So PUBLICATION and AUTHORSHIP are already separate.** Publishing is the one universal act every peer
> performs. **What varies is only what you publish** — something you wrote, or something you walked.
> **They are the same act on the same rails**, which is why nothing new is needed to do the second.

**Consequently the honest vocabulary is:**

| Instead of | Say |
|---|---|
| *a gatherer* | **a peer** |
| *running a gatherer* | **publishing a walk** |
| *the aggregation tier* | *(nothing — there isn't one)* |

**"Gatherer" survives only as a relative term** — *the peer whose walk I happened to find useful* —
which is a statement about the reader, not a role in the system. The document keeps the word below for
continuity with its own first draft; **the object is a published walk and the actor is a peer.**

> **⚠ This section over-corrected, and the operator pushed back the same day.** *"I don't think the
> noun goes — you're going too far in that too. It's not necessarily a required dedicated role, but
> there may still be dedicated roles that do stuff."* **Right, and the distinction is between REQUIRED
> and EXISTING.** What §4 proves is that no role is *required* and none is *privileged*; it proves
> nothing against roles that people choose to run, and there is at least one job that **only an
> always-on party can do at all**: *metering a non-publishing client's data into the tree.* **A peer
> that cannot publish cannot participate in the write direction without one.**
>
> **So the corrected claim is narrow: do not name a role in the mechanism.** The object is a published
> walk, the act is publishing, and any peer may do it — **and dedicated operators are a legitimate,
> expected deployment shape rather than an architectural necessity.** Deleting the noun from the
> vocabulary was the right move; deleting the *possibility* was not.

### §4.2 Organic and dedicated walking are the same mechanism at different intensities

**Because publishing a walk is optional and continuous, there is no service commitment anywhere** — no
uptime, no coverage promise, no SLA. A peer publishes what it happened to walk, when it feels like it.
That yields a **spectrum**, not two tiers:

- **Organic** — everyone publishes their own walks. Coverage emerges from attention.
- **Dedicated** — somebody systematically polls a declared scope. Coverage is deliberate.

**And organic walking has a property dedicated crawling does not: it is aligned with attention.** People
walk what they care about, so the union of published walks is weighted toward what was actually read —
**the web's link-graph signal, obtained without a crawler**, because the walk *is* the reading.

**The failure mode is the same one that comes with it, and it should be said in the same breath: this is
a popularity bias.** Content nobody walked is content nobody can find by this route. **Dedicated walking
over a declared scope is the corrective**, which is why the spectrum matters and why neither end is the
answer alone.

## §5 The flow, traced end to end, with nobody running a server

**Alice's laptop is closed and Bob's laptop is closed. This still works.**

1. **Alice publishes** an entry to her static origin. Signed, content-addressed. She goes away.
2. **Bob replies** — an entry in *his* namespace naming Alice's entry hash as the coordinate. Publishes
   to *his* static origin. He goes away.
3. **A gatherer wakes** on its own schedule. It holds a source list (Alice, Bob, and others). It polls
   each signed root, notices Bob's stream moved, walks the new entries, and sees one naming coordinate
   `C`.
4. **The gatherer publishes a participant set** for `C`: *"peers {Bob} have contributed to C, observed
   at T."* Signed, in the gatherer's own namespace, content-addressed. **It republishes nothing of
   Bob's** — index A only.
5. **Carol holds `C`** and wants the thread. She asks one or more gatherers she has chosen, unions the
   participant sets, and gets `{Bob, …}`.
6. **Carol fetches from Bob directly** (or from any mirror), verifies Bob's signature and that his
   entry names `C`. **She has now verified the thread without trusting any gatherer**, and without any
   party having held both halves.
7. **Alice learns of the reply the same way** — she reads a gatherer she subscribes to. **No inbox, no
   open delivery grant, no spam surface**, because she *pulled* from a party she chose.

**Step 7 is the important one and it changes an open item.** `PROPOSAL-THE-REPLY-HINT` exists because
`EXTENSION-INBOX` has no unsolicited route, so an author cannot learn of a stranger's reply. **A
gatherer dissolves that problem rather than solving it**: notification becomes an ordinary pull from a
source you chose. **The hint's own §5.3 named this as the strategic question — *"does route 4 make this
redundant?"* — and the answer now looks like yes.** The hint's remaining value is latency and the
no-gatherer case, which is a much smaller claim than it was written with.

## §5a Announce to the GATHERER — the push half, and it puts the notification where it belongs

**§5 step 3 has a gatherer polling. Polling is a latency floor and BitTorrent did not accept one:
`announce_peer` is the contributor telling the tracker directly.** The same move is available here and
it is better placed than the version this corpus tried first.

**`PROPOSAL-THE-REPLY-HINT` proposed sending a contentless hint to the AUTHOR**, which required the
author to publish an open delivery grant — a spam surface to be bounded. **Send the same hint to a
GATHERER instead and the problem evaporates:**

- **A gatherer's entire purpose is to receive these.** Accepting announces is not an imposition on it;
  it is the job. There is no unwilling recipient anywhere in the flow.
- **It verifies by fetching**, exactly as Webmention's receiver MUST — so an announce asserts nothing
  and a forged one dies on verification.
- **It can rate-limit, require registration, charge, or ignore announces entirely and just poll.**
  Policy is one party's, locally, with no protocol-wide open route.
- **Nobody's inbox is opened.** Alice never receives anything she did not pull.

> **The reply hint was the right mechanism pointed at the wrong recipient.** Notification is a
> *gatherer* concern, not an *author* concern — because the gatherer is the only party in the system
> whose job is to hear about things it did not ask for.

**Announce stays an optimization, never a requirement.** A gatherer that only polls is fully
conformant and slower; a contributor that never announces is still found. **That keeps the static tier
whole**: announcing requires a live recipient, and the recipient is the gatherer, which is the one
party in §3 that *is* always-on by construction.

## §5b It is a graph, not an index — and the reader is a gatherer

`[operator: "I don't really think of it as an index. It's more of a distributed database graph that
has navigational properties of discovery."]`

**That framing is more accurate than this document's title and it changes what is being built.**

**The graph already exists in full.** Every entry carries its outbound edges; the union of everyone's
streams *is* the database. **Nobody is constructing a graph — a gatherer materializes one direction of
a traversal that was always walkable forward.** So a participant set is not a new source of truth. It
is **a cache of a walk**, signed by whoever walked it, and it is *checkable against the graph itself*,
which is exactly why it needs no trust (§2.1).

**And the closure property (§4) means a reader who publishes what they walked IS a gatherer.**

> `[operator: "if you choose to share what you browse as well, you become a publisher of all this
> information… you can decide what's relevant to publish. Maybe it's just the metadata."]`

**That is the whole tier, and there is no threshold to cross.** A client already walks the graph to
render a thread. **Publishing that walk costs one signed entity, and it is immediately usable by
everyone else** — because it is the same type as everything else (§4). *Every reader is a gatherer that
has not published yet.*

**"Maybe it's just the metadata" is the right instinct and it is the design's own gradient:** a
publisher chooses **how much of the walk to publish**, and the choices form a ladder —

| What is published | Size | What a reader gets |
|---|---|---|
| **the participant set** — coordinate → peer-ids | ~32 B/participant | who to go ask (index A) |
| **+ entry hashes** | +32 B/entry | what to ask for, dedups against what they hold |
| **+ the entries themselves** | full | a mirror (`FEED` §4) — no second fetch needed |

**All three are the same object with more of the walk included**, all three are verifiable, and all
three merge by union with each other. **There is no separate "index format" and no separate
"aggregator protocol" — there is one publishable walk with a knob on it.**

## §5c The operator's four questions, answered as far as they can be

- **"How do you filter and aggregate on what you're interested in?"** — **the gatherer's declared
  scope**, and it is the mechanism *and* the product differentiator. A gatherer says what it covers
  (this source list / these coordinates / this topic) and that declaration is the filter. **Different
  scopes are different gatherers, and a reader picks.** Nobody has to agree on one view.
- **"What's the size?"** — §2.1's corrected numbers. **~32 B per participant; ~1.7 KB for a 50-reply
  thread; and the real lever is selectivity, not per-edge size.**
- **"How many?"** — **unbounded, and that is the point.** They are forkable, unprivileged, and
  composable; the degenerate one is a client walking its own follow list.
- **"How do you distribute it?"** — **no new mechanism.** Signed entries in the gatherer's namespace on
  a static origin, pulled exactly like a person's stream. **That is what §4's type identity buys**, and
  it is why there is nothing to design here.

## §5d What a firehose looks like when it is published instead of streamed

`[operator: "when we view the firehose now, we're connecting to an API and it's streaming us all these
events. If we had a firehose that publishes every 30 seconds or every minute in chunks of events that I
can page through…"]`

**That is the right shape and the corpus already specifies the structure — `FEED` §3.2.** A stream is a
**key-addressed page chain**: a head at `/{peer}/app/feed/index` naming the current page number, and
pages at `/{peer}/app/feed/index/{page}`. A reader polls the head and pages back to its cursor.

**A published firehose is that chain with walks as the page contents instead of the author's own
entries.** Nothing new: same head, same paging, same resume-at-cursor.

> **And §3.3's reason for key-addressing rather than hash-chaining matters more here than it does for a
> personal feed:** a hash back-chain makes page *N*'s identity depend on *N−1*, so **correcting one old
> chunk republishes the entire archive.** For a high-volume firehose that is fatal. Key addressing makes
> a correction O(tree depth).

**What changes versus a streaming API, and the first one is the practical unlock:**

| | streamed | published in chunks |
|---|---|---|
| **cacheable** | **no** — a socket cannot be cached | **yes — immutable content-addressed objects, ideal CDN objects** |
| consumer must be online | yes (or the server buffers per-cursor) | **no** — resume at any time from your page number |
| history re-servable | only by the origin | **by anyone who fetched the chunks** |
| gaps | invisible | **visible** — page numbers are dense |
| **latency** | sub-second | **the chunk period — the operator's 30–60 s** |

> **The trade is latency for cacheability, offline operation, and replay-by-anyone** — and for the
> stated use case (someone building an index or a search corpus) minute-latency is irrelevant, while
> **cacheability is what makes serving a firehose nearly free instead of an operating cost.** A
> streamed firehose is a per-consumer connection; a chunked one is a static object served identically
> to everybody, which is the same economics that make the whole publishing model work.

**Where the trade fails is where liveness is the product** — direct messaging, presence, typing
indicators. **That is the honest place for a live tier**, and it is a much narrower claim than "you need
a server to participate."

**Staleness needs no new machinery, and the operator's instinct is the ruled position:** *"it doesn't
really matter if someone's got an out-of-date view — you just see the out-of-date view."* Correct. A
walk carries `walked_at` and a stream carries a page number, so **staleness is visible; and it is
repairable by reading one more source.** What is *not* obtainable is proof of currency against a hostile
origin — `TREE` §3.3a already says so, and it is rung 3 of the verification ladder, not a gap in this
design.

## §6 What this costs, honestly

- **Coverage is unverifiable and always will be.** A gatherer's source list is its own choice; you
  cannot know you have every participant. **This is the append-only law** from the CT work
  (register **L-8a**): completeness requires a single writer at some scope, and a subject nobody owns
  has none. **Union of several gatherers improves coverage monotonically and never certifies it.**
- **Latency is a gatherer's polling cadence**, not a push. That is the RSS operating character, and it
  is the step of the day-one loop that is still unwritten (roadmap **P2**).
- ~~**Somebody has to run one.**~~ **Overstated — corrected per §4.1/§4.2.** Nobody *runs* anything:
  **publishing a walk is an act with no service commitment** — no uptime, no coverage promise, no SLA —
  and a peer that publishes once and stops has still contributed a permanently valid artifact (§3.1).
  **What genuinely costs something is DEDICATED walking over a declared scope**, and that is real work
  someone must choose to do. **The residual cost is therefore reach, not infrastructure**: without
  dedicated walkers you get organic coverage weighted by attention, with the popularity bias §4.2 names.
- **Bootstrapping: how does Carol find a gatherer for `C`?** Unsolved here. The obvious candidate is
  that the registry — which already maps names to peers — also names gatherers, which is the operator's
  *"tracker alongside the registry."* **Not derived; flagged.**

## §7 What would falsify this

| Claim | What refutes it |
|---|---|
| **§2 — the split is the whole win** | A workload where knowing *who* is useless without also holding *what* — e.g. a query over content the participants never publish in a walkable stream. **Full-text search is the obvious candidate and it may genuinely need the fused index** |
| **§4 — type identity is why they centralized** | An aggregation tier in the wild whose output *is* the same type as its input, that still centralized. **Nostr relays are the place to look** — a relay stores and serves events, which is close to type-identical |
| **§3 — a live DHT is unavailable** | An always-on tier dense enough to host one. If the hosted tier exists, a DHT becomes possible again — **so this is a claim about today's operating model, not a permanent one** |
| **§5.7 — a gatherer removes the notification problem** | A case where an author must be told *promptly*, where polling latency is unacceptable. **Direct messaging is the obvious one** |
| **§2.1 — index A cannot forge** | A participant claim that is *expensive* to disprove. Verification costs one fetch; an attacker naming 10,000 fake participants costs the reader 10,000 fetches. **Rate limiting and per-gatherer reputation are unaddressed here** |

## §7a The sharpest falsifier, run: does full-text search break the split?

**§7 named this as the most likely refutation, so it gets worked rather than listed.**

**The objection.** A participant set tells you *who*. **Search asks a question about content** — *find
entries containing "sourdough"* — and no amount of `coordinate → who` answers it, because the query
predicate is over bytes the index does not hold. If the flagship query of any real social product needs
the fused index, the split is a local optimization dressed as an architecture.

**The objection is correct about search and wrong about the split, and the distinction is precise:**

> **The split works for COORDINATE-ANCHORED queries and cannot work for CONTENT-ANCHORED ones.**
> *"What contributes to C"* has an anchor you already hold. *"What contains X"* has no anchor at all —
> it is a query over the whole corpus by construction, and **that is a property of the question, not a
> deficiency of the index.**

**What saves it is that the design already contains the answer, at the honest price.** §5b's knob has a
third rung — **publish the entries** — and a peer that publishes walks *at that rung, over a declared
scope*, is a search corpus. So:

- **Search needs someone holding content. That is unavoidable and is true of every system.**
- **It does not need a NEW mechanism** — it is the existing object with the knob turned up, so search
  does not fork the architecture.
- **Selectivity still does the work**: a search index over *one community, one forge, one topic* is
  tractable where a global one is not, and **scoped search is what most people actually want.** A
  global search over everything is the thing only a firehose consumer can offer, and **we should say
  plainly that we do not offer it** rather than pretend the split covers it.

**So: not a refutation, and a real limit.** The claim that survives is *"the split removes the firehose
from the queries that have an anchor"*, which is most navigation. **The claim that dies is any
suggestion that nobody ever needs to hold a lot of content** — for global content-anchored search,
somebody does, and that party is a mirror at scale, not a tracker.

**Second-order, and it is the interesting part:** because a search corpus is *also* just published
walks, **a search provider is not structurally different from any other peer** — no privileged position,
forkable, and its output (results) should itself be a published walk if it wants to compose. **The
type-identity rule (§4) is what keeps even the heaviest tier from becoming an operator.**

## §8 What this puts on the docket

1. **The published-walk object** — a coordinate, a set of peer-ids, an observation time, signed, with
   **§5b's optional rungs** (entry hashes, then entries). **One entity type with a knob, and a merge
   rule (set union)** — not three formats. This is the deliverable and it is proposal-shaped.
2. **The gatherer contract** — what it claims (*"I polled this source list at time T"*), what it must
   not do (substitute), and the **type-identity requirement** in §4, which is the load-bearing clause.
3. **`PROPOSAL-THE-REPLY-HINT` is not superseded — it is RE-AIMED.** §5a: the mechanism was right and
   the recipient was wrong. **Rewrite it as announce-to-a-gatherer**, which needs no open delivery grant
   from anyone, because the gatherer is the one party whose job is to hear about things it did not ask
   for. That is a better proposal than either the original or "delete it".
4. **Gatherer discovery** (§6, unsolved) — likely registry-adjacent.
5. **Route 4 in `THE-READERS-LOOP` §3.4 is this document**, arrived at from the general problem rather
   than from the forum, and it should be cross-referenced rather than restated.
