# PROPOSAL — the absence of a node is never an answer

**Status:** DRAFT
**Tier:** extensions — `EXTENSION-TREE` §3.5, §6.2; `EXTENSION-REGISTRY` §6a.3a, §6a.6, §11.1
**Origin:** `entity-browser-rust` `a0145a7` (F8), measured. Root cause is in our text, not theirs.
**Read at:** arch `639b6b3` · browser-rust `a0145a7` · core-rust `ed993bb` · core-go `88615f6` ·
core-py `79f38a9` — every claim re-derived from those trees, not from the filing seat's report.

---

## §0 Summary

`EXTENSION-REGISTRY` §6a.3a asserts, as landed normative text, that a hostile origin *"cannot omit a
node from the walk without the walk failing."* **No implementation has that property**, and the walk
pseudocode it depends on never specifies the branch.

The first draft of this proposal treated that as a completeness/tolerance trade-off and proposed two
walk modes. **That was wrong, and the reason it was wrong is the interesting part:** it assumed a
publisher serves a *filtered view of one large tree*, so incompleteness could be legitimate. **The
substrate does not work that way.** A publisher serves a **re-rooted tree built over exactly what is
published**. There is no legitimate incompleteness to accommodate, so there is no trade-off — there
is one invariant that three impls do not hold.

## §1 The design question, answered from the substrate

The practical tension is real: *a peer's tree holds everything, publishing is an implicit read-grant
to anyone, and different audiences get different subsets.* Nobody publishes their whole tree.

**The substrate already resolves this by re-rooting, not by filtering, and that is the stronger
design.** Two mechanisms, both landed:

- **`EXTENSION-TREE` §3.4 — a tracked prefix is its own trie.** `build_trie` runs over *only* the
  bindings under that prefix, keys stored **relative** to it, root tracked at
  `system/tree/root/{P}`. §3.3a signs that root and records its `prefix`. *(core-rust `ed993bb`,
  `core/peer/src/published_root.rs`: the prefix is "derived from the tracker" — a per-prefix trie,
  not the universal root.)*
- **`EXTENSION-TREE` §6.2 — `execute_extract` rebuilds.** `root_hash = build_trie(bindings)` over
  exactly the selected set, then `include_trie_nodes` bundles **every** node reachable from that new
  root.

**Publishing a subset means constructing a new trie containing only that subset.** Four consequences,
each of which is the property the operator's requirement actually needs:

1. **Nothing about the unpublished tree leaks** — not keys, not shape, not counts. Private bindings
   were never in this trie. A *filtered view* of a big trie would leak all three: a withheld `Link`
   still proves a subtree exists at that hash-prefix position.
2. **Node completeness is verifiable.** The publisher built the trie over exactly what they
   published, so a walk that meets an unresolvable child means **the origin is withholding** — full
   stop. **This is a claim about nodes and it does not extend to keys** (`entity-browser-rust`,
   `0e73f68` §5, and they are right): a clean `not_found` says *this trie does not contain K*, never
   *the publisher does not bind K*. Property 4 below makes that gap matter, and §4a rules it.
3. **Private churn does not move the published root.** No activity oracle, no spurious republish, no
   `seq` burn on changes the audience cannot see.
4. **Multi-audience is native.** N tracked prefixes = N tries = N independently signed roots.
   Content-addressed dedup shares the blobs underneath at no extra cost.

**So the design is right and needs no new mechanism.** What it needs is for the spec to stop
contradicting it in one paragraph, and for the walk to enforce it.

## §2 The contradiction — v3.x residue in §6.2

Directly beneath the rebuild algorithm, §6.2 says:

> *"When the `paths` filter is specified, the envelope includes only the trie nodes along the paths
> from the filtered paths to the root… **Unfiltered subtree nodes MAY be omitted.**"*

