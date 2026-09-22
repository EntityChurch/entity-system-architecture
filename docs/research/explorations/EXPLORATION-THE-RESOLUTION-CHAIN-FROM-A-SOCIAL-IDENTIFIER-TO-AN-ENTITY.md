# EXPLORATION — the chain from a social identifier to an entity, and the link that is missing

**Status:** Exploration (design record). Not a proposal, not normative.
**Question it answers:** *a person types a handle, a domain, a key, or clicks a link they found while
browsing. What happens between that and rendering someone's posts — and which parts of that chain do
we actually have?*

This is a **chain audit**. It takes the whole path end to end, rates each link by what exists rather
than what is intended, and maps each one against the deployed systems that already run it. The
companion documents are the naming landscape (which does link 1 in depth), the follow reduction
(link 4), and the prior-art source library (the axes). This one is the spine that connects them, and
it exists because **no document in the corpus traced the whole path at once.**

---

## §0 The first finding: the prior-art library has eight axes, and the one this question needs is not among them

The source library surveys the deployed field across **relay · inbox/outbox · DHT · federation ·
authority · user-facing identity · change detection · economics**. Eight axes, each a table across
the systems with primary sources.

**There is no axis for the data model.** No row anywhere for *what a post actually is* — not the
record schema, not the required fields, not the versioning rule, not how a consumer that has never
seen a type handles it. Verified by search across the library and the deployed-field survey: the
terms for record schemas in the two systems most relevant here return essentially nothing.

That is not a filing complaint. **It is the direct cause of §4** — the one empty link in the chain is
the one nobody surveyed, and the absence of the axis is why its emptiness never surfaced as a
finding. The ninth axis is supplied in §7.

---

## §1 The chain, link by link

Five links. They are at radically different maturity, and the gap is not where the conversation
usually puts it.

| # | Link | The question | State |
|---|---|---|---|
| **1** | identifier → peer | *who is this, and where* | 🟢 **designed, mostly built** — one real hole |
| **2** | peer → tree | *how do I get their data* | 🟢 **deployed** — live, walked end to end |
| **3** | tree → **the content model** | *what IS a post* | 🔴 **EMPTY** |
| **4** | the follow loop | *how do I learn there is something new* | 🟡 **designed, unbuilt** |
| **5** | the bridge | *how does this reach the systems people already use* | 🟡 **framed, not carried forward** |

**The load-bearing observation: link 3 is empty, and it sits between two links that are finished.**
We can resolve a name to a peer and pull their signed tree, and we have no vocabulary for what to put
in it. Everything downstream of that — the feed, the follow UI, the RSS emission, every bridge —
needs a content model to be about.

**And link 3 is the one where the cost of getting it wrong is highest**, because it is the link a
third party copies. A resolution mechanism is infrastructure a user never sees; a post schema is a
public format, and once anything is published against it, changing it is a migration rather than a
decision.

---

## §2 Link 1 — identifier → peer. The strong link, and the shape is deliberate

Four kinds of thing a person can type or click, and the design already has a home for each. The frame
is **Zooko's triangle** — human-meaningful, secure, decentralized, pick two — and the answer is not to
beat it but to expose all three corners as **different binding kinds under one resolver contract**:

| What the user has | Binding kind | Corner |
|---|---|---|
| a raw key / bare id | `self-certifying` | secure + decentralized, not memorable |
| a petname they assigned, or one they typed by hand | `local-name` | memorable + secure, not global |
| a global name (`alice.example.com`) | `peer-issued` | memorable + global, trust the issuer |
| a link they found while browsing | resolves to one of the above | — |

The user picks the corner per name, and a dispatch filter keeps the corners from leaking into each
other. **The privacy rule sits in that filter rather than in a backend** — a rule about which backends
may be *told a name at all*, applied to the configuration rather than to the query. This is stronger
than the DNS lineage manages: DNS took twenty years to get from "the resolver sees everything" to
encrypted transport, and encrypted transport still tells the resolver operator every name you look up.

