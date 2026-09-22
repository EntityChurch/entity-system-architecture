# EXPLORATION — Willow, read against its spec: how close we actually are, and the real distinctions

**Status:** Exploration (design record). Not a proposal, not normative.

**Operator-requested:** *"I'm curious about how close we are to Willow and Hyper-G and what's the real
distinction — that one seems like we really need to dig into."*

**This supersedes the Willow section of `EXPLORATION-THE-CONVERGENT-DESIGNS-…` §1**, which was
written from Willow's *comparison page* and flagged itself as such. **Two claims there are corrected
here** (§1.3, §4.2) and both corrections make the comparison sharper rather than softer.

**Sources opened this session:** `willowprotocol.org` **Data Model**, **Meadowcap**, and **Confidential
Sync** specifications. **Still unopened:** the encodings spec, the sideloading spec, and
`willow-rs`/`willow-js` source. Claims marked *(our reading)* are inference.

---

## §0 The one-paragraph answer

**Willow and this design agree on the shape of the problem and differ on exactly three things, and
all three follow from one root difference: Willow has one naming layer, we have two.** A Willow
Entry is named by `(namespace, subspace, path)` and *carries* a digest of its payload; our tree names
by `path` and *binds to* a content hash, with a second store keyed by that hash. **Their digest
verifies; our hash both verifies and names.** From that one difference falls: **dedup** (ours, free;
theirs, not a protocol concept), **traceless deletion** (both have it, by opposite mechanisms), and
**how a sync works** (theirs symmetric and therefore needing set reconciliation; ours asymmetric and
therefore needing only a root comparison).

**And one thing they have that we do not, correctly scoped after the operator's correction:** read
capabilities enforced *at the replication layer*, with a private-set-intersection handshake so that
**what you are interested in** does not leak. **That is a live-peer-session property and it is not
comparable to our public HTTP backend**, which is a deliberate catch-all read grant. §5.

---

## §1 The data model, exactly

### §1.1 An Entry is metadata, and the payload is separate

```
struct Entry {
  namespace_id:   NamespaceId,
  subspace_id:    SubspaceId,
  path:           Path,
  timestamp:      Timestamp,
  payload_length: U64,
  payload_digest: PayloadDigest,
}
```

- **Path** — *"a sequence of at most `max_component_count` many bytestrings, each of at most
  `max_component_length` bytes."* Bytestring components, not a string with a separator.
- **Timestamp** — a `U64`, recommended as **microseconds in TAI since J2000**.
- **`payload_digest`** — *"the result of applying `hash_payload` to the Payload."*
- A **namespace** is the logical join of all stores sharing a `NamespaceId`; a **subspace** is all
  Entries of a given `SubspaceId` within one.
- An **AuthorisedEntry** is an `(Entry, AuthorisationToken)` pair for which `is_authorised_write`
  returns true.

### §1.2 The correction: Willow does not eschew content *hashing* — it eschews content *addressing*

**`EXPLORATION-THE-CONVERGENT-DESIGNS-…` §1.2 quoted the authors' *"the only one to completely eschew
content-addressing on the protocol level"* and did not unpack it. Unpacked, it is narrower and more
interesting than it reads.**

`payload_digest` **is a content hash** and it sits in every Entry. What Willow declines is **naming
things by that hash.** Nothing in the protocol is looked up by digest; lookup is
`(namespace, subspace, path)`, and the digest is there so a receiver can verify the payload it was
handed.

> **The precise distinction: Willow is content-*verified*. We are content-verified **and**
> content-*addressed*.** Their digest answers *"are these the right bytes?"* Ours answers that **and**
> *"where do I find these bytes, and do I already have them?"*

**This is the operator's sentence, and it is the whole trade:** *"if you want de-duplication, we do
have the entity fidelity."* **Dedup is what the second naming layer buys, and it is the thing Willow
gave up.** Two Willow Entries at different paths with identical payloads are two payloads as far as
the protocol is concerned; the store may dedup underneath, but no peer can *ask* for a payload by
digest, so nothing dedups **across the wire or across peers.**

