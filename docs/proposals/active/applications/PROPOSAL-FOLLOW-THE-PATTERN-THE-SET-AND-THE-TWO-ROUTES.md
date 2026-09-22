# PROPOSAL — follow: the pattern, the set, and the two routes

**Status:** DRAFT 2026-09-04 — **first pass, and it is a stress test rather than a sign-off candidate.** The
purpose of writing it out is to find the gaps that a reduction could not; §6 is the substance and
several of its results are unresolved on purpose. **Not ready to rule. Not ready to implement.**

**Tier:** applications — with **one** candidate extension delta (§7 D2, the cursor site) that would
re-tier if it lands.

> **What this proposes, in one sentence.** That "following" a peer is a *reader-driven* pattern the
> substrate already supports by two independent routes, that the missing pieces are a durable
> **set**, a declared **cursor**, and a **refresh loop** — and that no new extension is required to
> get there.

**Rests on:** `ANALYSIS-REDUCING-FOLLOW-THE-PATTERN-THE-DATA-AND-THE-TWO-ROUTES` (the reduction) ·
`EXPLORATION-THE-OPERATING-MODELS-AND-THE-ALWAYS-ON-TIER` (why pull is the primary model) ·
`REFERENCE-PRIOR-ART-FEDERATED-AND-P2P-PUBLISHING-SYSTEMS-BY-AXIS` (the comparators).

---

## §1 The governing principle

**The publisher does not know who is following.**

Everything worth having in this design follows from that one property: it is why an intermittent
reader loses nothing, why a publisher's cost does not grow with popularity, why no fan-out tier is
required, and why the pattern degrades to a CDN under load. It is also the single property most
easily lost — the moment a publisher must maintain a follower collection, delivery becomes
`O(followers)` per post and the design has become a different one.

**Every requirement below is subordinate to keeping that property.**

---

## §2 What the substrate already provides — and the dependency question, answered

A design audit traced each part of the pattern to its defining section. **The result is that
following requires `EXTENSION-TREE` + `EXTENSION-NETWORK` and the core protocol. It does not require
`EXTENSION-REVISION`.**

`system/peer/published-root` (`EXTENSION-TREE` §3.3a) carries, in one signed entity:

| Field | Role in following |
|---|---|
| `peer_id` | the durable source reference |
| `root_hash` | the delta-fetch entry point |
| `prefix` | makes the trie keys interpretable |
| **`seq`** | **the change check** — MUST increase per republish — **and the rollback defense** |
| `predecessor` | the prior root's content hash: a history chain |