**The trust model inverts DNS's structural mistake.** In DNS the thing that resolves the name is also
the thing you trust, and a zone operator can answer differently for different queriers with nothing in
the protocol noticing. Here a peer id **is** a public key, so a binding verifies against the pinned
root at the consumer and a registry that lies produces a signature that does not verify. There is no
unsigned mode. The cost, stated plainly, is that we inherit key management, which DNS does not have.

### §2.1 One unbuilt backend is the bridge head for three ecosystems

The web-native backends are all still paper, and they are **not equal in value**. Ranked:

1. **`well-known-url` — build this first.** It covers **DID:web, the Nostr NIP-05 convention, and
   ActivityPub/Mastodon WebFinger with one backend**: one HTTPS fetch, one JSON parse, on the order of
   200 lines per implementation. It is the single highest coverage-per-effort item in the whole naming
   stack, and §7 shows why: the two systems the question names both resolve this way.
2. **`dns-txt`** — does the *same job* by a worse route (a second resolver dependency, a trust
   qualification to specify, a 255-byte record format to fight). Its one genuine advantage is working
   for a domain owner who has DNS but no web host.
3. **`did-web`** — mostly subsumed by (1); the delta is the DID-document schema.
4. **`consensus-anchored`** — real demand in one community, requires an external chain dependency.

**One constraint travels with `well-known-url` and must not be lost:** it is *name-transmitting*, so
it has to be pattern-scoped in the shipped resolver configuration. Unscoped, every bare name a user
types leaks to whatever host that backend points at — which is precisely the failure the dispatch
filter exists to prevent.

### §2.2 The one real hole: only-an-id has no answer

Three cases; two work.

| You have | Path | State |
|---|---|---|
| a **name** | resolve → the binding carries glue transports | ✅ works, three-way green |
| a peer you have **contacted** | its cached transport profile | ✅ works |
| **only an id**, never contacted, no name | — | ❌ **nothing** |

**Every id-first system in the field solves this and we do not.** libp2p's DHT (peer-id → multiaddrs
is its primary content), ATProto's DID document endpoint, Nostr's NIP-65 relay list. Three designs,
one shape: **the endpoint set is published by the id-holder, signed by the id-holder, fetched by id.**

DNS is no guide here, and that is the interesting part: **DNS is name-first by construction**, so
nobody ever holds a bare address and asks what it is. We are **id-first**, so bare ids circulate as
first-class references — in rosters, in grants, in peer sets — and the case DNS never had is our
common one.

There is a further structural point underneath it. A transport profile carries its peer id *inside*
the entity and nothing signs it; its authority is **positional** — it is trustworthy because it sits
in that peer's own namespace. **Position is exactly what is lost when the entity travels.** Fetched by
hash from a registry or a cache, all a consumer holds is a content-addressed blob asserting an id, and
the hash proves the bytes, never the authorship. This is where libp2p was before it introduced signed
peer records, with the damage currently bounded by the registry's signature — which is better, and is
still the wrong party signing.

---

## §3 Link 2 — peer → tree. Deployed, and it is the strongest asset

This link is done and running: resolve a name at a domain, find the signed root, walk the
content-addressed tree, render. Live on a real estate with a real CDN in front of it.

Three properties matter for everything downstream:

- **A signed mutable pointer** carries the root hash, a sequence number (change detection *and*
  rollback defence) and a predecessor link (a history chain).
- **The tree is canonical and history-independent** — the same bindings produce the same root hash
  regardless of the order they were written in, and cross-implementation byte-identical output is
  already a conformance proof point.
- **A static store is already a relay.** A CDN-hosted tree is a fully participating publisher. There
  is no always-on daemon requirement to be followed.

That third property is the one with the most leverage and the least recognition. **Every system in
§7 that survived in the wild has an always-on tier that is not the user's device**, and the
characteristic pain of the ones that struggle — content dying when seeders leave, pin-or-it-vanishes,
relay dependence — is the liveness tax for not having one. We have that tier, cheaply, because
content-addressed signed state does not care what is serving it.

### §3.1 The measurement that constrains link 4

