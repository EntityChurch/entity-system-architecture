# EXPLORATION — the operating models: what actually runs in the wild, why the pure-P2P ones decayed, and the tier we already shipped without naming

**Status:** Exploration (design record). Not a proposal, not normative. **Nothing here rules anything.**
**Question it answers:** *how do Mastodon, ATProto, Nostr, RSS and SMTP actually operate — not what
their primitives are — and what does that say about what this design still owes?*

**Companion:** `REFERENCE-PRIOR-ART-FEDERATED-AND-P2P-PUBLISHING-SYSTEMS-BY-AXIS` — the same systems decomposed axis by axis (relay ·
inbox/outbox · DHT · federation · authority · user-facing identity), with sources, for use while
designing.

---

## §0 Why this document exists, and the three framings it corrects

The design record surveys these systems **as primitive sets** — naming
(`EXPLORATION-NAMING-LANDSCAPE-AND-THE-DNS-MAPPING`), relay
(`EXPLORATION-RELAY-LANDSCAPE-AND-PRIOR-ART`), key rotation
(`EXPLORATION-ROTATION-AUTHORITY-AND-KEY-LIFECYCLE`). It had **never surveyed them as operated
systems.** Measured across the whole research corpus: `PDS→relay→appview` occurs **once**, as a
single table cell reading *"not P2P"*; `NIP-65` appears only as an address-discovery answer, never
as the outbox model. **The primitives were well studied; where the work and the cost live was not
studied at all.**

Three framings this document corrects, each load-bearing for what follows:

1. **"Cross-peer subscription means follows work" is an overclaim.** Subscription is **push** — it
   wants a live publisher *and* a live subscriber. The realistic scenario is **pull**: a client
   wakes, holds a list of followed peers, and refreshes — some offline, some reachable only through
   an intermediary. Those are different problems and only the second describes most users. **§5 is
   the answer, and it is better news than the overclaim was.**
2. **`GOSSIP` is unfinished, not abandoned.** It was a real V2.0 design (digest-based anti-entropy,
   explicitly epidemic broadcast) that was not carried into V7 for want of time and a settled view
   on which tier it belongs to — **not** because the question was closed. A survivorship table
   records what got carried; it is never evidence about *why*.
3. **A named entity type is not a capability.** RELAY Mode A is *"named for forward-compatibility;
   not normatively specified"* (§3.3) and is unimplemented. **§1 exists so that distinction is
   visible rather than assumed** — it is the single most common way a design record overstates what
   a system can do.

---

## §1 The status table — the instrument that would have prevented all three

**Every capability claim in this corpus should carry which of four rungs it is on.** They are not
degrees of confidence; they are different kinds of fact.

| Rung | Means | How you check |
|---|---|---|
| **S** — specified | normative text exists | open the spec |
| **I** — implemented | ≥1 engine has it | grep that tree at a commit |
| **X** — cross-impl verified | conformance/interop run | the run's artifact |
| **D** — deployed | running in the wild, unattended | a live URL |

| Capability | Rung | Note |
|---|---|---|
| Name → peer (`:resolve`) | **X** | three-way green |
| `http-poll` published-root fetch | **D** | this is what the live apexes serve |
| Content-addressed store + dedup | **D** | |
| Signed root + `seq` rollback floor | **D** | `seq` enforced as a rollback floor |
| **RELAY Mode S** (store-and-poll) | **D** | RELAY §3.2: *"a static-CDN-hosted peer's tree **IS** a Mode S relay namespace"* |
| RELAY Mode F (forward) | **S/I** | v1 ships it; interop breadth unmeasured here |
| Cross-peer SUBSCRIPTION (push) | **X** | verified in a lab; **assumes liveness** — §5.2 |
| Inbox-relay MX (`system/peer/inbox-relay`) | **S** | declared; deployment unmeasured |
| **RELAY Mode A** (aggregate) | **—** | named only. **No normative text. No implementation** |
| Internet-scale DISCOVERY backends | **—** | "later, driver-gated" |
| **GOSSIP** | **—** | unfinished, unranked (§0.2) |
| peer_id → endpoint | **—** | `PROPOSAL-PEER-TRANSPORT-SET` DRAFT |
| **Follow list as an entity** | **—** | **nothing models "who I follow"** — §6.1 |
| **Aggregator / read-only-consumer role** | **—** | searched; absent — §6.3 |
| Ordered timeline / feed index (RSS-shaped) | **—** | named deferred app extension |