**This describes navigating the original trie by path — the v3.x model.** §13 records that model as
replaced: *"Prior versions (v3.x) used a path-keyed compressed trie **where prefix queries DID
navigate the trie**."* Under v4.0 hash-keyed routing, paths do not navigate the trie, and the
algorithm four lines above **rebuilds** — so in the rebuilt trie there are no "unfiltered subtree
nodes" to omit.

At best the sentence is vacuous. At worst it licenses an implementer to build `extract` as a **view**
rather than a rebuild, emitting an envelope that is incomplete against its own root — and two impls
reading it differently produce envelopes that disagree, which is the cross-peer `MAY` divergence
`AGENTS.md` says to pin rather than leave.

**It is also the sentence that made the walk defect look defensible.** If incompleteness can be
legitimate, a tolerant walk is reasonable. It cannot be, so it is not.

## §3 The measurement — what the three engines actually do

| impl | commit | site | behavior on a declared-but-absent child |
|---|---|---|---|
| rust | `ed993bb` | `collect_bindings_into`, `core/tree/src/trie.rs` | `Entry::Link(h) => if let Some(sub) = load_trie_node(store, *h) { … }` — **no `else`**; returns `BTreeMap`, not `Result` |
| go | `88615f6` | `collectBindings`, `core/tree/trie.go` | `child, ok := LoadTrieNode(…); if !ok { continue }` |
| py | `79f38a9` | `_load_node` → `_collect_reachable_tuples`, `storage/trie.py` | `_load_node` returns `_empty_node()` when the entity is missing |

py states the governing assumption at the site: *"the trie's root contract guarantees every reachable
hash IS a trie node entity."* **That is exactly right for a locally-built trie and exactly wrong for
one fetched from a counterparty who chooses what to serve.**

Measured (`entity-browser-rust`, 24-name registry, one interior node withheld): **1 of 24 names
hidden, no error.** The signature still verifies — it commits to the root hash, which is intact — and
every binding returned is genuine. Two aggravations:

- **The return type has nowhere to put the miss.** A caller that *wanted* to check has nothing to check.
- **The hidden name stays resolvable.** Its `by-name` key can live in a surviving branch, so a
  consumer who already knows the name resolves it while the browse says it does not exist.
  **Two surfaces disagree and neither complains.**

## §4 The rule — one invariant, not two modes

> **R1 — the absence of a node is never an answer.** Every trie operation MUST distinguish:
> **(a)** a node that **resolved** and did not contain the sought key → `not_found`, *an answer*;
> **(b)** a node that **did not resolve** → **`tree/incomplete-walk`**, *a failure*.
> An implementation MUST NOT return (a) when it observed (b).

This is one rule and it covers every instance of the seam, at both depths:

- **Enumeration** — a withheld interior node stops shortening the result silently.
- **Lookup** — including `EXTENSION-REGISTRY` §6a.6's revocation check. *"No
  `by-target/{hex(binding_hash)}` key"* is precisely (a)-vs-(b) at leaf depth; §6a.6 already
  concedes *"an absent key and a withheld key are byte-identical at the consumer"*, which is this
  invariant stated as a limitation instead of enforced as a rule.

**R2 — completeness is structural, never heuristic.** A walk MUST derive children by decoding the
node (`system/tree/snapshot/node`) and reading its declared entries — every `Bucket` tuple's
`value_hash` and every `Link`. It MUST NOT derive them by scanning node bytes for hash-shaped
windows: a byte scan cannot tell *not declared* from *declared and withheld*, which is the whole
distinction. **This is the defect browser-rust found in their own publisher** — their closure walk
enqueued a child only when `fetcher.content(&c).is_ok()`, so `missing` was reachable for exactly one
hash in the tree, the root, and a guard recorded as landed since B14 **could not fire**.

**R3 — the error names the parent, not only the child.** `tree/incomplete-walk` MUST carry the
unresolvable child hash **and** the hash of the node that declared it. The cut point is what bounds
which keys are unaccounted for; a consumer holding only the child hash cannot locate the cut.

