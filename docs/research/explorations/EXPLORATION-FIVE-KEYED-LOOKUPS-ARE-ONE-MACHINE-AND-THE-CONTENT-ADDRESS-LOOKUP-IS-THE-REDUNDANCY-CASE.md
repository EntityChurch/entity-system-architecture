# EXPLORATION — five keyed lookups are one machine; what separates them is not the key, and the content-address lookup is the redundancy case

**Status:** EXPLORATION (2026-09-13). Written to be attacked. Nothing here is ruled.

---

## §0 The result

The corpus contains **five keyed lookups**. Four are specified — three landed, one drafted — and the
fifth, *"given a content hash, who holds these bytes?"*, is the one that keeps coming back as an
apparently structural gap.

**Three findings, and the third is the one that changes what gets built:**

1. **They are one machine.** Every one is: a key, a set of records, a precedence or merge rule, a
   capability scope, and a federation story. The resemblance is not superficial and the instinct that
   *"a registry is a name-to-key lookup and a routing table is a hash-to-next-hop lookup"* is correct.

2. **But the axis that separates them is not the key type — it is whether the answer is an
   ASSERTION or a HINT.** A name binding is a claim nobody can check, so it needs trust anchors,
   precedence, revocation and TTL; that apparatus is most of what `EXTENSION-REGISTRY` is. A
   hash answer is self-verifying on arrival, so **a wrong one costs one round trip and never costs
   correctness.** Modelling the content-address lookup on the registry imports machinery for a
   problem the hash key does not have, *and* imports first-hit-wins resolution semantics (§4.1.1)
   that are precisely backwards for it.

3. ⭐ **The content-address lookup is a REDUNDANCY set, and it is the only one in the system.**
   `EXPLORATION-THE-THREE-UNITS-OF-EXCHANGE-AND-THE-HANDOFF-NOBODY-HAS-WRITTEN` §4.3 established
   that the peer-set analogy from file-sharing **breaks** for a participant set, because every
   participant wrote something *different* — a partial list means you get less content. **For the
   content lookup the analogy does not break: every holder serves the same bytes, so a partial list
   costs nothing at all.** That is the one place in this corpus where the borrowed mechanism
   transfers with its completeness argument intact, and it is why this lookup is much cheaper than
   it has been treated as.

**Consequence, and it is a reframe rather than a design:** because
`APP-CONVENTION-REFERENCE` §1 already makes a reference routable by construction — *"a reference is
routable if and only if it names one \[a publisher]; bytes alone name no holder you can go and ask"*
— **a reader always already knows somewhere to ask.** A holder set therefore answers only *"somewhere
faster, or somewhere still up."* **It is an accelerator over a path that already terminates, not a
resolution path.** It carries no correctness burden, it blocks nothing, and it does not need to be
registry-grade. That is the honest reason it has sat open for months without anything breaking.

---

## §1 The five, stated so they can be compared

| | key → value | mechanism | state | is the answer checkable by the asker? |
|---|---|---|---|---|
| **1** | `name → peer` | `EXTENSION-REGISTRY` | landed | **No.** A name is claimed, not derived |
| **2** | `peer → transports` | `EXTENSION-NETWORK` §6.5.1c (`system/peer/transport-set`), §6.7 | landed | **Partly** — the set is signed by the peer; whether it is *current* is not checkable |
| **3** | `destination → next hop` | `EXTENSION-ROUTE` | landed | **No** — but the table is signed by a configuring authority and cap-scoped |
| **4** | `coordinate → who contributed` | `PROPOSAL-THE-COORDINATE-AND-WALK-DISCOVERY` + `PROPOSAL-THE-PUBLISHED-WALK` | **DRAFT** | **Yes, per claim** — fetch the participant's signed entry and check it names the coordinate |
| **5** | **`content hash → who holds these bytes`** | ⛔ **nothing.** Nearest is the `via: {tag: "peer"}` hint | **gap** | **Yes, completely** — the bytes hash to the key or they do not |

A sixth exists and is worth naming so it is not mistaken for the fifth: `EXTENSION-QUERY` §2.2's
**reverse hash index** is `hash → entities that reference it`, **strictly local to one peer's tree**,
MUST at Level 1. It answers *"what of mine depends on this"* — GC safety, impact analysis — and never
*"who else has it."* Its existence is part of why the fifth row reads as an omission: the local half
is built and mandatory.

---

