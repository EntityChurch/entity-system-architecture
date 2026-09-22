# EXPLORATION — the convergent article: what `EXTENSION-REVISION` already answers, why it is not a second design, and the ownership axis showing up for the third time

**Status:** exploration. **Nothing ruled, no spec text proposed.**
Predecessor: `EXPLORATION-THE-REPLY-PACKAGE-…` (the locator and the owned coordinate).

> **A note on method.** An earlier reading of this question concluded that `EXTENSION-REVISION` *"can
> host merge strategies and cannot host a CRDT"* from one sentence in the overview, having opened 130
> of ~3,800 lines. **This document opened §1.1–§1.3, §5.4, §7.2, §8.1, §8.7, §11.1 and §11.2 first, and
> reverses that conclusion.** For a spec over a few hundred lines, read its section list before ruling
> on it.

---

## §0 The result

1. **Merge is deterministic and convergence is O(1) — this is landed, normative text.** §7.2: a version
   entry is **`root` + sorted `parents` and nothing else**, so *"concurrent merges of the same inputs
   produce the exact same version entry hash."* Two peers who fold in the same contributions hold the
   **same article at the same hash**, with no coordination and no round-trip. §1.
2. **So the worry about divergence is real but is about a different variable.** Divergence does not come from
   the merge; it comes from **different input sets** — who you follow, accept, block. **The article's
   identity is its input set.** §2.
3. **Which makes disagreement a fork with a named common ancestor** rather than an incommensurable
   second document, and re-merging it is an operation the extension already defines. **"You are not
   going to get one concerted view" is right, and the design's answer is that the views are related in
   a way that is computable.** §2.1.
4. **The two designs are complements, not overlaps, and the seam is clean:** the **walk** converges
   *input sets*; the **version DAG** converges *state*. Neither can do the other's job. §3.
5. **`EXTENSION-REVISION` §8.7.3 independently reached the ownership rule**, months earlier and
   for its own reasons: *"syncing from the authoritative source first — if all peers sync from the
   namespace owner, subsequent cross-peer syncs share a common ancestor."* **That is the ownership rule arriving from
   the other direction.** §4.
6. **The ownership axis gets a third consequence, and it is the strongest one:** for an **owned**
   coordinate a walk's omissions are detectable, because `REGISTRY` §6a.3a's *"the walk is the
   authority, the listing is a menu"* makes a trie walk from a signed root commit to its whole key set
   — *"silently hidden becomes visibly incomplete."* **Bounded carefully in §4.2, because it is easy to
   overstate.**
7. **"Heavy" is about the git-shaped convenience surface, not the convergence core.** §11.2's core is
   **seven operations**; the article case uses four of them. §5.

---

## §1 What REVISION actually guarantees

**§7.2, verbatim and load-bearing:**

> *"**Concurrent merges converge in O(1).** With structural version entries (root + sorted parents),
> concurrent merges of the same inputs produce the exact same version entry hash. The second-order
> divergence that previously required an additional merge round does not occur… This works because the
> version entry contains only deterministic content: the trie root hash (deterministic merge of the
> same inputs) and sorted parent hashes. **No author, timestamp, or message fields differ between
> peers.**"*

**The last sentence is the whole mechanism, and it is a design decision made elsewhere in the
document.** §1.2's *Metadata is tree content*: **the version entry is purely structural, and commit
metadata — author, timestamp, message — is stored as entities at paths in the trie under application
convention.** Most version-control systems put the author and the clock *in the commit*, which makes
two peers' merges of identical inputs differ by construction. **Ours took them out, which is what
buys determinism.**

> is structural; everything human — who, when, why, how it renders — is tree content under app
> convention. **The same sentence that makes the article converge is the sentence that keeps
> presentation out of the protocol.**

**Convergence is stated as a guarantee with four supports** (§7.2): structural entries eliminate
second-order divergence · each merge round produces a descendant of all inputs · the number of distinct
heads is **monotonically non-increasing** · content addressing gives O(1) detection.

**And the one exception is named, with a remedy and a backstop.** Asymmetric strategies
(`source-wins` / `target-wins`) under `caller-perspective` ordering can produce different trie roots.
**Remedy: `deterministic` merge ordering. Backstop: oscillation detection.** So *"the settings are a
bit specific to get that right"* is an accurate description of a specified, bounded caveat — not of a
gap.

**CRDT is available and is not a storage format.** §5.4: a CRDT merge handler computes ops base→local
and base→remote, applies both, extracts merged state, and **returns a regular entity** — *"the
Eg-walker principle: CRDT is a computational artifact during merge, not a storage format."* The causal
record it needs is the version DAG's parent pointers, which is why no CRDT metadata has to persist.

---

## §2 The article's identity is its input set

**Nothing above says all peers reach the same article. It says peers who fold in the same
contributions do.** Those are different claims and the second one is the honest one.

| Variable | Who controls it | Converges? |
|---|---|---|
| **the merge function** | the spec | **yes — same inputs, same hash, O(1)** (§7.2) |
| **the input set** | each reader — follow, accept, block | **no, and it should not** |