**Where REVISION *is* required, and correctly so:** `revision:fetch-diff` (§4's Route A) and version-
DAG convergence via `revision:merge`. Both are genuine REVISION concerns. Neither is a requirement of
the *pattern* — only of one route through it.

**This proposal does not weaken that.** It states the dependency explicitly so implementers building
a minimal follower are not led to install an extension they do not need.

---

## §3 The records

### §3.1 `system/follow/entry` — one followed source *(application data)*

```
system/follow/entry := {
  fields: {
    target_peer_id: {type_ref: "system/peer-id"}   ; the followed peer (pubkey IS identity)
    prefix:         {type_ref: "system/tree/path"} ; what region of their tree is followed
    since:          {type_ref: "primitive/int"}     ; ms epoch, when following began
    label:          {type_ref: "primitive/string", optional: true}  ; display only
  }
}
```

**`prefix` is on the entry, not assumed**, because a published root declares its own `prefix` and a
follower may care about a narrower region than the publisher publishes.

### §3.2 `system/follow/cursor` — what I last accepted *(the candidate substrate delta)*

```
system/follow/cursor := {
  fields: {
    target_peer_id: {type_ref: "system/peer-id"}
    last_seq:       {type_ref: "primitive/int"}    ; highest accepted seq for this peer
    last_root_hash: {type_ref: "system/hash"}      ; BARE — the root that seq committed to
    checked_at:     {type_ref: "primitive/int"}    ; ms epoch, last successful check
  }
}
```

**Kept separate from the entry deliberately** — see §8 Q2, which is genuinely open. Separation lets
one source be followed by several consumers at different positions and keeps display data out of
the state a correctness rule reads.

**This is the one item with a substrate-shaped claim on it.** `seq` monotonicity is already a
normative consumer MUST; the value it is compared against has no declared home, so every consumer
invents one.

### §3.3 What is deliberately *not* a record

Timeline, ranking, mute/block/boost, reciprocity, notification preference, grouping. **All
application data with no interoperability requirement.** Naming them here so the boundary is
explicit rather than implied.

---

## §4 The two routes, and when each applies

| | **Route A — live publisher** | **Route B — static origin** |
|---|---|---|
| Change check | dispatch to the publisher | fetch `published-root`, compare `seq` |
| Delta | `revision:fetch-diff` computes it | walk the trie; dedup skips what is held |
| Work done by | **publisher** | **reader** |
| Publisher must be live | **yes** | **no** |
| Capability needed | **yes** — `revision:fetch-diff` on the prefix | **no** — public origin |
| Requires REVISION | **yes** | **no** |

**Route B gets its delta for free from content addressing.** There is no diff operation and no
negotiation: unchanged subtrees are already in the reader's content store and are skipped by the
walk.

**Proposed guidance `[SHOULD]`:** a follower **SHOULD** prefer Route B for any source it does not
have a live session with, and **MUST NOT** treat Route A's unavailability as the source being
unavailable. Route A is an optimization for peers already in session, not the base case.

---

## §5 The refresh loop

1. For each entry, fetch the head pointer at `{target_peer_id}/system/peer/published-root`.
2. Verify the signature against `target_peer_id`. **Reject `seq` lower than `cursor.last_seq`.**
3. If `seq` unchanged → done. **One request, no walk.**
4. If `seq` increased → walk from `root_hash`; fetch only nodes not held; advance the cursor.
5. On failure → back off; **do not advance the cursor**; **do not render the source as unchanged**
   (§6 T2).

**Adaptive cadence `[SHOULD]`.** Poll interval scales with observed change rate: a source that has
not moved in a year is checked on a long interval, not the default one. **This is reader-side only
and needs no protocol.** Without it, following N sources costs `N × (1/interval)` requests
regardless of how often anything changes — see T1.

---

## §6 Stress tests

**The point of this document. Each is a case the reduction did not cover; results are honest,
including the three that are unresolved.**

### T1 — the once-a-year publisher. **Partially resolved.**
1,000 follows at hourly polling is ~24,000 requests/day to learn nothing. Adaptive backoff (§5)
reduces it by orders of magnitude but never to zero, and the floor is set by how stale a reader will
tolerate being. **An untrusted change-feed aggregator collapses N requests to 1** and needs no trust,
because it reports *what to check* while the signed root reports *the truth* — it can waste time or
omit, never deceive. **Not specified here.** Named as the intended answer and left to its own
proposal.

### T2 — the withholding origin. **UNRESOLVED, and it is the most important result.**
`EXTENSION-TREE` §3.3a is explicit: *"a publisher that has not republished and an origin withholding
a newer root produce byte-identical results at the consumer — same signature, same `seq`, same
`published_at`."*

So **"Alice has not posted in a year" and "this origin is withholding Alice's newer root" are
indistinguishable**, and the naive follow UI renders the first as a fact.

**Proposed constraint `[MUST]`:** a follower MUST NOT present absence of change as evidence the
source has not changed. It may report *"no change observed as of T, from this origin."*

**What is unresolved:** whether anything better is achievable. Fetching the same root from a second
origin would detect divergence — but that requires knowing a second origin, which is a reachability
question this proposal does not own. **Flagged rather than solved.**

### T3 — the followed publisher re-keys. **STILL UNRESOLVED, but it is not unstudied — see §6a.**
`peer_id` *is* the public key. A publisher that re-keys is, to the protocol, **a different peer**. A
follow entry naming the old id is not stale — it is correct about a peer that has stopped
publishing, forever, with no signal distinguishing that from T2.

**This is not hypothetical**: a domain re-keyed off a compromised seed produces exactly this, and
every follower is silently pointed at an abandoned identity.

**What the design does not have:** any way for a publisher to say *"my new identity is X."* A
successor pointer signed by the **old** key is the obvious shape and is exactly what a compromised
key must not be able to assert.

**The prior work says where the answer lives and it is not here — §6a.** This proposal does not
attempt it, and the reason is now stronger than "out of scope": the answer is a **registry binding
kind**, not a follow-list field.

### T10 — "I only care about one subtree." **UNRESOLVED, and it is a substrate-shaped finding.**

A publisher who publishes sites, social posts and files publishes them under **one** root: the head
pointer is *"bound at that path and nowhere else"*, and there is deliberately **one `seq` stream per
peer** — forking it is what destroys rollback detection.

So **any** change bumps `seq` and changes `root_hash`, for every follower of every part of that
tree. A follower who only tracks social posts is woken by every unrelated site edit.

**That much is a cost problem. The sharper half is that the follower cannot cheaply find out whether
it cares.** The trie is a HAMT routed by `SHA-256(relative_key)`, and `EXTENSION-TREE` §3.3 states
the consequence directly: *"the trie structure is determined by hash bits, not by path-segment
locality."* The same section says the division of labor is deliberate — *"the trie's role is
content-addressed cross-peer convergence; the LocationIndex's role is path-keyed prefix scan."*

**The LocationIndex is local. It is not a published artifact.** So a *remote* follower has:

- **no subtree to check** — bindings under `social/` are scattered across the trie by hash;
- **no per-prefix hash** to compare — obtaining one means building a fresh HAMT over the filtered
  set, which requires already holding the bindings;
- **no way to ask the origin** *"what changed under prefix X"* — the `http-poll` routes are
  `TREE_GET` by path and `CONTENT_GET` by hash, neither of which answers a prefix-scoped
  change query.

**So "did anything I care about change?" is not answerable more cheaply than "what changed at all?"**
The walk is still efficient in bytes — dedup means unchanged nodes are not re-fetched — but it is
**whole-root in scope**, and a follower cannot scope its attention below the peer.

**Three candidate directions, none evaluated, and this proposal does not pick one:**

1. **Publish narrower.** A publisher re-roots at a narrower `prefix` — but there is one head-pointer
   path per peer, so a peer cannot publish two differently-scoped roots today. Would need a
   per-prefix published root, which re-opens the one-`seq`-stream rule that exists for good reason.
2. **A published prefix-index** — a signed, path-keyed digest per region of interest, i.e. the
   LocationIndex's job made publishable. Additive; costs the publisher a second artifact.
3. **Accept it.** Whole-root change detection with byte-efficient walking may simply be adequate,
   and the honest answer may be that the cost is a fetch of `O(depth)` interior nodes per refresh.

**This is the question most likely to change the design**, and it is squarely a substrate question
rather than an application one.

### T11 — the publisher adopts `EXTENSION-IDENTITY` and the follower does not. **UNRESOLVED.**

A capability that only one side of a peer boundary implements is a divergence with a delay. If
rotation continuity lives in an extension the *consumer* may not have, then a publisher who rotates
correctly is still invisible to a follower who did not adopt it — that follower sees T3.

**The corpus has already met this exact shape one layer down**, in the parked key-death work: a
single support flag with no general feature-negotiation surface, where *"two peers both advertising
`true`, one implementing a new marker family and one not, diverge while claiming the same tier."*

**The open question is not "should we adopt IDENTITY" — it is what a follower without it is entitled
to assume**, and whether the honest degraded mode (manual re-mapping on notice, as an operator
would do today) can be stated normatively rather than left to each implementer.

### T4 — the reader offline for three weeks. **Resolved, and it is the design's best case.**
Nothing breaks. The origin served the whole time; the cursor is durable; the walk fetches only what
changed. **No retroactive delivery problem exists because there was no delivery.** This is the case
that motivated the pull model and it costs nothing.

### T5 — following a stranger. **Resolved, and it constrains §4.**
Route A needs `revision:fetch-diff` capability **on the publisher's prefix** — a stranger has no
grant, so cannot use it. **Public following is Route B only.** This is a stronger argument for
Route B as the base case than the liveness argument, and it was not visible before writing §4's
table.

### T6 — the publisher restarts at `seq: 0`. **UNRESOLVED.**
A publisher that loses state and republishes from zero is **permanently unfollowable** by every
existing follower: the consumer MUST reject `seq` lower than one already accepted, and that rule is
the rollback defense — it is working as designed.

**No recovery path exists**, and this is an operational hazard for anyone who restores a publisher
from backup. Options — a signed `seq`-reset carrying the predecessor chain; consumer-side manual
re-trust; treating it as T3 — are all unattractive for different reasons. **Named, not solved.**

### T7 — the origin 404s. **Partially resolved.**
Distinguishable from "no change" (a failed fetch is not a `seq` comparison) but **not**
distinguishable from network failure, censorship, or the publisher's origin lapsing. §5 step 5's
back-off is the mechanism; the reporting vocabulary is undefined.

### T8 — the head-pointer path. **Verify, do not assume.**
The pointer path is pinned `[MUST, v4.3]` with **no peer-id segment**, after three implementations
diverged and one produced a path that fails at hop 0 for a consumer reading the pointer as a file.
**This is the exact path a follower fetches on every refresh.** Convergence across implementations
is a precondition of any follow implementation and is a measurement, not an assumption.

### T9 — the negative answer. **Resolved by existing normative text.**
`not_found` against a published root means *"this root does not bind K"* — never *"the publisher does
not bind K"*, and `prefix` is a bound rather than a completeness claim. **A follower carries both
limits and must not present a negative as authoritative absence.**

---

## §6a T3's prior art — what is already established, so it is not re-derived

**T3 was studied before this proposal, and the prior work is more advanced than "unresolved"
suggests.** Recorded here because the most expensive thing a future pass could do is re-derive it.

**1. The property we lack, named exactly.** `did:plc`'s advantage was never its rotation machinery —
it is that **the identifier is a hash of the genesis record, so keys rotate underneath a stable
name.** Ours *is* the key, so **no amount of rotation machinery produces that property.** T3 is a
consequence of that choice, not a gap in the follow design.

**2. Where the fix belongs is already ruled, and it is not core.** The core protocol places
cross-form correlation in the identity/registry/policy layers, *"never the address or capability
layer of core."* **So a stable-across-rotation identifier is a REGISTRY question**, and T3's answer
is a **binding kind**, not a follow-list field. This is why §9 declines it rather than deferring it.

**3. The shape is known and is cheap, with one conflict already found.** A genesis-hash identifier
form was proposed as a registry name; `EXTENSION-REGISTRY` has already landed the opposite for
`self-certifying`, where `name == target_peer_id` **and explicitly not a hash**. That is a shipped
v1 pin, so the genesis-hash form **cannot redefine it** — but REGISTRY's forward-compatibility rule
requires an unknown binding `kind` to be ignored with a warning while remaining valid on the wire,
so it lands as **a new kind beside `self-certifying`**: old peers skip it, new peers resolve it, no
wire break.

**4. A related defect is established by trace and is independently written up.** Rotation authority
and issuance authority are partitioned by *operator intent* — "the old key is still available" vs
"unavailable" — and **that partition is unverifiable**: the two events are identical on the wire, so
the stricter path gates the honest user while an attacker takes the cheap one. Its proposed rule —
**rotation MUST NOT require less authority than issuance** — is carried in its own extension-scoped
proposal, and a **redeemed pre-rotation commitment** substituting for the issuer's signature is what
keeps that rule adoptable. That is also the first argument making pre-rotation *needed* rather than
merely useful.

**5. The finding that matters most to this proposal, and it is uncomfortable.** A publisher whose
genesis is a **bare keypair with no attestation has no rotation path under any design in that
work.** Publishing paths that load-or-generate a keypair and mint no attestation therefore produce
peers for whom T3 is not merely unresolved but **structurally unavailable** — a hard re-key is the
only option they ever had.

**Two consequences for sequencing, both outside this proposal's scope and neither of them ours to
decide:**

- **This item has a real deadline, and it is the baseline argument.** Publishing freezes what
  consumers pin. Every follower created before a stable-identifier binding kind exists will hold a
  raw `peer_id` and will experience T3.
- **Whether publishers should be minting an attestation at genesis today** is a question for the
  identity thread, not this one — but a follow design that assumes rotation continuity will exist
  should not be written before that is answered.

---

## §6b Audience — what a publisher exposes, and the one thing the pull route cannot do

**Two concerns are commonly raised here and they have different answers: *"am I forced to publish my
whole tree?"* (no, and the design already handles it well) and *"is everything therefore public?"*
(on the static route, effectively yes — and that is structural, not an oversight).**

### §6b.1 Publishing is not exposing the tree. Two independent mechanisms already say so.

**1. A published root is a *re-rooted* trie, not a filtered view of the real one.** Extraction
rebuilds — `root_hash = build_trie(bindings)` over the selected set — so publishing a subset is
**re-rooting, not filtering**. Unpublished bindings are not present as absent siblings, do not
appear in the structure, and leak nothing: there is no trace of them to observe. Capability tokens,
settings and correspondence are simply **not in the artifact**.

**2. `serve_scope` is a literal capability token, evaluated by the same evaluator as the live
surface.** `EXTENSION-NETWORK` §6.5.6 is explicit — the published `serve_scope.cap` **is** the
effective cap handed to `check_permission`, so *"where the live surface asks 'does the connection's
cap-set permit `get(path)`?', the serving surface asks 'does `serve_scope.cap` permit
`get(path)`?' — one ACL machinery, no drift."*

**So the answer to "what I publish is distinct from my tree" is: yes, by construction, twice over.**
A follower sees a purpose-built artifact, not a window into the publisher.

**What this proposal adds:** nothing. It **carries** the property, and §3.1's `prefix` on the follow
entry is the follower-side half — a follower tracks a region, and the publisher independently
decides what that region contains.

### §6b.2 On the static route, audience control is cryptographic — it cannot be capability-gated

**This is the structural finding and it should be stated plainly rather than discovered later.**

`EXTENSION-NETWORK` §6.5.6: *"the read routes carry **no request auth** (the client may not speak
the protocol) — hash-knowledge (content) / path-presence (tree) is the read authority."*