On a real published tree of ~1,000 bindings, the trie is 52 nodes at max depth 2, about 139 KB. Two
numbers came out of it and they point in opposite directions:

- **Verifying a known region is cheap** — on the order of the tree's *depth*, a handful of nodes and
  ~11 KB, even while the publisher rewrites most of the estate.
- **Discovering what is new under a prefix costs the whole tree.** The structure is keyed by hash
  bits, not by path locality, so three keys under one site live in three different leaf nodes and all
  51 must be read to find them.

**That is the single most design-relevant fact in this document**, and §5 is where it lands.

---

## §4 Link 3 — the content model. This is the hole

**There is no post. There is no feed. There is no profile. There is no timeline.** Searched across
the specs and guides for the whole family of names a social content model would use: **zero results.**

What exists at the application tier is three landed conventions, and their entire vocabulary is:

| Convention | Type tags |
|---|---|
| Embed | `app/embed/markdown` · `app/embed/applet` · `app/embed/image/png` · `app/embed-output/*` |
| Semantic content site | `app/site-manifest` · `app/site-page` |
| Share | `app/share/record` · `app/share/audience-entry` · `app/share/follow` |

Rich content blocks, a site with pages, and a sharing/audience model. **Nothing that means "a thing
someone posted."**

### §4.1 The corpus already knows this, says so, and names the trigger

This is not an oversight discovered from outside. The content-site convention's own discovery section
deliberately cut an ordered page collection and wrote down why, and the reasoning is good: a flat
exhaustive index is the "download the whole index" anti-pattern, discovery should be lazy and
one-level-at-a-time, and ordering is a renderer presentation choice because **the tree carries no
semantic order** — enumeration is in hash-bit order, random with respect to names.

It pinned exactly one cross-implementation rule — a determinism floor, sort by name segment,
byte-wise — and then said this:

> **Semantic feeds are explicitly OPEN — flagged, not solved.** *"Newest-first," prev/next, RSS — an
> order that declares its meaning … is the one place a real contract is genuinely owed, and **there
> is something here we haven't fully found.*** … v1 deliberately does not guess a field/format ahead
> of the use case … and defers the semantic layer to a named feed/index extension **authored when a
> concrete feed use case arrives.**

**The trigger condition is written into the spec, and the concrete use case has now arrived.** The
question this exploration was opened on — follow a set of people across independent hosts and read
what they published, newest first — *is* the feed use case the deferral was waiting for. The
deferral was correct when written and it has expired on its own stated terms.

### §4.2 Why this link is the keystone rather than one gap among five

Three reasons, in increasing order of importance.

1. **Everything downstream is about it.** The follow loop follows *something*; the aggregator
   aggregates *something*; the RSS bridge emits *something*. Link 4 and link 5 cannot be specified
   past their mechanics without link 3.
2. **The affordability of the whole model turns on it.** §3.1 measured that *"what did they post that
   I have not seen"* is the expensive enumeration — the whole tree — while a known key is nearly
   free. **A feed entity at a known key converts the primary social operation from the O(N) path to
   the cheap path.** So an ordered feed index is not a later convenience; it is the thing that makes
   the core operation affordable, and it should be treated as a prerequisite of the follow work
   rather than a bridge feature after it.
3. **It is the format outsiders copy, so it is where the publish-once-and-live-with-it constraint
   bites hardest.** Low external attention is an asset with an expiry date. Every design decision
   that lands before adoption is a decision; every one that lands after is a migration.

### §4.3 What we have that the surveyed systems had to build, and what we are missing that they started with

This is the honest asymmetry and it is worth stating both ways.

**Already ours, and hard:** content addressing · a canonical Merkle structure with structural sharing
· signed mutable pointers with rollback defence · identity as keys rather than server tenancy ·
fine-grained attenuable capabilities that travel with the artifact.

**Missing, and easy:** *a record with an author, a timestamp, a body, and a place in an order.*

The systems in §7 started at the second and spent years retrofitting the first. We did the reverse.
That is a good position to be in, and it is also why the gap is invisible from the inside: the part
we are missing is the part that feels too simple to be a design problem, and it is the only part a
user or a third-party implementer ever sees.

