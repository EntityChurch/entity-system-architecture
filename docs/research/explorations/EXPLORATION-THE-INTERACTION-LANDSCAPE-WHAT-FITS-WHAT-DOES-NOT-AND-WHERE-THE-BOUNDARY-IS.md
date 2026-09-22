# EXPLORATION — the interaction landscape: what fits these four shapes, what does not, and where the boundary actually runs

**Status:** Exploration (design record). Not a proposal, not normative.

**The question:** *the four content shapes cover blogs, feeds, galleries, forums and comments. What
about everything else people do online — games, worlds, collaborative documents, marketplaces,
live events? Where does this model stop being the right tool, and can we say so before someone finds
out the hard way?*

**Why this is worth a document rather than optimism.** A convention that claims too much is worse
than one that claims a bounded amount: the first produces implementations that half-work and blame
the substrate, the second produces a clear edge people design around. **The most useful output here
is the list of things this should NOT be used for**, and §5 is that list.

**Rests on:** `EXPLORATION-THE-L5-CONTENT-TAXONOMY-AND-THE-FLOOR-THAT-STOPS-THE-EXPLOSION` (the four
shapes and the test) · `REFERENCE-PRIOR-ART-FEDERATED-AND-P2P-PUBLISHING-SYSTEMS-BY-AXIS` (the deployed-systems library).

**Honest sourcing note, up front.** The social-protocol material in this corpus was read from primary
specifications. **The categories in this document mostly have no specification to read** — games,
marketplaces and collaborative editors are products, not standards — so this reasons from their
observable behaviour and from the general system properties they must have. **That is a weaker input
and the conclusions should be held more loosely than the record-model work.**

---

## §1 The one distinction that predicts every verdict below

Everything in §§2–5 falls out of a single question, and finding it is most of what this document
does.

> **Is the state something a person OWNS AND PUBLISHES, or something that is SIMULATED AND
> CONTENDED?**

**Owned and published:** one author is the authority, they write into their own namespace, the value
is whatever they last said it was, and two people disagreeing is not a conflict — it is two people
with different data. **This substrate is extremely good at that.**

**Simulated and contended:** many actors mutate one shared state, the order of operations changes the
outcome, and somebody must adjudicate. **This substrate has no answer for that and should not grow
one at L5.** Local-view authority — *each peer is the authority for its own namespace* — is precisely
a statement that there is no shared mutable state, which is the property that makes everything else
cheap.

**The boundary is not "social vs games."** It runs straight through the middle of a game: a player's
inventory is owned, the world's physics is contended. **Every verdict below is an application of that
one line.**

---

## §2 Games — and the split is cleaner than expected

The operator's instinct to look here is right, and the reason is that games are the richest source of
data models in consumer software. **The result is that roughly half of a game fits well and the other
half does not fit at all**, and the halves are separable.

### §2.1 What fits, and some of it fits better than the social case

| Game concept | Shape | Why it works |
|---|---|---|
| **Character sheet / profile** | entry | one author, one authority, published |
| **Inventory, owned items** | collection | **ownership is a signed claim in your own namespace** — this is the model's home ground |
| **Achievements, trophies** | entry (or collection) | append-only, author-signed, never contended |
| **Guild / clan roster** | collection | an authored, ordered, bounded set |
| **Replays, demos, match records** | entry + attachments | **immutable content-addressed artefacts — near-perfect fit** |
| **Mods, maps, assets, skins** | attachments / embeds | content addressing gives dedup, integrity and CDN caching for free |
| **Leaderboards** | mirror | *"here is what I gathered"* — and it inherits omit-but-never-substitute |
| **Game forums, guides, screenshots, clips** | the whole social stack | it is social content that happens to be about a game |
| **Trade / gifting** | the grant layer | an item transfer is an authorization move, which the substrate already models |

**Two of these deserve emphasis because they are stronger than the social case, not weaker.**

**Mods and assets.** A content-addressed store with dedup is *exactly* what an asset pipeline wants:
identical textures across a thousand mods are stored once, integrity is free, a CDN caches them
forever because the hash is the name, and *"do I already have this?"* is a hash comparison. **The
existing distribution platforms for this are re-implementing content addressing badly.**

**Replays.** An immutable signed artefact with a permanent identity that anyone can verify came from
the player who claims it, hosted on cheap static infrastructure, referenceable from a post. **There is
no part of that this substrate is not already built for.**

### §2.2 What does not fit, and should not be made to

