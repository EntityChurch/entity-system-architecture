# WALKTHROUGH — I have a manifest and five hundred chunk hashes. How do I find them?

**Status:** WALKTHROUGH (2026-09-13). **Concrete trace, not a design.** Everything marked ✅ is landed
spec text you can run today; everything marked ⛔ is missing and named.

**Why this exists:** the synthesis before it argued *"publish a digest of `serve_scope`"* and never
traced a single fetch. **That was the wrong altitude, and it also blurred two different things** — see
§6. This document answers the question as asked.

---

## §0 The answer in six lines

1. **A chunk hash is a tree path.** `system/content/public/00a3f2…` — that is the whole trick, and it
   is landed (`EXTENSION-CONTENT` §6.4.2).
2. **The manifest came from a peer, and that peer is your first and usually only answer.** ✅
3. **Asking one peer *"do you have chunk H?"* is one HTTP GET.** ✅
4. **Asking ten peers about five hundred chunks that way is five thousand GETs.** That is ICP, and it
   is the thing that does not scale. ✅ *(it works; it is just expensive)*
5. ⭐ **Comparing a whole set costs ONE hash, because the trie already fingerprints every subtree.**
   `EXTENSION-TREE` §3.7.1. ✅ **specified** — ⛔ **but there is no operation to ask for it.**
6. **A stranger who has your chunks and whom you have never met is still unreachable.** ⛔ **Open, and
   the corpus's position is that this case is rarer than it sounds.**

---

## §1 The setup, concretely

You hold a `system/content/blob` — the manifest. Its `chunks` field lists 500 chunk hashes. You need
the bytes for all 500.

**The one fact that makes everything else work:**

```
{namespace}/{hex(H)}                      ; EXTENSION-CONTENT §6.4.2, landed
system/content/public/00a3f2...           ; 66 chars for SHA-256, leading `00` is the format byte
```

> *"The bound entity at that path is the `system/content/blob` **or** `system/content/chunk` entity
> itself."*

**So every chunk a peer serves is a leaf in its tree, at a path you can compute from the hash alone.**
You do not need to be told where it is. You need to know **who**.

---

## §2 Step 1 — ask the peer the manifest came from

**Where did the manifest come from?** In this architecture a reference is
`{tag: "pin", peer: P, hash: <manifest hash>}` — and `APP-CONVENTION-REFERENCE` §1 makes that
structural: ***a reference is routable if and only if it names a publisher; bytes alone name no holder
you can go and ask.***

**So you have `P`. Resolve `P` to an endpoint** (`system/peer/transport-set`, `EXTENSION-NETWORK`
§6.5.1c) **and ask:**

```
GET  {content-prefix}/{hex(H)}                     ; CONTENT_GET — the bytes
GET  {tree-prefix}/{P}/system/content/public/{hex(H)}.leaf
                                                   ; TREE_GET leaf — presence, returns the bound hash
```

Both routes are landed (`EXTENSION-NETWORK` §6.5.6 G2). **`P` answers iff `serve_scope.cap` permits
`get(system/content/public/{hex(H)})`** — one capability evaluator, the same one the live surface uses.

⇒ **500 GETs, parallel, done. This is the normal case and it needs nothing that does not exist.**

**And it is not a toy case** — it is what the winners do. OCI pulls from a registry you configured; Nix
pulls from a substituter list; Git pulls from a remote. *The publisher is the answer, and content
addressing is what makes it safe to accept the bytes from anywhere else later.*

---

## §3 Step 2 — `P` is missing chunks, or gone

**Now you need somebody else, and this is the part the arc has been circling.** Three options, in
increasing cost and increasing reach.

### §3.1 The hints that came with the reference ✅

`APP-CONVENTION-REFERENCE` §2.3's `via` list: `{tag: "mirror", value: <origin>}`,
`{tag: "peer", value: <peer id>}`. **Advisory, ordered, droppable, and MUST NOT be load-bearing.** Try
them in order. Cost: a few GETs. **Landed, and free.**

### §3.2 Ask each peer you know, per chunk ✅ — *this is ICP, and it is the expensive one*

```
for each peer Q you know:
  for each chunk H:
    GET {tree-prefix}/{Q}/system/content/public/{hex(H)}.leaf   → 200 | 404
```

**It works today with no new mechanism.** It is also **peers × chunks** requests — 10 × 500 = **5,000**
— which is exactly the fan-out cost that killed inter-cache lookup: *every added sibling multiplies
query fan-out per miss.*