**What they bought with it: the bootstrapping property.** *"Pure content-addressing faces a
bootstrapping problem: in order to actively distribute data to others, I must first inform them about
the hash of that data… After agreeing on an initial name, I can continuously bind new data to the
name, which automatically reaches those interested."* **That is a correct criticism of IPFS and it is
not a criticism of us** — our `path → hash` binding under a signed root is exactly *continuously
binding new data to an agreed name.* **We took both properties; they took one and were explicit about
which.**

### §1.3 Deletion — same property, opposite mechanism, and theirs has a cost ours does not

| | Willow | Us |
|---|---|---|
| Mechanism | **prefix pruning** — write an Entry whose `path` is a **prefix** of existing entries with a **newer timestamp**, and everything under it is deleted | **unbind the path.** The trie is canonical *"regardless of insertion or deletion history"*, so the result is byte-identical to a tree that never held it |
| Trace left | none | none |
| Requires | a **trusted clock** — the deletion is a *write* and it wins by timestamp | nothing — it is a tree edit, and the signed root moves |
| Granularity | a path **prefix**, so deleting one leaf means writing at that exact path | any single binding |
| Side effect | **deletes descendants**, which is the feature (directory delete) and the footgun | none |

**Both deliberately give up completeness proofs to get this**, and both say so. Willow: *"deliberately
avoids any cryptographic proofs of completeness, allowing for traceless data removal."* **That
convergence stands and it is the strongest single agreement between the two designs.**

**The distinction worth carrying: their delete is a write, ours is an edit.** A Willow deletion is
itself an Entry that replicates, so it *propagates* — a peer that syncs later learns the thing is
gone. **Ours does not propagate; a peer holding an old root simply never learns.** Theirs is better
for convergent removal across a swarm; **ours is better for the curation case**, because a removal
that propagates is a signal you removed something, and *"a removal leaves no trace"* was the point
(`BEARINGS-2026-09-04` §3 result 4). **Neither is strictly better and the difference is not
accidental — it follows from symmetric-sync versus asymmetric-publish.**

### §1.4 The timestamp is load-bearing for them and presentational for us — and this is where they pay for multi-writer

Willow resolves overwrite at a `(subspace_id, path)` by **newer timestamp wins**. That makes a
**wall clock a correctness input**: two writers (or one writer on two devices) with skewed clocks
resolve wrongly, silently, and permanently.

**We deliberately went the other way.** `PROPOSAL-APP-CONVENTION-FEED` §2.3.x: a timestamp is *"what a
renderer shows and sorts by when it has nothing better; the causal facts are `prev`."* **We have no
overwrite conflict to resolve, because a tree has one signing key and one root sequence.**

> **But that is also a gap we have not named, and Willow found it first.** Their stated reason for
> rejecting append-only logs is that a hash chain *"precludes the option of concurrent writes from
> multiple devices."* **They then answer multi-device with last-write-wins on a wall clock.** We
> reject the chain for a different reason (retroactive-edit cascade) and **we have not answered
> multi-device at all** — two devices sharing one key both advancing one signed root sequence is an
> unspecified race. **Open, and it is a real one:** §7.1.

---

## §2 Meadowcap against our capability model

### §2.1 Theirs, exactly

```
struct CommunalCapability {              struct OwnedCapability {
  access_mode: read | write                access_mode: read | write
  namespace_key: NamespacePublicKey        namespace_key: NamespacePublicKey
  user_key: UserPublicKey                  user_key: UserPublicKey
  delegations: &[(Area, UserPublicKey,     initial_authorisation: NamespaceSignature
                  UserSignature)]          delegations: &[(Area, UserPublicKey, UserSignature)]
}                                        }
```

- **Receiver** = *"the final `UserPublicKey` in the delegations, or the `user_key` if the delegations
  are empty."*
- **Area** = a restriction by **subspace × path-prefix × timestamp-range**.
- **Delegation** = a capability holder may *"mint new capabilities for the same resources but to
  another peer"* through cascading signatures, narrowing the Area.
