# EXPLORATION — the key as hub: naming as a topology, the content primitives, and the republication network

**Status:** Exploration (design record). Not a proposal, not normative.
**Questions it answers:**
*(a)* Does reaching someone always have to look like *a name at a host* — or is naming a **map** with
many routes to one point? *(b)* If I hold a public key, **how do I reverse it**? *(c)* If I lose a
domain, what have I actually lost? *(d)* What is the **minimum content vocabulary** that covers
posting, replying, boards, albums, feeds and forums across the systems people actually use?
*(e)* What kind of network do you get when **participating in a discussion is itself an act of
publishing**?

The companion document traces the chain from identifier to entity and finds the content model
missing. This one goes underneath it: the **shape** of the naming layer, the **reverse** direction
nobody specified, and a network topology that falls out of publishing rather than being designed.

---

## §1 Naming is a topology, and the key is its hub

The framing that produces bad designs here is *"a name is a lookup at a host"* — a chain, with one
canonical form, of which everything else is a variant. Every deployed system that started there has
spent years unwinding it.

**The accurate shape is a hub.** Many independent routes lead **inward** to a key; the key is the
only fixed point; and a separate set of routes leads **outward** from it. Nothing about the inbound
route survives into the outbound side, which is precisely the property that makes the routes
substitutable.

```
        INBOUND — many routes, no privileged one              OUTBOUND — from the key
   ┌──────────────────────────────────────────┐          ┌──────────────────────────────┐
   │ a domain, via a DNS record               │          │ where is it reachable        │
   │ a domain, via a well-known HTTPS path    │          │ what has it published        │
   │ a name issued by a registry              │  ──────► │ what names does it claim     │
   │ a petname I assigned locally             │   KEY    │ what does it serve           │
   │ a key typed or pasted by hand            │  ◄────── │ what does it say supersedes  │
   │ a QR code or short code, in person       │          │   it                         │
   │ a rendezvous label, met out of band      │          └──────────────────────────────┘
   │ a link followed while browsing           │
   │ a reference inside someone else's data   │
   └──────────────────────────────────────────┘
```

**Three properties of the hub, and they are what make the map work.**

1. **The key is self-certifying.** It is not *a record about* an identity; it **is** the identity —
   a public key. There is no authority to consult and nothing to trust, because verification is
   checking a signature against the thing you already hold.
2. **Every inbound route is an unverified claim until it terminates at a key**, and once it does,
   the route is discardable. Two people arriving by different routes hold **the same** identity, and
   neither has to care how the other got there. This is why a system does not need one canonical
   naming method, and why insisting on one is a design error rather than a simplification.
3. **The trust question is per-route, and it is the only place it lives.** A DNS-based route inherits
   DNS's trust properties; a hand-typed key inherits none and needs none; a registry-issued name
   inherits the issuer's. **Mixing these is safe precisely because they converge on a key**, and
   dangerous only if a system forgets which route it took.

**The three-corner framing already in the corpus is this map, seen from one side.** Human-meaningful,
secure, decentralized: the classical claim is pick two. The answer here is not to beat it but to
expose all three corners as **different binding kinds under one resolver contract** — self-certifying
(secure + decentralized, not memorable), local-name (memorable + secure, not global), issued
(memorable + global, trust the issuer). **The user picks the corner per name.** What §1 adds is that
these are not three alternatives to choose between at design time; they are three *edges into the
same hub*, and a real deployment uses all of them at once for different contacts.

### §1.1 What this rules out, and what it licenses

**Ruled out:** any design where a peer's identity is a function of *how it was found*. That is the
Mastodon-shaped mistake — identity bound to the instance that hosts it — and it is what makes
migration there a rebuild rather than a move.

**Licensed:** adding naming routes forever without touching anything downstream. A new inbound route
is a new backend under the same resolver contract; it terminates at a key like all the others; no
content model, no follow record and no stored reference changes. **The naming layer can keep growing
and the data layer never learns about it.**

---

## §2 The reverse direction — and it is two questions, not one

*"If I get a public key, how do I reverse it?"* splits into two questions that are usually conflated
and have different answers.

| | The question | Status |
|---|---|---|
| **R1** | key → **where**: how do I reach this peer | ❌ **specified nowhere** |
| **R2** | key → **who**: what names does this key claim, so I can show a human something | ❌ **specified nowhere**, and not previously identified as separate |

### §2.1 R1 — key to endpoint

This is a known hole and it is the one real gap in the naming layer. Three cases; two work:
holding a **name** works (resolution yields a binding carrying glue transports), holding a peer you
have **contacted** works (its cached profile), and holding **only a key** has no path at all.