**So *"you are not going to get one concerted view"* is correct, and it is a property of the social
layer rather than a failure of the merge layer.** Two peers with the same input set are **byte-identical
and know it by comparing one hash**. Two peers with different input sets hold different articles
**because they read different things**, which is the honest outcome.

### §2.1 What the DAG adds over "everyone publishes their version"

**A fork here is not an unrelated document.** Both heads descend from a shared history, so:

- **`find-ancestor` gives the common ancestor**, and the difference between two people's articles is
  **computable and displayable** rather than a matter of opinion about which file is which.
- **Merging them is an ordinary `merge`**, at any later time, by anybody — including a third party who
  read both and holds neither namespace.
- **§8.7.3 bounds even the worst case.** Two sources with *no* common ancestor conflict on every
  differing path **once** — *"this is a one-time cost — the merge version permanently grafts the two
  DAGs together, providing a common ancestor for all subsequent syncs."*

> **That is the difference between a fork and a schism.** *"Everybody publishes their own version"* is
> true in both; in one of them the versions are permanently incommensurable, and in ours they are
> related by an object anyone can compute. **The design does not promise one view. It promises the
> views stay mergeable**, which is the strongest thing a system without a gatekeeper can promise —
> and it is the same shape as the completeness result, arriving on state instead.

### §2.2 What is genuinely harder about an article than a thread, stated plainly

Not convergence. **Three other things:**

- **Prose is a bad fit for three-way merge**, so the article case leans on §2.3's per-path merge
  configuration and a text CRDT strategy (§5.4). That is a **strategy choice inside an existing
  framework**, and §1.1 already scopes it: real-time collaborative editing is *"application-level, uses
  the merge framework."*
- **There is no namespace owner**, so §8.7.3's mitigation — *sync from the authoritative source first*
  — is **unavailable.** §4.
- **Nobody has written the app convention** that says where an article lives, what its coordinate is,
  and what the per-path merge config should be. **That is the actual missing piece**, and it is an
  `APP-CONVENTION-*`-shaped hole, not an extension-shaped one.

---

## §3 The two mechanisms are complements, and the seam is sharp

store. They do different jobs and neither can do the other's.**

| | **the walk** | **the version DAG** |
|---|---|---|
| Converges | **input sets** — *who contributed to C* | **state** — *what the merged content is* |
| Unit | a grow-only set of pointers | a content-addressed trie root with lineage |
| Merge | **set union**, no conflicts possible | three-way / CRDT, conflicts are entities |
| Anchored at | a coordinate, possibly **ownerless** | a **prefix in one peer's namespace** (§3.1, keyed by prefix hash) |
| Answers | *whose work should I be looking at* | *what does the combined result say* |

**Neither substitutes for the other.** A version DAG cannot tell you a contribution exists that you
never fetched — it converges what you *have*. A walk cannot merge prose — it unions pointers.

> **Stated as one sentence: the walk is how input sets converge, and the version DAG is how state
> converges. The article case needs both, and it needs them in that order.** That is also why the
> arc's work is not a re-derivation of REVISION: it is the *upstream* half that REVISION assumes and
> does not provide.

**The apparent overlap is real and it is one shared primitive, not two designs:** both are
`EXTENSION-TREE` §3's trie, both dedupe by content address, both merge by hash comparison. **That is
the substrate doing its job in two places**, and it is the same reason a registry listing, a published
walk and a version snapshot are all "walk a trie from a signed root."

---

## §4 The ownership axis, for the third time — and REVISION got there first

**`EXTENSION-REVISION` §8.7.3, on reducing the cost of grafting independent DAGs:**

> *"**Syncing from the authoritative source first.** If all peers sync from the namespace owner,
> subsequent cross-peer syncs share a common ancestor."*

**That is the ownership rule — *publish the gathered set anchored at a coordinate you own* — reached independently, for a
different problem, and earlier.** REVISION needed a **graft point**; the walk needed a
**natural publisher**; both answers are *the owner*, for the same underlying reason: **the owner is
the one location every participant can derive without agreeing on anything.**

**And it inherits the same limit, which is now confirmed from two directions.** For an ownerless
coordinate — an article, a topic — there is **no authoritative source to sync from first**, so §8.7.3's
mitigation is unavailable and every pair of independent histories pays the graft. **The ownerless case
is hard in the walk layer and hard in the revision layer, and it is the same hardness.**

### §4.1 The consequence that is new: for an owned coordinate, omission by an intermediary is detectable

`EXTENSION-REGISTRY` §6a.3a — *"the walk is the authority, the listing is a menu"*:

> *"a hostile origin can omit an entry from a served listing undetectably, and **cannot omit a node
> from the walk without the walk failing.** Silently hidden becomes visibly incomplete, which is the
> strongest completeness property a static origin admits of."*

**Because the signed root commits to the whole key set**, a walk of an owned prefix is verifiably
complete **with respect to what the owner published.** So the owner's walk is not merely the
convenient one — **it is the only kind of walk whose omissions a reader can detect at all.**

### §4.2 The bound, because this is exactly where a safety claim gets overstated