**The two rows that matter most are the top and the bottom.** The **pull** path is at **D**. The
**social** rows are at **—**. It is easy to describe the bottom half in the vocabulary of the top;
this table exists to make that slip visible.

---

## §2 The central finding — every system that survived has an always-on tier that is not the user's device

This is the one sentence the whole survey collapses to, and it reframes the question from *push vs
pull* to **what is always on, who runs it, and what happens when it is absent.**

| System | The always-on tier | The user's device is… |
|---|---|---|
| **RSS** | the web server | a **poller**. Author offline is *meaningless* — the file is on a server |
| **SMTP** | the MX | a client that drains a **mailbox** that persisted while it slept |
| **Mastodon** | the instance | a view onto an always-on account host |
| **ATProto** | the **PDS** | a client. The PDS holds the repo and is dialable |
| **Nostr** | the **relays** | a client that writes to and reads from always-on relays |
| **IPFS / BitTorrent DHT** | ***nothing*** — peers **are** the network | **the network itself** |

**The last row is the one that decayed, and it decayed measurably.** §4.2.

The corollary is the useful half: **the always-on tier does not have to be trusted, and it does not
have to be us.** What it has to be is *reachable*. Which is exactly what a static origin is.

---

## §3 The systems, as operated

### §3.1 RSS — the one that never broke, and why

**Operating model:** author writes a file; a web server serves it; readers poll on their own
schedule. **No subscription state exists anywhere on the publisher.** The publisher does not know
who reads it, does not push, and cannot fail to deliver.

**What it costs:** one static file. Scales exactly like a web server.

**Its failure mode:** discovery (you must be told the URL), no social graph, polling is O(feeds ×
readers) with no coalescing.

**Why it matters to us:** it is the **only** model on this list where *the author being offline is
not a concept*. That is the requirement a real deployment has. And §5 is the finding that we
already implement this shape.

### §3.2 SMTP — the store-and-forward floor

**Operating model:** sender hands the message to an always-on MX for the recipient; the mailbox
holds it until the recipient drains it. Recipient liveness is decoupled from sender liveness by a
**third party that is neither**.

**What it teaches:** the MX record is a *per-recipient shard map* — mail for `@domain` goes to
that domain's MX and nowhere else, so the system shards perfectly with zero coordination. Our
horizontal-scaling analysis reached the identical structure independently: *"inbox-relay sharded by
recipient — the MX **is** the shard map."*

**Its failure mode:** spam, i.e. **open delivery with no admission control** — and the cost of
retrofitting admission (SPF/DKIM/DMARC, reputation) onto a protocol that shipped without it is the
single most expensive lesson available on this list.

### §3.3 Mastodon — federation is follow-driven, and nothing pulls you in

This answers the question a cluster-federation design must ask — *"how do federated instances connect together?
What pulls it into the network?"*

**Answer: nothing does, automatically.** An instance only sees content from remote servers that
**someone local already follows**. A brand-new instance is an empty room, and the documented
experience of running one is that timelines *"feel empty and disconnected"* without a steady
federation flow.

The three ways in, all opt-in and all manual:

1. **Local users follow remote accounts** — the base mechanism, and it is the *entire* transitive
   closure of what your instance can see.
2. **Relays** — the discovery accelerator. Mechanically this is ActivityPub itself: your server
   sends a `Follow` to the relay actor's `/inbox`, and the relay re-broadcasts participating
   instances' public posts. **Only admins subscribe.** Public relay lists **go stale quickly and
   many listed endpoints error** — a directory-decay problem worth noting, because it is the same
   shape as a stale connector set.
3. **Directories** — humans listing accounts in wikis.

**Its acknowledged weakness is precisely discovery:** inconsistent mechanisms make it hard to find
relevant content, against centralized platforms' unified search.

