# PROPOSAL — the locator table is a registry backend plus a substitute delta, and nothing else is new

**Proposes:** a **content-locator mechanism** — *given a content hash or a namespace, which peers will
serve it* — assembled from machinery that already exists. **One new backend, three normative deltas to
a landed extension, and one new entity type** — plus **two MUSTs the aggregation layer already carries**
`[rev 2]`. **Authorization needs nothing new** `[rev 3]`. Everything else is composition.

⚠ **The title is narrower than the design** `[rev 3]`: the subject is a **predicate**, and a content hash
is one of several. See §1.3 — the name is kept because the document is cited under it.

**Status:** **PROVISIONAL DRAFT 2026-09-13 · revision 4.** ⛔ **NOT ratifiable and not routable yet** —
but nothing blocks writing normative text. **Revision 4 does the grant CALL BY CALL (§6b), which is the
only way any of it reads, and corrects a conflation revision 3 committed.**

> ⛔ **`[rev 4]` Revision 3 wrote `peers: {include: ["/*/*"]}` — a RESOURCE path pattern in a
> PEER-IDENTITY field.** `resources` is a `path-scope`; `peers` is an `id-scope` whose members are peer
> identifiers. **Corrected in §6a.3; the distinction and its deliberate overlap are §6b.1; and two
> places the CORE SPEC blurs the same thing are §6b.4.**
>
> ⭐⭐ **`[rev 4]` And the framing that reorders the whole document: the traversal does not happen in
> hash space (§1.4).** *You step out of the content-address space into the semantic space, navigate the
> connected graph, and translate back down.* **The hash is a terminal, not a coordinate** — which is why
> *"you cannot route on a hash prefix"* was never an obstacle, why per-namespace is the default for a
> structural reason rather than a cheap one, and why a referral terminates.

> ⭐⭐ **`[rev 3]` Two corrections, and the second is larger than this document.**
> **① The capability grant is EXPRESSIBLE TODAY (§6a).** Revision 2's top blocker read one extension's
> summary of its own narrowing and concluded the capability system could not say *"consult a peer a
> table named."* **The core `grant-entry` has a `peers` dimension whose stated purpose is exactly
> outbound sub-dispatch at a foreign namespace, MUST-evaluated, fail-closed by default.** What survives
> is a smaller real defect: that extension **shadows** the core dimension (§6a.5).
> **② Whether to act on a claim about a third party is POLICY and belongs to the deployment (§6a.3).**
> The same grant carries enumerated devices for one person's fleet, any-peer-on-the-group's-say-so for
> an application network, and any-peer for a public archive. **A design that picks one has taken a
> decision that is not its to take.**
> **③ The subject is a PREDICATE, not a hash (§1.3).** *Asking "where do I get this hash" was a probe;
> what it revealed is the structure of the exchange, and that structure is the deliverable.*
>
> **Revision 2 folded an adversarial audit** (`REVIEW-THE-LOCATOR-ASSEMBLY-AUDITED-AGAINST-LANDED-TEXT`).
> Its three findings against the aggregation layer and the abort rule **all stand**; only its
> capability finding inverted. Changes are marked `[rev 2]` / `[rev 3]`.

**Depends (read):** `EXTENSION-REGISTRY.md` (v1.26) §1, §2.1, §4.1, §4.1.1, §8.2, §8.3, §12 ·
`EXTENSION-SUBSTITUTE.md` (v1.3) §1, §2.1, §3, §4, §5 · `EXTENSION-CONTENT.md` §6.4.2 ·
`SYSTEM-DATA-EXCHANGE.md` (v0.2) §1.1, §2.1, §2.2, §2.3, §2.5 · `SYSTEM-ARCHITECTURE.md` §13.1 ·
`EXTENSION-NETWORK.md` §6.5.6 · `[rev 3]` **`ENTITY-CORE-PROTOCOL.md` §3.5 (dispatch classes and where
Dimension 4 is evaluated), §3.6 (`grant-entry`), §5.2, §5.4, §5.6**

---

## §0 Summary

> ⭐⭐⭐ **The content lookup is a registry backend keyed on a hash instead of a name, consumed through
> the substituter's existing miss path, carried as ordinary published data, and merged by the reader.**

**The one sentence that decides the whole design:** *trust decides where you look, verification decides
what you accept, and only the first is transitive.* A locator entry is a **hint**. A hint that is wrong
costs one wasted fetch and **can never become bad data**, because `EXTENSION-SUBSTITUTE` §3 already
hash-verifies every byte it accepts and discards a mismatch. **That is why this mechanism needs almost
no trust machinery, and why its real risk is abuse rather than compromise.**

**The layering answer, which was the question asked:**

| layer | what happens | where it lives | new? |
|---|---|---|---|
| **data** | a locator table is an ordinary signed entity in the issuer's own namespace | the issuer's tree | ⛔ **nothing new** |
| **aggregation** `[rev 2]` | an aggregator **carries** many parties' claims **byte-identically**; the union is the **reader's view** over them, never a new merged object | `SYSTEM-DATA-EXCHANGE` §1.1's closure — ***the fixed point is `gathered → gathered`*** | ⚠ **free mechanism, two inherited MUSTs** — §2.5 paging and §2.1 byte preservation. **See §3.5** |
| **transport / currency** | the merged view is kept current by the copy-current loop | the data layer | ⛔ **nothing new** |
| **resolution** | `hash \| namespace → [peer]` | ⭐ **a new REGISTRY backend** — §1: additional backends *"compose on the substrate and ship in their own proposals"* | ⭐ **the new artifact** |
| **consumption** `[rev 2]` | on a local content miss, turn a bare hash into candidate sources and run the chain that already exists | ⚠ **`EXTENSION-SUBSTITUTE` §3 step 3, §3.1 and §4** | ⚠ **three deltas** |

⇒ **It is not a new extension in the sense of a new subsystem. It is one backend, three clauses, and a
type — plus two obligations the aggregation layer already carries and this design must honour.**

✅ **`[rev 3]` And a sixth row that turned out to need nothing: AUTHORIZATION.** The grant this mechanism
needs is expressible in the existing `grant-entry` — **`peers` is the dimension, it is MUST-evaluated
before an outbound sub-dispatch leaves the peer, and it is fail-closed when absent.** §6a.

⚠ **`[rev 3]` And a scope correction that is bigger than this document: the subject is a PREDICATE, not
a hash.** A hash, a namespace, a name, a peer, a destination, a coordinate, **a query** — same shape,
different predicate. §1.3.

⭐⭐ **`[rev 4]` Where the work happens, because it decides what the mechanism IS:** ***the traversal is
in the SEMANTIC space over the connected graph; the content-address space is where verification happens
and nothing is traversed there.*** The hash is a terminal reached by `tree:get` at a computable path
once navigation is over. §1.4.

⭐⭐ **`[rev 4]` And the authorization question is smaller than it looked: of the five calls in the
flow, FOUR ARE LOCAL and exactly one is an outbound sub-dispatch** — governed by the one dimension that
exists for that class, checked before the request leaves. §6b.2.

---

## §1 ⭐⭐ The mechanism uses the system to answer a question about the system

**This is the structural observation, and it is the reason four of the five layers need nothing.**

A locator table is **content**, named through the **tree**, kept current by the **data layer**,
resolved by the **registry**, and consumed by the **substituter**. **Every layer of the system,
applied to a question about the system's own bytes.** The mechanism is not beside the architecture; it
is the architecture pointed at itself.

### 1.1 The recursion terminates, and here is where

