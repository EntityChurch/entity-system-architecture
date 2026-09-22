# EXPLORATION — Hyper-G: the system that shipped what Xanadu designed, and why that was not enough

**Status:** Exploration (design record). Not a proposal, not normative.

**Operator-requested:** *"this Hyper-G — never heard of it either, man. This is great, this is new
stuff. Let's do a deeper exploration."*

**Why it earns its own document.** Xanadu is the famous failure and it teaches one thing: *a
requirement needing a global authority does not get built.* **Hyper-G is the harder case, because
Hyper-G was built.** Graz University of Technology shipped a system with bidirectional links, links
on read-only documents, guaranteed referential integrity, automatic full-text indexing of everything,
and a scalable server-to-server protocol for propagating updates across thousands of sites — **in
1995, in production, commercially, as Hyperwave.** Every criticism it made of the Web was correct.
**It lost anyway, and the reason is a design lesson rather than a business one.**

**The one-sentence result:** **every Hyper-G guarantee is a property of a closed world**, and the
moment a link crosses out of Hyper-G every guarantee evaporates — *while the Web's links point
outside everything, always, by construction.* **Our position is the synthesis neither of them
reached: we have Nelson's applicative property — links live outside the content they point at —
without Hyper-G's closed world, because the link database is not one database. It is every referrer's
own tree, and each referrer is authoritative only for their own references.** §5.

**Sources opened:** Andrews, Kappe & Maurer, *The Hyper-G Network Information System*, J.UCS 1(4),
1995 (primary, full text) · Kappe, *A Scalable Architecture for Maintaining Referential Integrity in
Distributed Information Systems* (the p-flood paper) · the mprove and Jaschke retrospectives · the
Old Vintage Computing Research 2025 write-up. Claims marked *(our reading)* are inference.

---

## §1 What Hyper-G actually was

### §1.1 The data model — collections, and it is a DAG rather than a tree

> *"Documents may be grouped into aggregate **collections**, which may themselves belong to other
> collections. **Every document must belong to at least one collection.**"*

Two things in that sentence are load-bearing and both are unlike a filesystem:

- **It is a directed acyclic graph, not a hierarchy.** A document can be in many collections at once.
- **The "at least one" is an invariant, not a convention.** There are no orphans. That is what makes
  their navigation, their scoped search and their deletion propagation all work — **every object is
  reachable from the collection graph by construction.**

**Clusters** are the special case: *"a special kind of collection, a cluster, groups documents into
logical entities"* — a text plus its images plus its video, or the German and English versions of one
page, presented together as one thing.

**Collections span servers.** A collection is not a directory on a machine; it is a graph node that
may aggregate objects held elsewhere.

### §1.2 The link model — the thing they got right, and it is Nelson's

> *"Links are **not stored within documents** (as in W3) but in a separate link database (as pioneered
> by **Intermedia**)."*

A link targets *"either a destination anchor within another document, an entire document, or a
collection."*

**What that one decision buys, all of it delivered:**

| Property | Why it follows from links-outside-documents |
|---|---|
| **anchors on any media type** | you are not editing the document to add an anchor, so the document need not be text or even writable |
| **links onto read-only documents** | third-party annotation without the author's cooperation — **Nelson's "applicative" property, actually implemented** |
| **backward traversal** | the database has both endpoints, so *"what links to what"* is a query |
| **the Local Map** | Harmony's viewer could render incoming **and** outgoing links for any document on request |
| **referential integrity** | *"when moving or deleting a document, it is important to know which other documents contain links to it"* — and here you do |
| **restoration** | delete a document and links to it vanish from every HyperWave server; **restore it and they reappear**, because the links were never destroyed, only their target |

**That last row is the one nothing else in the field has ever had**, and it is only possible because
the link is an independent object with a lifetime of its own.

### §1.3 The server architecture — one server per session, proxying

> *"Hyper-G clients talk to a **single** Hyper-G server for the entire session. Should information
> from a remote server be needed, **the local server acts as a proxy**, i.e. it fetches the object and
> passes it on to the client."*