**Every id-first system in the field solves this**, each in its own idiom — a DHT whose primary
content is key → addresses, a DID document's service endpoint, a self-published relay list. Three
designs, one shape: **published by the id-holder, signed by the id-holder, fetched by id.**

The reason it stayed open is instructive. DNS is no guide, because **DNS is name-first by
construction** — nobody holds a bare address and asks what it is. We are **id-first**, so bare keys
circulate as first-class references everywhere (in rosters, in grants, in peer sets, in *replies*),
and the case DNS never needed is our common one.

### §2.2 R2 — key to name, and why it is not the same question

R2 is what a browsing user hits immediately: you follow a link, you arrive at a key, and you want to
show a person something other than a base58 blob.

**The naive answer — a reverse index from key to names — is the wrong shape**, for the same reason
DNS's reverse lookup is vestigial: it requires an authority to maintain it, it is unverifiable, and
it is a privacy leak by construction (enumerating who has claimed what).

**The right answer is already demonstrated in the field, and it inverts the lookup.** The key
**publishes the names it claims**, and the consumer **verifies each claim in the forward direction**.
One deployed system states the resulting rule normatively:

> *"Handles should not be trusted or considered valid until the DID is also resolved and the current
> DID document is confirmed to link back to the handle."*

So: the key says *"I claim `alice.example.com`"*; the consumer resolves `alice.example.com` by the
ordinary forward route; if it lands back on this key, the claim holds and the name may be displayed.
**Neither direction alone is sufficient and both together need no authority.**

**This is a defect class we have independently: a resolver that checks a binding's signature without
checking that the binding is *for the name that was asked* lets a host answer one name with a binding
legitimately issued for another.** Same hazard, two systems, and theirs is written into the
specification as a precondition. Ours should be.

### §2.3 R1 and R2 want the same record, and that is the design consequence

Both are *a signed, self-published statement by a key about itself, fetched by key*. Sequence-bounded,
so a stale copy is detectable; servable by anyone, because the signature travels with it.

```
a self-published peer record, signed by the key it describes
  key           the subject (and the signer)
  endpoints     where it can be reached           ← R1
  also_known_as the names it claims               ← R2, each verified forward before display
  serves        what it offers, if anything       ← the per-peer service question, also open
  seq           monotonic; a stale copy is detectable
  supersedes    the predecessor, if this key replaces one   ← see §3
```

**Recommending this as one record rather than three is the substantive proposal in this section.**
The three questions have one signer, one lifetime, one distribution path and one freshness rule.
Splitting them would produce three fetches, three staleness windows and three chances to disagree.

**One structural warning it must answer.** A transport profile today carries its key *inside* the
entity and nothing signs it; its authority is **positional** — trustworthy because it sits in that
peer's own namespace. **Position is exactly what is lost when an entity travels.** Fetched by hash
from a cache or a third party, a consumer holds a content-addressed blob asserting a key, and the
hash proves the bytes and never the authorship. Anyone can mint one. This is the state one major P2P
stack was in before it introduced signed peer records — and a record that is *meant* to be fetched
from anywhere must be self-authenticating or it is worse than useless.

---

## §3 What survives what — domain loss, host loss, key loss

The question — *if I lose a domain, is it still mine?* — deserves a table rather than a
sentence, because the three losses are genuinely different and only one of them is fatal.

| Event | Identity | Followers | Published content | Human name |
|---|---|---|---|---|
| **Domain lost / expires / seized** | ✅ untouched | ✅ still follow the key | ✅ intact, content-addressed | ⚠️ that claim stops verifying |
| **Host / CDN lost** | ✅ untouched | ✅ | ✅ re-servable anywhere by anyone | ✅ |
| **Key lost or compromised** | ❌ **gone** | ❌ **pointed at an abandoned identity** | ⚠️ orphaned but still verifiable | ❌ |

**Reading the first row is the answer to the question, and it is the strong result.** Because
followers follow *the key*, and because names are *claims by the key* rather than the identity
itself, losing a domain costs you **a label, not an identity**. The forward-verification rule in §2.2
is what makes this true rather than aspirational: a name that stops resolving back simply stops being
displayed, and nothing else in the system referenced it.

**Compare the field.** One major fediverse implementation binds identity to the instance, so losing
the instance is losing the identity and migration is a partial rebuild. One major protocol saw this
and built an indirection layer specifically so that the handle is a *changeable label* over a stable
identifier — which is the same conclusion, reached by adding a layer where we get it from the key
being the identity in the first place.