⚠ **The obvious objection: if the locator table is content, does finding *it* require a locator table?**

**No, and the termination is structural rather than a special case.** A locator table is fetched
**by coordinate from a peer you already have a relationship with** — a name, a configured source, a
peer whose root you already follow — **never by bare hash.** It is the same bootstrap the registry
already has and the same one the substituter already has: *an ordered list of places, supplied by
configuration or by a relationship.*

⇒ ***the bare-hash problem is solved one level up by not having a bare hash at that level.*** The
table is the artifact that carries provenance for bytes that have none; the table itself always has
provenance, because you chose where to get it.

### 1.2 What this buys, stated as the trade

**You are trading storage and staleness for hops and reach, and every term is a local decision:**

- **hold more** → fewer hops, more disk, more to keep current;
- **hold less** → more hops, less disk, and **the penalty is logarithmic** — *relative hops scale as
  `log s₁ / log s₂`, so a hundredfold smaller table costs roughly 1.5–2× the hops*;
- **merge more sources** → broader reach, more exposure to bad hints, more volume to bound;
- **merge fewer** → narrower reach, less to verify, less to prune.

⭐ **None of these is a protocol parameter. All of them are a peer's own configuration**, which is the
property that separates this from a keyspace-partitioned overlay, where your share of the keyspace is
assigned to you.

### 1.3 `[rev 3]` ⭐⭐⭐ The subject is a PREDICATE, and the hash question was a probe

**This document is named for content hashes and that is too narrow.** The corpus already established
the general form and then lost it for five documents:

> ***Finding a hash is the same shape as finding contributions to a coordinate, entries in a feed, posts
> in a forum, or articles in an encyclopedia: a published set of claims about where things are, merged
> by the reader, with computation available over it. **Only the predicate differs.***

⭐⭐ **So the honest framing of what is being designed is not a content-locator. It is the exchange
substrate — *what gets published, how it is merged, how it stays current, and who is authorized to ask
whom* — of which the content hash is one subject among several:**

| subject | the claim | already a keyed lookup in this corpus? |
|---|---|---|
| a content hash | *I serve these bytes* | ⛔ the one with nothing behind it |
| a namespace | *I serve this peer's estate* | ⭐ the default (§3.1) |
| a name | *this name is that peer* | ✅ landed — the registry |
| a peer | *these are its transports* | ✅ landed |
| a destination | *this is the next hop* | ✅ landed |
| a coordinate | *these parties contributed* | ⚠ drafted |
| ⭐ **a query** | ***I can answer this*** — or *here is an aggregated view of it* | ⛔ **not designed, and it is the same shape** |

⇒ ***Asking "where do I get this hash" was a PROBE. What it revealed is the structure of the
communication the answer requires — publish, adopt, merge, expire, bound, authorize — and that
structure is the deliverable.*** **The hash is the easiest subject to reason about because a hash is
self-verifying; it was the right place to start and it is the wrong place to stop.**

**What this changes in this document, concretely:**

- ⭐ **`subject` is an open discriminated union, not a two-case enum.** §3.1's `{kind: "namespace"} |
  {kind: "content"}` gains *"and unknown kinds are MUST-ignore"*, so a query subject needs no new type
  and no new code path later.
- ⭐ **The three mechanisms that do the work are subject-independent** — expiry, per-issuer volume bounds,
  and reader-side merge. **None of them reads the subject.** That is the test that the generalization is
  real rather than aspirational, and it passes.
- ⚠ **What is NOT subject-independent:** the *derived-key* question (§7 item 6) is specific to hash
  subjects, and `evidence: probed` means *"I fetched a piece and it verified"*, which only content
  addressing makes cheap. **A query subject would need its own evidence notion, and that is out of
  scope here — named so it is not assumed solved.**

### 1.4 `[rev 4]` ⭐⭐⭐ Where the traversal actually happens — and it is not in hash space

**The mechanism steps OUT of the content-address space, navigates in the semantic space, and steps back
IN.** Stated as a loop:

```
hash  →  step back to the SEMANTIC subject it belongs to   (a namespace, a name, an estate)
      →  navigate the CONNECTED GRAPH                       (parties you know, tables you adopted,
                                                              referrals you follow)
      →  arrive at a party
      →  translate back DOWN to the hash                    (the tree: path → hash, §6.4.2's
                                                              {namespace}/{hex(H)})
      →  fetch, verify