**R4 — a partial result is an opt-in with a name that says what it costs.** The tolerant behavior
stays reachable for the case it is right for: a walk over a store **the walker itself populated** —
a partial local store, a peer mid-sync, §6.2's `included` semantics. It is requested explicitly, it
returns a subset, and it **MUST NOT** be presented as complete. **The discriminator is the trust
boundary, not a flag**: a walk over bytes a counterparty chose to serve has no partial mode.

**L12 — can the actor reach the inputs?** Yes, and it is worth stating because R1 assigns work to a
consumer. The walk needs (i) the node bytes it is already fetching — and their absence *is* the
signal, observed locally as a 404 or a non-node entity; and (ii) the declaring parent's hash, which
the consumer holds because it just decoded that parent to learn the child existed. **No new index,
no new request, no key material, no cooperation from the origin.**

## §4a What `prefix` commits to — the question §5-of-their-packet and §6-of-their-packet both turn on

`EXTENSION-TREE` §3.3a says **two incompatible things** about `prefix`, and a publisher sits on the
seam. Reading A: it is *"the operand needed to reconstruct a single absolute path."* Reading B: *"a
publisher at `prefix: "system/"` **commits to a strictly smaller set**… a consumer **MUST** read the
extent from `prefix`."* One is mechanical, the other is a completeness commitment enforced by a MUST
on the consumer.

**Ruled: `prefix` is the extent commitment. Reading A is what it is made of, not what it means.**
Reading B carries a consumer-side MUST, so a consumer is already entitled to act on it; leaving the
operand reading alongside it means publishers satisfy the weaker one and consumers rely on the
stronger.

Two rules follow, and together they answer both of their questions:

> **R5 — a publisher MUST publish completely under its declared `prefix`.** Every binding the
> publisher holds under `prefix` is in the committed trie, or `prefix` is wrong. **Disjoint published
> areas get separate roots** — `sites/` and `system/registry/` are two roots, not one root at the
> peer's universal prefix. Declaring broader than published is a **false extent claim**: a consumer
> resolving a key inside the declared prefix that was never published gets a clean negative and is
> entitled by the MUST to read it as authoritative.
>
> **R6 — a negative answer is scoped to the root that produced it.** `not_found` means *this root
> does not bind K*, never *the publisher does not bind K*. A publisher MAY serve different subsets to
> different audiences from the same prefix, and **no field on the published root can tell a consumer
> which subset it got.** So a consumer MUST NOT present a negative as authoritative non-existence,
> and a surface reporting "not bound" MUST scope it to the root it asked.

**R6 is a limit, not a mechanism, and that is the honest framing.** Signing what you published cannot
prove you published everything you hold. Re-rooting buys node completeness and buys it fully; it does
not buy key-space completeness and no signature can. **R5 is what keeps that limit small** — it
confines the ambiguity to genuinely audience-scoped publication instead of letting every subset
publisher emit a universal-prefix root whose negatives look authoritative for keys it never carried.

**Cost, stated plainly:** a publisher emitting two disjoint areas now signs two roots instead of one.
That is not overhead so much as the thing §3.4 already models — a tracked prefix is already its own
trie — and it strengthens property 3: registry churn stops moving the `sites/` root.

> **R7 — an incomplete walk is TERMINAL, not retryable.** *(Their §4, and the spec was silent —
> which means three engines each pick.)* `tree/incomplete-walk` is the origin failing to produce what
> its own signed root declares. Retrying grants a hostile origin unbounded attempts and turns a
> withholding into a hang; a **transport** failure fetching a blob is separately retryable and is a
> different condition. The discriminator: **the node resolved and its declared child did not exist**
> is terminal; **the fetch itself failed** is retryable. `entity-browser-rust` already draws exactly
> this line (*"a verification failure is TERMINAL. Only a fetch error re-enters the pump"*); the
> spec should say it rather than leave it to be re-derived three times.

## §5 Spec deltas (executable form)

