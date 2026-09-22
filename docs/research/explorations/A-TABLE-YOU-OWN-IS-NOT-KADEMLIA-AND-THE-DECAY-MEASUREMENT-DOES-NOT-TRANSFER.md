# A table you own is not Kademlia, and the decay measurement does not transfer

**Status:** EXPLORATION (2026-09-13). **Correction document.** It withdraws a claim this arc made five
times and restores one the corpus already held.

---

## §1 The error

**I wrote that this corpus "rejected the DHT." That is false, and the corpus says so in its own spec
text.** `EXTENSION-REGISTRY` §1 names **DHT** as one of the additional backends that *"compose on the
substrate and ship in their own proposals"*; §12 sequences it; §12 even anticipates *"oblivious DHT for
the DHT backend"* under query privacy. **A sequenced, unwritten backend is not a rejected one.**

**The deeper error is a category mistake, and it is one this arc has a name for.** I took a measurement
of **one implementation of one design** — Kademlia's provider-record decay, >70% unreachable — and
reported it as a property of **hash → location tables in general.** *A property of one expression
reported as a property of the architecture.* **Sixth instance in this arc's own catalogue.**

---

## §2 Two different things were sharing one word

| | **keyspace-partitioned overlay** *(Kademlia, Chord, R5N)* | **a table you own and publish** |
|---|---|---|
| who stores a record | ⛔ the node whose id is nearest the hash — **an unwilling stranger who does not hold the content** | ⭐ **you, about things you actually know** |
| why records rot | you hold records for content you do not have, from peers you never talk to, **with no way to check** | — |
| pruning | ⛔ **you cannot** — you do not know what is still valid | ⭐ **trivial and local: it failed, drop it** |
| agreement needed | the whole overlay must agree the keyspace partition | ⭐ **none** |
| liveness | the responsible node must be online to answer | ⭐ **it is an artifact — publish and walk away** |
| size | ⛔ forced: your slice of the keyspace | ⭐ **whatever you want. Two entries is a valid table** |
| composition | one global structure | ⭐ **many partial ones, merged by the reader; an aggregator is just a big one** |
| routing | ⛔ greedy descent through the overlay — **where restricted routes and the local-minimum trap bite** | ⭐ **no overlay routing at all; you fetch a document** |

> ⭐⭐ **Both recorded objections were objections to the left column.** The decay figure is a consequence
> of *storing strangers' records in a slice you did not choose.* The restricted-route / XOR-proximity
> objection is about *routing through an overlay.* **Neither touches a published locator table**, which
> stores nothing for strangers and routes through nothing.

---

## §3 The right column is the registry, and the corpus already specifies it

**Point for point, the substrate exists:**

| the model | already landed |
|---|---|
| *"I publish my own table; anyone can"* | §5 — *"**Anyone can publish a binding claiming any name. Receiver policy decides.**"* and §1 position 4 — *"**Registry is just a peer.**"* |
| *"you consult mine or you don't"* | §4.1 / §4.1a — the resolver-config **precedence dispatch list is the consumer's config** |
| *"maybe two entries, maybe thousands"* | nothing anywhere sets a size, a slice, or an obligation |
| *"maybe I'm an aggregator with thousands of peers"* | §8.2 — aggregator-as-meta-registry; *"the aggregator does **NOT re-sign**; receivers verify against the original issuer"* |
| *"if two tables disagree"* | §8.3 — **surface both, never silently pick**; consumer policy decides; fail-closed default |
| *"if a peer is dead I drop it"* | ordinary local state — **no coordination, because the table is yours** |

**And the federation status is not blocked, it is unwritten** — §8.2, corrected in v1.23:

> *"**Registry federation is unblocked; it is undesigned, which is a different and smaller
> statement.**"*

⇒ ⭐⭐ **The published-table model and the registry's model are the same model. The content lookup is a
registry backend keyed on a hash instead of a name.**

---

## §4 And keyed on a hash it is *cheaper* than every other backend

**This arc already established why and then drew the wrong conclusion from it.** A name is a **claim**
nobody can check, so the registry spends eight trust-anchor variants, precedence, revocation, TTL and
receiver policy making an uncheckable assertion usable. **A hash is a commitment to its own answer.**

