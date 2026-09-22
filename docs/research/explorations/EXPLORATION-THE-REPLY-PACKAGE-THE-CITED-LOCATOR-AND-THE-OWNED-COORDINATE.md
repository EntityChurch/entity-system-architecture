# EXPLORATION — the reply package: what a citer actually publishes, why the locator travels with the citation, and who the natural walker is

**Status:** exploration. **Nothing here is ruled and no spec text is proposed yet.**
Predecessor: `EXPLORATION-THE-TWO-LEVEL-INDEX-…` (the walk) and `EXPLORATION-THE-READERS-LOOP-…`
(the four routes). Handoff: `HANDOFF-2026-09-07-…`, whose §3.2 (discovery) and §3.3 (publish policy)
this document moves.

---

## §0 The result

1. **A reply publishes three objects, not one**, and only the third is new: the **entry**, a **mirror
   of what it cites**, and **a locator for the cited peer**. §1.
2. **"I copied it in" is right in intent and wrong in shape** — `FEED` §1.2 forbids inlining another
   entry's bytes, for a reason that is load-bearing (an inlined copy gets a new hash and stops being
   the same object, which kills union). **The correct shape is to carry it alongside as a mirror**, at
   the same origin, same hash. The visitor still fetches nothing extra. §1.1.
3. **The cited locator needs no new type and, it turns out, barely needs a rule.**
   `PROPOSAL-PEER-TRANSPORT-SET` already builds a **peer-signed, anyone-may-serve** reachability record
   and dissolves *"who may serve it?"* with **"anyone"** — explicitly naming *"ask whoever you are
   already talking to"* as a legitimate retrieval path. **The cited locator IS that path**, and all
   that is owed is the convention that a citer serves one. §2. It fills a hole the corpus names
   outright (`NETWORK` §6.5.4: transport discovery is out-of-band in v1; `[OPEN-FEED-5]`), and it is
   safe from a stranger for **exactly the reason a `participants` claim is** — `REGISTRY` §5.1's
   connection-authority invariant. §2.3.
4. **That gives half of discovery for free.** A citation that carries a locator makes the content graph
   traversable with no registry at all — **the bootstrapping question splits**, and the
   half that survives is narrower. §3.
5. **Publish policy has a rule, and it is not a heuristic: publish the walk anchored at a coordinate
   you own.** The author of an entry is the one party every reader can *derive* the location of, and
   the one party with a standing interest in every reply. Ownerless coordinates keep the whole problem;
   owned ones dissolve it. §4.
6. **A gap this exposes:** `PROPOSAL-THE-PUBLISHED-WALK`'s `coordinate: reference` pins **one entity**,
   so *"replies to anything I wrote"* — the natural subscription unit —
   **is not expressible.** §5.
7. **A testable hypothesis, with falsifiers**, in §6.

---

## §1 What a reply actually publishes

Bob replies to Alice's entry `E`. Written out, Bob's origin now serves:

| # | Object | Type | New? |
|---|---|---|---|
| 1 | **the reply** | `app/feed/entry` with `reply: {root, parent}` carrying a `reference` pin | no — `FEED` §2.2.1, §3 |
| 2 | **a copy of `E`** — original bytes, Alice's detached signature | a **mirror** (`FEED` §4) — which is a walk at the top rung over `E`'s coordinate | no |
| 3 | **how to reach Alice** — the reachability record Bob used | **Alice's own signed transport-set** if she publishes one; otherwise a `self-certifying` binding carrying Bob's observation | **the open one — and it is a convention, not a record** |

**Only row 3 has no home today**, and rows 1–2 are the answer to *"does the visitor need to go grab
it?"* — **no, if Bob publishes row 2.** That is one extra entity at Bob's origin and no change to the
reply.

### §1.1 The correction that matters: alongside, not inside

`FEED` §1.2 is a MUST and it is right: ***an entry MUST NOT contain another entry's bytes*** — *"an
inlined copy gets a new hash inside a new parent and is no longer the same object to anybody."* Inline
`E` into the reply and two readers who assembled the same conversation independently no longer hold
`E` at the same name, so **merging stops being set union** and every property downstream of that goes
with it.