⚠ **And a 404 here is `absent`-or-`could-not-look`, never proof.** It means *not bound under that
namespace, at an origin that answered.* The publisher may hold it elsewhere, or be scoping you out.

### §3.3 ⭐ Compare the whole set once, instead of asking per chunk ✅ *specified*, ⛔ *not askable*

**`EXTENSION-TREE` §3.7.1, landed:**

> *Given any subset of bindings under a prefix, an implementation builds a fresh trie over that subset
> and obtains a deterministic root hash; two implementations building the same subset produce
> byte-identical results.*

**So:**

```
my_fingerprint   = trie_root_over(system/content/public/*)      ; local, no publication needed
their_fingerprint = <ask Q for the same>                        ; ⛔ NO OPERATION EXISTS
equal  ⇒ identical sets. Zero further work, 500 chunks answered.
differ ⇒ descend: compare child fingerprints, recurse only into subtrees that differ
```

⭐ **That is range-based set reconciliation, and the trie is already the backend it needs** — a total
order with a fingerprint at every interior node, computed for other reasons. Cost is **proportional to
the difference**, not to either set.

⇒ **The 500-chunk question becomes: one hash per peer, descend only where they differ, then answer all
500 locally.**

### §3.4 ⛔ The three things that are actually missing

**1. There is no operation to ask a peer for a prefix fingerprint.** The computation is specified; the
way to request it is not. `DATA-EXCHANGE` §11 `Q9` is exactly this question, open:

> *"What is missing is only an **operation** to ask for one."*

**2. Reconstruction is too slow to run in a loop, and this is measured, not assumed.**

| bindings under the prefix | rebuild cost | per binding |
|---|---|---|
| 1,000 | **34 ms** | 34 µs |
| 10,000 | **455 ms** | 46 µs |
| 50,000 | **2.62 s** | 52 µs |

⇒ **Right for a one-off comparison. Wrong as a per-pass witness**, because it makes the *source* pay
seconds of CPU per requesting reader per pass. **A maintained sidecar index pays `O(log n)` per write
and `O(1)` per read — and nobody has built one in any implementation.** *(The landed spec's own
"microseconds to low-ms" claim is wrong inside its stated range by ~30×; already corrected as `D19`.)*

**3. It only reaches peers you already know.** Reconciliation tells you what *your* peers have. A
stranger holding your chunks, whom you have never met and who was named by no reference, is **not
reachable by any of this.**

---

## §4 So what does the whole thing look like, end to end

```
have: manifest + 500 chunk hashes, and a reference naming publisher P

1. resolve P -> endpoint                     EXTENSION-NETWORK §6.5.1c        ✅
2. GET content/{hex(H)} x500 from P          EXTENSION-NETWORK §6.5.6         ✅
   |
   +-- all 500 arrive -> DONE. This is the common case.
   |
   +-- some missing, or P unreachable:
       |
       3. try `via` hints on the reference   APP-CONVENTION-REFERENCE §2.3    ✅
       |
       4. for peers you know, EITHER
       |    (a) probe per chunk              TREE_GET .leaf, 200|404          ✅ expensive
       |    (b) compare prefix fingerprints  EXTENSION-TREE §3.7.1            ⛔ no operation
       |        then fetch only the delta
       |
       5. nobody you know has it             -> ⛔ NO ANSWER
```

**Everything above the dashed line runs today.** The gap is narrow and it is two items: **an operation
to ask for a prefix fingerprint**, and **a maintained index so that operation is cheap in a loop.**

---

## §5 Why the capability story matters here, concretely

**Because a peer that says *"yes I have it"* and then refuses is worse than one that says nothing**, and
that failure killed inter-cache lookup: ICP's `HIT` did not account for access control, so *an object
present in cache but not accessible for a sibling cache* produced a false hit.

**Here the same evaluator decides both sides.** A chunk is at `system/content/public/{hex(H)}`;
`serve_scope` is a capability over paths; **whether the leaf is visible and whether the GET succeeds are
the same check on the same path.** So:

- A fingerprint computed over *what this asker may see* cannot promise something the fetch will refuse.
- `DATA-EXCHANGE` §6.2.1 already requires `absent` (*the publisher holds nothing here, by choice*) and
  `refused` (*we have it and have not shared it with you*) to be **distinct, non-mergeable outcomes** —
  which is precisely the discrimination ICP lacked.