---

## §5 Link 4 — the follow loop, which is the RSS question

Reduced against the reference systems, following is **five parts**: a durable source reference · a
**reader-held cursor** · a cheap change check · a delta fetch · **no publisher-side per-follower
state**. All five are already present in the substrate.

**Two routes exist and they are not interchangeable:**

| | Route A | Route B |
|---|---|---|
| Mechanism | live dispatch to the publisher | walk the published static root |
| Work happens at | the publisher | the reader |
| Needs a capability grant | **yes** | no |
| Publisher may be offline | no | **yes** |

**A stranger has no grant, so public following is Route B only.** That single sentence settles a
surprising amount: the public social path is the static path, the publisher does not need to be
online, and there is no per-follower state anywhere — which is the property that makes the fan-out
economics work and that the server-centric systems in §7 do not have.

**Route B's delta is free from content addressing.** Unchanged subtrees are already held, so a diff
costs what changed rather than what exists. There is no diff operation to specify and nothing to
negotiate.

What is genuinely unwritten is the operational character — cadence, backoff, coalescing, what a
consumer does with a root it cannot fetch, and where the cursor lives. **This is where RSS's entire
character actually lives**, and §3.1 established that it is *not* cost-constrained, so the open
questions are operational rather than structural.

**Three honest holes in the loop**, each of which a reasonable implementer would ship something
plausible and wrong for:

- **A quiet publisher and a withholding origin are byte-identical at the consumer.** A follow UI
  cannot honestly say "no new posts" without a second origin to compare against.
- **A publisher restored from backup at sequence zero is permanently unfollowable** — the rollback
  defence working exactly as designed.
- **Attention cannot be scoped below the whole peer.** One root, one sequence number, and a structure
  keyed by hash bits, so there is no per-prefix hash to check without holding the bindings. §3.1's
  measurement is this hole quantified.

---

## §6 Link 5 — the bridge. Framed once, correctly, and never carried forward

The prior framing is sound and worth keeping verbatim in substance: **"bridge" is a neighbourhood,
not a domain.** There is no single bridge extension. Each specific bridge — HTTP, mail, a source-code
forge, a social protocol — is its own single-extension domain, with install-time **egress-only /
ingress-only / both** modes, because the two directions flip the threat model, the direction
capabilities flow, idempotency and the failure surface without justifying two extensions.

It also extracted **seven questions every bridge specification must answer**, from a live bridge built
against a real external system: authority assignment · capability translation in both directions ·
namespace convention · sync-state visibility · cache invalidation · **hash-identity translation** ·
idempotency on egress. That list is the scaffold for any bridge written from here, and the sixth item
is the hard one for this subject — an external system's identifiers are not content hashes, and
something has to own the mapping.

**Transferable hazards, also from that prior art:** external identifiers are adversarial input (the
escape-the-root class shows up as redirects, refs and symlinks alike); time-of-check/time-of-use under
external mutation; and a bridge introduces a **third sync state** beyond have/don't-have — *fetching /
stale / remote-unreachable* — which has to be surfaced rather than collapsed into an error.

**What does not exist:** any bridge to a social protocol, in any form, anywhere.

### §6.1 The cheapest bridge is the one pointing outward, and it is nearly free

**Emitting a feed from a published root** is close to free once §4 exists, and it makes the network
legible to every reader on the internet on day one. It is the highest-leverage external integration
available and it does not require anyone else to adopt anything.

**Ingress is a different order of work** and should be sized honestly rather than assumed symmetric.
Reading someone else's social protocol means running their resolution chain, mapping their record
schema onto ours, and answering hash-identity translation for objects that were never content
addressed. That is a real extension per protocol, not a shim — and §7 is the input to sizing it.

---

## §7 The ninth axis: what a post *is*, in the systems people actually use

Supplied here because the source library has no such axis and this chain cannot be reasoned about
without one. Read from the primary specifications.

### §7.1 The resolution chains, side by side