**Publishing `E` beside the reply has none of that cost and all of the benefit.** Same bytes, same
hash, same signature, fetched from Bob instead of Alice — which is precisely what `FEED` §4.1 already
authorises (*republish original bytes; omit but never substitute; a mirror is not authorship*). A
reader that already holds `E` pays nothing; one that does not pays one fetch to an origin it was
already talking to.

> **So "self-contained" is a property of the ORIGIN, not of the entry.** Bob's site carries everything
> a visitor needs; Bob's *reply* carries only the reply. **The unit of self-containment and the unit of
> addressing are different, and conflating them is what the §1.2 MUST prevents.**

---

## §2 The cited locator

### §2.1 The hole is named in the corpus, in normative text

`EXTENSION-NETWORK` §6.5.4: *"How a consumer learns of a publisher's transport profiles is out of scope
of this section. For v1: out-of-band (well-known URL, hard-coded, manually configured)."* And §6.5.4
again, on a static publisher's own profile: *"out-of-band too."*

`EXTENSION-REGISTRY` reaches the same wall from the other side (§6a.3, the peer-issued `transports`
MUST): *"§4.1.2's 'transport resolution per NETWORK §6.5 finds reachable endpoints' … **has no static
counterpart**: NETWORK §6.5.4 makes profile discovery out-of-band in v1, so for a statically published
peer there is nothing to find. A consumer resolves a peer-id and stops."*

**So a bare peer-id is a dead end unless the reader already holds a binding for it** — which is
`[OPEN-FEED-5]` (`FEED` §11, *"bare-id → origin"*), currently parked on
`PROPOSAL-PEER-TRANSPORT-SET`. A `reference` is `{peer, hash, ?path}` (`FEED` §2.2): it says **who**
and **what** and, optionally, **where in their tree** — and nothing about **where on the network.**

**Region searched, so this negative is reviewable:** `NETWORK` §6.5.1–§6.5.6; `REGISTRY` §3, §3b, §4.1,
§5–§5.2, §6–§6.7, §6a.1–§6a.3, §8; `FEED` §2.2, §4; `GUIDE-NETWORKING-MODEL` §7; plus a corpus grep for
`locator` / `resolution hint` / `cached binding` across `specs/`, `guides/` and `docs/proposals/`.
**What is absent is the practice, not the type** — see next.

### §2.2 The record exists, it is already *designed* to be served this way, and there are two rungs

**This was very nearly written as "a new convention over `system/registry/binding`." That would have
been a worse answer, and reading the proposal instead of the specs is what caught it.**

**Rung 1 — the peer's own signed transport-set, republished by the citer. This is the right answer.**
`PROPOSAL-PEER-TRANSPORT-SET` §1 dissolves *"who may serve it?"* in terms: ***"Anyone. There is no rule
to write.** A registry, a relay, a CDN, a peer that met P once, a QR code on a business card. Serving a
signed record confers nothing on the server and requires nothing from it."* And §1 again, on lookup
shape: *"none of the retrieval paths is a trust boundary… **'ask whoever you are already talking to'
becomes a legitimate implementation.**"*

> **The cited locator is not a new mechanism; it is that sentence's most obvious instance, and nobody
> had written it down.** §5's R2 is *"servable by any party"* and R1 is *"self-authenticating against
> `peer_id` alone"* — so **Bob republishing Alice's transport-set requires zero trust in Bob**, detects
> a dropped member (the set is the signed unit, §2), and detects a stale answer (`seq` + `expires_at`,
> R3). *(R1 is about **verifying** with the peer-id alone, not **finding** — the record still has to
> reach the consumer from somewhere, which is exactly the gap §2.1 measures and this closes.)*

**Rung 2 — a `self-certifying` binding, when the peer has published no set.** `REGISTRY` §3's `kind`
enum includes **`"self-certifying"`**: *"no issuer signature; `name == target_peer_id`; trivially
verified by checking that `name` Base58-decodes to a valid V7 §1.5 peer-id."* Carrying `transports`,
that object reads as ***this is peer P — which you can check yourself — and here is where I found
them***, and it makes **no naming claim**, which is why it is the right fallback atom. §5 permits the
publication outright (*"Anyone can publish a binding claiming any name. Receiver policy decides"*) and
§3 adds that *"aggregators MAY re-publish at their own path."*