**And "migrate your followers" is nearly free in the first two rows.** If a follow entry names a key,
moving hosts is invisible to every follower — there is nothing to migrate, which is the honest and
slightly surprising answer. What we are missing is not migration; it is the *third row*.

### §3.1 The third row is the real open problem, and it is not solvable by wanting it

A peer that re-keys is, to the protocol, **a different peer**. A follow entry naming the old key is
not stale — **it is correct about a peer that has stopped publishing, forever** — and there is no
signal distinguishing that from a publisher who simply went quiet, or from an origin withholding
newer data. All three look identical to a follower.

**The obvious fix is the one that must not work.** A successor pointer signed by the **old** key is
the natural shape and is exactly what a compromised key must not be able to assert: whoever stole the
key can redirect every follower. So the naive design hands an attacker the entire follower graph.

**Two honest observations about the shape of any real answer.**

1. **It cannot live in the follow list.** A successor claim is a statement about an identity, so it
   belongs beside the identity — a binding kind, or the self-published record of §2.3 — not as a
   field each follower patches locally.
2. **The degraded mode is a specification item, not an implementation detail.** If continuity depends
   on an optional layer, then a publisher who rotates *correctly* is still invisible to a follower who
   did not adopt that layer, and both are conformant. **What a follower without it is entitled to
   assume has to be written down**, or two implementations will diverge while claiming the same tier.

**The question — *"can I have a cryptographically secure identity without putting a key
somewhere?"* — has a precise answer: no, and the layers above the key do not change that.** What they
change is *recovery*: a bare key with nothing attesting to it has **no rotation path under any
design**, whereas a key with prior attestations, a quorum, or a pre-committed successor has one.
**That is the whole value of an identity layer here — not security, which the key already provides,
but survivability of the identity across key change.** Which is why it is optional for a publisher
and load-bearing for a *long-lived* publisher, and those are different postures that should be named
as such.

---

## §4 The content primitives — ten products, six shapes

The systems named — the two decentralized protocols plus the mainstream social products and the
forum/board lineage — look like ten different data models and are not. Read as data rather than as
products, they reduce to **six shapes**, and the differences are which shapes a product emphasizes
and what its interface does with them.

| # | Shape | What it is | Appears as |
|---|---|---|---|
| **1** | **Entry** | authored content: author, time, body, optional media | post · note · status · photo · pin · article · link · comment |
| **2** | **Reference** | a *typed edge* from an entry to another object | reply-to · quote · repost/announce · link-to · mention |
| **3** | **Collection** | an ordered or curated set of entries | feed · timeline · board · album · gallery · subreddit |
| **4** | **Reaction** | a lightweight typed edge, valued in aggregate | like · favourite · upvote · reaction |
| **5** | **Relation** | an edge between two identities | follow · connection · subscribe · block |
| **6** | **Context** | the container an entry is *in*, distinct from who it is *for* | thread · discussion · board · group · conversation |

**Worked across the products, to show the reduction is real and not a tidy-up:**

- A microblog post = **Entry**, in the author's **Collection**, optionally carrying a **Reference**
  (reply/quote) and accumulating **Reactions**.
- A pinboard = **Entry** (the pin) whose defining feature is a **Reference** to an external source,
  organized purely by **Collection** (the board). The product *is* the collection layer.
- A photo service = **Entry** with media as the body, one **Collection** per author, **Reactions**
  and threaded **References**.
- A professional network = the same, with **Relation** made *symmetric* (a connection requires
  consent) — a constraint on shape 5, not a new shape.
- A forum = **Entry** where a comment and a top-level post are the *same* shape distinguished only
  by whether **Reference** is present, inside a **Context** (the board), with **Reactions** driving
  the ordering of the **Collection**.
- One decentralized protocol wraps **Entry** in an activity verb and gives it a globally unique URI;
  the other stores **Entry** as a typed record in a content-addressed repository. Both then express
  2–6 as further record types.

**Two findings from doing this.**

**(a) The vocabulary is small and it is stable across thirty years.** The same six shapes describe a
1990s newsgroup, a 2000s forum, and every deployed social system since. **A content model that gets
these six right does not need per-product extension** — the products differ at the interface layer,
which is exactly where they *should* differ.

**(b) Shape 6 is the one that gets collapsed, and collapsing it is the expensive mistake.**
*Context* (what a thing is part of) and *audience* (who may see it) are orthogonal, and most systems
conflate them because in a server-centric design they coincide — the board you post to is also the
board that controls visibility. **Here they must not be conflated**, because audience is already
modelled at the authority layer and context is content. Keeping them separate is what allows a
private reply in a public thread and a public reply in a closed group, both of which are ordinary
and neither of which is expressible if the two are one field.