| Game concept | Why it fails |
|---|---|
| **Authoritative world simulation** | many writers, one contended state, order-dependent outcomes. **The core property is that there is no shared mutable state** |
| **Real-time position, physics, combat** | tens of milliseconds. The reader loop is a cadence measured in minutes and the transport is a fetch |
| **Ephemeral state** | a player's position is not a durable signed publication and should never be one — publishing every tick is absurd on every axis |
| ~~**Anti-cheat**~~ | **corrected — see §2.2a. The score is an attestation, not a self-claim, and the substrate has the primitive** |
| **Matchmaking, queues, instancing** | a global scheduler over contended resources |
| **Scarcity, drops, first-come-first-served** | requires global uniqueness, which requires a gatekeeper — see §5.1 |

### §2.2a WHO signs is the semantic axis — and this corrects the row above

An earlier form of this document said **"a signature proves authorship, never truth"** and filed
anti-cheat as not fitting. **The first half is right and the conclusion was wrong, because it treated
all signatures as one kind of object.**

> **A signature's evidential weight comes from the relationship between the signer and the claim's
> subject.** A peer signing a claim **about itself** proves only that it said so. A peer signing a
> claim **about someone else** is a different object entirely — it is an assertion by a party with
> independent knowledge and something to lose, and its worth is exactly the reader's trust in that
> party.

A player signing *"I scored 4,000,000"* is worth nothing. **The game server signing *"this player
scored 4,000,000"* is worth whatever the server is worth to you** — which is how every leaderboard
that has ever worked, works.

**And this is not a gap to fill: it is a landed substrate primitive.** `EXTENSION-ATTESTATION`
defines `system/attestation` as ***"Peer A asserts that B has property X" — the edge type in the
system's signed graph***, where hash-addressed entities are nodes and signed claims are edges. It
ships supersession chains, revocation lookup, liveness checks with time-travelling validation, and
chain walking; **K-of-N lives beside it in `EXTENSION-QUORUM`**, so *"three referees agree"* is
expressible without inventing anything.

**So the corrected verdict.** *Attesting an outcome* fits and has a primitive built for it — a score,
a match result, a ban, a credential, a verified badge, a moderation label, a reputation. **What does
not fit is the simulation that produced the outcome**: the contended real-time state, which is §1's
line and is unchanged. **Adjudication is not out of scope; the authority just has to be somebody, and
saying who is the reader's choice rather than the protocol's.**

**Two consequences that reach past games.**

- **The social tier gets its positive half.** Moderation-by-omission is native and covers exclusion.
  **Endorsement, labels, credentials and reputation are attestations** — third-party signed claims,
  revocable and supersedable, which readers subscribe to exactly as they subscribe to block lists.
  That closes a side of the curation model that was previously only addressed negatively.
- **"Trust the signature" is never the right instruction to an implementer.** The right one is
  **"decide whose attestations you accept, then check the signature"** — and those are two steps that
  a UI collapses at its peril.

### §2.3 The shape that would actually serve games, and it already has a name

**An authoritative game server is the *live* rung of a dial this design already has** — static store ·
hosted origin · **live peer** — plus a capability others do not hold. It is not a different
architecture, and the one thing the static rungs structurally cannot do is **evaluate a grant at fetch
time**, which is exactly what an authority does.

**So the honest statement for a game builder is:** publish what players own, run a live peer for what
the world adjudicates, and **the two halves interoperate because they are the same substrate at
different rungs.** A player's inventory does not stop being theirs because a server arbitrates the
battle.

**MMOs and RPGs are the same answer at a larger scale**, with one addition: **persistent world state
is the contended half**, and a "player-hosted world" is simply a live peer with a lot of state and a
lot of grants. **A single-player or asynchronous game — turn-based, play-by-mail, roguelike ladders,
idle games — needs no live rung at all**, and that is a genuinely underserved category that this model
serves unusually well.

---

## §3 The rest of the landscape, briefly and honestly

| Category | Fit | The determining property |
|---|---|---|
| **Blogs, feeds, galleries, forums, comments** | ✅ **designed for it** | owned and published |
| **Newsletters, podcasts, RSS-shaped anything** | ✅ **excellent** | the pull precedent is 25 years old and our check is cheaper |
| **Wikis** | 🟡 **partial** | a personal wiki is a site; **a shared wiki is contended** and needs merge policy |
| **Collaborative documents** | 🟡 **needs a CRDT layer** | contended, but the contention is *mergeable* — this is the one contended case with a real answer |
| **Code forges** | 🟡 **partial** | repositories are content-addressed DAGs already; **issues and reviews are social**; CI is contended compute |
| **Marketplaces / classifieds** | 🟡 **listings yes, settlement no** | a listing is an entry; **payment and escrow need adjudication** |
| **Messaging** | ✅ with its own machinery | roster, delivery, membership — the conversation is a real second thing |
| **Live streaming / video calls** | ❌ | real-time media, no relation to published state |
| **Auctions, ticketing, booking** | ❌ | **contended scarce resources; needs a single arbiter by definition** |
| **Search** | ❌ | **nothing about this design produces it**, and it has no owner |
| **Ratings, "1.2M likes", trending** | ❌ **as commonly understood** | see §5.2 |