**The two rungs differ in exactly one property and it is worth naming: whose claim it is.** Rung 1 is
**Alice's** claim, carried by Bob. Rung 2 is **Bob's** claim about Alice. The first is checkable before
you dial; the second is checkable only by dialing. Both are cheap to be wrong about (§2.3) — but a
consumer holding both should prefer the first, and **rung 2 exists because a peer that has published
no set is otherwise unreachable-by-citation, which is the case §2.1 says is unsolved today.**

**So what is owed is a convention and not a mechanism:** *a citer SHOULD serve, alongside a citation,
the best reachability record it holds for the cited peer*, plus the consumer-side preference order.

> **A local-name binding is NOT the vehicle**, and the reason is worth recording so nobody reaches for
> it: `REGISTRY` §6.3 gives it **no signature** on the explicit grounds that *"the user IS the trust
> source; the binding lives in the user's local store; signing is meaningless"*, and §6.1 says
> local-names *"have no global meaning."* It is the right shape for **holding** an observation and the
> wrong one for **publishing** it.

### §2.3 Why a locator from a stranger is safe — and it is the same argument twice

`REGISTRY` §5.1, verbatim: ***"Resolution never confers connection authority.** A binding from an
untrusted or low-trust backend can be safely consumed for its `transports` field because dialing those
transports does NOT admit the peer — IDENTIFY is the gate."*

**That is structurally the same sentence as `PROPOSAL-THE-PUBLISHED-WALK` §2.1's added rule** — *"a
`participants` claim is a POINTER, never evidence… a fabricated participant costs one wasted fetch and
nothing else."* Both say: **the claim is checkable at the destination, so being wrong about it is
cheap, so no trust in the source is required.**

> **One principle, on a second noun: anything a peer learned
> by walking can be republished as a pointer, because pointers are validated where they land.** A walk
> is the pointer *who contributed*; a locator is the pointer *where to reach them*. Neither is evidence
> and neither needs to be.

**The cost of a hostile locator is at most one failed dial, and at rung 1 it is zero.** A substituted
**transport-set** fails Alice's signature and is refused **before** anything is dialed; a substituted
**binding** costs one dial that fails at IDENTIFY. Either way the attacker cannot do the one thing that
would matter — make you believe something Alice did not sign. **The residual is denial (a locator that
sends you nowhere) and metadata leak (the attacker learns you looked)** — both real, both cheap, both
already true of every other hint in the system.

### §2.4 Staleness is the expected state, and the corpus has the vocabulary