⚠ **And the real consequence for the fingerprint, which is a genuine open question:** a per-asker
fingerprint is correct and leaks the authorization boundary; a single public fingerprint over the
`published-set` leaks nothing new — *within `serve_scope`, hash-knowledge is already the read
authority* — **but it cannot answer for a private namespace at all.** ⇒ **the public-set case is easy
and is probably all that should ship first.**

---

## §6 Corrections to the synthesis that preceded this

**Two, and the first is the one that caused the confusion.**

1. ⛔ ***"Publish a digest of `serve_scope`"* blurred two different things.** **`serve_scope` is a
   capability — it decides WHICH LEAVES are in the set. The digest is the trie's subtree hash — it
   summarizes WHAT IS IN IT.** They are not the same object, and the digest is not a new artifact to
   design: **the trie already computes it.** The correct sentence is *compare the fingerprint of the
   content subtree, evaluated under the asker's scope.*
2. ⚠ **The synthesis said staleness is "already paid for" by the 30-second convergence republish.**
   True for a **published root**. **Not true for two live peers on a LAN who have published nothing** —
   which is the topology a product actually ships in, and the case `DATA-EXCHANGE` §9.7 was written
   about. **For live peers the fingerprint must be computed on request, which is where the cost table
   in §3.4 bites.**

---

## §7 What to build first, if anything

**In order, and the first is small:**

1. ⭐ **An operation to ask for a prefix fingerprint.** The computation is already normative; this is a
   request shape and a result type. **It makes §3.3 real and it is `DATA-EXCHANGE` `Q9`.**
2. **A maintained subtree-fingerprint index**, so the operation is `O(1)` rather than seconds.
   **Nobody has one; the trie's structure means it is an incremental update, not a rebuild.**
3. **Only then** anything resembling a published holder set — and §5 says the **public namespace** case
   is the one worth shipping first.

**And the honest framing for the whole arc: this was never a missing lookup service.** It is a missing
*question you can ask a peer you already talk to*, over a structure that already exists.

---

## §8 What this does not answer

- ⛔ **The stranger case** (§3.4.3). Unchanged, and it is the one genuinely open discovery question.
- ⛔ **Abuse**: §3.3 lets an unauthenticated stranger ask a peer to do trie work. **Uncosted, and it is
  the next review.**
- ⛔⭐ **`EXTENSION-CONTENT` CONTRADICTS ITSELF ABOUT WHETHER `ingest` BINDS INTO THE TREE — and
  everything in this document depends on the answer.** Two landed sentences in one spec:

  > **§6.4.1:** *"`system/content:ingest` into namespace P writes to the content store **AND** binds at
  > the canonical path in the tree (§6.4.2)."*
  >
  > **§6.3:** *"**Content store only.** No tree writes, no subscriptions fire, no cascades."*

  **These cannot both be true**, and §6.3's pseudocode agrees with §6.3's prose — it shows
  `content_store.put(entity)` and no binding step.

  ⭐ **The probable resolution, and it is the better design:** **binding is a separate act from
  ingesting.** You ingest bytes (hash-addressed, no path), and you **separately bind the hashes you
  choose to serve** at `{namespace}/{hex(H)}`. That is what makes a published set **a deliberate
  publication rather than everything you happen to have fetched** — and it **strengthens** §5's privacy
  argument rather than weakening it: the served set is what you chose to expose, not your whole store.

  ⚠ **But it is not this document's call, and until it is ruled:** whether a peer that merely *fetched*
  a chunk can answer for it is undefined, which changes how much §3.3's comparison actually covers.
  **Rule this before building anything in §7.**

---

## §9 Cross-references

`EXTENSION-CONTENT` §6.3 (ingest), §6.4, §6.4.1, §6.4.2 (Hash Tree Presence — the load-bearing one) ·
`EXTENSION-TREE` §3.7.1 (subtree hash by reconstruction), §3.3a · `EXTENSION-NETWORK` §6.5.6 (routes,
`serve_scope`), §6.5.1c (`transport-set`) · `APP-CONVENTION-REFERENCE` §1, §2.3 ·
`PROPOSAL-KEEPING-A-COPY-CURRENT…` §6.2.1 (the outcome taxonomy), §9.7 and §9.7.0 (the prefix witness
and its measured cost), §11 `Q9` · `SYNTHESIS-THE-FIVE-LOOKUPS-REDUCE-TO-ONE-LOOP…` (corrected at §6) ·
register rows `LK-1`…`LK-58`.