**The pattern in the middle column is the point:** the 🟡 rows are all *"the owned half fits and the
contended half needs something else"* — the same line as §1, arriving in six different products.

---

## §4 Two categories worth calling out because they look easy and are not

### §4.1 Collaborative editing is the one contended case with a real answer

**Everywhere else, contention needs an authority. Here it does not**, because text and structured
documents have well-developed conflict-free replicated types with two decades of literature behind
them. Two people editing one paragraph converges without a server.

**But it is a genuinely different data model from the four shapes** — a document under a CRDT is a
*log of operations that converge*, not an immutable published artefact — and **collapsing it into the
entry would be exactly the over-unification the taxonomy test forbids.** A conformant consumer must
behave completely differently: it merges operations rather than holding an immutable object.

**Recommendation: not a fifth content shape. A separate convention when someone needs it**, with the
entry as the *published snapshot* of a document whose live editing is a different mechanism. **The
seam is publication:** you collaborate in an operation log and you publish an entry.

### §4.2 Events and calendars are the case most likely to be underestimated

The operator named "event hosting" alongside social early on. **An event looks like an entry and is
not one**, and the reason is instructive: an event has **RSVPs**, which are *other people's writes
about your object*.

**Which the model already handles, in the way it handles everything else:** an RSVP is an entry in the
*attendee's* namespace with a `context` pointing at the event, and the host assembles a **mirror** of
the RSVPs they gathered. **The host cannot fabricate an attendee** (no key) and **cannot see attendees
who never told anyone** (no gatekeeper).

**So an event needs no new type — and the thing it cannot do is the thing event platforms sell:** a
guaranteed-complete guest list and a capacity limit. **Capacity is a contended scarce resource**
(§5.1), and *"27 of 30 spots left"* is not expressible without an arbiter. **A host who needs a hard
cap runs a live peer that issues admission grants**, which is §2.3's rung again.

---

## §5 What you get instead of a global answer — coverage, convergence, and the two that really are closed

> **Reframed, and the frame was wrong before.** An earlier version called this "the impossibility
> list." **Most of these are not impossible — they are unavailable *as instantaneous global facts* and
> available *as converging estimates with a stated coverage*.** That is a different claim, it is the
> honest one, and it changes what gets built: an impossibility gets designed around, **a convergent
> estimate gets measured and improved.**

### §5.0 The frame: your answer is bounded by what you have had contact with

**Global state is not absent; no observer has instantaneous access to it.** What any participant holds
is the result of what has propagated to them — their event cone — and everything below is a
consequence of that one fact.

**This is not a peculiarity of decentralization.** Every distributed system has a consistency horizon,
including the centralized ones; a platform's "global" count is that platform's aggregation over the
shards it happened to reach, presented without error bars. **The difference here is not that we have
less; it is that we cannot hide it, and therefore should report it.**

**So the shape of the answer, in every row below, is the same three-part one:**

1. **a value over your coverage** — honest, computable, and yours;
2. **a coverage estimate** — how much of the reachable set produced it;
3. **monotone convergence** — because entries are signed and content-addressed, **aggregating two
   observers' sets is union**, so combining views can only move the value *toward* the true one and
   never away. **You never have a wrong count. You have a low one.** That is §4.2's grow-only result
   applied to measurement instead of to threads.

**And it composes hierarchically, which is what makes it feasible at scale.** An aggregator that
reads a thousand origins publishes what it counted; a second-level aggregator reads aggregators.
**Each level states its coverage and can omit but never substitute**, so a reader picks a depth and
knows what they bought.

### §5.1 Global uniqueness and scarcity — **genuinely closed**

**No gatekeeper means no global uniqueness**, and unlike the rows below this one does not converge:
"who was first" is a total order over a set nobody can enumerate, and two observers can hold
irreconcilable answers **forever** rather than merging. First-come-first-served claims, unique
usernames as property, one-of-a-kind items and hard capacity limits all need an arbiter.