### §4.1 The one shape the field is worst at, and where our position is strongest

**All three decentralized systems are public-broadcast-first**, and each added private or permissioned
data late and partially. Their unit is *a post to the world*; anything narrower is retrofitted onto an
architecture that assumed publication.

We start from the opposite side: fine-grained, attenuable, revocable authority that travels with the
artifact. **The honest form of the claim is narrow** — the *substrate* supports it and the content
model that would use it does not exist yet — but it is the one axis where the starting position is
genuinely better rather than merely different.

---

## §5 How a reference points — and why content addressing changes the answer

Shape 2 is the load-bearing one for §6, so it is worth being exact about what a reference contains.

**The two deployed answers:**

| System | A reply points by |
|---|---|
| The activity-vocabulary lineage | `inReplyTo` — **a URI**. Location only. |
| The repository lineage | a **strong reference**: a URI **plus a content hash** of the referenced record |
| Mail, and the newsgroup lineage before it | a **Message-ID**, plus a `References` header carrying **the whole ancestry chain** |

**The second one is the interesting one and it is a deliberate correction of the first.** A URI alone
says *where the thing is* and nothing about *what it is*, so a reply is only as trustworthy as the
server currently answering that URI — the parent can be edited or replaced underneath the reply.
Adding the content hash makes the reference say *"this exact content, which was at this location."*

**We get that property for free and one step further.** The tree is `path → hash` and the store is
content-addressed, so the natural reference is:

```
reference = { peer, path, hash }      ; who published it · where they put it · exactly what it was
```

The **hash** is the assertion and the **(peer, path)** is a hint about where to find it. A consumer
that already holds the bytes needs no fetch at all; one that does not can get them **from anybody**,
because the hash validates them regardless of source. **A reference is therefore verifiable without
contacting the party being referenced**, which is the property §6 is built on and which neither
location-only referencing nor a server-mediated thread can offer.

**The mail lineage supplies the other half.** Carrying the **full ancestry chain** rather than only
the parent is what lets a participant who holds *fragments* of a conversation reconstruct its
structure without a server and without the missing pieces. That is a thirty-year-old solution to
precisely the problem a decentralized thread has, and it is worth adopting deliberately: **a reply
carries `root` and `parent` at minimum**, and carrying the chain is what makes partial views
assemblable.

**Recorded honestly:** the newsgroup flood-fill lineage and the gossip-first social protocol are both
on this corpus's own list of unread prior art, and §6 is exactly the region where they would bite.
The observations here are drawn from mail and from the two protocols read directly; **the newsgroup
and gossip lineages should be read before §6 is turned into a specification.**

---

## §6 The republication network — participation as an act of publishing

This is the architecture the whole document is for, and it is not a variant of the deployed models.

### §6.1 The model

Every participant publishes a static, content-addressed, signed tree to whatever cheap always-on
storage they like. Then:

1. I read a discussion by fetching entries from the peers that published them.
2. I write a reply. It is an **Entry** in **my** tree, carrying a **Reference** to the parent —
   `{peer, path, hash}` — and naming its **Context**.
3. **Because I engaged with it, I also republish what I gathered**: the entries I fetched, unchanged
   and still carrying their original authors' signatures, into a thread view **in my own tree**.
4. My published tree now contains a *reassembled* view of that discussion — other people's entries,
   verifiably theirs, plus mine.
5. Someone who reaches **my** tree can read the whole thread, verify every entry against its author's
   key, and **discover every other participant** from the references.
6. They reply, and republish, and the fan-out repeats.

**The network is not a thing anyone maintains. It is the residue of people participating.**

### §6.2 Why this works here and does not work in the deployed systems

Four properties have to hold simultaneously, and this substrate is the reason they can.

1. **Republishing does not forge.** An entry carries its author's signature, so my copy of your
   comment is *your* comment. In a location-addressed system, "hosting a copy" and "asserting
   authorship" are indistinguishable to a reader; here they are cryptographically distinct.
2. **Republishing costs nothing.** Content addressing means my copy of your entry is the same bytes
   under the same hash — no duplication in the store, no divergence, and a reader who already holds
   it fetches nothing.
3. **No coordination.** I do not need permission, and you do not need to be online — for me to
   republish, or for a third party to read it later. **An offline author's contribution stays fully
   available and fully verifiable.**
4. **Cheap hosting is a first-class participant.** A static store is already a relay; there is no
   always-on daemon requirement to be part of the network.