Behind that seat: an object-oriented database guaranteeing *"consistency and integrity of data"*, a
document cache, and a full-text index into which *"every document and collection is **automatically**
indexed upon insertion."*

**Search was free and scoped**, which nothing on the Web had for another five years — you could search
*within a collection*, because the collection graph told the index what "within" meant.

### §1.4 `p-flood` — referential integrity at scale, and it is the best-engineered part

The problem statement is exact and is still true of the Web today: *"the lack of support for
maintaining referential integrity in WWW and Gopher means that whenever a resource is moved or
removed, dangling references from other resources can occur."*

`p-flood` is *"a scalable, robust, prioritizable, **probabilistic** server–server protocol for
efficient distribution of update information to a large collection of servers"*, designed for
**thousands** of them, and — the detail that matters — *"in principle it could also have been added on
to existing WWW and Gopher servers."*

**It is the same family as Usenet's flood-fill, pointed at metadata instead of content.** Usenet
floods articles and dedups on `Message-ID`; p-flood floods *update notifications* and prioritizes
them probabilistically so the network is not saturated by churn. **It is a good algorithm and it was
never the problem.**

---

## §2 Why it lost, and the reasons are structural

The sympathetic account is *timing*: presented in 1995, when the Web was *"more exploding than
growing."* True, and insufficient — Hyper-G had a five-year head start on every feature it shipped.
**Four structural reasons, each independently sourced:**

### §2.1 The guarantees held only inside its own world

Referential integrity across sites required **every remote site to be running a HyperWave server.**
A link to a Web page was just a link; it dangled like anyone else's. **So the value of the guarantee
scaled with the fraction of the world running your software** — which is the worst possible shape for
an adoption curve, because the product is worth least exactly when you have fewest users.

**The Web's property is the opposite and it is why it won:** a URL's failure mode is *local*. It
breaks alone, and nothing else breaks with it.

### §2.2 The session model made the server a bottleneck by design

One server per session, proxying everything, versus Web clients opening connections to many servers
directly. **That is not an implementation detail; it is a statement about who is in the middle.**
Every remote fetch cost the local server's bandwidth, its cache, and its availability — and it made
the local admin a load-bearing party in every user's every action.

### §2.3 Interoperating meant conceding the standard

The surviving server had to *"bend backwards"*: parse HTML though it preferred HTF, speak HTTP though
it preferred HG-CSP. **Once you speak the other protocol well enough to be useful, you have taught
your users that they do not need the rest of you.** By 2000, *"neither HTF, nor Hyper-G clients, nor
the HG-CSP protocol played any role anymore"* — only the server survived, in a niche.

### §2.4 The monolith lost to components

By 2000 every differentiator — indexing, search, link checking, log analysis, dynamic content, load
balancing — *"could be achieved by combining a classic WWW server like Apache with modules and
third-party products"*, and Apache ran more than half the Web. **Hyper-G's advantage was that it did
all of these things together and none of them separately.** A retrospective puts it bluntly:
companies producing monolithic software instead of components were on the wrong side of history.

---

## §3 The transferable rule, and it is aimed at us

> ***p-flood was an elegant solution to a problem the Web won by declining to solve.***

**Before building a mechanism, ask two questions:**

1. **Does the competing design simply live without this property?** If yes, the mechanism is not a
   feature, it is a **tax you pay and they do not** — and you must be able to say what it buys that is
   worth more than the tax.
2. **Does the property survive at the edge of your own deployment?** A guarantee that holds only among
   peers running your software is a guarantee whose value is a function of your market share.

**We have live instances of both questions right now**, and this is why the rule is worth carrying:

- **`EXTENSION-RELAY`'s mode set.** L11 already records that arch nearly pruned the design space here.
  Question 1 applies to every mode: what does it buy that a plain origin fetch does not?