- **Two namespace kinds.** **Owned:** *"the person who created the namespace is the owner of all its
  data"* — access needs the namespace keypair. **Communal:** *"each subspace is owned by a particular
  author"*, proving ownership by signature over their own subspace's data.

### §2.2 The axis comparison, and we are not simply "ahead"

`EXPLORATION-THE-CONVERGENT-DESIGNS-…` §1.4 said *"we are strictly more expressive."* **That is true
on one axis and false on two, and the corrected reading is better.**

| Axis | Meadowcap | Ours |
|---|---|---|
| **What is scoped** | **data** — which Entries | **dispatch** — which handler, which operation, which resource path, which peer |
| Verb granularity | **two**: read / write | **arbitrary**: `operations` is an `id-scope` over named operations |
| Path scoping | prefix-based Area | `path-scope` with **`include` AND `exclude`** |
| **Time scoping** | **first-class** — a timestamp range is one of the three Area dimensions | **absent from the grant.** We have `max_delegation_ttl` (how long a *delegation* lives) but no *"you may read entries from March"* |
| Delegation | cascading signatures, Area narrows | strict narrowing (**child MUST retain all parent constraint keys, MUST NOT add keys**) + `delegation-caveats` (`no_delegation`, `max_delegation_depth`, `max_delegation_ttl`) |
| **Enforced during sync** | **yes** — read caps gate replication | **no** — enforced at dispatch, which is the same thing for a live peer and does not exist for the static backend |
| Namespace ownership model | **owned vs communal, as a first-class protocol distinction** | one model: a namespace is its key-holder's |

**Three real findings there:**

1. **We are richer on the verb and the exclude.** *"You may call `tree:get` but not `tree:put` on
   `x/*` except `x/private/*`"* is not expressible in Meadowcap, and it is ordinary for us.
2. **They are richer on time, and we have nothing.** A grant scoped to a time range is a genuinely
   useful thing we cannot say. *(Whether we want it is a separate question — our entries do not carry
   an authoritative timestamp, per §1.4, so a time-scoped grant would be scoping on a field we treat
   as presentational. **That may be the honest reason we do not have one**, and if so it is worth
   writing down rather than leaving as an absence.)* — §7.2
3. **Owned-vs-communal is a framing we should steal outright.** Their rationale is the best sentence
   either project has on the subject: owned namespaces exist *"to mimic the curative function of a
   fediverse instance host without tying it to computational resources"*, while communal namespaces
   are *"a model that couldn't realistically be replicated in the fediverse unless everybody ran their
   own server."* **That is our capstone's use case 1 and use case 3, named better than we named
   them** — and *curation without hosting* is exactly what our mirror is.

---

## §3 Sync — and this is the largest structural difference

### §3.1 Theirs is symmetric; ours is asymmetric

**Willow's Confidential Sync is two peers reconciling.** Neither is authoritative. So the protocol is:

1. **PIO** — each peer sends **salted hashes** of its `PrivateInterest` (namespace + area). Matching
   hashes reveal an overlap; *"peers can discover common interests without disclosing any non-shared
   information to each other."*
2. **Read capability exchange** — only for the confirmed overlap.
3. **3D range-based set reconciliation** — fingerprints via `hash_lengthy_authorised_entries` over
   ranges in `namespace × subspace × path`; recursive subdivision where fingerprints differ.
4. **Entry and payload transfer**, plus eager forwarding of new entries afterwards.

Over **five logical channels** with independent flow control (Reconciliation, Data, Overlap,
Capability, PayloadRequest), each using resource handles, *"so as not to overload each other."*

**Ours is a reader pulling from a publisher.** The publisher is authoritative for its namespace. So:
fetch the signed root, compare the sequence, walk the changed interior nodes. **Measured at roughly
ten kilobytes and a handful of nodes even while the publisher rewrote most of their estate**, and
unchanged means no further work.

> **The sharp sentence: Willow needs set reconciliation because its namespaces are multi-writer. We
> need only a root comparison because ours are single-writer.** Set reconciliation is what you build
> when *neither side knows what the answer is supposed to be.* A signed root is what you fetch when
> **one side does.**

### §3.2 …and this is exactly where the pocketed RBSR item comes back