**What it teaches us:** *a "hundreds or thousands of independent clusters" model is
viable — Mastodon proves the social and governance half works.* But **it also proves the federation
graph does not form on its own.** If we ship the cluster model with no equivalent of relays and
directories, we ship Mastodon's cold-start problem with none of Mastodon's decade of workarounds.

### §3.4 ATProto — and the sentence that should change our roadmap

**Operating model:** three tiers. **PDS** holds a user's signed repo and is always-on. **Relay**
crawls all PDSes and emits a merged **firehose**. **AppView** consumes the firehose and builds
indexes, timelines and search.

**What it costs, measured:** a full-network relay was ~**$150/mo** in mid-2024; after Sync v1.1
removed the requirement to keep an archival on-disk copy of the network, a demonstrated full relay
runs **~$34/mo** on a VPS, with community operators on Raspberry Pis. The "cheap" tier is quoted as
**10 GB RAM / 500 GB SSD / 50 Mbps**. Firehose throughput ~**600 events/sec** typical with **2000/sec**
sustained peaks observed (April 2025 figures — 2026 rates are higher and were not re-measured here).
**Bandwidth, not storage or CPU, is the binding constraint**, and it grows as *events × subscribers*.
Their own mitigations are fan-out tiers: **Rainbow** (re-broadcasts the firehose to many clients) and
**Jetstream** (a lighter merged stream), plus **sharding on the account DID**.

**The sentence that matters most:** *relays are a **performance optimization** and are not strictly
necessary — clients may receive events directly from PDSes.*

**What it teaches us:** **the firehose is not an architectural requirement; it is an index-building
convenience.** It is tempting to read "we have no firehose" as a hole in the *protocol*. It is a
hole in the *product tier*, and it only exists once someone wants a global index. **A user who
follows 200 accounts never needs a firehose** — they need 200 cheap change-checks, which is §5.

### §3.5 Nostr — the live experiment in shipping without a reconciliation layer

This is the most important negative result available to us, because **Nostr built approximately the
model we would build, and it is centralizing anyway.**

**Operating model:** `npub` is the pubkey. Relays are dumb stores. Clients are smart. **NIP-65
(the outbox model)** has each user publish which relays they write to (**outbox**) and read from
(**inbox**), so clients can fetch from the right places instead of a global few.

**What actually happened, per a March 2026 critique and the implementers' own notes:**

- **Silent fallbacks.** If a declared relay is slow, the client writes to a default relay **with no
  indication that it happened.**
- **The inbox side is worse.** Most clients query the same **10–15 big relays** and *assume*
  replication got the note there. Sometimes it did. Often only partially.
- **The gossip layer was never built.** NIP-65 depends on relays gossiping with each other, and
  **that layer does not exist** — so a client cannot trust that fetching from declared relays
  returns a complete picture, and falls back to the big relays.
- **Enforcement was reverted.** Clients that enforced outbox strictly saw notes go missing, users
  complained, and enforcement was rolled back — leaving NIP-65 with *"lip service in readmes and
  silent override in production."*
- **A centralizing feedback loop.** Big relays accumulate everything because clients keep writing
  there as backup, which makes them more complete, which makes clients depend on them more.

**This is the single most transferable finding in the document.** The intuition — *"maybe
gossip works better, because you come and you join"* — is pointing at exactly the layer whose
**absence is the documented cause of Nostr's centralization**. Nostr shipped the publish-where-you-
like model *without* reconciliation and got de-facto centralization plus silent data loss. **We have
the same hole (GOSSIP unfinished) and have not yet shipped the social tier. That ordering is an
advantage and it is perishable.**

Also note the honest economics: free relays shut down routinely; guidance is to expect churn and to
support **paid** relays.

### §3.6 IPFS / the DHT — the ghost-town failure, quantified

The common intuition: *"with DHTs and BitTorrent and IPFS — nodes come on, they go away, the network
becomes a ghost town, and you're passing around a bunch of old-ass data for nodes that aren't even
live."*

**Measured, and worse than the intuition:**

- Provider records carry a **48h validity**, republished every ~**22h** because of churn.
- In one Kubo measurement, **over 70% of provider records pointed to unreachable peers** — the
  remainder mostly reachable only via a relay.