## §2 The axis: an assertion needs trust; a hint needs none

Lay the five out by what it costs to be wrong, and the apparent family resemblance separates cleanly.

| | a wrong answer costs | so the mechanism must supply |
|---|---|---|
| `name → peer` | **you talk to the wrong party**, believing you reached the right one | trust anchors, precedence, receiver policy, revocation, TTL, negative caching |
| `peer → transports` | a failed dial, then fallback | ordering, expiry, a fallback loop |
| `destination → next hop` | a misrouted or dropped envelope | signing, cap-gated writes, metric precedence, fail-closed default |
| `coordinate → who` | **one wasted fetch**, plus a silently incomplete view | union merge; an acceptance policy that is the reader's |
| **`hash → holders`** | **one wasted fetch. Nothing else** | union merge, and nothing |

`EXTENSION-REGISTRY` is 1,900 lines and eight trust-anchor variants (§2.4) because of the first row,
and its §1 says so directly: *"the substrate gates no name claims. Anyone can publish a binding
claiming any name. Whether a receiver TRUSTS that binding is the receiver's policy."* **Every
expensive thing in that document exists to make an uncheckable assertion usable.**

The hash key does not have that problem, and it does not have it *by construction rather than by
good fortune*: the key **is** a commitment to the answer. Nobody can lie about which bytes hash to
`H`; the worst a hostile holder record achieves is to waste a request. **A design that spends
registry-grade machinery on the fifth row has bought nothing it needed.**

> **The trap this closes.** *"Registry, routing table and content lookup are all key-value lookups,
> so one mechanism should serve all three"* is a true premise with a false conclusion. They share a
> *shape*; they do not share a *threat model*, and the threat model is where all the cost lives.
> **Generalize the shape, never the apparatus.**

### §2.1 And one semantic that is actively wrong to inherit

`EXTENSION-REGISTRY` §4.1.1 — **single binding per name per resolution** — returns the *first hit
that validates*, and the lower-priority backend's binding is *"never surfaced for that resolution."*
That is correct for names: two answers to *"who is `alice.example`"* is a conflict to adjudicate.

**For holders it is the opposite of what is wanted.** Two answers to *"who has `H`"* is not a
conflict; it is a better answer than one, and ten is better still. **The fifth row wants union and
breadth where the first row wants precedence and a single winner** — and the two cannot be served by
one resolution rule, whatever they share structurally.

---

## §3 So what *is* the registry — and why it is deliberately off this path

Stated plainly, because the answer has been derived twice and is worth having in one sentence:

> **The registry is the cold-start entry point and the authority for `name → peer`. It is not on
> the traversal path, and putting it there is a known failure.**

Two independently-derived results already say this, and neither is about the key type:

- The graph is traversable without it. A coordinate names its owner for three of its four kinds;
  citations carry locators; forward traversal from a follow set discovers parties you do not follow.
  **Reaching a peer you hold a coordinate for needs no registry at all.**
- The surveyed federated systems that centralized did so because a *lookup* became a *prerequisite
  for every traversal* — a fusion, not a policy choice. **Any design that answers a discovery
  question with "consult the registry" should be suspected on that ground before it is drawn**, and
  that caution is worth more than the gap it guards.

`PROPOSAL-THE-COORDINATE-AND-WALK-DISCOVERY` §4.3 states the same boundary from the other side:
`name-coord` *"is not a name-resolution mechanism — it is a subject identifier that happens to be
spelled with characters, and it confers no authority over anything."*

**So: registry-adjacent is the wrong neighbourhood for the fifth row, and the reason is not that the
key is a hash. It is that a holder lookup would sit on every fetch.**

---

## §4 What the fifth row would be, if it is built

It is **not a new mechanism.** It is a fifth coordinate kind and a third predicate on an object
already drafted.

`PROPOSAL-THE-COORDINATE-AND-WALK-DISCOVERY` §2 declares four coordinate preimages —
`entry` (a pinned reference — owned), `namespace` (a peer — owned), `path` (peer + subtree — owned),
`name` (scheme + value — **ownerless**). **A bare content hash is none of them.** `entry-coord`
wraps a *reference*, and a reference carries a publisher by requirement; the fifth row's whole
premise is that you hold bytes' identity and **no** publisher.

```cddl
content-coord = { kind: "content", hash: content-hash }   ; these exact bytes — OWNERLESS
```

Predicate: **"I hold these bytes and will serve them."** Everything else is inherited:

| inherited from | what it gives the fifth row |
|---|---|
| §3.1 constructable path `/{holder}/system/walk/{hex(coordinate)}` | the address of any reachable peer's holder claim is **computable**; no lookup service is consulted at any point |
| §3.2 tracked-prefix declaration | a 404 becomes a **verifiable negative** instead of could-not-look, if the holder paid one declaration for it |
| §4.1's generalization | *"a grow-only set of independently-signed claims about one coordinate, merged by union, with the reader applying an acceptance policy… the information flow generalizes; what is in the claim does not have to"* |
| `EXPLORATION-THE-THREE-UNITS…` §4.2 | a **published artifact, not a live service** — still there after the publisher leaves, verifiable offline, re-servable by anyone, merges with a partial answer, and **the same type as everything else** |

**This is the discharge of a normative obligation, not an optional nicety.**
`ENTITY-CORE-PROTOCOL` §3.5 exempts *fetching* a held hash (*"content-addressed lookup is scheme-free
and exempt"*) but **finding a holder record is discovery of a new entity**, and §3.5 requires any such
entity to declare Strategy A (observational) or Strategy B (a core-general constructable path).
**Strategy B is available here for the same reason it is available for a walk, and Strategy A fails
here for the same reason: it finds only holders you have already seen.**

### §4.1 Why this is not the DHT that was already rejected

The rejection on record is specific and survives: a DHT needs **live nodes to answer**, and a
publisher who closes a laptop can neither answer a lookup nor store an announce — measured against a
deployed DHT where **>70% of provider records were unreachable**, with every deployed mitigation
moving back toward addressability.

**A published holder set is not a live lookup, and the difference is pull versus push.** The holder
writes a static artifact at a path anyone can compute and serves it like any other bytes; nothing has
to be online to *answer* beyond serving a file. **That is exactly the property the DHT lacked**, and
it is why the same conclusion — *the index is published* — lands here as it did for the two-level
index.

---

## §5 The three things that could kill it, honestly

### §5.1 ⚠ A holding claim is about the PRESENT, and the reply claim is not — this is the sharpest objection

A participant claim is monotone: *someone replied* stays true forever. **"I hold these bytes" is a
statement about now, and it decays.** Union merge over a grow-only set accumulates stale holders
indefinitely, which is the provider-record decay above **reappearing inside our own mechanism rather
than being escaped by it.**

**It is survivable, and the reason is specific rather than optimistic:**

- **You need one live holder out of N, not all of them.** This is §0's third finding doing real work
  — for a redundancy set, decay degrades *latency*, where for a composition set it degrades *content*.
- **The authoritative path still exists underneath.** The reference names a publisher; if every
  holder claim is stale you are exactly where you were before the mechanism existed. **The floor is
  the current behaviour, not failure.**
- **A claim can carry `expires_at`**, as `EXTENSION-ROUTE` §2 already does for a route entity — so a
  holder decides how long it is willing to be quoted.

**What must NOT happen: the holder set becoming load-bearing.** `APP-CONVENTION-REFERENCE` §2.3
already pins the discipline for exactly this class of term — *"a reader that ignores every `via` hint
MUST reach the same answer as one that uses them, or fail"* — and **the same rule has to bind a
holder set or the decay stops being a latency problem.** A hint that becomes required is a link that
rots.

### §5.2 ⚠ Volume — one leaf per blob is not obviously payable

`PROPOSAL-THE-COORDINATE-AND-WALK-DISCOVERY` §3.3 answers the *snapshot* cost (growth is in what
changed, not in what is held) but **not the leaf count**, and the fifth row is the kind with the worst
cardinality by far: a walk covering 10,000 subjects is a large walker, while a peer holding 10,000
blobs is **an ordinary peer.** Publishing one leaf per blob held is a different order of magnitude
from anything the walk proposal costed.

**The likely shape of the answer, flagged as a lead and not a conclusion:** you do not publish
per-blob. You publish per `namespace-coord` or `path-coord` — *"I serve this peer's estate"* — and the
reader, holding a reference that already names a publisher, asks *"who mirrors **that publisher**"*
rather than *"who has **this hash**."* That is one claim per mirrored estate instead of one per blob,
and it composes with the two-level index result rather than competing with it.
`EXTENSION-QUERY` §2.5's path-prefix zone summaries are a candidate aggregation structure and have
**not** been read for this purpose. ⛔ **This is the open question that decides whether the fifth row
is cheap or expensive, and it has not been costed.**

### §5.3 ⚠ Tier

`PROPOSAL-THE-COORDINATE-AND-WALK-DISCOVERY` §4.4 already flags that a constructable-path convention
**cannot live in a document only some clients read** — §3.5 forbids path-construction against an
extension-private scheme, so a client that does not know the convention cannot construct the address.
**The fifth row inherits that argument and strengthens it**: holder claims are useful in proportion to
how many unrelated clients publish them, which is the definition of an interoperability-floor
convention. **Flagged, not decided — and it is the same open flag, not a second one.**

---

## §6 What this changes

**On the question as it is usually asked** — *"is the content-address lookup a DHT, a registry, a
routing table, or something we are missing?"*:

- **Not a DHT** — for a reason already measured, and the published-artifact form escapes it.
- **Not the registry** — not because of the key, but because the registry is a cold-start entry point
  and this would sit on every fetch.
- **Not a routing table** — `EXTENSION-ROUTE` is `destination peer → next hop`, deliberately a storage
  plane with no computation, and its own §3 notes that peer-ids are *"a flat, non-hierarchical space
  with no aggregation structure."* A holder lookup is a different key answering a different question,
  and ROUTE's **deferred computed-routing hatch** (§4) is the seam a live-lookup backend would use —
  which is the seam this design is specifically declining.
- **A fifth coordinate kind and a third predicate on the walk** — no new mechanism, which is the
  result worth having.
- **And an accelerator, not a resolution path** — which is why it is not blocking, and why it should
  not be designed as though it were.

**What is genuinely owed**, in order:

1. **The volume question (§5.2).** Per-blob versus per-estate decides the whole shape. Nothing else
   should be drawn first.
2. **The non-load-bearing rule (§5.1)**, stated wherever this lands, in the words
   `APP-CONVENTION-REFERENCE` §2.3 already uses.
3. **The tier flag (§5.3)** — resolved together with the walk's, because it is one question.

**What is NOT owed:** a format, a resolution algorithm, a registry backend, or a mechanism. Two
existing drafts and one landed convention already carry all three, and the remaining work is
composition plus one cost measurement.

---

## §7 What would test this

- **A second holder.** The redundancy-set claim in §0 is structural and cheap to falsify: put the same
  bytes behind two independent publishers and check that a reader with a partial holder list is not
  worse off than one with the full list. If it is worse off, the set is not a redundancy set and §0's
  third finding is wrong.
- **A count.** One peer's blob inventory, measured, against §5.2's per-blob leaf assumption. This is a
  measurement, not an argument, and it has not been taken.
- **A stale-holder run.** Populate a holder set, take every holder offline, and confirm the reader
  lands on the publisher path with only a latency penalty. If it lands anywhere else, §5.1's
  non-load-bearing rule is not actually being honoured by the implementation.

---

## §8 Cross-references

- `EXTENSION-REGISTRY` §1, §2.4, §4.1.1 — the name lookup, its trust apparatus, and its first-hit rule
- `EXTENSION-ROUTE` §1, §3, §4 — the routing table, its match rule, and its deferred computed hatch
- `EXTENSION-NETWORK` §6.5.1c, §6.7 — `peer → transports`, and the reachability facts under it
- `EXTENSION-QUERY` §2.2, §2.5 — the local reverse hash index; the candidate aggregation structure
- `APP-CONVENTION-REFERENCE` §1, §2.3 — routable-iff-it-names-a-publisher; the hint discipline
- `ENTITY-CORE-PROTOCOL` §3.5 — discovery locality; Strategy A / B; the content-fetch exemption
- `PROPOSAL-THE-COORDINATE-AND-WALK-DISCOVERY` §2, §3.1, §3.2, §3.3, §4.1, §4.3, §4.4
- `EXPLORATION-THE-THREE-UNITS-OF-EXCHANGE-AND-THE-HANDOFF-NOBODY-HAS-WRITTEN` §4.2, §4.3
- `EXPLORATION-THE-OPERATING-MODELS-AND-THE-ALWAYS-ON-TIER` — the DHT measurement, and why an index is published rather than served
- `EXPLORATION-THE-TWO-LEVEL-INDEX-COORDINATE-TO-WHO-AND-THE-GATHERER-CLOSURE` — the split this
  document's §5.2 lead composes with