- **The registry / naming tier.** `EXPLORATION-NAMING-LANDSCAPE-…` establishes that we deliberately
  bind names to keys rather than locations, and that **every route terminates at the key and is then
  discardable.** *That is question 2 answered correctly by construction* — losing a route costs a
  label, not an identity — and it is worth recognizing that Hyper-G failed exactly the test our naming
  model passes.
- **Anything we might build to give references integrity.** §5.

---

## §4 The comparison table

| | Hyper-G (1995, shipped) | Us |
|---|---|---|
| where links live | **outside documents**, in a link database | **outside documents**, in the referrer's own tree |
| link direction | **bidirectional**, guaranteed | **one-way**, with backlinks existing only where someone published one |
| who owns the link space | **the server** — authoritative for its region | **each referrer**, authoritative only for their own references |
| annotate a read-only document | ✓ | ✓ — a reference in *your* tree needs nobody's permission |
| "what links to this?" | a database query, **complete** | walk the reference graph from peers you can reach, **incomplete and monotone** |
| delete a document | links to it vanish network-wide (p-flood) | nothing propagates; a stale reference is a `404` at fetch time |
| restore a document | **links reappear** | **the same, and for free** — the reference never went anywhere; it names a hash |
| orphans | **impossible** — every document is in ≥1 collection | **allowed and deliberate** — an unlisted entry reached by reference is valid (`FEED` §3.3) |
| aggregate structures | collections (DAG, cross-server) + clusters | `app/feed/collection` (ordered, cross-peer via references); **EMBED's `renditions` is a cluster** |
| full-text search | automatic, on insertion, scope-aware | **absent** — capstone §8.5: *"search does not fall out of anything"* |
| session shape | **one server, proxying** | reader contacts origins directly |
| integrity at the edge | **evaporates** outside Hyper-G | the hash is the integrity; it does not depend on who serves the bytes |

**Two rows in that table are worth stopping on.**

**The restore row.** Hyper-G's most impressive property — *delete the target and links vanish; restore
it and they come back* — **we get for free and more strongly.** A reference names `{peer, hash,
path?}`. If the bytes are gone, the fetch fails; if they come back **anywhere**, from any peer, the
reference resolves and **verifies**, because the hash is the identity. Hyper-G's version needed a
database, a lifetime model and a flooding protocol. **Ours needs nothing, because we never pointed at
a location in the first place.** *(our reading, and it is the strongest single argument in this
document for content addressing over link databases.)*

**The search row.** This is where Hyper-G is straightforwardly ahead and we should say so. *Every
document and collection automatically indexed upon insertion*, with **scope-aware** search because the
collection graph defines a scope. We have no search and no reason to expect one to emerge. **Hyper-G's
answer only works because of the closed world** — you can index everything when everything is in your
database — so it is not an answer we can copy. **But it is the clearest demonstration that the closed
world bought something real**, and any honest account of the trade has to include it.

---

## §5 The synthesis — what we have that neither of them had

**Nelson's insight** (§1.2 of the lineage document): links are **applicative** — *"applying from
outside to content which is already in place with stable addresses"*, permitting *"thousands of
overlapping links on the same body of content, created without coordination by many users around the
world."*

**Hyper-G implemented it and paid for it with a closed world**, because their answer to *"where do
links live if not in the document?"* was **a database**, and a database has an owner and a boundary.

**The Web declined it entirely** — links embedded, one-way, dangling — and got scale.

**Our answer is a third one, and as far as this read found, nobody else has it:** links live outside
the content they point at, **in the referrer's own tree**, which they sign and control. So:

- **the applicative property is preserved** — you can reference anything, the author cannot stop you,
  and no coordination is required;
- **there is no link database and no owner of the link space** — the "database" is the union of every
  publisher's own references, and it is exactly as complete as the set of publishers you can reach;
- **integrity is per-reference and does not depend on a server** — the hash is the assertion, the
  location is a hint;
- **and the guarantee does not evaporate at a boundary**, because there is no boundary: a reference
  into a tree hosted by someone who has never heard of this protocol still verifies if you can get the
  bytes from anywhere at all.