**Mastodon / ActivityPub.** ActivityPub itself does **not** specify how to turn a handle into an
actor — the fediverse uses WebFinger for that, and it is mandatory in practice for federation. A
handle `@user@domain` becomes a query to `https://domain/.well-known/webfinger?resource=acct:user@domain`,
which returns a JSON Resource Descriptor whose `links` entry with `rel: "self"` and type
`application/activity+json` points at the actor. The actor is then fetched with content negotiation.
**You may not guess the actor URL from the handle** — the handle domain and the actor host can
legitimately differ, which is exactly why the indirection exists.

**ATProto / Bluesky.** A handle resolves to a DID by **two methods in parallel**: a DNS TXT record at
the `_atproto` subdomain carrying `did=…`, or an HTTPS endpoint at `/.well-known/atproto-did`
returning the DID as plain text. On conflict the DNS result is preferred. The DID document then
carries the service endpoint for the user's data host.

**The rule to steal, and it is one we independently found from the other direction:**

> *"Handles should not be trusted or considered valid until the DID is also resolved and the current
> DID document is confirmed to link back to the handle."*

That is **bidirectional verification**, and its absence is a live defect class in our own registry
design — a resolver that checks a binding's signature without checking that the binding is *for the
name that was asked* lets a host repoint one pointer file and answer one name with the binding
legitimately issued for another. Two systems, the same hazard, and theirs is written into the
specification as a normative precondition. **Ours should be too.**

**What this means for link 1:** both chains terminate in an HTTPS `.well-known` fetch. That is one
backend. The ranking in §2.1 is not a guess — it is the observation that the two ecosystems named
here, plus the third one that uses the same convention, are one implementation away.

### §7.2 The record models

| | ActivityPub / ActivityStreams 2.0 | ATProto Lexicon |
|---|---|---|
| Unit | an **object** (`Note`, `Article`) wrapped in an **activity** (`Create`, `Like`, `Follow`) | a **record** in a typed collection in a repository |
| Naming | JSON-LD vocabulary terms | **NSIDs** — reverse-domain namespaced identifiers |
| Identity of an item | objects **MUST** have globally unique URIs unless intentionally transient | an AT-URI composed of the DID, the collection, and a record key |
| Where it lives | an actor's `outbox`; actors **MUST** have `inbox` and `outbox` collections | a content-addressed repository, Merkle-structured |
| Evolution | vocabulary extension via JSON-LD | **explicit compatibility rules** — see below |

**Lexicon's evolution rule is the one to read closely**, because it is the discipline our own §4 is
about to need:

> *"all old data must still be valid under the updated Lexicon, and new data must be valid under the
> old Lexicon."*

Concretely: **new fields must be optional; types cannot change; fields cannot be renamed; a breaking
change requires a new schema name.**

**We already have the wire half of that rule and not the vocabulary half.** Unknown fields are
MUST-ignore at the protocol level and the locked core is never renumbered — that is the same
principle, one layer down. What we do not have is any equivalent discipline over the application-tier
type vocabulary, and the consequence is not hypothetical: two independent implementations of the same
landed sharing convention currently emit **different type tags for the same object**, so a query from
either side returns a correct, complete, empty answer and neither can see the other. **A naming
divergence has no discovery path except someone reading both trees** — it never surfaces as a byte
mismatch, because the two sides never hold each other's data at all.

**So the lesson from Lexicon is not "copy the schema." It is that a published vocabulary needs a
stated compatibility contract and something that checks it**, and that is owed at the moment link 3
is written rather than after.

### §7.3 The shared blind spot, and it is the strongest thing in our position

**All three decentralized social systems are public-broadcast-first**, and each added private or
permissioned data late and partially. Their unit of sharing is *a post to the world*; anything
narrower is retrofitted onto an architecture that assumed publication.

We have the opposite starting point — fine-grained, attenuable, revocable authority that travels with
the artifact — and the audience question is already modelled at the sharing layer. **That is the
genuinely earned differentiator on this axis**, and it is worth stating carefully rather than loudly,
because the honest form of the claim is narrow: the *substrate* supports it, and the *social content
model that would use it does not exist yet*, which is §4.

