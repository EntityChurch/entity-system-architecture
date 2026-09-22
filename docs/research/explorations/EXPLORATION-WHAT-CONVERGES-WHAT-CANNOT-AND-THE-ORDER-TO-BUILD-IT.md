# EXPLORATION — what converges, what cannot, and the order to build it

**Status:** Exploration (design record). Not a proposal, not normative.

**This is the capstone of the social-substrate work — read this one if you read one.** The preceding
four explorations found the chain, the naming topology, the complete stack map, and the convergence
thesis. This one does three things none of them did: it states **what the system is, in one page**;
it works out the **two genuinely hard problems** — forum aggregation without a gatekeeper, and
divergent interpretation of a shared data model; and it puts the work in **an order where the first
step is a thing a person can actually use.**

**The organizing distinction, stated up front because it resolves most of the confusion:** *publishing
a feed and following other feeds* is easy, designed, and has almost no open questions. *A forum* is
hard for one specific reason. *Encrypted groups* are hardest. **These are three different problems and
treating them as one project is what makes the whole thing look intractable.**

---

## §1 The system, in one page

A person holds a **key**. That key is their identity — not an account, not a name, not a record on
somebody's server. They publish **entities** into a tree they control: content-addressed, signed,
laid out however they like. They put that tree wherever bytes are cheap — a static host, a CDN
bucket, a service that does it for them, their own machine.

To be reachable, they publish a **signed root**: one pointer saying *this is my current state, at this
sequence number*. Anyone who wants to know if there is something new fetches that pointer and compares
it. Unchanged means done — a handful of interior nodes, on the order of ten kilobytes, measured. **The
publisher does no work per follower. A million followers and one cost the same.**

To be **found**, there are many routes and no canonical one: a domain with a DNS record or a
well-known path, a name issued by a registry, a petname you assigned, a key typed by hand, a QR code,
a rendezvous label, or a reference inside somebody else's data. **All of them terminate at the key,
and once they do the route is discardable.** Losing a domain costs a label, not an identity.

To **read** someone, you walk their tree, verify against their key, and hold what you want. To
**reply**, you publish an entry in *your own* tree carrying a reference to theirs — `(peer, path,
hash)`, where the hash is the assertion and the location is a hint. To make a conversation durable,
you **republish what you gathered**: their entries, unmodified, still carrying their signatures,
assembled into a view in your tree. **You cannot forge them — you do not hold their key — and you
cannot alter a byte without the signature failing. All you can do is carry what they already chose to
publish.**

That last property is the one everything else rests on. It means **assembly is a by-product of
participation** rather than a role somebody has to occupy, it means **an aggregator can omit but never
substitute**, and it means **blocking is native** — omission is the only operation the assembly layer
permits, so choosing what to republish *is* moderation, and choosing whose views to read *is* choosing
whose moderation you accept.

**What you stop needing:** an account, a server, a name someone else controls, their ranking, their
delivery infrastructure, their moderation applied to you, their advertising in your exchanges.

**What that costs:** no global takedown, ever. No platform to appeal to. No guarantee you have seen
everything. And no network effects — a better architecture does not move people; it lowers the cost of
leaving.

---

## §2 Four use cases, three difficulty tiers

The single most useful thing in this document is probably this table, because it stops one hard
problem from making three easy ones look hard.

| | Use case | Shape | Difficulty | State |
|---|---|---|---|---|
| **1** | **Publish a feed, follow feeds** | one author per space; readers pull | **easy** — and *nothing here is in doubt* | designed; vocabulary missing |
| **2** | **Reply and thread** | references across spaces | **moderate** — designed, needs the mirror | designed |
| **3** | **A forum** | many authors, one logical space | **hard** — §4 | theory, no design |
| **4** | **Private / encrypted groups** | many authors, membership, secrecy | **hardest** | out of scope so far |

**Why (1) is genuinely easy and should be said plainly.** There is no shared space. I publish into
mine, you publish into yours, I read yours because I chose to. There is nothing to agree about beyond
the entity format, nothing to reconcile, no contention, no consensus, no completeness question — *I
know exactly where to look because you told me where you are.* **Two deployed ecosystems already prove
the pattern works socially.** This is the realizable chunk and it should ship first for that reason,
not as a stepping stone.

**Why (2) is only moderately harder.** A reply is an ordinary entry that happens to carry a reference.
The new thing is the mirror, and the mirror is four rules. **Nothing about (2) requires solving (3).**

**Why (3) is the hard one, precisely.** In (1) and (2) I always know *where to look* — the author told
me. In a forum I want *everyone who commented*, and **there is no place that knows who they are.** No
gatekeeper means no index means no completeness. That is not an implementation gap; it is the shape of
the thing.

---

## §3 Three kinds of convergence, and only one of them must hold

Most of the worry about *"will everyone really agree?"* dissolves once these are separated, because
they have different answers and only the first is a correctness requirement.

