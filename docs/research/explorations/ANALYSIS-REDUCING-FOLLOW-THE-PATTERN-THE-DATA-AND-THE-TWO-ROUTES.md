# ANALYSIS — reducing "follow": the pattern, the data, and the two routes that already exist

**Status:** Analysis (design record). Not a proposal, not normative. **Deliberately does not decide
where this lives** — §6 lays out the options and the evidence for each, and the decision wants peer
review before it is made.

**Question it answers:** *"follow" looks like it might be an application convention, and it might be
a generalizable mechanism. Which parts are which, and does the substrate already do it?*

**Method note.** The reduction in §3 is derived from the reference systems
(`REFERENCE-PRIOR-ART-FEDERATED-AND-P2P-PUBLISHING-SYSTEMS-BY-AXIS`) and then checked against the landed specs, in that order —
so the pattern is not simply a restatement of what we happen to have built.

---

## §0 The cut this starts from, and one correction it forces

The framing that opened this: **the follow *list* — what I follow, with what labels — is application
data. The follow *pattern* — "tell me what the latest is, then I decide what to do" — is
generalizable.** That cut holds up under examination and everything below is organized by it.

**But the analysis immediately corrects a claim made before it was done.** An earlier note asserted
that *nothing models following*. **That is wrong and the error is worth naming**, because it is the
same class as the two before it: a search for the *word* rather than the *mechanism*.

`EXTENSION-REVISION` §12 already specifies **two canonical follow patterns by name** — it literally
calls the first *"local follow chain"* — and carves out a deliberately light path:

> **Apply step is content-only.** `fetch-diff` + `tree:merge` mirrors *content* into the follower's
> namespace. It does **not** advance the follower's local version DAG. A follower that also wants
> its DAG to converge calls `revision:merge` — a deliberately separate concern. **Keeping the
> canonical follow recipe at `tree:merge` is what keeps content-mirroring and DAG-mirroring from
> re-entangling.**

So the intuition that *"revision sync is heavy"* is **true of `revision:merge`** — the version DAG,
the merge framework, conflict entities — and **already answered for the light case.** The
separation exists, it is deliberate, and it is documented in the spec.

**What is actually missing is much narrower than "a follow mechanism," and §4 is where it is.**

---

## §1 What "follow" reduces to, across the references

Stripping each system to the mechanism underneath its vocabulary:

| System | Source reference | Cursor | Change check | Delta fetch | Publisher holds follower state? |
|---|---|---|---|---|---|
| **RSS** | a URL | `ETag` / `Last-Modified` | conditional GET → `304` | re-fetch document | **no** |
| **git** | remote + ref | last-known commit | ref advertisement | pack of missing objects | **no** |
| **Nostr** | pubkey (+ relay hints) | timestamp / event id | query since T | events since T | **no** (relay holds events, not followers) |
| **ATProto** | DID | repo `rev` / firehose cursor | repo head or stream position | records since cursor | **no** |
| **Mastodon** | actor URI | — (push) | — | — | **YES — the follower collection** |

**Four of five put the state on the reader.** Mastodon is the outlier, and Mastodon is also the one
where delivery is `O(followers)` work for the publisher on every post (`REFERENCE-PRIOR-ART-FEDERATED-AND-P2P-PUBLISHING-SYSTEMS-BY-AXIS`
§7). **That is not a coincidence — it is the same fact stated twice.**

### The reduction

**Follow = five parts:**

1. a **durable reference** to a source
2. a **reader-held cursor** — what I last saw
3. a **cheap change check** — "has it moved past my cursor?", ideally O(1) and cacheable
4. a **delta fetch** — give me only what I lack
5. **no publisher-side per-follower state**

**Part 5 is the load-bearing one.** It is what makes the pattern work when readers are intermittent,
what makes it scale without fan-out, and what distinguishes it from subscription. Everything good
about the pattern follows from the publisher not knowing who is watching.

---

## §2 What the substrate already provides — verified, not assumed

| Part | Mechanism | State |
|---|---|---|
| 1. source reference | `peer_id` (is the pubkey); name → binding | **present** |
| 2. reader-held cursor | REVISION names `A.last_seen_head_for(B)` in a call pattern | **named, no declared home** — §4.1 |
| 3. cheap change check | `published-root` + `seq` monotonic floor | **present** (static route) |
| 4. delta fetch | **two routes, both real** — §3 | **present, twice** |
| 5. no publisher state | true of both routes | **present** |