A static origin cannot authenticate a requester. **`serve_scope` bounds what the origin will answer
for *at all*, uniformly, for every caller** — it is a publication boundary, not a per-recipient
ACL. So on Route B there is exactly one lever that distinguishes recipients: **whether they can
decrypt what they fetched.**

**This maps cleanly onto the two routes and sharpens both:**

| Audience | Mechanism | Route |
|---|---|---|
| **Public broadcast** | plaintext in the published tree | **B** — works today |
| **Closed audience** | **encrypted payload**; audience = who holds the key | **B** |
| **Closed audience, live** | capability check at dispatch | **A** — but §6 T5: needs a grant, so not for strangers |
| **Bilateral private** | encrypted payload via a store-and-poll intermediary | **B** over a relay namespace |

**The last row is already a shipped shape, and the analogy is exact.** Store-and-poll is *"pull-shaped,
persistent, single-author per envelope, addressed-by-namespace"*, and a static-CDN-hosted peer's tree
**is** such a namespace. An intermediary that stores encrypted envelopes it cannot read, for a peer
that wakes later and drains them, **is encrypted SMTP** — and it needs no new primitive.

### §6b.3 Revocation, stated honestly

**Bytes already fetched cannot be recalled**, and no design in this corpus pretends otherwise —
content addressing makes republication trivial for anyone who holds the bytes.