- A distinct failure: **address staleness** — the record survives while its addresses rot.
- **K=20** is chosen from *observed average churn*; when real churn exceeds the assumption, the
  guarantee degrades silently.
- **NAT'd / residential nodes are named as the biggest contributor to ghost records** — announced
  but undialable.
- The deployed mitigations are a retreat toward addressability: **delegated routing over HTTP**,
  provider prioritization, cached address books with active probing, IPNI.

**What it teaches:** a DHT does not fail loudly. **It answers.** It returns a record, the record is
well-formed and signed, and the peer is not there — and the consumer cannot distinguish that from a
withholding peer. This corpus already has a name for that indistinguishability problem, recorded
three times in this design record's own history. **A DHT would import it as a steady-state
condition affecting the majority of records.**

---

## §4 What the two failures teach, stated as rules

**§4.1 — A publish-where-you-like model without a reconciliation layer centralizes.** (Nostr.) The
absent layer is not a nice-to-have; its absence *is* the centralizing force, because every client
must hedge by also writing to the big aggregator, and hedging is what makes the aggregator complete.

**§4.2 — A liveness-derived index decays toward ghosts, and it decays quietly.** (IPFS, >70%.) Any
answer whose truth depends on a peer still being up must be either (a) refreshed faster than churn,
(b) verifiable at the consumer, or (c) both. Content addressing gives us (b) **for the bytes** and
nothing for **the pointer**.

**§4.3 — The corollary we are unusually well placed on.** In every system above, the aggregator must
be trusted to some degree: Nostr relays can omit; Mastodon relays can omit; an ATProto AppView is
believed. **For us an aggregator serves bytes and the hash verifies them.** `tree = path → hash` +
dedup + a signed root means **an untrusted aggregator can be structurally correct** — it can lie
only by *omission*, never by *substitution*, and omission is exactly what a signed `coverage:
complete` manifest is designed to catch. **That is a genuinely strong position and no surveyed
system has it cleanly.** It is what makes a *"I am just an aggregator / content backup"*
role cheap for us and expensive for them.

---

## §5 Where we actually stand — the pull path is the shipped path

**This is the answer to the wake-and-refresh scenario, and it inverts §0.1's overclaim.**

### §5.1 We already built an RSS-shaped tier and did not name it

- **`http-poll` is sessionless by construction** — NETWORK §6.5's own table: *"one-way fetch · no
  session · **consumer polls; publisher never dials**."*
- **A static origin is a first-class peer surface**: `published-root` + signed pointer + the
  Amendment-10 transitive content closure, uploaded as files.
- **RELAY §3.2 says it outright**: *"**a static-CDN-hosted peer's tree IS a Mode S relay
  namespace**"* — and Mode S is **shipped in v1** and **deployed** on the live apexes today.

**So the always-on tier of §2 is not missing. It is the thing we are already running.** The
publisher can be offline, asleep, or a browser that closed three weeks ago; the origin still serves
the signed root and the closure beneath it. **That is RSS's property, with content addressing and a
signature on top.**

### §5.2 Why the earlier "subscription" answer was the wrong half

SUBSCRIPTION is push and is verified — but *push* means the publisher fires on its own tree change,
which requires the publisher to be running. For a cohort of mobile/browser peers that is the
exception, not the rule. **Push and pull are both correct and they serve different populations:**
push for always-on peers and low latency; **pull for everything else, which is most things.**

### §5.3 The wake-and-refresh scenario, honestly traced

*"I go to my site, it wakes up, it says I'm following 10 peers, it tries to refresh."*

| Step | Rung | What is missing |
|---|---|---|
| Know the 10 | **—** | **no follow-list entity exists** (§6.1) |
| Locate each | **X** if named / **—** if bare id | name→binding works; bare `peer_id` is the drafted gap |
| Ask "did it change?" | **D** *(mechanism)* | signed root + `seq` gives a cheap check; **the refresh loop is unspecified** (§6.2) |
| Fetch what changed | **D** | `http-poll` + content closure; dedup means unchanged bytes cost nothing |
| Handle offline | **D** | **a static origin has no offline state** — this is the good news |
| Handle relayed | **S/D** | Mode S namespace poll; inbox-relay MX declared but undeployed |
| Merge into a timeline | **—** | deferred app extension |