Two guards on the surrounding claims, kept because it is easy to overreach here. **Content addressing
is not novel** — two of the surveyed systems have it, one of them with a Merkle-structured signed
repository that is the nearest prior art for a static store acting as a relay. **Keys-not-servers
identity is not novel** either. The defensible contribution is the *synthesis* plus posture mobility:
the same state can be a live peer, a passive CDN-hosted store, or dormant, and move between those
without changing what it is or who may read it.

---

## §8 What the chain implies for sequencing

Ordered by what unblocks the most, with the constraint from §4.2.3 applied: shapes a third party
would copy get more care and less speed than internal mechanics.

1. **The content model (link 3).** A post/entry record and an ordered feed index at a known key.
   **This is the keystone and it is the next design.** It carries three requirements the analysis
   above produced rather than assumed: the feed index exists because §3.1 measured that the primary
   operation is otherwise the expensive path; the vocabulary needs a **stated compatibility contract**
   per §7.2; and it needs **conformance artifacts**, because the one convention that shipped without
   them diverged between two implementations in six days.
2. **`well-known-url` (link 1).** One backend, three ecosystems, the bridge head for everything in
   §7.1. Scoped by pattern so it cannot leak names.
3. **The follow loop (link 4)** — the follow set as a record, the refresh loop's operational rules,
   and the read-only follower as a first-class role. That last one is the cheapest user in the whole
   system: publishes nothing, needs no origin, no name, no inbound reachability, and **nothing
   currently says so.** It is the on-ramp.
4. **Feed emission outward (link 5).** Nearly free once (1) exists; makes the network readable by
   every existing reader.
5. **Bidirectional name verification** — §7.1's rule, applied to our own resolver, as a normative
   precondition rather than an implementation habit.
6. **Only-an-id resolution (§2.2)** — signed, portable, sequence-bounded, published by the id-holder.
   Three consumers are already waiting on it.

**Deliberately not on this list:** a distributed hash table. Its unique job is lookup when there is no
always-on origin for the thing, so its value is *inversely proportional* to how well the origin tier
works — and the origin tier is the strongest component we have. The honest trigger, written down so
the decision is not made by drift: **a DHT becomes interesting the first time we need to find content
whose publisher has no origin and no name.** Nothing in this chain needs that.

Also not on the list: rebuilding a naming hierarchy for delegation. The direction is pinned to a
known model, and every post-DNS system surveyed declined to rebuild the hierarchy deliberately —
it is also DNS's central point of political failure. It should be deferred *knowingly*, and the thing
that breaks first at scale is not resolution but **configuration distribution**.

---

## §9 Open questions

**These gate design, not research.**

1. **Is the content model one convention or two?** A post/entry record and a feed index are different
   objects with different lifetimes — the record is immutable-ish and content addressed, the index is
   mutable and ordered. Splitting them is probably right and it is not obvious.
2. **What is the ordering contract?** The determinism floor exists (sort by name, byte-wise) and it is
   deliberately *not* semantic. A feed that declares its meaning — date-descending — is a real
   cross-implementation contract and the one place the corpus says a contract is genuinely owed.
3. **How much of an external system's model do we mirror on ingress?** The full record schema, or a
   reduced projection that keeps author, time, body and a link back to the original? The second is
   much cheaper and loses fidelity; the choice determines whether a bridge is a shim or an extension.
4. **Does an aggregator claim completeness, and how?** The property worth protecting is that **an
   aggregator can omit but never substitute**, because the hash verifies the bytes. No surveyed system
   has this cleanly, and it is the strongest differentiator available at the aggregation tier — but a
   consumer needs a way to detect omission for it to mean anything.
5. **Is the identity layer a prerequisite or an enhancement here?** The chain above works with bare
   keys throughout. Rotation, recovery and durable identity make it *better* and are not on the
   critical path for any link — but a follower pointing at an abandoned identity after a re-key is a
   real failure mode with no current answer, and it should be an explicit choice rather than a
   discovery.