**What is achievable is exclusion from future publication**, and on Route B that means **rotating the
content key**: a removed member keeps what they have and can decrypt nothing published after the
rotation. Whether that rotation is per-post, per-epoch, or per-membership-change is a real design
question with real cost, and **this proposal does not answer it.**

**Known adjacent gap, flagged rather than restated:** group-as-audience via minted per-member tokens
has a recorded defect — on the request-initiated path the granter does not hold the token hash that
revocation is keyed by, so the revoke verb is unreachable for the party the ruling assigns it to. It
is reachable on the handshake path, where the delivered capability is written to a per-peer session
record. **A group-audience design for following must state which path it assumes.**

### §6b.4 Three models, kept separate

Conflating these is how a design ends up serving none of them:

1. **Public feed** — no audience management. The 100M-follower case; degenerates to a CDN.
2. **Closed audience** — a set that changes over time; needs encryption on the pull route, key
   rotation for exclusion, and a group mechanism for membership.
3. **Bilateral private** — two peers over an untrusted store. Encrypted store-and-poll.

**This proposal specifies the follow *pattern*, which is common to all three** — the reader-held
cursor, the change check and the delta fetch are identical whether the payload is public or opaque.
**What differs is only the audience mechanism**, and (2) and (3) are out of scope here.

**The open question that decides how much is out of scope: is a private follow the same mechanism
with an encrypted payload, or a different mechanism?** The evidence above says the same — the loop
never inspects content — but that has not been tested against a real group-membership change, which
is where it would break if it breaks.

