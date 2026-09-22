# EXPLORATION — the interop floor: what a peer must have to participate, four candidate sets, and the structure left after peeling

**Status:** exploration. **Nothing ruled, no spec text proposed.**
Predecessors: `EXPLORATION-THE-REPLY-PACKAGE-…` · `EXPLORATION-THE-CONVERGENT-ARTICLE-…`

---

## §0 The result

1. **The corpus's own interop ladder already names this rung and declares it ungoverned.**
   `SYSTEM-ARCHITECTURE` §5's top row is ***"Secondary convergence | L5 applications | Applications may
   compose if patterns followed (**not guaranteed**)."*** **That "not guaranteed" is the whole
   question**, and this document is about what would make it guaranteed. §1.
2. **The existing tier table does not answer it, and cannot — it is a different axis.** §13.1's Tier 1
   calls eleven extensions *"irreducible — a peer fundamentally needs these to run"*, while
   **`REGISTRY` and `NETWORK` sit in Tier 2** as things *"a single peer can run without."* **Yet the
   base case needs perhaps three of the eleven and both of the Tier-2 ones.** The tiers answer *what
   does a peer need to RUN*; this asks *what does a peer need to BE SEEN*. §2.
3. **Four candidate sets, built and then peeled.** §3–§4.
4. **What survives is a structural claim, and it is smaller than any of the four candidates: the
   interop floor is a set of ENTITY TYPES, PATH CONVENTIONS and DISCOVERY STRATEGIES — not a set of
   handlers.** A conforming participant runs **no operations for anybody**. It publishes artifacts and
   reads artifacts. **Handlers are what you add to get a *live* surface, and the live surface is
   optional at every rung.** §5.
5. **That makes the floor four independent axes rather than one list** — artifact · live · private ·
   convergent — *"there may not be one single answer"* turning out to be structurally true, with a
   genuinely small common core. §6.
6. **`ENTITY-CORE-PROTOCOL` §3.5 already binds the discovery half, normatively, and the published walk
   does not satisfy it.** *"An extension introducing a new discoverable entity MUST state which
   strategy it uses."* **Strategy B — a core-general constructable path — is the answer for the
   ownerless coordinate**, reducing it to *ask anyone you already talk to, at a computable address*. §7.
   **It also determines which layer the convention must live at.** §7.2.
7. **Revision granularity is a prefix choice and nesting is already specified**, so *"revision all of
   Wikipedia"* is a thing the design lets you decline. §8.

---

## §1 The rung already exists and says "not guaranteed"

`SYSTEM-ARCHITECTURE` §5, *Multi-Level Interop* — *"Interop is not a single property. It's a ladder;
each rung gives something distinct."*

| Level | Boundary | What agreement gives you |
|---|---|---|
| Mathematical coherence | L1 primitive structure | same analytical picture |
| **Class S** | L0 + L1 category (b) | **data interchange — hashes match, frames parse, signatures verify** |
| Class T | + COMPUTE + L2 | running-program transfer |
| Developer interop | + L3 SDK facade | consistent programming experience per language |
| Pattern interop | + L4 SDK patterns | standard compositions produce identical entities |
| **Secondary convergence** | **L5 applications** | ***"applications may compose if patterns followed (not guaranteed)"*** |

**Class S is real and is achieved** — the cohort's conformance suite is exactly that rung. **The top
rung is a hope.** The goal — *participate as expected, build your own client, interoperate* — **is a
request to convert that row from *may* to *does*.**

> **So the deliverable is not a list of extensions. It is a rung definition**, and the reason the
> question has felt slippery is that it has been asked in the vocabulary of the layer below it
> (*"which extensions?"*) when its content is *which artifacts, at which paths, discoverable how.*

---

## §2 Why the tier table is the wrong instrument