**Six of seven steps have a mechanism; four are deployed.** The genuinely absent pieces are the
**follow list**, the **refresh loop**, and the **merge** — and none of them is network architecture.
**They are L5 application conventions over a substrate that already works.** That is a much smaller
and much better-shaped gap than "we need gossip before we can do social."

---

## §6 The gaps, honestly ranked

1. **The follow list has no home.** Nothing in the corpus models *who I follow*. It is a signed,
   per-user, ordered set of `(peer_id, name?, since)` — trivially expressible, entirely unspecified.
   **Cheapest item with the highest unlock; nothing else in §5.3 can be built without it.**
2. **The refresh loop is unspecified.** Poll cadence, backoff, coalescing, what a `seq` comparison
   costs, and what a consumer does with a root it cannot fetch. **RSS's whole operational character
   lives in this loop and we have written none of it.**
3. **The aggregator role does not exist.** Searched: no read-only-consumer or aggregator role in any
   spec. Per §4.3 this is the role we are *best* placed to define and it is undefined. It is also
   a want a real network has (*people join and say "I am an aggregator, a content backup"*)
   and the answer to the 100M-follower question in §7.3.
4. **`GOSSIP` — unfinished, unranked, and §4.1 says the clock is running.** Nostr is the live proof
   that shipping the social tier without a reconciliation layer centralizes. **But** §3.4's ATProto
   sentence says a firehose is an optimization, not a requirement — so **GOSSIP is not on the
   critical path to a working follow experience**, only to a *global index*. Those are different
   products and conflating them would over-scope the next year.
5. **peer_id → endpoint** — drafted (`PROPOSAL-PEER-TRANSPORT-SET`), three consumers waiting.
6. **Timeline/feed index** — named deferred app extension.

**Note the shape:** items 1–3 are **L5 conventions**, cheap, and unlock §5.3's scenario.
Items 4–6 are protocol work. **The product is closer than the protocol gap suggests.**

---

## §7 Three scenarios, traced

### §7.1 The read-only follower — *"maybe I don't publish, I just want to read"*

**This is the cheapest user in the system and nothing currently says so.** They need: a follow list,
a refresh loop, and `http-poll` fetch. They publish no root, need no origin, need no name, need no
inbound reachability, and are invisible to every peer they follow. **In Mastodon they need an
instance; in Nostr they need relay connections; in ATProto they need a PDS. For us they need
nothing but a client** — because pull requires no identity at the far end.

**That should be stated as a first-class role.** It is the on-ramp, it is most users, and it is the
strongest single argument for the pull model being primary.

### §7.2 The independent-cluster model — hundreds or thousands of registries

Mastodon proves the governance model works and that **the graph does not form by itself** (§3.3).
What we would need, in order: a **directory** of clusters (Mastodon's weakest link — its public relay
lists rot), a **cross-cluster follow** path (works today via name→binding), and eventually
**delegation** (`NS`-shaped, direction already pinned to the DNS-zone/TUF model, deliberately
deferred). **The failure to design for is not resolution — it is configuration distribution**, which
the naming exploration already identified as what breaks first at scale.

### §7.3 100M followers

Fan-out to 100M *push* subscribers is not a thing any surveyed system does, and neither should we.
**What they actually do:** the publisher writes **once** to an always-on origin, and the read side
scales by **caching and aggregation**, not by fan-out. ATProto: PDS writes once, relay/Rainbow/
Jetstream fan out to *subscribers of the firehose*, not to users. Mastodon: the instance holds it and
other instances pull.

**Ours degenerates well, and this is the payoff of §4.3.** The publisher writes one signed root to a
static origin. **That origin is a CDN.** 100M readers polling a content-addressed, immutable-beneath-
the-root tree is *the single best-understood scaling problem in computing*, and unlike every system
above, **an untrusted cache cannot corrupt the answer** — the hash verifies. **The 100M-follower case
is closer to solved than the 200-follower case**, because the 200-follower case needs the follow list
and refresh loop that do not exist yet, and the 100M case needs a CDN.