| # | File | Section | Edit |
|---|---|---|---|
| **D1** | `EXTENSION-TREE.md` | §6.2 | **Delete the "Unfiltered subtree nodes MAY be omitted" paragraph.** It is v3.x residue contradicting the rebuild algorithm above it, and it is the only text in the corpus that makes a published tree legitimately incomplete. Replace with the property that is true: an extract's envelope is **complete against its own root**, because the root is built over exactly the extracted bindings. |
| **D2** | `EXTENSION-TREE.md` | §3.4 / §3.3a | State the publication model where a publisher will read it: **publishing a subset is re-rooting, not filtering** — the four properties of §1. Today a publisher has to infer this from `build_trie`'s call site. |
| **D3** | `EXTENSION-TREE.md` | §3.5, `walk_entry_collect` | R1 + R2 — the missing-node branch, derived structurally. The pseudocode gains the failure path it never had. |
| **D4** | `EXTENSION-TREE.md` | error surface | `tree/incomplete-walk` carrying `missing_hash` + `declared_by` (R3). |
| **D5** | `EXTENSION-TREE.md` | §6.2 | R4 — the partial walk named, opt-in, and barred from being presented as complete. |
| **D6** | `EXTENSION-REGISTRY.md` | §6a.3a | The completeness claim now cites `EXTENSION-TREE` §3.5 R1 as the thing that delivers it. **Currently it asserts a property nothing implements.** |
| **D7** | `EXTENSION-REGISTRY.md` | §6a.6 | Apply R1 to the revocation lookup. §6a.6 currently *documents* the absent-vs-withheld collapse as an accepted limitation and routes the reader to a signed-root walk for better — but that walk inherits the same collapse, so the escape hatch does not currently escape anything. |
| **D8** | `EXTENSION-TREE.md` | §11 | Vectors (§7). |
| **D9** | `EXTENSION-TREE.md` | §3.3a | **SPLIT — half landed, half WITHDRAWN.** R6 (a negative is scoped to the root that produced it) stands and is the load-bearing half. **R5 — the completeness MUST — is withdrawn in full at TREE 4.2**; see §9. TREE 4.0.2 → 4.1 → **4.2**. |
| **D10** | `EXTENSION-REGISTRY.md` | §4.1 step 2, §4.1a | **LANDED** — the catch-all rule is re-keyed from **remoteness** to **name transmission**, which is the property its own rationale names. `peer-issued` resolved per §6a.4 through the signed root joins the catch-all; the four query backends stay banned. §6a.4 + §6a.3a already make the signed-root path mandatory, so this costs no integrity that was not already required. The residual disclosure (hash-prefix oracle on a miss; a public binding's blob on a hit) is stated rather than claimed away. REGISTRY 1.7 → 1.8. |
| **D11** | `EXTENSION-TREE.md` | §3.5 | R7 — terminal vs retryable. |

## §6 Cohort impact

| Seat | Owed |
|---|---|
| **`entity-core-rust`** (`ed993bb`) | R1 at `collect_bindings_into`. `collect_node_closure` in the same file already holds the invariant — it inserts the node hash **before** loading, so a gap survives into the output. The enumerating twin dropped it. |
| **`entity-core-go`** (`88615f6`) | Same, at `collectBindings` — the `if !ok { continue }`. |
| **`entity-core-py`** (`79f38a9`) | Same. **Do not change `_load_node` globally** — its `_empty_node()` fallback is shared with the write path (CHAMP collapse on delete), where the store is local and the contract genuinely holds. Add the checked path at the walk. |
| **`entity-browser-rust`** (`a0145a7`) | **Publisher side already conformant; it is the reference implementation of R2.** Consumer side still calls the tolerant upstream primitive and closes when the engines land D3. |
| **`entity-workbench-go`** | Any published-root or registry enumeration presented to a user. |
| **oracle / keystone** | §7's vectors. **No re-pin** — a new failure case, no encoding change. |

## §7 Vectors — required for ratification (charter #5)

- **`TREE-WALK-WITHHELD-1`** — root commits to N keys; serve every node but one interior node; walk.
  MUST fail with `tree/incomplete-walk`. **The assertion is the error, not the count** — asserting
  `len(keys) == N` also passes on an impl that returns N by luck of which branch was cut.
- **`TREE-WALK-CONTROL-1`** — the same trie fully served MUST return exactly N keys and no error.
  Without it, a walk that always fails scores green on the vector above.
- **`TREE-WALK-PARENT-1`** — the error's `declared_by` equals the hash of the node holding the
  dangling `Link` (R3).
- **`TREE-WALK-VALUE-1`** — a withheld **bucket value** entity, not a `Link`, also fails. Cut the
  other half and an impl that only checks links reports clean.
- **`TREE-EXTRACT-COMPLETE-1`** — a `paths`-filtered extract's envelope is **complete against its own
  root**: walking the returned root touches no entity outside `included` (D1).
- **`TREE-WALK-PARTIAL-1`** — the explicitly-requested partial walk returns the short list and no
  error, pinning that R4's opt-in exists and is reachable.
- **`REG-BROWSE-WITHHELD-1`** (`EXTENSION-REGISTRY` §11.1) — a registry browse over a signed root
  with one node withheld surfaces the failure rather than a shorter name list.

## §8 What this does NOT claim

- **It does not refute enumeration.** A signed root genuinely commits to the complete key set; that
  ruling stands. The defect is the collector's error handling one layer down.
- **It does not close F2.** A host withholding a revocation is not *prevented* by this. R1 makes the
  withholding **visible** where today it is a silent `404`, which is strictly better and is not the
  same as fixed.
- **It does not make a full walk cheap.** O(N) fetches against a remote origin is not a first-paint
  primitive. §6a.3a's served-listing-as-a-menu shape is unchanged — what changes is that the
  *fallback* now provides the guarantee it is advertised as providing.
- **It does not touch the wire.** No encoding change, no renumbering, no new entity type.

---

## §9 The seam — fourth appearance, and the general form

`"absent"` and `"withheld"` keep arriving as the same value, and each time the fix was to make
incompleteness a failure rather than a shorter answer:

1. B14 returned `Ok(None)` for an unwalkable tree.
2. Upstream `collect_bindings_into` — the short walk (§3).
3. `--verify` reported *16 pointers verify, 0 broken* over an incomplete closure.
4. The guard written to close (3) enqueued only already-present children, so it could not fire.

Four instances, two trees, one shape — and §6a.6's revocation `404` is a fifth wearing different
clothes. **The general form, for the anti-pattern catalog:** *a lookup that returns "not found" for
both "does not exist" and "you were not given it" is a silent authorization surface whenever the
party answering chooses what to serve.* It is invisible in testing because the fixtures serve
everything, and the honest answer and the hostile answer are byte-identical at the consumer.

---

## 9. R5 withdrawn — the completeness MUST was unsatisfiable, and the ruling beside it was the disproof

**`[2026-08-18, same day it landed. Refuted by `entity-browser-rust` at `688395d`; all four grounds
verified here against source before withdrawal. TREE 4.1 → 4.2.]`**

R5 landed as: *"`prefix` is an extent commitment, and a publisher MUST publish completely under it.
A publisher whose published areas are disjoint publishes a separate root per area."* `-l` §4 assigned
`entity-browser-rust` the restructure. **They went to implement it and it did not survive being read.**

### The four grounds, each sufficient alone

**(1) Unsatisfiable by construction — and this is the durable part.** Publishing binds two entities
*under the peer prefix*: the head pointer at `/{peer_id}/system/peer/published-root` and the signature
at `/{peer_id}/system/signature/{hex(H)}` (`published_root_head_path` / `invariant_signature_path`,
`core/peer/src/published_root.rs`, `entity-core-rust` `a701e13`). **H is the content hash of the entity
that commits to `root_hash`.** Committing to a trie that binds H changes `root_hash`, which changes H —
a hash fixed point, twice over for the signature. **An anchor cannot sit inside the tree it anchors.**
`Peer::publish_root` defaults to `prefix = "/{peer_id}/"` (`core/peer/src/lib.rs`), so the MUST failed
on **every publisher's first publish, including the reference implementation named when it was
written**, and on a live peer the default prefix is the peer's whole tree.

**(2) Its justification was revoked one paragraph below it, in the same commit.** R5's entire stated
harm: *"a consumer resolving a key that falls inside the declared prefix but was never published
receives a clean negative and is entitled to read it as authoritative."* **R6, added in the same diff:**
*"A consumer MUST NOT present a negative from a published root as authoritative non-existence."* The
harm was already closed, four lines away, by the better mechanism. R5 imposed a restructure on every
publisher in the ecosystem and delivered a consumer nothing.

**(3) It forbade what R6 permits.** R6: *"a publisher MAY serve different subsets to different
audiences from the same prefix."* A per-audience subset is incomplete under its prefix by definition.
One section, opposite rulings on what a publisher may do; an implementer's choice decided by reading
order.

**(4) "Disjoint areas" was never defined.** `sites/a/` and `sites/b/` are disjoint in exactly the sense
`sites/` and `system/registry/` are. Any literal reading degenerates to one root per key, and nothing
in §12 tests publisher completeness — it carries extent only as a consumer-side read obligation.

### What this cost and what it teaches

**The engine seats were told too.** `ROUTING-2026-08-18-m` §4 put "TREE §3.3a two MUSTs" on all three
engines' owed lists. `entity-core-go` `5b86b2b` answered that the prefix MUST *"is already satisfied"* —
so a seat reported conformance to a rule that cannot be satisfied, which is what a rule nobody can
fail looks like from inside.

**The lesson is not "we shipped a bad MUST."** It is that **R5 and R6 were derived from the same
finding, in the same sitting, and only one of them was checked against the artifact.** R6 came from
asking what a consumer can know; R5 came from asking what a publisher should promise — and the second
question was answered without opening `published_root.rs`, which is the file the answer lives in.
**L4 says a claim about a tree is checked by opening it; a claim about what an implementation *should*
do is still a claim about that tree.** A rule authored for a surface is not exempt from reading the
surface because it is prospective.

**And browser-rust's own framing is the sharper version, adopted here:** *a landed MUST is a claim, not
a fact.* We told them last round to verify claims in `specs/` rather than in the packet describing
them. They then took a MUST from `specs/` on its word and began implementing — and forty lines of
`published_root.rs` refuted it. **The verification obligation does not stop at the spec boundary.**

### Delta

| # | § | Change | Verified |
|---|---|---|---|
| W1 | §3.3a | R5's completeness MUST **removed**; replaced by an explicit *"`prefix` bounds scope, it is not a completeness claim"* | ✅ present |
| W2 | §3.3a | Withdrawal box — four grounds, the anchor-fixed-point invariant stated generally | ✅ present |
| W3 | §3.3a | The pre-existing *"commits to a strictly smaller set"* sentence re-worded to **bound**, so the section stops disagreeing with itself | ✅ present |
| W4 | header | **4.1 → 4.2** | ✅ present |

**R6 is untouched and remains the operative rule.** No mechanism replaces R5: signing what you
published cannot prove you published everything you hold, which R6 already says.

### Cohort impact

| Seat | Owed |
|---|---|
| `entity-browser-rust` | **Nothing. The assigned restructure is withdrawn** — do not split your publish into two roots. Your `AGENTS.md` guard against it is correct and should stay |
| `entity-core-{go,rust,py}` | **The prefix MUST is off your `-m` §4 list.** R6 (negative scoping) and the terminal-vs-retryable ruling stand unchanged |