**What we give up is completeness of the backlink set**, which is Hyper-G's headline property.
**And we already accepted that trade for a different reason** — capstone §4.1's *"completeness is
unattainable and will remain so"* — so it costs nothing new. **The backlink question is the forum
question wearing different clothes**, with the same answer: *your view is incomplete, incompleteness
is monotonically repairable, and every new contact can only add.*

> **Stated as a design principle worth keeping: Hyper-G made the link a first-class object owned by a
> server. We make the link a first-class object owned by the party who made it. That is the same
> insight with the ownership moved, and moving the ownership is what removes the boundary.**

---

## §6 What to take, concretely

1. **The Local Map is a UI we can approximate and should name.** *Incoming and outgoing links for this
   document, on request.* Outgoing is free (they are in the entry). Incoming is **the mirror set**:
   the union of published mirrors and replies you can reach. **It is Hyper-G's best user-facing
   feature, we can build 80% of it, and nothing in our corpus describes it.** — §7.1
2. **Clusters ≈ renditions, and the equivalence is worth writing down.** Hyper-G's cluster (one
   logical entity presented as a group — multilingual variants, multimedia aggregate) is what
   `APP-CONVENTION-EMBED`'s `renditions` list already is, and their framing (*"a logical entity"*
   rather than *"a list of alternatives"*) is the better one for a spec.
3. **"Every document belongs to at least one collection" is an invariant we deliberately do not
   have** — and we should be able to say why. **Theirs makes navigation total and deletion
   propagable; ours makes an unlisted-but-referenced entry valid**, which `FEED` §3.3 rule 3 states as
   a MUST. **Both are coherent; only one of them is written down as a choice.** — §7.2
4. **The two questions in §3** as a standing habit for any mechanism proposal.

## §7 What this opens

1. **A backlink / "who referenced this" view has no home in the corpus.** The taxonomy floor probably
   says it needs no new type (it is a mirror pointed at a document rather than a thread), but **nobody
   has asked the question**, and Hyper-G is evidence that it is the single feature users noticed.
2. **Write down why we permit orphans**, since Hyper-G's opposite invariant is defensible and ours is
   currently an absence rather than a stated choice.
3. **Search remains genuinely open** (capstone §8.5) and Hyper-G is the clearest evidence that the
   closed world is what made it easy. **No action** — recorded so the next person who asks *"why is
   there no search?"* gets the reason rather than a shrug.

## §8 Sources opened

[Andrews, Kappe & Maurer — *The Hyper-G Network Information System*, J.UCS 1(4), 1995](https://www.jucs.org/jucs_1_4/the_hyper_g_network/Andrews_K.html) **(primary)** ·
[Kappe — *A Scalable Architecture for Maintaining Referential Integrity in Distributed Information Systems* (p-flood)](https://link.springer.com/chapter/10.1007/978-3-642-80350-5_8) ·
[Kappe, Pani & Schnabel — *The Architecture of a Massively Distributed Hypermedia System*, Internet Research 3(1), 1993](https://www.researchgate.net/publication/242934482_The_Architecture_of_a_Massively_Distributed_Hypermedia_System) ·
[Vision and Reality of Hypertext and GUIs — Hyper-G/HyperWave](https://mprove.de/visionreality/text/2.1.15_hyperg.html) ·
[*Why Hyperwave?*](https://jaschke.net/hyperwave.html) ·
[prior-art-dept.: the hierarchical hypermedia world of Hyper-G (2025)](http://oldvcr.blogspot.com/2025/05/prior-art-dept-hierarchical-hypermedia.html)

**Unread, named:** the **HG-CSP** protocol specification and the **HTF** technical report (Kappe,
IIG, 1995) — the wire and markup formats, neither of which surfaced in a searchable form; **Maurer,
*Hyperwave: The Next Generation Web Solution* (Addison-Wesley, 1996)**, which is the book-length
statement; and **Intermedia**, named in the primary source as the origin of the separate-link-database
idea and not examined here. **Microcosm's Distributed Link Service** is the third member of that
family and is also unread.