**What does *not* degenerate well:** anything requiring the publisher to know who its followers are.
**Do not build that.**

---

## §8 Can we support them all? Can we bridge to them all?

Preliminary, and this section is the least researched — flagged rather than concealed.

| System | Bridge shape | Difficulty |
|---|---|---|
| **RSS** | emit RSS/Atom from a published root; consume RSS into a follow list | **Low — and it should probably be first.** Pure pull both ways, no identity, no liveness. NETWORK §6.5.4 already anticipates sibling transport profiles |
| **SMTP/IMAP** | named in NETWORK Amendment 1 as an expected sibling profile | Medium |
| **Nostr** | NIP-05 ⇒ our `well-known-url` backend (already ranked #1 to build); events ⇔ signed entities | Medium; both are pubkey-first, which is the hard part already aligned |
| **ActivityPub** | WebFinger ⇒ same `well-known-url` backend; needs an always-on actor endpoint | **Higher — it is push/inbox-shaped**, which is the half we are weakest on |
| **ATProto** | `did:web` ⇒ `well-known-url`; a PDS-shaped origin is close to our static origin | Medium; the repo/commit model is genuinely close |

**The recurring answer is `well-known-url`** — the naming exploration already ranked it the single
highest coverage-per-effort backend because it covers **DID:web + NIP-05 + WebFinger** in one, ~200
LOC. **It is the bridge head for three ecosystems and it is not built.** Note the constraint
the resolution design imposes: it is a name-transmitting backend, so it must be pattern-scoped in the shipped
resolver config or every bare name a user types leaks.

---

## §9 What we do not know — the honest list

- **No capacity numbers exist anywhere in this corpus.** Every scaling claim is structural
  (*"scales like a CDN"*, *"O(domain) not O(users)"*). **Not one is quantified.** ATProto publishes
  theirs; we should be able to state ours.
- **Nothing has been proven at multi-peer scale.** The largest measured run is two devices.
- **Mode F's interop breadth is unmeasured** in this document.
- **The inbox-relay MX is specified and, as far as this survey went, undeployed** — which matters,
  because it is the offline path for *addressed* messages.
- **Whether GOSSIP or an aggregator role is the right answer to §6.3 is genuinely open.** They are
  different: gossip is peer-to-peer epidemic reconciliation; an aggregator is an operated tier. §3.4
  suggests the operated tier is the pragmatic first answer and §4.1 suggests the reconciliation
  layer is what stops it centralizing. **Probably both, in that order — but that is a hypothesis and
  this document does not rule it.**
- **2026 ATProto figures were not re-measured**; the numbers quoted are 2024–early-2025 vintage and
  are certainly low.

---

## §10 What this suggests

**Nothing here is a ruling**, and §9 lists too many open questions for one to be safe.

1. **State the pull path as the primary model** and the always-on static origin as the tier that
   makes it work. It is deployed, and it is RSS's property. Push becomes the optimization for
   live peers rather than the base case.
2. **Specify the follow list, the refresh loop and the read-only-follower role** — three L5
   conventions, cheap, and they unlock the whole §5.3 scenario without touching the protocol.
3. **Define the untrusted-aggregator role** on §4.3's argument, which is our genuine differentiator.
4. **Build `well-known-url`** — one backend, three ecosystems, and the bridge head for §8.
5. **Treat GOSSIP as the reconciliation layer for the aggregator tier, not as the prerequisite for
   follows** — §3.4's *"relays are a performance optimization"* is the licence to sequence it later,
   and §4.1's Nostr result is the reason not to sequence it *never*.
6. **Quantify something.** One capacity model, even rough, would be the first in the corpus.

**The reframe worth carrying out of this document:** the question was *"how do we get to a functioning
peer-to-peer network at global scale."* The survey's answer is that **none of the systems that work
are peer-to-peer at the data tier** — they are federations of always-on origins with smart clients,
and the ones that tried to be true P2P have measured decay. **We already run the origin tier. What we
have not built is the small, unglamorous application layer that turns it into following someone.**