**Our design becomes multi-writer in precisely one place: the forum** — capstone §4.2's *"a thread is
a grow-only set, and merging two views is set union."* **That is the shape RBSR is for, and it is the
only part of our system that has it.**

So the honest positioning of the pocketed item is now much more specific than *"do bloom filters
help?"*:

- **Publish-and-follow (use cases 1 and 2) will never want RBSR.** A root comparison is cheaper and
  strictly better, because one party is authoritative.
- **The forum (use case 3) is where two peers hold overlapping partial views of one grow-only set and
  neither is authoritative.** That is Willow's problem statement verbatim, and RBSR is the published,
  analyzed, twice-deployed answer.
- **And our trie is already a range-summarizable ordered store**, which is RBSR's one structural
  requirement.

**Still pocketed** — it sits behind the vectors and the two seats' reviews — but it is now a
*targeted* item with a named consumer, not an exploratory one. §7.3.

### §3.3 The sideloading axis we have no equivalent for

Willow ships a **sideloading** specification: transferring a set of Entries out of band — a USB
stick, a file, a courier — such that the receiver can verify it without ever contacting the author.

**We can do this and have never specified it.** A signed root plus the interior nodes plus the
entities is a self-verifying bundle by construction; nothing in our verification path requires a live
connection. **A "publish to a file, verify from a file" convention is close to free and we do not have
one.** Filed — §7.4. *(This is also the honest answer to a category of question — sneakernet,
air-gap, archival hand-off — that the corpus has never been asked.)*

---

## §4 The scorecard, corrected

| Property | Willow | Us | Note |
|---|---|---|---|
| identity = key | ✓ | ✓ | |
| data named, not only hashed | ✓ | ✓ | the convergence that matters most |
| content **verified** by digest | ✓ | ✓ | |
| content **addressed** — lookup by digest, dedup across peers | **✗ by design** | **✓** | §1.2 — the root difference |
| traceless deletion | ✓ (prefix prune, propagates) | ✓ (unbind, does not propagate) | §1.3 — same property, opposite mechanism |
| overwrite needs a trusted clock | **✓ — yes, it does** | ✗ | §1.4 |
| multi-device write | ✓ (LWW) | **undesigned** | §7.1 |
| capability: verb granularity | read/write only | arbitrary operations | |
| capability: time scoping | ✓ | **✗** | §7.2 |
| capability: exclude sets | ✗ | ✓ | |
| **read caps enforced during sync** | **✓** | live peer: equivalent at dispatch · static backend: **N/A by design** | §5 |
| **interest hiding (PIO)** | **✓** | **✗** | §5, §7.5 |
| symmetric reconciliation | ✓ (3D RBSR) | ✗ (root compare) | §3.1 — a difference, not a gap |
| offline/sideload transfer spec | ✓ | **✗** | §3.3 |
| owned vs communal namespaces | ✓, first-class | one model | §2.2 finding 3 |

---

## §5 The metadata-privacy question, corrected — the operator's point, and it holds

`[operator, 2026-09-04, paraphrased: "there are two different things we're dealing with. The HTTP-poll
public backend is public on the internet — essentially a catch-all capability token for anyone to grab
and use, that's the theory. Whereas running the full live peer with the full dimensional grant, you're
communicating directly with peers — you have all the capability, permission, privacy, whatever you
want; you connect to a peer and give them directly what they access."]`

**`EXPLORATION-THE-CONVERGENT-DESIGNS-…` §8.1 filed metadata privacy as a flat gap. That was
under-specified and the correction changes the finding, not just its wording.**

**We have two modes and Willow has one.** Willow is *only* live peer-to-peer sync; it has no
static-backend rung at all. So the comparison has to be taken per-rung:

| Rung | What it is | What privacy means there | Willow's equivalent |
|---|---|---|---|
| **Static / HTTP-poll backend** | signed root + trie on cheap public hosting | **none, deliberately.** Publishing to a public origin *is* a catch-all read grant; anyone who has the key can walk it. **That is the design, not a leak** | **none — Willow has no such mode** |
| **Live peer** | full four-dimension grants, direct connection, per-peer authorization | real, and already enforced at dispatch — including §9.1's *"listing entries for paths the capability does not cover MUST be omitted"* | Meadowcap read caps at the replication layer |