---

## §7 Deltas

| # | Delta | Where | Kind |
|---|---|---|---|
| **D1** | `system/follow/entry` + the follow set | new application convention | additive |
| **D2** | `system/follow/cursor` — **the declared cursor site** | **candidate substrate delta** — see §8 Q1 | additive |
| **D3** | The refresh loop + adaptive cadence `[SHOULD]` | application convention | additive |
| **D4** | Route guidance: prefer Route B; Route A is an in-session optimization | `guides/` — resolution/serving guidance | clarifying |
| **D5** | `[MUST]` — a follower MUST NOT present absence of change as evidence of no change (T2) | application convention, carrying TREE §3.3a | additive |
| **D6** | State the extension dependency explicitly: following needs TREE + NETWORK + core, **not** REVISION | `guides/` | clarifying |

**No delta modifies `EXTENSION-REVISION`, `EXTENSION-SUBSCRIPTION`, or the wire core.**

---

## §8 Open questions

1. **Does D2 belong in a spec at all, or is a convention sufficient?** The cursor is purely
   reader-local — nothing crosses a peer boundary — which argues convention. Against: `seq`
   monotonicity is a normative MUST and the state it reads has no home, which is how divergence
   starts. **Most decision-relevant question here.**
2. **Cursor bundled into the entry, or separate?** §3.2 separates them; bundling is simpler and
   forecloses multiple consumers at different positions.