**What exists instead is issuer-scoped uniqueness** — a registry guarantees a name is unique *among
the names it issues*, which is real and useful **and is not the same thing.** Two registries may issue
the same label by design. **Where a hard guarantee is needed, an authority issues it and the reader
chooses to trust that authority** — §2.2a's shape again.

### §5.2 Global counts — **not closed; convergent, and this is the interesting one**

Total likes, follower counts, view counts, trending. **What is available is a count over your coverage
plus a coverage figure**, and combining observers converges upward.

**The honest presentation is therefore not "no count."** It is *"340 across the mirrors I read"*, with
coverage visible, and **a number that grows as you connect more** — which is a shape people already
accept from search result estimates and poll margins.

**Two design consequences worth carrying:**

- **Coverage is a measurable quantity and nobody has defined it here.** *"What fraction of the
  reachable network have I sampled?"* is a diffusion/propagation question with real literature behind
  it, and **it is currently an open research item rather than a solved one** — see §6.4.
- **Nobody can inflate a number nobody computes centrally**, which removes engagement metrics as a
  manipulation surface. Whether users read a coverage-qualified count as *honest* or as *broken*
  remains the biggest product risk in the design, and it is empirical.

### §5.3 Enforced deletion, expiry, view-once — **genuinely closed**

Covered in the feed proposal. **Anything of the form "make them stop having it" is unavailable**, it
does not converge, and a design implying otherwise lies to a user about something that matters.

### §5.4 Sybil resistance and abuse — **partially closed, and attestation is the lever**

**Identity is free**, so an adversary can mint unlimited keys and there is no rate limit on being a
stranger.

**Two mitigations, and the second was under-stated before §2.2a.** Nothing is pushed and every route
is opt-in, so **an unknown key with no path to you is not in your view at all** — genuinely stronger
than an open inbox, and it fails exactly where openness is the point, which is the forum. **And the
open case is where attestation does the work**: *"who vouches for this stranger"* is a third-party
signed claim, revocable, and readers choose whose vouching counts. **That is a web-of-trust shape with
a landed primitive underneath it**, and it is the most promising unexplored direction in the corpus.

### §5.5 Recovery from key loss — **open, and the one a normal user actually hits**

Unsolved; the natural fix — a successor signed by the old key — is exactly what a compromised key must
not be able to assert.

---

**The old framing, kept because the correction is the lesson:** calling all five "impossible" made
three of them look like walls when they are **gradients with a measurable position on them**. Only
§5.1 and §5.3 are walls. **A wall you route around; a gradient you climb, and you need an instrument
to know how far up you are.**

### §5.1 Global uniqueness and scarcity

**No gatekeeper means no global uniqueness.** First-come-first-served claims, unique usernames as
scarce property, one-of-a-kind items, capacity limits, "only 100 will ever exist" — **all require
somebody to decide who was first**, and there is nobody.

**What exists instead is issuer-scoped uniqueness**: a registry can guarantee a name is unique *among
the names it issues*, which is a real and useful guarantee **and is not the same thing.** Two
registries may issue the same label, and that is by design.

### §5.2 Global counts

**You cannot count what you cannot enumerate.** Total likes, view counts, follower counts, trending —
each presumes a vantage point that sees everything, and no such vantage exists.

**What is honest and available is a per-reader count**: *"of the mirrors I read, 340 people liked
this."* **That is a different number, it varies between readers, and it must be presented as what it
is.** A UI that renders it as a global total is lying, and the lie is the kind users will not forgive
once they notice two clients disagreeing.

**The interesting half:** this removes the substrate of engagement metrics as a manipulation surface.
**Nobody can inflate a number nobody computes centrally.** Whether users experience that as honest or
as broken is a genuine open question and probably the single biggest product risk in the whole design.

### §5.3 Enforced deletion, enforced expiry, enforced view-once

Covered in the taxonomy and in the feed proposal. **Anything of the form "make them stop having it"
is unavailable**, and a design that implies otherwise is lying to a user about something that matters
to them.

### §5.4 Sybil resistance and abuse at scale

**Identity is free**, which is a feature for a person and a problem at scale: an adversary can mint
unlimited keys. **There is no rate limit on being a stranger.**

**The mitigation the architecture does have is real but narrow:** because nothing is pushed to you and
you read only what you chose to follow or mirror, **an unknown key with no path to you is not in your
view at all.** Spam requires a route, and the routes are all opt-in. **This is genuinely stronger than
the open-inbox model**, and it is not a general answer — it fails exactly where openness is the point,
which is the forum.

**This is the most under-examined risk in the corpus and it deserves its own study.**

### §5.5 Recovery from key loss