**What this dissolves is the aggregator problem.** Every deployed decentralized system needs someone
to assemble a view — a relay, an index, a big instance — and that role reliably centralizes, because
clients hedge toward the most complete one and hedging makes it more complete. **Here assembly is a
by-product of participation**, so the aggregation is distributed across exactly the people who cared
enough to reply.

**And the sharpest form of the property:** an aggregator in this model **can omit but cannot
substitute**, because every entry verifies against its author's key. That reduces the trust question
from *"is this view honest?"* to *"is this view complete?"* — a strictly smaller question, and the
only one left.

### §6.3 What this costs, stated plainly

Not free, and the costs are structural rather than incidental.

- **Completeness is unknowable from one vantage.** N partial views and no way to know if any is
  whole. This is the same question §6.2 reduced the trust problem *to*, and it does not disappear —
  it is the honest residue. Comparing multiple republished views is the mitigation, and it is also
  the natural one, since a reader following several participants gets several views for free.
- **Retention.** Republishing means holding other people's bytes, and nothing currently reclaims
  content. A thread mirror grows monotonically. **Retention has to be designed with this, not after
  it** — this is the one place the model knowingly writes a debt.
- **Edits and deletions do not propagate**, by construction. A republished entry is a snapshot. This
  is *correct* for verifiability and *wrong* for the expectation every user brings from every other
  platform, and the gap has to be a stated product decision rather than a surprise.
- **Moderation has no natural home.** Nobody can remove anything from anyone else's tree. Blocking,
  filtering and reputation are all reader-side, which is coherent but is a real answer that has to be
  designed. **This corpus has no moderation axis at all**, and at the point participation becomes
  republication it needs one.
- **A reply is unsolicited republication of someone's words into a context they did not choose.**
  Worth naming as a design constraint now, because it is a social property of the architecture and
  not a feature that can be added later.

### §6.4 Discovery falls out, and that is the elegant part

One phrase for it is exact: *the network reveals itself.* Concretely — each republished thread
names every participant by key; each key resolves outward via §2.3; a reader arriving at any one
participant can walk to all the others; and a follow set grows by **encountering people in
discussions** rather than by consulting a directory.

**That is a discovery mechanism with no index, no crawler and no registry**, obtained from the
reference graph that threading requires anyway. It does not replace naming — you still need §1's
inbound routes to find the *first* peer — but it means the cold-start problem is bounded to the first
contact rather than being permanent. **Every deployed federated system that struggles with discovery
struggles because the graph never forms by itself; here the graph is the data.**

---

## §7 What is missing, ordered

Distinguishing what needs *design* from what needs *building*, because they are different asks.

| # | Item | Kind |
|---|---|---|
| 1 | **The content model** — the six shapes of §4, with Context and audience kept separate, and a **reference** shaped as `{peer, path, hash}` carrying root and parent | **design** — the keystone; everything else is about it |
| 2 | **The ordered feed index at a known key** | **design** — makes *"what is new"* a cheap read instead of a whole-tree walk |
| 3 | **The self-published peer record** of §2.3 — endpoints, claimed names, services, `seq`, `supersedes` | **design** — closes R1 and R2 in one artifact |
| 4 | **Bidirectional name verification** as a normative precondition | **design**, small — the rule is already written down by someone else |
| 5 | **`well-known-url` resolution** — one backend, three ecosystems, pattern-scoped so it cannot leak names | **build** — already ranked highest value-per-effort |
| 6 | **The follow loop** — the set, the cursor, cadence, backoff, and the read-only follower as a first-class role | **build**, designed |
| 7 | **The thread-mirror convention** — what a republished view claims about itself, and what completeness it does *not* claim | **design** — §6, and it is genuinely new |
| 8 | **Retention** | **design** — §6.3, and it is now load-bearing rather than a tidy-up |
| 9 | **Key-change continuity** — §3.1, including what a non-adopting follower may assume | **design** — the one genuinely hard problem here |
| 10 | **Moderation as an axis** — reader-side filtering, blocking, reputation | **not started, not surveyed** |

**Two things deliberately not on the list.** A distributed hash table: its unique job is lookup when
there is no always-on origin, so its value is *inversely proportional* to how well the origin tier
works, and the origin tier is the strongest component here. The honest trigger, written down so the
decision is not made by drift: **it becomes interesting the first time we need to find content whose
publisher has no origin and no name.** And a delegation hierarchy for naming: every post-DNS system
surveyed declined to rebuild it deliberately, and it is also DNS's central point of political
failure.

**The reading still owed before §6 becomes normative:** the newsgroup flood-fill lineage and the
gossip-first social protocol, both already on this corpus's unread list, and both squarely about a
network that assembles itself from partial views.