**Nothing in the five-part pattern is absent from the substrate.** That is the headline, and it is
the opposite of what a search for the word "follow" suggests.

---

## §3 The two routes, and this is the part nobody has written down

**Delta fetch exists twice, by mechanisms that share no code and have different preconditions.**

### Route A — live publisher, operation dispatch

`peer_at(B).revision_fetch_diff(prefix, base = A.last_seen_head_for(B))`

`revision:fetch-diff` is executor-local and **cross-peer dispatch of it MUST NOT be rejected** —
the spec is explicit. B computes the diff against A's stated base and returns an envelope; A applies
it with `tree:merge`, content-only, DAG untouched.

**Precondition: B is live and answering dispatch.** The work happens on the publisher.

### Route B — static origin, structural walk

Fetch `published-root`; compare `root_hash` against the cursor. If unchanged, **done in one
request**. If changed, walk the trie — and because the tree is a HAMT with **structural sharing**,
every unchanged subtree is already in the local content store and is skipped. Only genuinely new
nodes are fetched.

**Precondition: an origin serving files. The publisher may be offline, asleep, or gone.** The work
happens on the reader.

### Why this matters more than either route individually

**Route B gets delta-fetch *for free from content addressing* — there is no diff operation, no
publisher cooperation, and no negotiation.** The dedup that already exists *is* the delta. That is
an unusually good property and no surveyed system has it in this form: RSS re-fetches whole
documents, git negotiates a pack, ATProto streams a cursor range.

**And the routes are not interchangeable:**

| | Route A (dispatch) | Route B (static walk) |
|---|---|---|
| Publisher must be live | **yes** | **no** |
| Work done by | publisher | reader |
| Request count when unchanged | 1 dispatch | **1 conditional GET** |
| Needs capability grant | **yes** (`revision:fetch-diff` on the prefix) | no — public origin |
| Freshness | immediate | bounded by publish cadence |

**Nothing in the corpus says which to use when.** That is the *"is it clear to a developer when or
why I would use one or the other?"* question, and the honest answer today is no.

---

## §4 What is actually missing — four narrow things

### §4.1 The cursor has no declared home

`last_seen_head_for(B)` appears **inside a call pattern** as though it were a given. It is
reader-held state — per-source, durable across restarts, and it is the thing that makes the whole
pattern work. **There is no declared site for it in any spec.**

This is a known failure shape in this corpus: *a rule or mechanism that names a value with no
declared place to put it is a mechanism two conformant peers cannot both implement the same way.*
Every consumer that has needed this has invented its own location.

### §4.2 There is no set — only one source at a time

Both routes describe following **a** peer. The pattern's whole value is following **N** of them:
which sources, in what order, with what per-source cursor and cadence. **The set is genuinely
absent**, and it is where the application data and the mechanism meet.

### §4.3 The loop is unspecified

Cadence, backoff, coalescing, what to do with a source that fails, and — the cost question below —
how not to make N requests forever against sources that never change.

### §4.4 No guidance on route selection

§3's table does not exist anywhere. A developer facing an intermittent publisher has no way to know
Route B is the one that survives it.

---

## §5 The cost problem: the once-a-year publisher

**This is the sharpest open question and it has no answer yet.**

Following N sources with a fixed poll interval costs `N × (1/interval)` requests **regardless of how
often anything changes.** For sources that publish yearly, essentially 100% of that is waste. At
1,000 follows and hourly polling that is ~24,000 requests/day to learn almost nothing.

**Route B's conditional GET makes each check cheap but not free** — it is still a round trip, and
the cost is borne by the reader and by every origin.

Four mitigations, in increasing order of how much new machinery they need:

1. **Adaptive backoff by observed change rate.** What every mature RSS reader does: a source that
   has not moved in a year gets checked weekly, not hourly. **Purely reader-side, needs no protocol,
   and should probably be part of §4.3's loop regardless.**
2. **A change-feed aggregator.** One request to a party that has already polled 1,000 sources,
   returning only *which heads moved*. Reduces N requests to 1.
3. **A push hub** (WebSub-shaped). Cheapest steady-state, but requires publisher cooperation and
   re-introduces publisher-side state — **which is part 5 of the pattern, so it should be an
   optional accelerator and never the base case.**
4. **Gossip between aggregators**, so no single aggregator is authoritative about what moved.

### The observation that makes (2) unusually attractive here