Named for completeness: it is already on the open list, it is unsolved, and **it is the failure a
normal user is most likely to actually hit.** The natural fix — a successor signed by the old key — is
exactly what a compromised key must not be able to assert.

---

## §6 What this suggests for sequencing, without reordering anything

**No change is proposed to the build order.** Three observations that would otherwise have to be
rediscovered:

1. **The asynchronous-game category is nearly free and is a strong demo.** Turn-based play, ladders,
   replay sharing and mod distribution need **nothing beyond the four shapes plus what exists**, and
   they exercise attachments and collections harder than a text feed does. **A more convincing
   proof-of-life than another blog, and cheaper than it looks.**
2. **The live rung has three independent customers now** — an authoritative game server, a hard-capacity
   event, and a marketplace with settlement. **That is the promotion evidence a live-serving surface
   would need**, and it was previously a single app-tier request.
3. **§5's list belongs in front of adopters, not in a design record.** *"This cannot do global counts,
   scarcity, or enforced deletion"* is the kind of thing that builds trust when said early and
   destroys it when discovered late. **The natural home is the domain charter or a public FAQ**, and
   it should be written for a builder deciding whether to adopt.

---

### §6.4 Two research items §5's reframing creates, stated as hypotheses so the next pass can refute them

**Neither is answered here, and both became askable only once "impossible" became "convergent."**

**(a) Coverage as a measurable quantity.** If a count is *"N over my coverage"*, then coverage is the
number that matters and nothing in the corpus defines it. **Hypothesis: it is a propagation/diffusion
estimate** — how much of the reachable set a walk of depth *d* from your contacts is expected to
touch — and the epidemic-broadcast and anti-entropy literature has directly applicable results.
**What to establish: a definition, an estimator a client can actually compute, and what it costs.**

**(b) Distributed set summaries — probabilistically, where the trie does not reach.** The corpus
already holds **Merkle-set reconciliation and calls the hard half shipped**: two peers compare
canonical root hashes, equal means identical, unequal means descend, and unchanged subtrees cost
nothing. **That works when the two sides share a tree. It does nothing for *"which of these
ten thousand entry hashes do you already hold?"* across peers whose layouts have nothing in common**
— which is exactly the mirror, coverage and aggregation case §5.2 needs.

> **Hypothesis to test: a compact probabilistic set summary is the right instrument there, and it is
> a different tool from a DHT rather than a smaller one.** A DHT answers *"who has K?"* and pays for
> it with routing state that decays as peers churn — the failure this corpus has already measured and
> declined. **A set summary answers *"do you have any of these?"* between two parties already in
> contact, so it carries no routing table, no membership, and nothing to go stale.** It is an
> optimization on an exchange that is already happening.
>
> **What must be established before it is more than an intuition:** the false-positive semantics
> (a summary that says *maybe* is only useful if *maybe* is cheap to resolve), the size/accuracy
> curve at realistic set sizes, whether it composes with the canonical trie we already have or
> duplicates it, and **whether it is measurably better than exchanging sorted hash ranges**, which is
> simpler and has no false positives.

**Both belong in the next research pass** and neither should be designed before it.

---

## §7 What this document does not establish

1. **The game material is reasoned, not surveyed.** No game data model was read from a specification,
   because they are products and mostly do not publish one. **Treat §2 as a hypothesis a game
   developer could refute in a conversation.**
2. ~~**The CRDT recommendation is a pointer, not a design.** §4.1 says a separate convention; it does
   not say which family, and that choice is consequential.~~ **ANSWERED — state-based (CvRDT), in its
   delta-state form; not operation-based.** Derived from our delivery model rather than from
   preference: op-based requires every operation **reliably delivered**, and *"most operation-based
   CRDT designs require causal delivery"* — a precondition this substrate has declined to build at
   every layer, since replication is pull-based, partial and coverage-bounded. **State-based asks for
   exactly what we have**: two parties in contact exchanging state in any order, possibly twice, with
   merge idempotent, commutative and associative. **Delta-state is unusually cheap here** — each delta
   is an immutable content-addressed entity, so dedup is automatic and merge is a fold over whatever
   deltas you hold; and the literature's *"full state on first contact"* bootstrap **is** the published
   snapshot, which makes §4.1's *"the seam is publication"* the literature's own step rather than a
   convenience. `EXPLORATION-THE-DATA-MODEL-LADDER-…` §8.
3. **§5.2's product risk is unmeasured.** Whether people accept per-reader counts is an empirical
   question about users and nobody here can answer it from first principles.
4. **Nothing here was reviewed by an application seat.** It is arch reasoning about categories it does
   not implement, which is the position that has produced the corpus's most confident errors.