3. **Is there a second, non-social consumer of this exact loop?** A change-feed aggregator following
   many sources runs the identical mechanism at larger N; so, plausibly, do package-update tracking
   and configuration pull. **If two of those fit without contortion, the pattern is a mechanism and
   D2 belongs in a spec. If they need contortion, it is a convention.** This is the test that should
   settle Q1, and it is answerable with evidence rather than taste.
4. **What vocabulary should T7's failure states use?** Undefined, and a follower needs it to say
   anything honest.
5. **Do T3 and T6 belong to this proposal at all?** **Answered for T3 by §6a: no.** Its home is a
   registry binding kind, the placement is already ruled, and the shape is known. **T6 remains
   genuinely unowned** — a publisher restored from backup is neither a rotation nor a compromise,
   and no existing thread covers it.
6. **T10 — can a follower scope its attention below the peer?** Today it cannot: one root, one
   `seq`, and a hash-keyed trie with no path locality. **This is the question most likely to change
   the design**, it is a substrate question, and the three candidate directions are unevaluated.
7. **T11 — what is a follower without `EXTENSION-IDENTITY` entitled to assume?** The honest degraded
   mode is manual re-mapping on notice. Whether that can be stated normatively, or whether rotation
   continuity must be legible to consumers who did not adopt the extension, is undecided.

---

## §9 What this proposal is NOT doing

- **Not proposing a new extension.** §2 removes the argument that one is needed.
- **Not specifying an aggregator, gossip, or a DHT.** T1 names the aggregator as the intended answer
  to the cost problem; specifying it is separate work.
- **Not touching the version DAG, merge, or conflict handling.**
- **Not defining a timeline, ranking, or any social verb.**
- **Not solving identity rotation** (T3) or publisher state loss (T6).
- **Not claiming the design is ready.** T2, T3 and T6 are unresolved and two of them are structural.

---

## §10 Validation needed before this is ruled

1. **T8 measured** — do implementations agree on the head-pointer path today?
2. **A follow loop run against real origins**, including one deliberately stale and one 404ing.
3. **A cost measurement** for T1 at realistic N. No capacity figure exists for any part of this
   system; this would be the first.
4. **Q3 tested** — attempt the loop for a non-social consumer and report whether it fits.
5. **Cohort review of T2 and T6**, which are the two places where a reasonable implementer could
   ship something honest-looking and wrong.
6. **T10 measured on a real tree** — for a publisher carrying several unrelated regions, what does a
   refresh actually cost when the region the follower cares about did not change? That number
   decides between T10's three directions and is obtainable today.
7. **Whether any live publisher has an attestation at genesis** (§6a.5). If none do, T3 is not
   merely open for future peers — it is already foreclosed for every peer now publishing, and that
   changes how urgent the binding-kind work is.