**A change-feed aggregator needs no trust.** It reports *what to check*; the publisher's own signed
root reports *the truth*. A lying or incomplete aggregator can waste the reader's time or omit a
source — **it cannot cause the reader to accept a wrong value**, because the reader verifies against
the signed root either way.

**That is a much weaker trust requirement than any aggregator in the reference survey**, and it
means the expensive part of the cost problem can be solved by an untrusted, easily-run service.
It also means the aggregator role and the follow loop are **the same design question** viewed from
two ends: an aggregator following 10,000 sources runs the identical mechanism a user following 10
does.

---

## §6 Where it lives — the options, not a decision

**No recommendation is made as a decree. The evidence for each:**

**(a) Application convention only.** Build the list, the loop and the cursor at L5 over what exists;
see what generalizes.
*For:* matches the working method — does the substrate support it, write the app, amend the
substrate only when it does not. Avoids inventing an abstraction from one consumer.
*Against:* §4.1's cursor is a substrate-shaped gap, and leaving it to applications is exactly how
four seats invent four incompatible keys.

**(b) Amendment to `EXTENSION-REVISION`.** Give the existing follow recipes a declared cursor site
and route guidance.
*For:* **the follow patterns are already there and already named.** REVISION's stated scope includes
*"sync protocol — version negotiation, delta transfer, integration between peers."* This is the
smallest possible change and it puts the answer where a reader already looks.
*Against:* REVISION carries the version-DAG apparatus, and a light path homed inside a heavy spec
may keep reading as heavy — which is the perception problem that started this.

**(c) A new extension in the network family.** A pull-side counterpart to SUBSCRIPTION.
*For:* the shape is real and recurring across every surveyed system; the family already separates
*addressed* (RELAY) from *unaddressed* (GOSSIP), and this is a third mode — **self-service, where
the reader initiates and the publisher is passive.** Notably RELAY Mode S is already described as
*"pull-shaped, receiver polls"* — for messages. This is the same shape for *state*.
*Against:* the over-generalization risk named at the outset — an abstraction nobody can name the
purpose of is worse than a duplicated pattern.

**(d) Split.** The mechanism amends REVISION (b); the social data model is an app convention (a).
*For:* it is the original cut, applied literally.
*Against:* needs (b) and (a) to be cleanly separable, which §4.2's set — part data, part mechanism —
may not be.

### The test that would settle it, and it may already be passing

This repo promotes a rule on **a second instance in a different shape**, never on speculation. **The
same ladder applies to an abstraction.** So: *is there a second, non-social consumer of the
identical mechanism?*

Candidates, and they are not hypothetical:

- **an aggregator** following many publishers (§5 — same mechanism, larger N)
- **a peer tracking a package or extension registry** for updates
- **a device pulling configuration** from a controller
- **a workbench following a published code root**

**The aggregator is the strongest, because it is already on the roadmap for independent reasons.**
If the aggregator and the social follower run the same loop with the same cursor, the pattern has
two consumers on day one, and that is the argument for (b) or (c) over (a).

**This is the question to take to the peer review**, and it is a better question than *"extension or
convention?"* because it is answerable with evidence.

---

## §7 What stays application data, in any of the options

Not generalizable, and should not be dragged into a mechanism:

- **display name, avatar, labels, notes** on a followed source
- **grouping** — lists, circles, categories
- **ordering and ranking** of the merged result
- **mute, block, boost, like** and every other social verb
- **the timeline** — how N sources' items interleave into one view
- **the social meaning of "follow"** — reciprocity, visibility, notification preferences

**The line the analysis suggests:** the mechanism owns *"has this source moved past my cursor, and
give me what I lack."* Everything about *why you care* is application data.

---

## §8 Open questions

1. **Is there a second non-social consumer of the identical loop?** §6's test. **Most decision-
   relevant question in this document.**
2. **Does the cursor belong to the follow entry, or is it separate state?** Bundling makes it one
   object; separating lets one source be followed by several consumers at different positions.
3. **Should Route A and Route B present as one interface with two implementations, or as two
   things a developer chooses between?** §3's table argues they are genuinely different — different
   preconditions, different capability requirements, different work distribution. **Hiding that
   behind one interface may be a false unification.**
4. **Does the change-feed aggregator (§5.2) need a protocol, or is it a well-known URL returning a
   list of `(source, head)` pairs?** The second is nearly free and may be sufficient.
5. **What does a follower do with a source whose head moved backwards?** The `seq` floor rejects it —
   but *reject* and *report* are different, and a silently-rejected rollback is indistinguishable
   from no change.