is; `seen` asserts only this is what was there when I linked. An author who omits it is saying they
have no expectation to offer."* A published locator is an **observation with a timestamp**, never an
assertion of current reachability, and a reader that fails on it falls through to whatever it would
have done anyway (a registry, another citer's locator, nothing).

**And staleness is self-healing along the graph**, because every subsequent citer republishes what
*they* observed. A peer that moves is re-pointed by the next person who reaches it, with no
coordination and no revocation. **The freshest locator for a peer is held by whoever talked to them
most recently, which is exactly the population most likely to be citing them.**

---

## §3 What this does to discovery

**The open bootstrapping question — how a reader finds a gathered set for a coordinate it holds —
with *registry-adjacent* flagged as a candidate and not a derivation.** The cited locator splits that
row in half.

- **Reaching a peer you have a coordinate for: solved, and with no registry.** The coordinate names
  the peer; the citation that gave you the coordinate carries the locator. **You can walk the content
  graph from any starting point using only bytes served by the peers you already reached** — which is
  the property that a visitor to a citer's site has what they need, and it is
  forward traversal becoming *reachable* rather than merely *nameable*.
- **Finding a walk for a coordinate whose participants you have never heard of: still open**, and §4
  narrows it further.

**The registry does not go away and its job gets clearer.** It is the authority for
**name → peer** (a human-meaningful name is a claim someone has to stand behind) and the **cold-start
bootstrap** (your first peer). It is **not** the discovery path for the graph, and building on the
assumption that it is would put a lookup service on the path of every traversal — which is the
concentration failure this design avoids on every other axis.

---

## §4 The owned coordinate — and this is the publish-policy rule

**This is route 3 (`THE-READERS-LOOP` §3.3), with its prerequisite discharged the way `THE-TWO-LEVEL-
INDEX` §5 step 7 discharges it** — the author *pulls* from a walker they chose, so no inbox is opened
and no delivery grant is issued. What is new is not the loop; it is what the loop implies about *who
should publish*.

### §4.1 The rule

> **Publish the walk anchored at a coordinate you own.**

Because for an owned coordinate, three things are simultaneously true and they are true of nobody else:

1. **You are derivable.** A coordinate is a `reference` and a `reference` names its peer. **A reader
   holding `C` can compute where to look for a walk over `C` without asking anyone** — it is the
   namespace `C` already names.
2. **You have a standing interest in every contribution**, which no third party does; a walker covers
   what it chose, and that selection is a popularity bias.
3. **You are the only party for whom the scope is not arbitrary.** *"Replies to my entries"* is a
   complete, self-delimiting scope. *"Replies to things about topic X"* is not, and never will be.

### §4.2 It bounds redundancy without a heuristic

`PROPOSAL-THE-PUBLISHED-WALK` §7.3 asks *"there is already a registry doing this, why would I
republish"* and calls a bad default here the way a distributed database becomes a swarm of
near-identical documents. **The ownership rule answers it structurally rather than by tuning:**

**K people reply to Alice's entry. Under the rule, exactly ONE party republishes them — Alice.** Not
because anyone forbade the others, but because for each of the other K−1 the coordinate is not theirs
and their natural locus is elsewhere. **The count of natural publishers per coordinate is the count of
owners, which is one.** `subsumes` remains the bound for everything else; **the rule removes most of
what it would otherwise have to bound.**

### §4.3 The honest limits, both of them

- **The author's walk is the most convenient and the least neutral.** Curation and moderation are the
  same act — the corpus already says so — and the party with the strongest interest in a conversation
  is the party with the strongest interest in shaping it. **`FEED` §4.1's *omit but never substitute*
  is the whole protection**, and it is a real one: Alice can leave a reply out, and she cannot forge
  one or alter one. **Union with any other walker's coverage is the corrective**, which is why the rule
  is a default and not an exclusivity.
- **Ownerless coordinates keep the entire problem.** A topic, a tag, a subject nobody wrote — no
  derivable location, no natural publisher, redundancy fully live, discovery fully open. **That is the
  hard half and it is unchanged.** What the rule buys is that the *common* case — a conversation
  anchored on something somebody wrote — is not the hard case.

---

## §5 The gap this exposes: a walk cannot say "anything in my namespace"

`PROPOSAL-THE-PUBLISHED-WALK` §2 declares `coordinate: reference`, and `FEED` §2.2's `reference` is
`{peer, hash, ?path}` — **a pin on one entity.**

**So the everything-about-me case is not expressible.** *"All the replies to my entries"* is a walk over
Alice-the-namespace, not over one entry, and under the current schema it is N walks for N entries —
which is absurd at any volume and is also the wrong subscription unit, since the reader wants to
subscribe once and keep getting them.

**The fix is to widen what a coordinate may be**, and the constraint that decides it is already stated:
`THE-TWO-LEVEL-INDEX` §2.2a's whole reason the split works is that **a coordinate is computable by both
parties with no coordination** (the BitTorrent infohash property). Candidates that satisfy it:

| Coordinate | Computable by anyone? | Owned? |
|---|---|---|
| an entry pin — `reference` | yes | **yes** — the entry's author |
| a **namespace** — a peer-id | yes | **yes** — that peer |
| a **tree path** — `{peer, path}` | yes | **yes** — that peer (`R-6`: authority is a property of the pair) |
| a **topic string** | yes | **no** — and this is where §4.3's hard half lives |

**The first three are the owned cases and they are the ones §4's rule applies to.** Note that a
namespace-scoped or path-scoped walk is a *published* form of something `EXTENSION-QUERY` §2.2 already
computes locally — its path link index is keyed `referenced_path → {source_path, …}`, and paths are
namespace-prefixed, so *"all backlinks into `/{alice}/…`"* is a prefix read. **The primitive exists
locally; what is missing is the published shape.**

**Not proposed here.** It is a schema change to a DRAFT proposal and it should be made once, with §7.1
(*what a walk should contain*) answered beside it, rather than twice.

---

## §6 The testable hypothesis

**The claim under test, stated so it can fail:**

> **A reader can traverse the content graph, verify everything it renders, and reach peers it has never
> heard of, using only bytes served by static origins it was already talking to — with no registry, no
> aggregator, no live peer, and no out-of-band configuration beyond its first origin.**

**Falsified if** completing the walk requires contacting a party that neither the reader nor the peers
in the chain chose.

**The fixture: four peers, four static origins, nobody online.** Cold client, empty store, no
resolver-config beyond one origin.

| # | Claim | Setup | Measurement | Falsifier |
|---|---|---|---|---|
| **1** | **Reachability** | A publishes `E`. B publishes: the reply, a mirror of `E`, and a self-certifying binding for A. C follows **only B** and has never heard of A | The set of hosts C contacts, and **which bytes taught it each host**. Expect `{B, A}`, with A learned from B's binding | C cannot reach A, or reaches A via a party nobody in the chain chose |
| **2** | **No trust in the intermediary** | as above | `E` verifies under **A's** signature from B's mirror | flip one byte of B's copy → **C must refuse it, not render it** |
| **3** | **A hostile locator is cheap** | B's locator points at an attacker origin serving a valid but different peer. **Run twice** — once with a transport-set (rung 1), once with a binding (rung 2) | Cost of the failure: **rung 1 refused at signature check, zero dials; rung 2 refused at IDENTIFY, one dial** — then fall-through | C accepts the substitute, the failure is not recoverable, or rung 1 costs a dial |
| **4** | **The owner hosts the conversation** | A, offline throughout, later pulls one walker and republishes at rung 3. D follows **nobody** | D gets the whole thread from **A's origin alone** | requires the coordinate widening of §5 — **expected to fail today, and that is the point of running it** |
| **5** | **Redundancy is bounded** | K repliers, M walkers, `subsumes` in use | total published walk **bytes** grow `O(K+M)`, not `O(K·M)` | superlinear growth |

**1–3 are cheap and need no new spec text beyond the §2 convention.** 4 needs §5. 5 needs `subsumes`
implemented. **1–3 are also the strongest evidence available**, because they are the base case — *follow a hundred people, pull them when you go online, no
aggregator* — and nothing in this design is worth much if that does not run.

**It is a cross-impl fixture, not a thought experiment.** `http-poll` static publication is deployed
(`THE-READERS-LOOP` §4 steps 3–4), so this is buildable against three implementations rather than
argued.

---

## §7 What this does not settle

- **`PROPOSAL-THE-PUBLISHED-WALK` §7.1 — what a walk should CONTAIN — is untouched.** It remains the
  biggest open item and §5 above should be answered with it, not before it.
- **Ownerless coordinates** — discovery, redundancy and neutrality are all still fully open there
  (§4.3).
- **Whether the §2 convention is `EXTENSION-REGISTRY` text, an `APP-CONVENTION-*` rule, or SDK
  guidance.** It touches a REGISTRY type from a FEED-shaped use, and that boundary is undecided.
- **The size question.** A citer publishing a mirror of everything it cites has a cost nobody has
  measured; content addressing dedups it within a store and not across peers (`FEED` §1.2's *"honest
  scope"*), so the bill is real. **No cost model exists** — same row as the handoff's *"anything about
  real client cost at 100 follows."*
- **Whether rung 2 is worth having at all.** If `PROPOSAL-PEER-TRANSPORT-SET` lands and adoption is
  broad, the `self-certifying` fallback covers only peers who published no set — and **T9's posture in
  that proposal is that publishing one is opt-in**, so that population is not empty by construction.
  It is a judgement about how much to spend on the un-adopted tail, not a design question.
- **`[OPEN-FEED-5]` is parked on `PROPOSAL-PEER-TRANSPORT-SET`, and this does not unpark it.** The
  record is that proposal's; what is added here is one retrieval path for it. **The dependency runs
  one way and should be stated that way in any fold.**

> **A note on how this document was nearly wrong, since the correction is the useful part.** §2 was
> first written as *"the cited locator is a new convention over `system/registry/binding`"* — derived
> correctly from `REGISTRY` and `NETWORK`, and **wrong because the relevant text was in a DRAFT
> proposal rather than in `specs/`.** `PROPOSAL-PEER-TRANSPORT-SET` §1 had already dissolved the
> question and named this exact retrieval path. **The transferable lesson: *the
> answer to a design question is at least as likely to be in a proposal as in a spec* — and the
> corrected answer is strictly better: a peer-signed record needs no trust in the citer, where a
> binding needs a dial.