**It catches a *serving intermediary* withholding. It does not catch the *owner* omitting.** Alice can
decline to add a reply to her tree, and no walk of her signed root will ever reveal a thing that was
never in it. So:

| Party | Can they omit? | Detectable? |
|---|---|---|
| a CDN / mirror / relay serving Alice's tree | yes, by withholding a node | **yes — the walk fails** |
| **Alice, by never publishing it** | **yes** | **no, and it never will be** |
| anyone, by substituting content | **no** — signatures | n/a |

**`FEED` §4.1's *omit but never substitute* is unchanged and is still the whole protection against the
owner.** What §6a.3a adds is one rung below it, against everyone *between* the owner and the reader.
**Both are worth having and they are not the same guarantee** — and the completeness law still stands: no
cross-author completeness, ever, because a subject nobody owns has no single writer at any scope.

---

## §5 Is REVISION heavy?

**§11.2's core conformance is seven operations** — `commit`, `log`, `status`, `merge`, `resolve`,
`fetch`, `fetch-entities` — plus *store version entities*, *heads at
`system/revision/{H}/head`*, *entries contain `root` and `parents` only*, *`parents` sorted*, and
*three-way merge as default*. **The other twelve operations** (`branch`, `checkout`, `tag`, `diff`,
`cherry-pick`, `revert`, `config`, `merge-config`, `fetch-diff`, `pull`, `push`, `find-ancestor`)
**are convenience, and §11.1 makes core a declared conformance level on its own.**

**The article case uses four of the seven.** `commit`, `merge`, `fetch-entities`, `resolve`.

> **So the weight is the git-shaped ergonomics, not the convergence machinery** — and the convergence
> machinery is four fields (`root`, sorted `parents`) plus a merge cascade. **Nothing here has
> proposed a lighter mechanism for state convergence, and the reason is that this one is already about
> as small as the property allows.** *(What §1.2's "prefix-scoped versioning — you version what
> matters" also means: nothing obliges an article client to version anything else.)*

---

## §6 Where the locator convention lands

**The ruling, recorded:**

- **Not new REGISTRY text and not a new namespace.** The citer **republishes what the registry already
  holds** — the same `system/registry/binding` / transport-profile entities, at their own origin.
  **`REGISTRY` §3 already permits it** (*"aggregators MAY re-publish at their own path"*), which is why
  this needs no extension change.
- **The convention is `APP-CONVENTION-*` plus SDK guidance** — *what a client publishes alongside a
  citation, and the consumer-side preference order.*
- **The registry stays the agreed-upon namespace**, and this does not compete with it.
- **It may graduate.** *"Normative application extensions… participating in the network the way we
  expect, so you can build your own client and interoperate"* — the W3C-shaped layer. **Not now, and
  the trigger is refinement, not a date.**

### §6.1 "Anyone can be a registry" is already specified

**This needs nothing built. Three landed clauses already say it:**

- **`REGISTRY` §5:** *"No `system/capability/registry-publish-binding-for-name-X` cap exists. **Anyone
  can publish a binding claiming any name. Receiver policy decides.**"*
- **`REGISTRY` §4.1 / §4.1a** — precedence order and the dispatch list are **the consumer's config**,
  so *"I will read what Alice says without putting her in my normal lookup chain"* is a resolver-config
  arrangement and not a protocol question.
- **`REGISTRY` §6a.3a** — a registry's name set is a trie walk from its signed root, so **Alice being
  browsable costs her a published prefix and nothing else.**

**So the registry is an entry point and a browsable structure, not a prerequisite for exploring the
network** — which is §3 of the predecessor document (a citation carries the locator, so traversal needs
no registry at all).
**Both halves check out and they are the same conclusion.**

---

## §7 What this leaves open

- **The article app convention.** Where does an article live, what is its coordinate, what is the
  per-path merge config, and who publishes the walk when nobody owns the coordinate. **The
  extension-layer machinery is present; the convention is not written.** *(And the gathered set's
  `coordinate` cannot yet name a namespace or a path, which this case needs.)*
- **Ownerless coordinates, still.** Third document in a row to arrive here. **It is now clearly the
  single hardest open item in the whole arc**, and it is hard in two layers at once (§4).
- **The minimum interoperable extension set** — *"what gets you interoperable: content, registry,
  identity, group, revision, compute?"* **This is a well-posed and answerable question and nothing in
  the corpus answers it today.** Named here, not attempted; it deserves its own pass and it is
  probably the next one.
- **Private groups.** *"Encrypt once, per-member keys, lock someone out by restructuring the
  cryptography. Recording only that **Shamir sharing is the
  wrong primitive for the stated requirement** (it splits one secret among parties who must cooperate,
  not grant N independent revocable accesses) and that the literature to read is broadcast /
  key-wrapping and group-rekeying — **and that no claim about it should be made before that reading**,
  and the primary-source reading is a precondition of ruling inside a design space.
- **Nothing here is a proposal.** The one thing that arguably *should* become one is §6's convention,
  and that waits on the extension-set pass so it is written once.