**So the fair comparison is live-peer ↔ Willow sync, and there the residual gap is narrow and
specific: it is not access control, it is *interest hiding*.** We can already refuse to serve what a
grant does not cover. What we cannot do is let two peers **discover what they both care about without
each revealing what they care about** — Willow's PIO, salted interest hashes matched before any
capability is exchanged.

**When that would matter for us:** a live peer connecting to a peer it does not fully trust, in order
to sync a shared interest, without disclosing its full follow set or its topic list. **That is the
forum case again** (§3.2) — the same rung, the same use case, and the only one where two of our peers
meet as equals. **Everywhere else the question does not arise**, because the reader is pulling from a
public origin and there is nothing to hide from.

**Revised finding:** not *"we lack metadata privacy"* but *"our live-peer rung has no interest-hiding
handshake, and the only use case that needs one is the forum, which is also the only use case that
needs RBSR."* **Two gaps, one consumer, and it is Stage 3.** That is a much better-shaped item than
the one it replaces.

---

## §6 What to take

1. **The owned-vs-communal namespace framing** (§2.2). Better than our use-case-1/use-case-3 language
   and it names *why* — *curation without computational resources.*
2. **Sideloading as an axis** (§3.3). Nearly free for us, unspecified, and it answers a class of
   question nobody has asked yet.
3. **Their honesty about what content-addressing costs** (§1.2). They named the bootstrapping problem
   precisely and chose against it with eyes open. **Our corpus should be able to state the converse as
   crisply: what we pay for the second naming layer.** *(We pay: an entity's identity changes when its
   bytes change, which is why a transform is a new entity — the exact point `PROPOSAL-UNKNOWN-FIELD-
   PRESERVATION-…` §2a now makes.)*
4. **The five-channel flow-control model** (§3.1) as prior art for any future live-sync surface. Not
   needed now; needed the moment two peers reconcile rather than one pulling.

## §7 What this opens

1. **Multi-device publishing is undesigned.** Willow answers it with LWW on a wall clock and pays for
   it with clock trust. We reject the chain for a different reason and have no answer. **Two devices
   sharing one key advancing one signed root sequence is a race nothing in the corpus describes.**
   *Highest-value item in this document* — it is a normal-user scenario, not an exotic one.
2. **Time-scoped grants: absent, and possibly correctly.** Meadowcap has them; we do not. The honest
   reason may be that our timestamps are presentational (§1.4), which would make a time-scoped grant
   a scope over a field we do not trust. **If that is the reason, write it down.**
3. **RBSR has a named consumer now** — the forum, and only the forum (§3.2). Still pocketed.
4. **A sideload/offline-verify convention** (§3.3).
5. **Interest hiding on the live-peer rung** (§5) — same consumer as (3).

## §8 Sources opened

[Willow — Data Model](https://willowprotocol.org/specs/data-model/index.html) ·
[Willow — Meadowcap](https://willowprotocol.org/specs/meadowcap/index.html) ·
[Willow — Confidential Sync](https://willowprotocol.org/specs/sync/index.html) ·
[Willow — comparison to other protocols](https://willowprotocol.org/more/willow_compared/index.html) ·
[range-reconcile](https://github.com/earthstar-project/range-reconcile) ·
[Meyer, *Range-Based Set Reconciliation*, SRDS 2023](https://arxiv.org/pdf/2212.13567)

**Unopened, and named so nobody assumes coverage:** the **encodings** spec (their wire format, which
is the direct counterpart of `ENTITY-CBOR-ENCODING` and the thing a bridge would have to read — see
`EXPLORATION-THE-BRIDGE-…`), the **sideloading** spec, **LCMUX** (the channel multiplexer), and
`willow-rs` / `willow-js` source. **Every §1 and §2 struct above is quoted from the spec; every
comparison to our corpus is this record's own and should be checked by whoever proposes anything from
it.**