`SYSTEM-ARCHITECTURE` §13.1 classifies extensions into five tiers. **Tier 1 — *"irreducible add-ons — a
peer fundamentally needs these to run… together with Tier 0 and V7, they define what a peer is"* — has
eleven members.** Tier 2 — *"needed for production multi-peer deployments; a single peer can run
without these"* — holds `IDENTITY`, `NETWORK`, `DISCOVERY`, `RELAY`, `REGISTRY`, `ROLE`, `GROUP`,
`QUORUM`, `ATTESTATION`.

**Check the base case against it** — the loop in `THE-READERS-LOOP` §4: *follow a hundred people, pull
them when you go online, no aggregator.*

| Tier 1 member | Needed to participate? |
|---|---|
| `TREE` | **yes** — the trie is the published artifact |
| `CONTENT` | **yes** for anything with bytes in it |
| `CLOCK` | **borderline** — you need *a* timestamp; `FEED` §2.3.1 already says it is the author's own unverifiable clock |
| `INBOX` | **no** — the base case pulls; no delivery is required |
| `SUBSCRIPTION` | **no** — you poll a signed root |
| `CONTINUATION` | **no** — no cross-peer dispatch in the base case |
| `COMPUTE` | **no** |
| `QUERY` | **no** — indexing replies by parent is a client-side dictionary |
| `REVISION` | **no** — not for a feed |
| `HISTORY` | **no** |
| `TYPE` | **no** — §13.1 already notes it is deferred in implementation |

| Tier 2 member | Needed to participate? |
|---|---|
| **`NETWORK`** | **yes** — §6.5 published-root and transport profiles are the publication surface |
| **`REGISTRY`** | **yes for names**, no for traversal (§3) |
| `IDENTITY` `ROLE` `GROUP` `QUORUM` `ATTESTATION` `DISCOVERY` `RELAY` | **no** |

> **Three of eleven "irreducible" ones, and two of the "you can run without" ones.** The tier table is
> **not wrong** — it answers *what does a peer need to run* — but **that is a different axis from *what
> does a peer need to be seen by other peers*, and nothing in the corpus measures the second.** Reading
> the tiers as the interop floor would put a client on the hook for eight extensions it never calls and
> leave out the two it cannot do without.

---

## §3 Four candidate sets

Built as designs to be attacked. **Each is stated as *what it can do* and
*what it must carry*.**

### Candidate A — READER

**Can:** fetch a peer's signed root, walk the trie, verify signatures, render entries and embeds,
follow the graph forward one hop at a time.
**Cannot:** be seen. Contributes nothing; nobody can cite it.
**Carries:** core (ECF, hashes, signatures) · `TREE` trie walk · `NETWORK` §6.5 published-root +
transport profile · `CONTENT` · the entry/reference/embed conventions.

**Verdict: necessary, not sufficient.** It is a **consumer of the network, not a member.** Worth naming
because it is the honest description of a web-embedded viewer, and because **it needs no writes at
all** — which is why a browser can do it.

### Candidate B — PUBLISHER

**A + publish.** Build a trie over your own namespace, sign a root, serve it, republish on change with
monotone `seq`.
**Carries:** A + the composer contract (`L-4`: one signature per entry, signed root + served closure,
republish within the freshness ceiling).
**Verdict: this is the real floor for *existence*, and it is still not participation** — you can be
read, but you cannot be *found*, because nothing points at you.

### Candidate C — PARTICIPANT

**B + the reply package.** Cite with a `reference` pin · mirror what you cite · **serve the cited
peer's reachability record.**
**Carries:** B + `REGISTRY` binding/transport-profile *types* (not its handler) + the citation
conventions.
**Verdict: this is the interop floor.** At C the graph is **traversable by strangers with no registry,
no index and no aggregator** — every citation you publish is a discovery edge for everybody
downstream. **A and B are strictly smaller and neither closes the loop.**

### Candidate D — CONTRIBUTOR