| | Question | Must converge? | How |
|---|---|---|---|
| **Set** | do we hold the same entries? | **yes** | union of signed content-addressed entries — §4 |
| **Order** | do we sort them the same? | **where cheap** | chronological is free; ranking is §5 |
| **Selection** | do we show the same subset? | **NO — divergence is the feature** | filtering, muting, blocking are per-reader by design |

**Selection is supposed to diverge.** The whole exclusion model depends on it: my view omits what I
chose to omit, and a reader of my mirrors inherits that visibly. **A system where selection converged
would be one with a central moderator**, which is the thing being replaced.

**So the anxiety about interpretation divergence is really about the middle row**, and §5 argues it is
smaller than it looks.

---

## §4 The forum problem — and the convergence result

### §4.1 The honest statement

**Completeness is unattainable and will remain so.** With no central gatekeeper there is no place that
knows every participant, so *"have I seen every comment?"* has no answer. Somewhere there is a reply
by someone you have never encountered, on a host you have never contacted.

This is not a defect to engineer away. It is what *no gatekeeper* means.

### §4.2 What is true instead, and it is stronger than it first sounds

**Partial views merge losslessly, and the merge is verifiable.**

A thread is a **set of immutable, content-addressed, signed entries**. Merging two views is **set
union**. Union is idempotent, commutative and associative, so:

- merging in any order gives the same result,
- merging twice changes nothing,
- **two people with completely disjoint views of a thread who meet produce the union, with no
  conflict, no reconciliation, and no coordination**,
- and nobody can inject a fake element, **because every element carries its author's signature** —
  which is a property ordinary convergent data structures do not have, since they assume cooperative
  replicas.

**So a thread is a grow-only set, and the network of threads is a set of those** — which is the
"CRDT of CRDTs" intuition, and it is exact rather than metaphorical. Deletion is the operation such a
structure cannot express, and this design already does not have it: a revision is a *new* entry
referencing the old, and *which one you display* is an interpretation question, not a set question.

### §4.3 The consequence worth building on

**You never have a wrong view. You have an incomplete one, and incompleteness is monotonically
repairable — every new contact can only add.**

Put beside the alternative, that trade is good. A centralized forum can show you a view that is
**wrong**: ranked against you, quietly filtered, with comments removed and no trace. **This can only
show you one that is short**, and shortness is visible (compare two mirrors) and fixable (meet more
people).

**And this is why the network converges in practice even though it cannot in principle.** Every reply
republishes what its author gathered. So the more a thread is discussed, the more copies of it exist,
the more paths lead to any given participant, and **the more likely any newcomer's view is nearly
complete.** Activity produces redundancy; redundancy produces completeness. Two subnetworks that
discussed the same thing independently and later meet **merge into one thread with no work at all** —
their entries were content-addressed the whole time.

### §4.4 What is actually missing — and it is navigation, not aggregation

**Aggregation is not the unsolved part; it is the part §4.2 solves.** What is missing is **navigation
under a bound**:

- Nobody can replicate the whole network, so traversal must be **lazy** — one level at a time, on
  demand, budgeted. (The content-site convention already reached this exact conclusion for pages and
  its reasoning transfers directly.)
- **What do I hold and why?** Republishing is a storage decision, and *"is this worth keeping"* has no
  answer in the corpus. This is the retention gap, and the mirror model is what made it load-bearing.
- **How do I discover a topic I have no contact for?** Following people is a solved route. Following
  *ideas* — a content-addressed topic identifier — is a different one, and it is the forum's actual
  entry point.

**That last item is the missing primitive, and it is small.** A topic that is a content hash gives a
stable global name that anyone can compute independently and nobody owns — *hash the subject, and
everyone who cares about the same subject computes the same address without coordinating.* What it
does not give is a place to look, which is exactly the same shape as the key-to-endpoint problem one
layer down and probably wants the same answer: **participants publish their topic participation, and
discovery walks the reference graph from anyone you already know.**

---

## §5 The interpretation problem — the last 10%

The concern is right: **the data model says what exists; it does not say what you see.** Two clients
holding identical entries can present different views, and a version change can shift a ranking under
everyone. That is real. It is also more tractable than it looks, for three reasons.

**First, most of the surface is trivially convergent.** Chronological order needs no agreement beyond
reading the same field. Threading order follows the reference graph, which is in the data. **The
overwhelming majority of what a reader actually does — read a feed newest-first, read a thread in
reply order — converges for free.**

**Second, ranking is an opinion, and this architecture already knows how to handle opinions: publish
them.** The instinct is to standardize the algorithm so everyone computes the same score. That is the
wrong move and it is the centralizing one — it makes the algorithm a thing someone must own. **The
right move is to make a ranking a publishable, subscribable artifact**, exactly as block lists are.
Then a ranking is something you *choose*, several can coexist, disagreement is expressible, and nobody
has to win. **The nearest live prior art is a feed-generator model in one deployed system and it
works.**