```

⭐⭐ **So the hash is a TERMINAL, not a coordinate you move through.** You never traverse hash space —
there is nothing to traverse in it, because a hash carries no information about location. **Every hop
of every walk in this design happens over live, connected, semantic structure: peers you have a
relationship with, namespaces, adopted tables, referrals.**

**Three things that follow, and each one settles a question the arc kept reopening:**

1. ⭐ **This is why *"you cannot route on a hash prefix"* was never an obstacle.** Routing aggregation
   needs the key's prefix to correlate with topology, which a content hash never does — **but nothing
   here routes on the hash.** The hash is consumed at the last step, by a `tree:get` at a computable
   path, after the navigation is already over.
2. ⭐⭐ **This is the real reason per-namespace is the default** and not merely the cheap option: the
   namespace **is** the semantic handle the graph is navigable by. A per-hash claim is a shortcut that
   skips the semantic step — useful, and it forfeits the structure that makes the walk work.
3. ⭐ **And it is why a referral terminates** (§5): a referral narrows in *name* space, which is
   hierarchical and finite, where nothing narrows in hash space at all.

⇒ ***The content-address layer is where verification happens. The semantic layer is where navigation
happens. This mechanism is the translation between them, and the connected graph is the thing it walks.***

---

## §2 What tier, and the tests are already written

`SYSTEM-ARCHITECTURE.md` §13.1's own tests decide it:

| test | answer |
|---|---|
| Does a **single peer** need it to run? *(Tier 1 is "irreducible… they define what a peer is")* | ⛔ **No.** A lone peer has nothing to locate ⇒ **not Tier 1** |
| Is it needed for *"production multi-peer deployments"*? | ⚠ **No — but not for the reason revision 1 gave.** `[rev 2]` **Not installing it leaves a peer exactly where it is today**, which is a weaker claim than *the path already terminates* and is the true one ⇒ **not required at Tier 2** |
| So where? | **Tier 2b, optional, as a companion to `EXTENSION-SUBSTITUTE`** — the same shelf, and the same posture |

> ⚠ **`[rev 2]` The correction matters and it runs against this proposal's own interest.** Revision 1
> argued *"the path already terminates without it: a reference names its publisher, and the publisher
> serves the bytes."* **That is true only while the publisher is reachable — which is exactly the case
> the mechanism does not exist for.** Provenance travels at the top of an artifact and is **lost at its
> leaves**: a manifest's chunk hashes carry no locator, so once the publisher is gone you hold orphans
> and a third party holding one of those chunks is unreachable by anything in the system.
> ⇒ ***this is an accelerator in the common case and the ONLY mechanism in the tail case***, and
> calling it an accelerator flatly is how something ends up under-specified.

⭐⭐ **It MUST stay optional, and the rule to copy is `EXTENSION-SUBSTITUTE` §1 position 1 verbatim:**
*"Installing this extension is additive; not installing it leaves CONTENT's miss behavior (404)
unchanged."*

✅ **`[rev 2]` And optionality has an enforcement point that already exists.** `EXTENSION-SUBSTITUTE`
§8's consult-cap *"MAY narrow via `constraints` … `substitute_types` (restrict to specific handlers)."*
**A locator backend is its own `substitute_type`, so a deployment that does not want it simply does not
grant it.** No new machinery, and the optionality claim becomes checkable rather than asserted.

**Why this is a requirement and not modesty:** the single best predictor of survival across every
system surveyed in this arc is ***scale-invariance — does the system do something useful for its first
user, with no adoption at all?*** A locator mechanism that becomes a prerequisite for ordinary
fetching would trade away the one property the outcome column says is decisive. **One peer must remain
a working system. Two peers must remain a working system.**

---

## §3 The new artifact — one entity type

**Provisional shape.** Every field below is derived from a recorded finding and is cited; **the schema
is not the contribution and is expected to change.** The contribution is that there is only one.

```
type: "system/locator/claim"
data: {
  subject:      locator-subject,        ; §3.1 — what is being located
  holders:      [system/hash],          ; peer ids that will serve it
  issuer:       system/hash,            ; whose claim this is
  expires_at:   time,                   ; REQUIRED, not optional — §3.2
  evidence:     locator-evidence?,      ; §3.3 — probed, asserted, or relayed
  provenance:   [system/hash]?          ; §3.4 — the merge path, if this was aggregated
}
```

### 3.1 `subject` is a namespace by default and a hash by exception

⭐⭐⭐ **Per-namespace is the default; per-hash is the sharp case.** Four independent derivations reach
this granularity — hierarchical naming is what permits aggregation and a content hash is flat; leaf
count (a peer holding 10,000 blobs is an ordinary peer); the privacy and abuse surface; **and the
referral (§5), which a per-hash table structurally cannot express.**

```
locator-subject = { kind: "namespace", path } | { kind: "content", key }
```

⚠ **And the two kinds are not symmetric on privacy.** A namespace under an already-published root
discloses nothing new. **A bare content hash is a confirmation oracle in both directions** —
publishing *"I hold `H`"* confirms possession to anyone who can compute `H`, and asking *"who has
`H`"* tells the answerer what you want.

⇒ ⭐ **the rule that follows, and nobody has written it down:** *a `namespace` subject carries the path
plainly; a `content` subject carries a **derived** key, not the bare hash.* Four deployed systems agree
on the principle — **the lookup key is derived from the thing that authorizes reading** — and the
construction is §7 item 5's open work.

### 3.2 `expires_at` is REQUIRED

**Not a field with a default — a required field, and the design fails without it.**

- A holder claim is about the **present** and decays. *Someone replied* stays true forever; *I hold
  these bytes* does not.
- ⛔ **The author cannot retract.** Once an aggregator has unioned your table, a dead peer's entries
  keep being served by parties the author cannot reach — **the author is precisely the party who
  cannot fix it.**
- ⭐ **Measured in a live federation:** merged blocklists do not propagate retractions, and the
  deployed workaround is *a separate retraction feed* — which is the known-bad shape, because a
  subscriber who misses the second feed holds the first one's errors indefinitely.
- **Every deployed shared index makes the holder re-assert.** Four independent systems, one answer.

⇒ **an expiring claim retracts itself, and that is the only retraction mechanism this design gets.**
⛔ **An aggregator that unions without honouring expiry is building the orphaned-data problem on
purpose.**

### 3.3 `evidence` — a claim and a probed claim are different objects

**Three values, and the discrimination is load-bearing:**

| value | meaning |
|---|---|
| `asserted` | the issuer says the holder serves it. Cheap, unverified |
| `probed` | the issuer fetched a small piece and it verified. **Content addressing is what makes possession cheaply provable** |
| `relayed` | the issuer got this from someone else. See `provenance` |

⚠ **A probe proves possession at probe time to the prober. It does not prove the holder consented to
serve *your* readers** — see §7 item 7.

### 3.4 `provenance` — and the reason it is not optional in an aggregate

⭐ **Origin attribution alone is insufficient, and the largest deployment of this model proves it:**
validating the *origin* of an announcement does not catch a **leak**, where the origin is legitimate
and the *propagation* is wrong. **A merge path is a different fact from an issuer.**

✅ **And the substrate already does the right thing here:** `EXTENSION-REGISTRY` §8.2 — *"the
aggregator does NOT re-sign aggregated bindings; receivers verify against the original issuer's
signature. Aggregator is transport."* **Every entry stays attributable to whoever minted it.**

### 3.5 `[rev 2]` ⛔ What the aggregation layer already obliges — and revision 1 omitted both

**The closure property is free. The two conditions it carries are not, and they are MUSTs.**

#### 3.5.1 Every claim is individually signed, and that settles the granularity question

`SYSTEM-DATA-EXCHANGE` §2.2: a republished object **MUST** carry *"a detached per-entity signature at
the invariant pointer `/{signer_peer_id}/system/signature/{target_hash_hex}`, or an inclusion proof"*,
and one with neither **MUST NOT** be presented as attributed.

⇒ **the signature is per CLAIM, not per table, because the claim is what travels.** *(Which is
`EXTENSION-SUBSTITUTE` §2.1's existing pattern exactly.)*

⭐⭐ **And that turns §3.1's granularity choice from a preference into arithmetic — a peer holding
10,000 blobs mints 10,000 signatures per-hash and ONE per-namespace.** **A fifth independent
derivation of per-namespace-by-default, and the first that is a bill rather than an argument.**

#### 3.5.2 ⛔ An aggregator CARRIES; it does not merge

`SYSTEM-DATA-EXCHANGE` §2.1: a republished entity **MUST** be bound **byte-identically** and an
implementation **MUST NOT** re-encode it. **An aggregator that merges N tables into one combined table
has re-encoded every claim it carries** — and §2.1 names the consequence: *"Move the hash and the
signature no longer names the entity"*, producing *"a complete, verifiable, correctly-walked
publication in which nobody wrote anything."*

⚠ **§2.1 also names why this is the likely defect rather than a careless one:** *"A gatherer aggregates
types it did not write, so the case that breaks it is the normal case, and it raises no error at any
layer."*

> ⭐⭐ **Normative shape: an aggregator carries claims byte-identically. The union is the READER'S view,
> computed at read time, never materialized as a new signed object.**

✅ **`EXTENSION-REGISTRY` §8.2 already says this in registry vocabulary** — *"The aggregator does NOT
re-sign aggregated bindings; receivers verify against the original issuer's signature. **Aggregator is
transport.**"* **Two specifications agree; the design must not disagree with both.**

#### 3.5.3 ⛔ The growth rule binds, and silence is itself the violation

`SYSTEM-DATA-EXCHANGE` §2.5: a republication format whose membership **grows with participation**
**MUST NOT** carry its members as an unbounded collection in a single entity — it **MUST** be a
**bounded head plus key-addressed pages**, never renumbered, merged or compacted. **And a second MUST:
a specification introducing a member collection MUST state which side of the test it falls on.**

**This proposal introduces two, and they fall on different sides:**

| collection | side | consequence |
|---|---|---|
| **the table** (the set of claims a party publishes) | **self-issued: bounded by the issuer.** ⛔ **Aggregated: participation** — it is the *definition* of an aggregator that its membership grows with how many sources it merges | **an aggregate table MUST be a bounded head plus key-addressed pages** |
| **`holders`** within one claim | **self-issued: bounded by the issuer.** ⚠ **Aggregated: participation, and this needs a ruling** | ⚠ **open — see §7 item 7** |

⭐ **Why this is not a formatting requirement, in §2.5's own words:** *"The cost is the reader's, and it
is invisible from the format that causes it. A flat list is correct, verifiable, cheap to write, and
passes every check at the size its author tested — it becomes a defect only in somebody else's fetch,
at a size no fixture has."* ⇒ **the aggregator is precisely the case that rule was written for.**

✅ **Checked clean:** §2.3 rule 4 forbids a republication format from *providing a field in which to
claim completeness*, and the §3 schema has none. ⚠ **But a `partial`/`complete` holder discriminator —
which the survey recommended borrowing — sits close enough to that prohibition that it must be checked
against rule 4 before it is added**, not assumed distinct because its subject differs.

---

## §4 The resolution layer — a registry backend, with one contract difference

`EXTENSION-REGISTRY` §1 already reserves the slot: *"Additional backends (peer-issued, DID:web,
DNS-TXT, DHT, consensus-anchored, aggregator) compose on the substrate and ship in their own
proposals; this spec defines the contract they bind to."* §12 repeats it. **A sequenced backend is not
a rejected one, and this proposal is one of the sequenced backends arriving.**

**What the substrate already supplies, and it is nearly everything:**

| the design needs | the substrate has |
|---|---|
| anyone may publish; no permission to assert | §1 position 2 — *"Anyone can publish a binding claiming any name. Whether a receiver TRUSTS that binding is the receiver's policy"* |
| the reader chooses sources and their order | §4.1 precedence, §4.1a the default dispatch list — **the consumer's config** |
| no special infrastructure | §1 position 4 — *"Registry is just a peer"* |
| aggregation without re-signing | §8.2 |
| two sources disagree → surface both | §8.3 — *"does not silently pick"*, fail-closed default, explicit pin override |
| abuse bounds owned by the backend | ✅ §12 — *"Anti-squatting / abuse prevention. Per-backend concern; substrate has no opinion"* |

### 4.1 ⛔ The one contract difference: cardinality

**`EXTENSION-REGISTRY` §4.1.1 is *Single binding per name per resolution*** — `meta_resolve` returns
the first hit that validates, and the lower-priority backend's binding is never surfaced.

⭐ **That is correct for names and backwards for holders.** A name resolving two ways is a **conflict to
adjudicate**; a hash resolving two ways is **a better answer, and ten is better still.** This is the one
place where the locator backend wants something the substrate's default resolution rule does not give.

⚠ **Open: is this a backend-local reading, or a substrate change?** The honest answer is that it is
not yet known — it turns on whether `ResolutionResult` can carry a set without a substrate revision.
**§7 item 5.**

---

## §5 The referral — and this is the part that makes small tables work

> ⭐⭐ **A locator entry says `key → where the bytes are`. A referral says `key-range → where to ask
> next`. They are the same record; only the kind of thing "where" names differs.**

**Without referrals, every consumer must be configured with every table it might ever need.** The
corpus already found this hole from the naming side: there is no way for a registry to say *names
under this prefix are answered by that registry*, so delegation *"is configured per-consumer, so it
scales with the number of consumers rather than the number of zones."*

⇒ ⛔ ***The reason a small device cannot hold a small table today is configuration, not storage.*** Its
table is small; its config must be complete.

**Two things the referral needs, and both are already specified elsewhere:**

1. ⚠ **A bounded walk.** Nothing forces a referral to narrow, and `A → B → A` is expressible. The
   bounded fan-out query — **a hop budget plus a set of already-visited parties, with progressive
   partial results** — is specified at two pre-split core revisions and is listed as deferred future
   work in `EXTENSION-QUERY` §1.2. ⭐ **It now has a caller: referral walking does not terminate
   without it.**
2. ⛔ **A carve-out in `EXTENSION-SUBSTITUTE` §4** — see §7 item 4.

**Why the hierarchy objection does not bind this.** Delegation was deferred on a good argument: a
rooted hierarchy is also a central point of political failure. ✅ **That argument is about
hierarchy-as-AUTHORITY.** A referral has **no root** (any table may carry one), is **not
authoritative** (§8.3 surfaces conflicts rather than picking), and is **not binding** (a wrong referral
costs one round trip). **Hierarchy-as-hint is already safe under machinery the corpus has.**

---

## §6 The consumption layer — two clauses in `EXTENSION-SUBSTITUTE`

⭐⭐⭐ **The substituter is already the interpreter this design needs, and it already refuses exactly the
case this design answers.** §1: *"A peer that misses on a local `system/content:get` SHOULD be able to
consult an ordered chain of substitute sources before returning the terminal miss. The model follows
Nix substituters: ordered, priority-driven."*

**Its chain algorithm (§3) does this, in order:** pending-sidecar check → capability gate → ⛔ **step 3,
*"if `hash.claimed_source_peer_id == null`: return 404"*** → enumerate entries matching that source →
consult in priority order → verify hash → ingest → advance on mismatch.

> ⭐⭐⭐ **Step 3 is the hole, and the locator table is exactly what fills it.** §4 says so in its own
> words: ***"No wildcards in v1. A bare-hash query (no claimed `source_peer_id`) does NOT trigger
> consultation. Wildcard-source entries are out of scope for v1."***
>
> **A locator lookup SUPPLIES the `claimed_source_peer_id` that step 3 requires.** It runs *before*
> step 3, turns a bare hash into a candidate list, and **the entire rest of the chain applies
> unchanged** — hash verification, mismatch-discard-and-advance, ingest-denied handling, chain
> exhaustion, error codes. **None of it is redesigned.**

**Delta 1 — §3 step 3.** A bare hash no longer returns 404 outright when a locator backend is
installed; it is resolved to candidate sources first, and each candidate is then consulted as an
ordinary entry. ⚠ **Fail-closed when no backend is installed** — behaviour is unchanged, per §2.

**Delta 2 — §4's wildcard clause.** The *"out of scope for v1"* sentence becomes a pointer to the
mechanism that scopes it, rather than a flat refusal.

**Delta 3 `[rev 2]` — §3.1's abort rule, and this one is a live denial-of-service.**

§3.1: *"A convention handler returning **`cap_denied`** … **ABORTS** the chain — advancing to another
URL won't help if access is denied at the cap layer."*

⭐ **Sound for the chain it was written for and false for this one.** In a deployment-curated chain
`cap_denied` means *you genuinely cannot fetch, and the next entry will not change that.* **For an
unvetted third-party candidate it means only *that peer will not serve you*, which says nothing about
the next candidate.**

⛔ **The attack is ONE CLAIM long.** Publish a locator claim naming a peer that will refuse the asker;
a reader who merged it aborts on that candidate and returns 404 **with every good candidate behind it
unread.** No capability, no volume, no collusion — **and the volume circuit breaker does not fire,
because the volume is one.**

> **A `cap_denied` from a LOCATOR-SUPPLIED candidate MUST ADVANCE, not ABORT** — or locator candidates
> are consulted in a sub-chain whose abort does not propagate to the deployment-curated chain.

⚠ **Preserve the distinction while fixing it:** aborting on a peer's *own* configured entries is
still correct, and flattening both into *always advance* would remove a real protection. **The rule
keys on where the candidate came from — which is the same discriminator §7 item 1 needs, so the two are
one missing concept rather than two fixes.**

⭐ **Everything else in that spec is reused, not amended** — and the reuse includes the properties this
design depends on most: bytes are verified against the hash regardless of who served them, a mismatch
discards and advances rather than failing, and the consumer's local ordered set is the only chain.

---

## §6a `[rev 3]` ⭐⭐⭐ Authorization — the grant, and it is expressible today

**Revision 2 filed *"the capability model cannot express this"* as the top blocker. That was wrong, and
reading the core protocol's capability system instead of one extension's summary of it is what shows
why.** ⛔ **Revision 2's item 1 is withdrawn.**

### 6a.1 The grant has five dimensions, and one of them exists for exactly this case

`ENTITY-CORE-PROTOCOL` §3.6 — `system/capability/grant-entry`:

```
handlers:    path-scope   ; include / exclude, tree paths or patterns
resources:   path-scope   ; the data paths
operations:  id-scope     ; operation names
peers:       id-scope     ; ⭐ WHICH PEERS a grant may be used AGAINST — optional
constraints: map          ; narrowing, handler-interpreted, byte-equal under delegation
allowances:  map          ; expanding, handler-interpreted
```

⭐⭐⭐ **A substitute fetch from a candidate peer is INTERNAL DISPATCH, and the core protocol says the
`peers` dimension is the dimension that exists for it:**

> §3.5: *"**Internal dispatch (handler sub-requests).** A local handler executing a sub-request MAY
> target a remote peer's namespace… The peer runtime recognizes that the canonicalized path targets a
> different peer and routes the request outbound."*

> *"On **internal dispatch**, a locally-originated sub-request MAY name a foreign namespace… **This is
> the class the `peers` dimension exists for**: it scopes which peers a grant may be used *against* on
> an outbound sub-dispatch, not which peers may call in."*

> **The enforcement point is normative and already exists:** *"An implementation **MUST** evaluate
> `check_permission` **before a locally-originated sub-dispatch leaves the peer**, with all four
> dimensions applied and `target_peer = extract_peer(uri, local_peer_id)`."*

✅ **And the default is fail-closed in the right direction:** *"A grant carrying no `peers` scope
therefore defaults to `{include: [local_peer_id]}` and is **still checked**… Implementations MUST NOT
skip the check on an absent `peers` field; doing so authorizes a foreign namespace under a grant nobody
scoped for one."*

⇒ ***the locator mechanism requires an explicit `peers` grant, and without one it is denied. That is
exactly the right default and nothing had to be invented to get it.***

### 6a.2 The calls, the operations and the resources

**Modelled on the resolver-handler contract (§2.1), which is two operations for every backend:**

| | |
|---|---|
| **handler** | `system/locator` — the resolution side. *(The consumption side is `system/substitute/sources`, which already exists.)* |
| **operations** | `resolve(subject, [hints]) → LocatorResult` · `invalidate-cache(subject \| null)` — **the registry substrate's own two, unchanged**; plus `publish` / `retract` on the issuing side |
| **resources** | the **subject space** a grant covers — i.e. *which namespaces may I ask about, and which may I answer about.* `path-scope`, so `include`/`exclude` and patterns work |
| **peers** | ⭐ **the candidates this grant may be spent against.** §6a.3 |
| **constraints** | the genuinely locator-specific narrowing: **`locator_issuers`** (whose tables I will consult) · **`max_candidates`** (how many I will try) · **`min_evidence`** (`asserted` / `probed`) |

⚠ **Two things this list does NOT need**, because they fall out of dimensions that already exist: a
*"which tables do I trust"* mechanism separate from `constraints`, and a *"may I fetch from a stranger"*
flag separate from `peers`.

### 6a.3 ⭐⭐⭐ The policy spectrum — one mechanism, and the answer is the deployment's

**There is no single right answer here and the design must not pick one.** *Whether a peer will act on
a claim about a third party is policy, owned by whoever runs the network — and the grant-entry already
has the dimensions to say it:*

> ⛔ **`[rev 4]` CORRECTION — revision 3 wrote `peers: {include: ["/*/*"]}` and that is a RESOURCE
> pattern in a PEER-IDENTITY field.** The two dimensions are different types and conflating them is
> the specific confusion this area produces. **See §6b.1 for the distinction and §6b.4 for two places
> the core spec itself blurs it.** Corrected below.

| the deployment | the grant |
|---|---|
| **my own programs and files; authority given directly to the devices I own** | `resources: {include: ["/*/system/content/*"]}` · `peers: {include: [dev-1, dev-2, dev-3]}` — **the path shape is open, the party list is closed.** No stranger is ever fetched from |
| **a group, or one application's network** | `resources: {include: ["/*/system/content/*"]}` · `peers: {include: ["*"]}` · `constraints: {locator_issuers: [the group's aggregator]}` — **any party, but only on the group's say-so** |
| **a public content network** | `resources: {include: ["/*/system/content/public/*"]}` · `peers: {include: ["*"]}`, no issuer constraint, **plus the volume bound** (§7) |

⭐⭐ **Read the columns, not the rows: the RESOURCE pattern barely changes across the three, and the
PEERS list changes completely.** That is the shape of the whole policy question — *the same paths, a
different set of parties* — and it is why both dimensions are needed and neither is redundant (§6b.1).

⇒ ***The same claim type, the same resolution path, the same substituter chain. What differs between a
three-device household and a public archive is a grant, not a design.*** **That is the property to
protect, and it is also why *"is a third-party claim allowed?"* is not a question this document can
answer** — see §7 item 2.

### 6a.4 ⭐⭐ Consent-to-be-listed is already specified, and it is not new machinery

**The abuse review's one unresolved item was: *a probe proves possession at probe time, not that the
holder consented to serve YOUR readers.*** The core protocol answers it:

> *"**Presented** — a credential that is not the propagated caller capability but a distinct capability
> minted **by the target peer**, naming **this peer** as `grantee`. **It relaxes the handler grant's
> `peers` dimension, because the party that decides what may be done at a peer is that peer and it has
> already decided**; requiring the dispatcher's grant to have anticipated the target asks the wrong
> party."*

⭐ **That is consent, stated normatively, with a five-check verification recipe** (root `granter`
resolves to the target; leaf `grantee` to the local peer; unexpired; unrevoked; chain-verified — *"a
credential failing any of these is not presented authority"*).

⚠ **And it carries a limit the design must respect: *"The target answers WHERE; the handler's grant
still answers WHAT."*** A holder-minted credential relaxes **Dimension 4 only**. **A locator design must
not expect a consenting holder's credential to widen what operations or resources the asker may reach** —
that is the confused-deputy shape the correction at §3.5 exists to close.

### 6a.5 ⛔ The real finding: `EXTENSION-SUBSTITUTE` §8 shadows Dimension 4

**What revision 2 mis-read as a blocker is a genuine but different defect.** §8 documents the
consult-cap's narrowing as *"`constraints`… **`source_peer_id`** (restrict to specific publishers),
`substitute_types` (restrict to specific handlers)"* — **and `source_peer_id` duplicates the core
protocol's `peers` dimension with strictly weaker semantics:**

| | core `peers` dimension | §8's `source_peer_id` constraint |
|---|---|---|
| include **and** exclude | ✅ | ⛔ |
| pattern matching (§5.4) | ✅ | ⛔ |
| normative evaluation site | ✅ §5.2 Dimension 4, MUST, before the sub-dispatch leaves | handler-interpreted |
| delegation | attenuable — a child may narrow a wildcard | ⛔ **byte-equal** (§5.6), so it can only be matched exactly, never narrowed |

⇒ **Two places narrow the same thing and the extension's is the lesser one.** A deployment narrowing via
the constraint gets no wildcard, no exclusion, no pattern and no attenuation; a deployment narrowing via
`peers` gets all four and is not mentioned by the extension that most needs it.

> **`EXTENSION-SUBSTITUTE` §8 should REFERENCE Dimension 4 rather than shadow it.** ⚠ This is a small,
> routable finding against landed text, **and it is the thing revision 2's blocker was actually
> touching.** `substitute_types` is fine and stays — it narrows something the core has no dimension for.

---

## §6b `[rev 4]` ⭐⭐⭐ The whole grant, call by call — because no dimension makes sense alone

**A dimension read in isolation is unreadable. The way to see this is to take each call in the flow, name
the handler it reaches, then work out the resource, then the party.** That is what this section does.

### 6b.1 `resources` and `peers` are different types, and the overlap is the point

**This is the distinction that dissolves the apparent confusion, and getting it wrong is what revision 3
did:**

| | `resources` | `peers` |
|---|---|---|
| type | **`path-scope`** — tree paths | **`id-scope`** — peer identifiers |
| values look like | `/*/system/content/public/*` · `system/tree` · `/{alice}/local/files/*` | `<peer_id>` · `*` |
| what it answers | ***what path*** | ***whose*** |
| wildcard form | **`/*/`** prefix — *"this shape, at any peer"* (§1.4 calls these *"grant **resource** patterns"*) | **`"*"`** — matched by `matches_pattern`'s `if pattern == "*": return true` |

⚠ **Why they look redundant and are not.** On an outbound sub-dispatch the URI is `/{target}/path`, so
**the resource path's first segment IS the target peer** and `target_peer = extract_peer(uri, local)`.
Both dimensions therefore constrain the same call. ⭐⭐ **But they constrain different axes of it, and
the fleet case proves the difference:**

```
resources: {include: ["/*/system/content/*"]}     ; any party's content subtree
peers:     {include: [dev-1, dev-2, dev-3]}       ; but only these three parties
```

***You can pin the shape without pinning the party, or the party without pinning the shape.*** A design
that used only `resources` could not express *"my three devices, wherever their content lives"*; one
that used only `peers` could not express *"their content namespace, and nothing else of theirs."*
**Both, always.**

### 6b.2 The five calls, and the grant each one spends

**A reader holds a hash `H`, its content store misses, and it has a locator backend installed.** Five
calls happen. `L` = the local peer, `I` = a table's issuer, `C` = a candidate holder.

| # | the call | handler | operations | resources | peers | outbound? |
|---|---|---|---|---|---|---|
| **1** | resolve the subject | `system/locator` | `resolve` | `system/locator/*` *(peer-relative ⇒ local)* | — *(absent ⇒ local, §3.5)* | **no** |
| **2** | read the adopted tables | `system/tree` | `get` | ⭐ **`/*/system/locator/claims/*`** — cached remote data under other peers' namespaces (§1.4) | — *(absent ⇒ local; it is a LOCAL read of cached data)* | **no** |
| **3** | consult the substitute chain | `system/substitute/sources` | `consult` | `ctx.resource_target` — the triggering target namespace (§8) | — | **no** |
| **4** | ⭐ **fetch at a candidate** | `system/content` | `get` | **`/*/system/content/public/*`** — or narrower | ⭐ **`{include: [C…]}`** or `{include: ["*"]}` | ⭐ **YES — this is the only one** |
| **5** | ingest what came back | `system/content` | `ingest` | the **target namespace** the bytes land in — which decides whether *you* become a holder (§7 item 8) | — | **no** |

⭐⭐ **Four of the five calls are local, and the whole authorization question lives in call 4.** That is
the honest scope of the *"how is this authorized"* problem: **one outbound sub-dispatch, governed by a
dimension that exists for exactly that class, checked before the request leaves the peer.**

⚠ **Call 2 is the one that surprises.** Reading a table you adopted is **not** an outbound call — it is
a `tree:get` against **cached remote data in your own tree** (§1.4: *"entities under other peers'
namespaces… this is the local peer's knowledge of what those peers have"*). **Refreshing that cache is
a separate act**, owned by the copy-current loop, and it is the loop's grant that pays for it — not the
locator's. ⇒ *adopting a table and consulting it are different operations with different grants, and the
design should not merge them.*

⚠ **Call 5 is where a real decision hides.** `ingest` binds under `target_namespace_from_cap`, and in
the production topology binding is what makes you servable. ⇒ ***the resource dimension on call 5
decides whether fetching makes YOU a holder***, which is the question §7 item 8 now carries.

### 6b.3 What the locator-specific narrowing actually is

**Having done §6b.2, the `constraints` list shrinks to what no existing dimension covers:**

| constraint | why it cannot be a dimension |
|---|---|
| **`locator_issuers`** | *whose TABLE I will believe* — the issuers of claims. **Not `peers`**, which is whom I will *fetch from*. ⭐ **Two different parties, and this is the trust/verification split showing up as two separate fields** |
| **`max_candidates`** | a count. No dimension is numeric |
| **`min_evidence`** | `asserted` \| `probed`. A property of the claim, not of the call |

⚠ **`locator_issuers` versus `peers` is the sharpest thing in this section.** *The party that told you
where to look and the party you look at are different, and a grant that cannot separate them cannot
express "I trust my group's index, and it may point me anywhere" or its opposite.* **Both rows appear
in §6a.3's table, and they differ in exactly these two fields.**

### 6b.4 ⚠ Two places the core spec blurs the same distinction

**Filed because the confusion is not only in this document, and both are small.**

1. **§1.5's canonical-form note illustrates a `peers` pattern as a PATH:** *"Cap patterns are
   form-bearing strings (`peers: include = ["/{peer_id_form}/..."]`)."* ⛔ **But `peers` is an
   `id-scope` whose members are peer identifiers, and §3.6's own default is `{include: [local_peer_id]}`
   — a bare id, not `/{id}/...`.** The note's *subject* is cap-pattern form-coherence, which makes the
   example's shape especially unfortunate. **A one-token fix.**
2. **§3.6's `id-scope` says its values *"use `matches_pattern` (§5.4)"* — but §5.4's stated contract is
   for PATHS:** *"Both path and pattern MUST be canonicalized (absolute) before calling"*, and
   `canonicalize` on a bare peer id would produce `/{local}/{peer_id}`, which is nonsense. **So the
   id-scope case uses the matcher outside its own precondition.** ⚠ *In practice the two arms that
   matter behave correctly — `pattern == "*"` returns true and an exact id falls through to exact match
   — so this is an under-specification rather than a live defect, and it should be said rather than
   relied on.*

⇒ **Both are routable findings against `ENTITY-CORE-PROTOCOL`, both are clarifications rather than
changes, and together they are a large part of why this area reads as confusing.**

---

## §7 ⛔ The seams — what must be settled before a normative revision

**These are the deliverable of this revision. Items 1–5 are collisions with landed normative text.**

### Item 1 `[rev 3 — WITHDRAWN]` ✅ *"The consult-cap cannot express this mechanism"* — it can

⛔ **Revision 2 ranked this first and it was wrong.** It read `EXTENSION-SUBSTITUTE` §8's summary of its
own narrowing and concluded the capability system had no way to say *"consult a peer a table named."*
**The core protocol's `grant-entry` has a `peers` dimension whose stated purpose is exactly outbound
sub-dispatch at a foreign namespace, with a MUST-level evaluation site and a fail-closed default.**
**See §6a — the grant is expressible today and nothing has to be invented.**

⭐ **The transferable half, because this is the toolkit's own recurring defect arriving in a design
argument:** *an extension's summary of a core mechanism is not the mechanism.* §8 lists two `constraints`
keys; it does not, and was never trying to, enumerate the core's dimensions. **Reading a narrowing
example as the narrowing surface produced a confident blocker against a system that already did the
job.**

✅ **What survives is smaller, real, and routable: §8 SHADOWS Dimension 4** — its `source_peer_id`
constraint duplicates `peers` with strictly weaker semantics (no exclude, no patterns, no attenuation,
handler-interpreted rather than MUST-evaluated). **§6a.5 states it; it is a defect in landed text worth
a packet, not a blocker on this design.**

### Item 2 `[rev 3 — REFRAMED; it was never an arch question]` ⚠ What `EXTENSION-SUBSTITUTE` §4's *"invalid"* is

**The landed text:** *"**No transitive trust.** An entry claims authority for `source_peer_id`'s content
only. **Peer B serving peer A's content without A's signature is invalid.**"*

⛔ **Revision 2 filed this as *"should a third-party claim be admissible?"* — a question for arch to rule
globally. That framing is wrong, and the correction is the more important content:**

> ⭐⭐⭐ **Whether this peer will act on a claim about a third party is POLICY, and it belongs to whoever
> runs the deployment. It is a `peers` scope and a `locator_issuers` constraint — a grant somebody
> writes, not a sentence arch writes.** §6a.3 shows the same mechanism carrying three different answers:
> enumerated devices for one person's own fleet, any-peer-on-the-group's-say-so for an application
> network, any-peer for a public archive. **A design that picks one has taken a decision that is not
> its to take.**

⇒ **The narrow question that remains, and it is answerable:** *is §4's sentence a **wire MUST**, or a
**default** a grant may widen?* **Read against §1 position 2 — *"a substitute fetch is trustworthy when
the returned bytes hash-match… regardless of who served them"* — it can only be the latter**, because
the bytes are already closed and what is left to protect is wasted work. ⇒ **§4 states the
conservative default; the `peers` dimension is how a deployment departs from it.** ⚠ **That still wants
confirming in the text rather than inferred here — but it is a one-clause clarification, not a
ruling about what networks are allowed to want.**

### Item 3 ⚠ §3 step 3 and §4's wildcard refusal — the delta is small but it is a v1 reversal

Both clauses are correct *as v1 scope statements*. The question is whether they become
*"scoped by the locator backend"* or stay *"out of scope"* with the locator mechanism living
elsewhere entirely. **Named so the choice is deliberate.**

### Item 4 ⚠ §4's "no transitive following" versus the referral

**The landed text:** *"**No transitive following.** Bytes fetched from a substitute are hash-verified +
ingested locally; any substitute metadata embedded in the result is **advisory, never a new chain to
follow.** The only chain is the consumer's local `system/substitute/sources/` set."*

⛔ **A referral is, on its face, a new chain to follow.** ✅ **But the two cases are genuinely
different and the distinction is defensible:** §4 forbids *following hints embedded in bytes you just
fetched* — an unbounded, attacker-supplied redirect. A referral is *an entry in a table you
deliberately adopted*, walked under a hop budget the reader sets.

⇒ **A carve-out is needed and it must be written narrowly**, or the referral is non-conformant on its
face. ⚠ **And the rule has more than one home** — restating it in one place while another keeps the
flat prohibition is how a rule ends up contradicting itself at MUST level.

### Item 5 ⚠ Resolution cardinality — backend-local, or a substrate change?

§4.1.1 returns one binding. The locator backend wants the union. **Unknown whether the handler
contract can express that without a substrate revision.** Requires reading §2.1's `ResolutionResult`
shape against a set-valued answer. **Smallest of the four and the most mechanical.**

### Item 6 ⚠ The derived key for `content` subjects is specified nowhere here

**The principle is settled** — *the lookup key is derived from the thing that authorizes reading*, on
four deployed derivations across three domains. **The construction is not chosen**, and the candidates
differ in what they cost: a tagged hash of an encryption key needs the key; a blinded index needs an
agreed rotation epoch and unpredictable-but-agreed randomness; a double hash gives only
anonymity-over-a-prefix.

⚠ ⭐ **The cost to check first, because it may be small:** *per-namespace is the default, and a
namespace under a published root is already public.* **If the sharp per-hash case is rare enough, this
item may be deferrable rather than blocking** — but that is an argument to make explicitly, not to
assume.

### Item 7 ⛔ The abuse bound — the shape is now known, the numbers are not

**The attack is real and has been run:** a shared structure that anyone may append claims to, about
subjects who did nothing, with readers paying the cost — that is a deployed, CVE'd failure in another
ecosystem, and the flooding problem was *known for years with proof-of-concept attacks* before anyone
ran it.

✅ **This design has the structural defence by construction:** a claim lives in **the issuer's own
table**, signed by them, and the aggregator does not re-sign — **so a poisoner's million claims inflate
the poisoner's own table, not the victim's.**

⛔ **The exposure is exactly where that stops being true: an aggregator unioning into a subject-keyed
index.** ⇒ ⭐⭐ **the rule: an aggregator bounds contribution PER ISSUER, never per key.** A per-key
bound is unimplementable against a flood (the attacker picks new keys) and punishes the popular hash.

**Three mechanisms, all cheap, none numbered yet:**

| mechanism | what it does | status |
|---|---|---|
| **per-issuer volume bound** | a source that suddenly asserts far more than usual is dropped automatically. **Twenty-five years deployed** in the largest instance of this model | ⭐ **required**; thresholds unknown |
| **corroboration as ORDER** | entries asserted by several independently-chosen sources are tried first | ⭐ cheap — it is a sort order. ⛔ **never an admission gate**: a rare hash is known to one source by definition |
| **consent to be listed** | the subject marks a third-party claim as attested. **The only mechanism that proves consent rather than possession** | ⚠ **shape known, unspecified** |

⚠ **And the honest ceiling, which should be stated in the spec rather than discovered:** *between
compromise and finding out, a trusted source can do unbounded damage.* The volume bound is what closes
that window. **After thirty years the equivalent system's posture is layering plus monitoring, not a
solution** — anyone expecting attack-proof should read that as the answer.

### Item 8 `[rev 4, NEW]` ⚠ Does fetching make YOU a holder? — the resource on call 5 decides

**`EXTENSION-CONTENT` §6.4.1's production topology binds on ingest, and binding is what makes a hash
servable under a namespace.** The substitute chain ends in `ingest(bytes, hash, target_namespace_from_cap)`.

⇒ ***So whether a successful fetch makes you a holder is decided by the `resources` scope on the ingest
grant, and nothing else.*** Two positions, both legitimate, and a deployment must be able to pick:

| | |
|---|---|
| **ingest into a served namespace** | you become a holder by using the system. **This is what makes the mechanism compound** — the holder set grows with traffic, which is the property that dissolves *"a locator only names peers the publisher knew about"* |
| **ingest into an unserved namespace** | you get the bytes and answer for nothing. **Also correct**, and it is what a device with no wish to serve should do |

⚠ **What must NOT happen is for this to be decided accidentally, by whichever cap the triggering request
happened to carry.** ⭐ **It is a deployment posture and it deserves to be stated as one** — and it is
the cleanest example in the whole design of the principle §6a.3 rests on: ***where the answer differs
between one household and a public archive, the answer is a grant, not a specification.***

---

## §8 What is deliberately NOT proposed

- ⛔ **No universal-reach guarantee.** A keyspace-partitioned overlay offers *any node finds any
  published key in `O(log n)` hops with no prior relationship*; this offers **reach bounded by your
  table graph.** ⭐ **The trade is deliberate and belongs in the spec as a stated limit** — the
  guarantee being given up is one the field measured as **not kept** (a large majority of provider
  records unreachable), and **an unkept guarantee is worse than an honest absence, because systems get
  designed against it.**
- ⛔ **No answer for a truly bare hash with no reachable table that carries it.** If nothing you can
  reach knows `H`, you do not find `H`. **This is a stated limit, not an open gap** — it has been
  re-opened repeatedly precisely because it kept being filed as a question.
- ⛔ **No global content-anchored search.** Already ruled out; still ruled out.
- ⛔ **No reputation system.** What replaces it: **your own adoption decisions**, provenance so a bad
  entry is attributable, and a volume bound. A source producing bad hints **falls out of readers'
  precedence lists** — not by a score, but because each reader independently stops consulting it, with
  no coordination. ⚠ **This is three-quarters of a self-regulating system; the missing quarter is the
  compromise window, which is what the volume bound is for.**
- ⛔ **No erasure-coding story.** A claim over whole objects cannot express *3 of 10 shares*. The known
  resolution is to code first and address the shares, which leaves this design unchanged and pushes
  the question one layer down. **Named, not solved.**
- ⛔ **No placement policy.** *How many copies does this data want* is a separate mechanism from *who
  has it*, and conflating the two is what makes placement inexpressible. Out of scope here.

---

## §9 Open questions

| | |
|---|---|
| **Q1** | ⛔ **Pruning cost at aggregator scale** — genuinely unmeasured, and the largest remaining unknown in the design |
| **Q2** | **Who pays for refresh?** Expiry makes claims soft state; the re-assertion cost falls somewhere and it has not been costed |
| **Q3** | **Does the fidelity axis belong in v1?** A lossy, false-positive-admitting summary is a legitimate published table, is cheaper, and **cannot be read back out as an inventory** — but it has no per-entry attribution, so it is publishable by its holder only and never merged from third parties. **Cleanly separable; probably not v1** |
| **Q4** | **Does the aggregator wait for deferred federation machinery?** §8.2's meta-registry pattern is deferred on relay-mode text — but §8.2's own correction records that the underlying cross-peer mirror is specified and cross-implementation verified, so ***federation is unblocked; it is undesigned***. Which path the locator aggregator takes is a choice, not a constraint |
| **Q5** | ⚠ **Citation hygiene, and it affects this document.** The closure property is `SYSTEM-DATA-EXCHANGE` **§1.1**; the five further layers — subject, source, witness, position, intent — are **specified and not yet folded**, so their section numbers resolve only in the source proposal. **Anything routed to another party must cite the landed section or say plainly that it is citing unfolded work** |

---

## §10 What a conformance check set must discriminate

**Not written here, and deliberately** — check sets are an output of cross-implementation convergence,
not an input to a first draft. **What this proposal owes instead is the list of requirements a check
set will have to satisfy**, and it is short because the mechanism is small:

1. **A bare-hash miss with no locator backend installed returns exactly what it returns today.** The
   optionality claim in §2 is falsifiable and must be falsified.
2. **A wrong locator entry costs one fetch and nothing else** — the bytes fail their hash, the chain
   advances, and the final answer is identical to the answer with no entry at all.
3. ⭐ **A reader that ignores every locator hint reaches the same answer as one that uses them, or
   fails.** The hint is never load-bearing. *(This discipline is already stated for reference hints and
   transfers verbatim; a logged production instance in another ecosystem shows the failure it
   prevents — a false positive that returned an error instead of falling through.)*
4. **An expired claim is not served, not merged, and not counted.**
5. **An aggregated entry verifies against its original issuer, not against the aggregator.**
6. **A referral walk terminates** under a hop budget, including on a cycle.

---

## §11 What this revision establishes, and what it does not

✅ **Establishes:** the layer map (§0); that the mechanism is optional and must stay so, with an
enforcement point rather than an assertion (§2); that the resolution layer is a sequenced registry
backend rather than a new subsystem (§4); that the consumption layer is **three clauses** in an existing
extension rather than a new path (§6); **the two MUSTs the aggregation layer already carries** (§3.5);
and **seven named seams**, five of which are collisions with landed text (§7).

⭐ **`[rev 2]` And one thing an adversarial audit did NOT find, which is the more useful result because
it was the thing most likely to be wrong: no finding contradicts the central claim that this is a
registry backend consumed through the substituter's miss path.** Every finding is about *what that
costs* and *what it must carry* — **none is about whether the join is real.** The join held.

✅ **`[rev 3]` also establishes:** that the grant is **expressible in the existing capability system**,
with the dimension, the enforcement point and the default all already normative (§6a); that the
own-devices, group and public cases are **one mechanism at three grant scopes** (§6a.3); that
**consent-to-be-listed is the already-specified *Presented credential* arm** (§6a.4); and that **the
subject is a predicate rather than a hash** (§1.3).

⛔ **Does not establish:** the schema, the derived-key construction, the volume-bound thresholds, the
cardinality question, or the `holders`-list growth ruling. **None of those can be settled by assertion
here.**

⭐ **`[rev 3]` And there is no longer an item that blocks writing normative text.** Revision 2's two
blockers are closed — one withdrawn, one handed back to the deployment where it belongs. **What remains
is fill-in work plus one measurement** (pruning cost at aggregator scale), and **one routable defect
against landed text** that is somebody's to fix rather than this design's to wait on (§6a.5).

---

## Document history

- **2026-09-13 · revision 4** — **does the grant call by call (§6b)**, on the principle that no
  dimension is readable alone: five calls, of which **only one is outbound**, with handler / operations
  / resources / peers named per call. **Corrects revision 3's conflation** of a resource path pattern
  with a peer identity (§6a.3), states the distinction and its deliberate overlap (§6b.1), shrinks
  `constraints` to what no dimension covers (§6b.3) — where **`locator_issuers` versus `peers` is the
  trust/verification split appearing as two separate fields** — and files **two places the core spec
  blurs the same distinction** (§6b.4). **Adds §1.4: the traversal is semantic, never in hash space.**
  Two consequences recorded: **adopting a table and consulting it are different grants**, and **call
  5's resource decides whether fetching makes YOU a holder**.
- **2026-09-13 · revision 3** — closes both of revision 2's blockers, in opposite directions.
  **The capability grant is expressible today** (§6a): a substitute fetch at a candidate peer is
  internal dispatch, and the `peers` dimension exists for exactly that class, MUST-evaluated before the
  sub-dispatch leaves, fail-closed when absent. Revision 2's item 1 is **withdrawn**; what survives is
  that `EXTENSION-SUBSTITUTE` §8 **shadows** Dimension 4 with a weaker `constraints` key (§6a.5) — a
  routable defect, not a blocker. **Item 2 is reframed**: whether to act on a third party's claim is
  **policy owned by the deployment**, carried by the same grant at three very different scopes
  (§6a.3), and consent-to-be-listed turns out to be the already-normative *Presented credential* arm
  (§6a.4). **And §1.3 widens the subject to a predicate** — the hash question was a probe; the exchange
  structure it revealed is the deliverable.
- **2026-09-13 · revision 2** — folds the adversarial audit
  (`REVIEW-THE-LOCATOR-ASSEMBLY-AUDITED-AGAINST-LANDED-TEXT`). **Three new blockers**: the consult-cap
  cannot express a locator-supplied source (§7 item 1, now ranked first); `cap_denied` aborts the whole
  chain so one hostile claim kills a lookup (Delta 3); and the aggregation layer carries two MUSTs the
  first revision omitted (§3.5 — byte-preserving carriage and the growth rule's paging). **The tier
  argument's reasoning is replaced** (§2) — the conclusion survives, the justification did not. **The
  central claim survived the audit unchanged.**
- **2026-09-13 · revision 1** — created as a **provisional** assembly. Answers the layering question,
  inventories what already exists against what is new, and files the six seams. **Not routable; not
  ratifiable.**