**C + publish walks.** Coverage past your own follow graph; `subsumes` to bound redundancy.
**Verdict: optional and additive by construction.** Nobody is worse off if you skip it — which is
exactly *the residual cost is reach, not infrastructure*. **Not floor; the first optional
rung.**

### Candidate E — COLLABORATOR

**D + `REVISION` core (7 ops, 4 used).** Converge on a shared mutable asset.
**Verdict: needed only for shared mutable assets.** A feed, a photo album and a storefront listing
never need it. **Not floor.**

---

## §4 Peeling — what turns out redundant or wasteful

**Wasteful in every candidate: requiring HANDLERS for anything in A–C.** A, B and C describe a peer
that **answers no operations for anybody**. It fetches static artifacts and publishes static artifacts.
`INBOX`, `SUBSCRIPTION`, `CONTINUATION` and the dispatch surface buy **latency and liveness**, not
membership — and that trade is already priced (a chunked firehose is a static object; a stream is a
per-consumer socket).

**Redundant: `REGISTRY`'s handler at the floor.** C needs REGISTRY's **entity types** — a binding, a
transport profile — and not its resolver contract, because the citer republishes bytes and the consumer
verifies them. **`REGISTRY` §5's *"anyone can publish a binding claiming any name; receiver policy
decides"* is what makes the type usable without the machinery.**

**Redundant: `QUERY` at the floor.** *"Index replies by parent"* is a dictionary in the client. QUERY's
path link index is how a **dedicated** walker does it at scale — Candidate D's tool, not C's.