⇒ **The entire trust apparatus is unnecessary for this backend.** A hostile or stale entry **costs one
wasted fetch and can cost nothing else.** That is not a reason to avoid the mechanism — **it is a
reason this is the easiest backend the registry will ever have.**

**One real delta, and it is the same one noted at the start of the arc:** §4.1.1 returns **a single
binding per name**, first hit wins. **A hash wants the union** — two answers are better than one and
ten are better still. So the backend's result cardinality differs from the name backends', which is a
**statable difference in the resolution contract**, not an obstacle.

---

## §5 What this dissolves

**The "permanent limit" I asserted one document ago — *a locator names only peers the publisher knew
about* — is wrong, and federation is why.**

**A third party who acquired the content later publishes their own locator claim.** A reader who
consults both tables, or an aggregator who unions them, finds it. **Nothing requires the original
publisher to have known anything.** That is precisely §5's *anyone can publish* plus §8.2's aggregator,
applied to a hash key.

⇒ **The gap I called permanent is the undesigned half of a landed extension.**

---

## §6 The structural point, which is the one to keep

> **Finding a hash is the same shape as finding contributions to a coordinate, entries in a feed, posts
> in a forum, or articles in an encyclopedia. It is a published set of claims about where things are,
> merged by the reader, with computation available over it. Only the predicate differs.**

**The corpus already holds this as a general property** — `DATA-EXCHANGE` §5.0's closure: *a gathered
result is the same kind of object as its sources, so aggregation can be aggregated.* **A locator table
is a gathered view whose predicate is *where*, and an aggregator of locator tables is a gatherer.**

⭐ **This is also what `LK-1`/`LK-2` said in the first document of the arc — *they are one machine* — and
what five subsequent documents lost.** The arc did not fail to find the answer. **It found it, then
argued itself out of it by treating a trust-cost difference as a mechanism difference.**

---

## §7 What is actually owed

1. ⭐ **A locator-claim type and a registry backend over it** — `hash → [where]`, signed, published,
   union-merged, reader-policy-selected. **The substrate is landed; this is the backend proposal §1
   already anticipates.**
2. **The resolution-contract delta** — §4.1.1's single-binding rule does not fit a union key. State it.
3. **Registry federation's normative text** (§8.2) — *undesigned, not blocked*, and it is what makes
   aggregation work.
4. ⚠ **Pruning and decay as a stated property rather than an inherited fear** — a table you own prunes
   on failure. **What that costs at aggregator scale is genuinely unmeasured**, and it is the real
   open question here, not whether the mechanism is sound.

---

## §8 Corrections issued

| claim | status |
|---|---|
| *"the corpus rejected the DHT"* | ⛔ **withdrawn.** §1 sequences it as a backend; §12 anticipates an oblivious variant |
| *"there are four structures and we rejected three"* | ⛔ **withdrawn.** Structure 2 conflated keyspace-partitioned overlay routing with federated published tables; **only the first is rejected** |
| *"a locator names only peers the publisher knew about — permanent"* | ⛔ **withdrawn** (§5) |
| *">70% decay is evidence against hash→location tables"* | ⛔ **withdrawn.** It is evidence against **storing strangers' records in a slice you did not choose** |
| *"restricted routes rule this out"* | ⛔ **withdrawn.** That objection is about **overlay routing**; a published table routes through nothing |

---

## §9 Cross-references

`EXTENSION-REGISTRY` §1 (backends sequenced, incl. DHT; *registry is just a peer*), §4.1/§4.1a
(consumer precedence), §4.1.1 (the single-binding delta), §5 (*anyone can publish*), §8.2 (federation —
unblocked, undesigned), §8.3 (conflict surfacing), §12 (sequencing; oblivious DHT) ·
`PROPOSAL-KEEPING-A-COPY-CURRENT…` §5.0 (closure) · register rows `LK-1`, `LK-2`, `LK-2a`, `F-13`,
`F-40a` · and `THE-BARE-HASH-HAS-FOUR-POSSIBLE-ANSWERS…`, **corrected by this document**.