**Third — and this is where transferable compute earns its keep — a *published* ranking can be an
auditable one.** If the ranking function is itself a content-addressed program, then: it is identified
by hash, so *"we shipped a new version"* is a **different hash** rather than a silent behaviour change;
anyone can re-run it over the same inputs and check the output; and two people can verify they are
seeing the same view because they ran the same program over the same set. **Version skew stops being a
coordination problem and becomes a content-addressing problem, which is the one thing this substrate is
best at.**

**So the honest ordering:** ship (1) and (2) with chronological and reply order, which need no
convergence machinery at all. Treat ranking as publishable data when it arrives. Reach for transferable
compute **only where auditability is actually wanted** — which is the small, high-value case, not the
common one. **The 90% does not need the last 10%, and the last 10% has a design when it is needed.**

**The residual risk, stated because it is real:** where clients diverge *without* publishing why, users
experience it as the system being broken rather than as two clients disagreeing. **The mitigation is
that a view declares what produced it** — which ranking, which mirror set, which exclusions. A view
that names its own provenance is debuggable; one that does not is indistinguishable from a bug.

---

## §6 Scale — what following a thousand people costs

**The precedent is good and it is worth leaning on.** Feed-pull scaled to millions of publishers for
25 years, and the failures were **client-side** — readers that refetched when nothing had changed —
never the model. Nobody's publisher fell over from being subscribed to.

**Our version of the check is cheaper than that precedent's**, which is the encouraging half.
Comparing a signed root is a small fetch of interior nodes — measured at roughly ten kilobytes and a
handful of nodes even while the publisher rewrote most of their estate — and **unchanged means no
further work at all.** The older technology refetched a whole document to discover nothing had changed.

**The linear term is unchanged and is the real constraint:** N follows means N origins to check. At a
thousand follows that is ordinary reader behaviour and well within what the precedent demonstrated. At
a hundred thousand it is not a client's job.

**And the answer at that scale is a tier this design already has for another reason.** Someone who
already checked those origins can publish a combined view; you check *their* root with one fetch and
walk what changed. **That is the mirror, pointed at fan-in instead of at threads** — and it inherits
the same guarantee: it can omit but never substitute, so using one costs you completeness, never
integrity. **You can also check several, or check the origins directly for the ones you care most
about.**

**The client discipline that the precedent says decides this:** conditional checks, honest backoff,
coalescing, and never refetching what a sequence number says has not moved. **The failure mode is a bad
client, and it has been a bad client for 25 years.** That is a specification item for the refresh loop
and it is cheap.

---

## §7 The order to build it

Ordered so that **the first thing that ships is a thing a person can use**, because *"what is the point
of this?"* is answered by a working artifact and never by an explanation.

### Stage 1 — publish and follow *(the demo, and the answer to "what's the point")*

The entry, the reference, the paged index, the follow set, the refresh loop. **A person publishes to
cheap static hosting under a name they chose, and other people read them in a normal-looking client.**
No server, no account, no company.

**Why this is the right first artifact:** it has no open design questions, it is the pattern two
deployed ecosystems already validated socially, and **it is demonstrable in a sentence** — *"that
website is my feed; here is my key; follow me and nothing between us is anyone's business."*

### Stage 2 — reply, thread, mirror

References across publishers, and republication. **This is where the architecture stops looking like
a blog with extra steps** — the first moment where participation visibly builds shared structure.
Gates the aggregator work, and supersedes most of it.

### Stage 3 — the forum

Topic-as-content-hash, lazy navigation, retention policy. **Do not start this before Stage 2 is
running**, because §4.3's convergence-through-activity only becomes observable once there is activity.

### Stage 4 — private and encrypted

Confidential entries ride the same rails (nothing the format reads lives inside the body). Membership,
key distribution and rotation are the genuinely hard part and are properly last.

### Running alongside, not blocking

Naming routes as they are wanted · the hosted publishing tier (the on-ramp for people who will not
self-host) · transferable compute, where auditability is wanted · retention, which Stage 2 makes
load-bearing.

---

## §8 What remains genuinely unknown

Kept short and honest; these are not rhetorical.

1. **Does the social layer converge in practice?** Byte-level convergence is provable and being
   proved. **Whether independent people adopt one vocabulary is a social question and it is the open
   one.** The lever we have is not enforcement; it is that divergence costs interoperability and
   convergence is free.
2. **Multi-publisher scale is unmeasured.** The largest real run is two devices. Every number in §6
   is either a single-publisher measurement or an inference from a precedent.
3. **Retention has no policy.** Republication makes storage grow monotonically and nothing reclaims
   it.
4. **Key-change continuity is unsolved.** The natural fix — a successor signed by the old key — is
   exactly what a compromised key must not be able to assert.
5. **Search does not fall out of anything.** Unlike moderation, which turned out to be native, there
   is no reason to expect search to emerge from this design. It is a separate problem with no owner.
6. **Nobody has run the forum case even in theory at size.** §4 is an argument, not a result.

**The through-line, and it is the fair summary of the whole arc:** the parts that are hard to build
are largely built, and the parts that are easy to build are the ones missing. What remains genuinely
uncertain is not whether the mechanism works — it is whether people agree on a vocabulary, which no
architecture can force and this one at least makes cheap.