**Redundant: `TYPE` at the floor**, and §13.1 says so itself (*"currently deferred in implementation
because the team is close enough to the ground to enforce manually"*).

**Not redundant and easy to miss: `CONTENT`.** The moment an entry carries an image the content store
is on the interop path, and content-addressed dedup — which the operator correctly notes *"solves
itself"* — **only solves itself if the addressing is stable.** the inline prohibition is the same
concern one layer up: **churn in the content-addressing layer is the one thing that silently destroys
the dedup property**, and it is a conformance concern rather than a design one.

---

## §5 What survives: the floor is artifacts, not extensions

> **A peer is interoperable when it publishes the right ENTITY TYPES at the right PATHS, discoverable
> by a declared strategy. It does not have to run anything for anybody.**

**Every clause of C decomposes into one of three things, and none of them is an operation:**

| | Content of the floor |
|---|---|
| **Types** | ECF-encoded entities + `system/signature` · trie nodes · `system/peer/published-root` · `system/peer/transport/*` · `system/registry/binding` · the entry / `reference` / embed vocabulary |
| **Paths** | the invariant signature pointer · the published-root slot (`manifest_url_prefix`, and `NETWORK` §6.5.4's MUST that nothing else is served there) · the citer's published prefix · a tracked-prefix declaration per `TREE` §3.3a |
| **Discovery strategies** | per `ENTITY-CORE-PROTOCOL` §3.5 — A (observational) or B (constructable path) — **declared, not assumed.** §7 |

**Three consequences worth stating separately:**

1. **The floor is implementable by a program that cannot listen.** A static site generator, a
   background job, a browser tab. *(A background passive service that wakes on a schedule and republishes
   its tree is a **conformant participant at the floor**, not a compromise.)*
2. **The floor is verifiable from outside.** Every clause is checkable by fetching a peer's origin and
   reading bytes — so a conformance check for this rung needs **no harness, no live peer and no
   handshake.** That is unusual and it is a direct consequence of the floor being artifacts.
3. **It gives the missing rung a definition rather than a hope.** *"Applications may compose if
   patterns followed"* becomes *"a peer at the artifact floor is composable by construction, because
   everything it emits is fetchable, verifiable and mergeable by union."*

---

## §6 The four axes — why "no single answer" is structurally right

**Above the floor, capability is bought on four independent axes.** Nothing forces you up any of them,
and going up one does not oblige another.

| Axis | Buys | Costs | Extensions |
|---|---|---|---|
| **Artifact** *(the floor)* | existence, citation, traversal, being mirrored | a publishing step | core · `TREE` · `CONTENT` · `NETWORK` §6.5 · `REGISTRY` types |
| **Live** | latency, notification, real-time | **an always-on surface** | dispatch · `INBOX` · `SUBSCRIPTION` · `CONTINUATION` · `RELAY` · `SIGNALING` |
| **Private** | audience control, groups | key management, revocation | `IDENTITY` · `ROLE` / `GROUP` · `QUORUM` · `ENCRYPTION` |
| **Convergent** | shared mutable assets | merge policy per path | `REVISION` core (7 ops) |
| *(Reach)* | coverage past your follow graph | walking work | `QUERY` indexes; Candidate D |

**The claim to test: these are genuinely independent.** A private feed needs Private and not Convergent.
A wiki needs Convergent and not Live. A chat needs Live and not Convergent. **A forge needs Convergent
and Reach and neither of the other two.** *If any pair turns out to be entangled, that is a finding and
it belongs here.*

> **This is the answer to *"what set of extensions"*: a floor of five things nobody can skip, plus four
> axes you buy per application.** There is no single answer, and the reason is that the *floor* is single and the *axes* are orthogonal — which is a much better shape
> than a menu, because a client author can implement the floor and be a real participant on day one.

---

## §7 The discovery half is already normative, and the walk does not satisfy it

**`ENTITY-CORE-PROTOCOL` §3.5, *Discovery locality — general principle (normative — v7.45)*:**

> *"**any entity that generic, cross-peer or cross-consumer core machinery must DISCOVER (not merely
> fetch by a hash it already holds) MUST be discoverable by one of two strategies — (A)
> observational/recorded discovery… with a fail-closed default when unresolved; or (B) a core-general
> constructable path (e.g. the invariant pointer `/{signer}/system/signature/{target_hex}`,
> constructable with no extension knowledge). It MUST NOT rely on path-construction against an
> extension-private scheme** — that is the defect class (the discoverer must know a convention it
> cannot derive; works locally and for the issuer, springs apart cross-peer/cross-consumer, and is
> typically masked by conformance tests that assert the divergent convention)."*
>
> *"An extension introducing a new discoverable entity **MUST state which strategy it uses**."*

**`PROPOSAL-THE-PUBLISHED-WALK` introduces a new discoverable entity and states neither strategy.**
That is a checkable defect against landed normative core text, and **it is the same gap the handoff has
been carrying as §3.2 "discovery"** — restated in the core protocol's own vocabulary, with a rule
attached, three weeks before anybody asked the question.

### §7.1 Strategy B dissolves the ownerless coordinate

**The ownerless coordinate is the hardest open item** — no owner, so no derivable location for a
gathered set. **Strategy B is precisely a derivable location for a thing whose owner you do not
know.**

Publish a walk at a path constructed from the coordinate — the shape the invariant pointer already
uses, `/{walker}/…/{hex(coordinate)}` — and:

- **Knowing only the coordinate, you can ask ANY peer whether they walked it.** No index, no registry,
  no aggregator, no *"who covers this topic."*
- **The peers you can ask are the ones you already reached** — your follow set, plus everyone their
  citations introduced you to (§3).
- **It composes with union**: ask five, union five answers, each independently verifiable.

> **That is the tracker's job done with no tracker** — *coordinate → who*, obtained by construction
> instead of by lookup. **The invariant-pointer pattern is normative and generalizes past signatures by
> design.**

**Not proposed here, and two things must be settled first:** *(a)* what `hex(coordinate)` is when the
coordinate is a **name** rather than a hash — a normalization rule, and normalization is where this
class of design usually goes wrong; *(b)* whether a per-coordinate path is acceptable at volume, or
whether the walker publishes an index the path points into. **Both are answerable; neither is
answered.**

### §7.2 And this is the technical reason the tier ruling was right

**§3.5 forbids path-construction against an *extension-private* scheme.** So a walk's path convention
**cannot** live in a document only some clients read — that is definitionally the defect class. It must
sit at a layer **every participant knows**.

> **Which is exactly *"it may reach the level of normative application extensions… participating in the
> network as expected, so anyone can build their own client and interoperate.***
> **The layering question and the discovery question are the same question**, and §3.5 answers it: a
> convention that clients must construct paths against belongs at the interoperability floor, or it
> does not work.

---

## §8 Revision granularity — a prefix choice, and nesting is specified

**Correct, and the design already lets you decline it.**

- **`REVISION` §1.2: *prefix-scoped versioning — "you version what matters."*** Versions cover a
  subtree, not the tree.
- **§8.7.2 tabulates the scopes** — own namespace, single foreign namespace, domain subtree, full tree
  — and warns that full-tree versioning *"is appropriate for milestone snapshots but not recommended
  for continuous auto-versioning."*
- **§8.2 is repositories-of-repositories, already specified:** *"if `project/` is versioned and
  `project/frontend/` is independently versioned, their version DAGs are independent. A commit to
  `project/frontend/` does NOT automatically create a version for `project/`… the parent's trie is only
  captured when someone explicitly commits at `project/`."*

> **So an encyclopedia is article-per-prefix with independent DAGs, and an encyclopedia-level version
> is a deliberate milestone that costs one commit when somebody wants one.** Nothing accumulates per
> edit at the parent. **And big repositories still work** — Linux-scale is the *domain subtree* row,
> which is the ordinary case, helped by §8.7.2's note that trie snapshot cost is
> `O(changed_paths × depth)` **regardless of prefix size.**

**What is missing is not mechanism, it is convention:** *what is an article's prefix, how does a topic
name become one, and what is the per-path merge config.* **That is `APP-CONVENTION-*` work and it is
the same document §7.1's normalization rule belongs in.**

---

## §9 The registry, refined — cached vs asserted

**The corpus carries this distinction already, and it is carried by the signature rather than by
location.** `REGISTRY` §3: every non-`self-certifying`, non-`local-name` binding **MUST** carry an
`issuer_signature` whose `data.signer == issuer_peer_id`, plus an optional `issuer_attestation`.

- **Cached** — Alice serves a binding signed by **the original registry**. Alice is a *transport*. Its
  authority is the issuer's, unchanged, and **Alice cannot alter it without breaking the signature.**
- **Asserted** — Alice serves a binding signed by **Alice**. Its authority is Alice's, and a consumer
  weighs it by whatever policy it has for Alice.

**So the two are already distinguishable by any consumer, mechanically, with no new field** — and
§3's step 2a is the guard that keeps a cached one honest (*"checking `binding.name` against the name it
was located under… a signature proves who issued a binding, never what it was issued for"*). **What is
missing is only that a client should SHOW the difference**, which is a UI/SDK concern and belongs with
the tier-ruling convention.

---

## §10 What this leaves open

- **§7.1's two questions** — name normalization into a coordinate, and per-coordinate path volume.
  **These are now the critical path**, because they are what converts *the hardest open item* into a
  mechanism.
- **The floor as a checkable artifact.** §5's claim that this rung is verifiable by fetching an origin
  suggests a **conformance profile with no harness** — worth a proposal, and it would be the first
  conformance surface in the ecosystem that needs no live peer.
- **Whether the four axes are truly independent** (§6). Stated as a claim to attack, not a result.
- **Private groups** are covered by `EXTENSION-ENCRYPTION` §8's group key-wrap mode and are not an
  open question at this layer; the open part is rekey granularity, which is a policy choice.
- **Nothing here is a proposal**, and the one that should be written next is §7.1's, because it unblocks
  §8's convention and §5's profile at once.
