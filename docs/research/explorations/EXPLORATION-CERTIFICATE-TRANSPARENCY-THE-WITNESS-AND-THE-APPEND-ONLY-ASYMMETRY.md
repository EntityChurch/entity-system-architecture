# EXPLORATION — Certificate Transparency read at source: the witness, the failed gossip, and the append-only asymmetry

**Internal working document.** Not a proposal, not normative. **Nothing is folded by this document.**

**Predecessor:** `EXPLORATION-THE-P2P-COVERAGE-AUDIT-…` ranked this GAP 1 and called it *"worth more
than the other four together."* That ranking was right. **Its framing was wrong**, and correcting the
framing is where most of the value in this document came from.

---

## §0 What was read, and how

**Primary sources, opened directly — not search summaries.** This is the sourcing standard the
predecessor document failed on Radicle (`radicle.dev` 403'd; that debt is still open, see §9).

| Source | What it is |
|---|---|
| **RFC 9162** (CT v2.0) | Consistency proofs §2.1.4, inclusion proofs §2.1.3, STH §4.10, **trust model §1.6** |
| **RFC 6962** (CT v1.0) | SCT-as-promise §3, MMD, **gossip §5 / §7.3** |
| **C2SP `tlog-witness` v1.0.0** | The witness protocol — state, wire format, error conditions |
| **C2SP `tlog-cosignature`** | Checkpoint format and what a cosignature commits to |
| **`draft-ietf-trans-gossip-05`** | The gossip design that **expired without becoming an RFC** |
| **atproto.com/specs/repository** | The MST — for §7 |

**Corpus-side measurement, invocation quoted** (L7's standing rule):

```
grep -rlw -i "witness" specs/ guides/          → 0 files
grep -rn -i -e witness -e cosign -e countersign specs/extensions/EXTENSION-ATTESTATION.md → 0 hits
```

**`witness` occurs zero times in the normative corpus**, in either `specs/` or `guides/`, including in
`EXTENSION-ATTESTATION` — which `WORKSTREAMS.md` D-45 names as *"the home"* for it. D-45's own
disposition is **"flagged, not designed."**

---

## §1 The premise inversion — CT does not solve split-view, and says so in terms

The predecessor document, the handoff and register row **C-2** all carry the same sentence: CT is
*"twelve years of deployment on exactly the limitation `TREE` §3.3a states and does not solve,"* and its
*"consistency proofs + client gossip + witnesses"* map onto rung 3, rung 4 and D-45.

**Read at source, that is wrong in its most important clause.** RFC 9162 §1.6:

> *"The log auditing mechanisms described in this document can be circumvented by a misbehaving log
> that shows different, inconsistent views of itself to different clients. **Therefore, it is necessary
> to treat each log as a trusted third party.**"*

and, in the same section:

> *"While mechanisms are being developed to address these shortcomings and thereby avoid the need to
> blindly trust logs, **such mechanisms are outside the scope of this document.**"*

**Witnesses are not defined in RFC 9162 at all.** Client gossip was specified separately, in
`draft-ietf-trans-gossip`, which **expired at version -05 and never became an RFC** (§4).

**So the correct statement is the inverse of the one we published.** CT is not twelve years of a
solved answer to our open rungs. **It is twelve years of the same open problem, by the best-resourced
deployment of a transparency log that exists**, with the standardized part solving strictly less than
we assumed and the actual answer arriving only recently and outside the RFC series.

**This is worth more to us than the answer we expected to find.** Our rung-3 gap is not a hole in our
design that a mature field already fills. **It is the state of the art**, and `TREE` §3.3a's honesty
about it — *"a publisher that has not republished and an origin withholding a newer root produce
byte-identical results at the consumer"* — is the same sentence RFC 9162 §1.6 writes about logs. Two
independent systems stopping at the same rung is the register's **L-9** pattern, now with a third
instance and by far the largest.

> **The method note, because it is the recurring one.** The false framing came from ranking a gap by a
> **term census** (`split-view` 0, `consistency proof` 0, `transparency log` 0) and then describing what
> the unread literature contains. **A zero count establishes that we have not read something. It carries
> no information about what the thing says** — and the sentence we wrote about CT's contents was a claim
> about documents nobody had opened. That is **L4** on foreign sources rather than our own, and it is the
> same shape as the four corrections recorded in `HANDOFF-2026-09-06-c` §3.

---

## §2 We already have the checkpoint, and ours is richer

CT's signed artifact and ours are **the same object**, arrived at independently.

A C2SP checkpoint is three lines — **origin, tree size, root hash** — signed. An RFC 9162 STH commits
to **timestamp, tree_size, root_hash**. Ours, at `EXTENSION-TREE` §3.3a:

| CT checkpoint / STH | `system/peer/published-root` | Notes |
|---|---|---|
| origin line | `peer_id` + **`prefix`** | Ours is cryptographic identity, not a label. **`prefix` has no CT counterpart** |
| `tree_size` | `seq` | Both monotonic. Theirs is also a content measure; ours is not — **§5** |
| `root_hash` | `root_hash` | Same |
| `timestamp` | `published_at` | Both publisher-asserted. **Both specs warn against reading it as freshness** |
| — | `predecessor` | **We have a hash chain over heads. CT has no per-head backpointer** |

**Two observations, and the second is the load-bearing one.**

*(1)* `prefix` is ours alone and it is a real advantage. A CT log's origin line, per `tlog-cosignature`,
*"MAY match a publicly reachable endpoint or not"* — it is a disambiguation string. Our `prefix`
is a **normative scope bound**: *nothing under this trie lies outside `prefix`*. A witness cosigning
one of our roots is cosigning a claim whose extent is stated in the artifact. CT witnesses cosign a
claim whose extent is "whatever this log is."

*(2)* **`predecessor` and a consistency proof are not the same thing, and the difference is exactly
§5's asymmetry.** RFC 9162 §2.1.4's consistency proof proves *"that the first m inputs D[0:m] are equal
in both trees"* — a claim about **contents**. Our `predecessor` is a signed backpointer: it proves
**lineage**, that the publisher claims root N as the parent of root N+1. It does **not** prove that
root N+1's binding set contains root N's.

**We should not describe `predecessor` as our consistency proof.** It is our non-rollback chain, which
is what `TREE` §3.3a already calls it, and that is the correct and narrower word.

---

## §3 The witness protocol — and D-45 is designable today, with no new type

`tlog-witness` v1.0.0 is small, and the smallness is the finding.

**Witness state:** *"the witness only needs to keep track of the latest checkpoint it observed and
verified"* — per log. **O(1) per publisher.** Not a copy of the log; not an index; one checkpoint.

**The protocol:** a log POSTs `/add-checkpoint` carrying `old <size>`, a consistency proof, and the new
signed checkpoint. The witness replies with a cosignature, or:

- **409 Conflict** — the old size does not match the witness's recorded latest, **or the sizes match
  and the root hashes differ.** *That second condition is the split-view catch, and it is one
  comparison.*
- **422** — the consistency proof does not verify.
- **403** — no signature verifies against a trusted key.

**What a cosignature means**, per `tlog-cosignature`:

> *"A v1 cosignature is a statement that, as of the specified time, the consistent tree head with the
> largest size the cosigner has observed for the log identified by the origin line has the specified
> root hash."*

### §3.1 The mapping onto our substrate

**Every element of this exists in our corpus already.** A witness, for us, is **a peer that signs
another peer's `published-root`**:

- **The artifact to cosign** — `published-root`'s `content_hash`. Already content-addressed.
- **The signature path** — `/{witness}/system/signature/{hex(target)}`, the **invariant pointer**
  (`ENTITY-CORE-PROTOCOL` §3.5, register **S-2**). Constructable with no extension knowledge, which
  means **a reader can find a witness's cosignature without asking the witness what it is called.**
- **Third-party carriage** — register **S-3**, already landed: *"when Carol asks Alice for Bob's data,
  Alice can include Bob's signatures."* **So a mirror can carry the witness cosignature.** This is the
  property CT had to invent gossip for and we get from the addressing model.
- **The witness's check** — signature valid · `seq` strictly greater than the held one · `predecessor`
  equals the held root's hash.
- **The equivocation evidence** — **two signed roots at the same `(peer_id, prefix, seq)` with
  different `root_hash`**, or two roots claiming the same `predecessor`. Either is a self-contained,
  transferable, cryptographic proof of publisher misbehaviour that names its own signer.

**D-45 said the witness form is *"a signature over a signature"* — targeting `content_hash(signature
entity)`, meaning *"this signature existed and I saw it."*** Reading CT sharpens that, and I think
**corrects it**: the useful target is the **`published-root` entity**, not the signature over it. CT's
cosignature is over the *checkpoint*, not over the log's signature on the checkpoint. Signing the root
directly is what makes the cosignature independently meaningful — a witness asserting *"the highest
root I have seen for Alice is H"* is a claim about Alice's publication, whereas a witness asserting
*"Alice's signature entity existed"* is a claim about a byte string.

**Both forms are well-formed on our substrate; §3.5 property 4 already governs the first.** The
distinction D-45 draws is real and the *choice between them* is what D-45 left open. **CT chose, and
its reason is the one above.**

### §3.2 What this does not give — stated plainly, because CT states it

A cosignature *"proves the witness verified consistency with its previously recorded checkpoint state,
not absolute log correctness."*

- It does **not** prevent equivocation. It **converts an undetectable equivocation into a detectable
  one**, and only if the reader consults someone who is not the publisher.
- It does **not** establish content completeness (§5).
- A witness that never sees the second fork detects nothing. **Coverage is a deployment property, not a
  cryptographic one** — which is why `tlog-policy` exists and defines **quorums** of named witnesses.

---

## §4 Why CT's gossip failed — and which of the three causes are ours

`draft-ietf-trans-gossip` specified three mechanisms (SCT Feedback, STH Pollination, Trusted Auditor).
**It expired in 2018 without publication.** Three causes, and the sorting matters more than the list.

**(a) The STH is a tracking beacon. — DOES NOT TRANSFER.**
Logs were limited to one STH per hour and clients ignore STHs older than 14 days, *"given a maximum
issuance rate of one per hour, an attacker still has 336 unique STHs per log available for tracking."*
A rare STH handed to an anonymous browser is a supercookie.
**This is a consequence of anonymous clients, and our readers are peers with cryptographic
identities.** A peer that already dialled Alice's namespace is not additionally identified by holding
Alice's root. **The mechanism that killed CT's answer is absent from our setting** — not mitigated,
structurally absent.

**(b) Anonymous proof-fetching is hard. — DOES NOT TRANSFER, for the same reason.**
CT clients must fetch an inclusion proof without revealing which certificate they hold, *"most likely
through DNS,"* and the draft concedes *"any method that is anonymous for some users … may not be
anonymous for others."*
**Our reader fetches from the publisher's namespace by construction.** Interest in Alice is already
disclosed by the read. There is no second, worse disclosure to arrange.

**(c) The bootstrapping deadlock. — TRANSFERS DIRECTLY, AND IS THE DESIGN CONSTRAINT.**
*"Each mechanism's failure mode was defined in terms of the others being deployed."* Clients visiting
servers that had not deployed SCT Feedback could be attacked undetectably; the strongest-privacy
configuration was the one with **zero** auditing. Nothing could be deployed first.

> **The transferable rule: a detection scheme that only works once everyone runs it will not be run by
> anyone.** CT's successor design abandoned emergent gossip for an **explicit quorum policy** —
> `tlog-policy` names witnesses and witness groups, and a proof is verified against *"a policy of
> trusted log and witness public keys."* **A quorum of N named witnesses bootstraps at N=1 and is
> useful immediately. A gossip mesh is useful at scale and useless below it.**

**This is the single most actionable thing in the document**, and it points away from the shape our own
notes have been drifting toward. The coverage audit named *"gossip / dissemination — gossipsub,
plumtree"* as GAP 4 on the theory that *we have discovery and no dissemination.* **For the
equivocation axis specifically, the deployed field tried gossip, failed, and moved to named
quorums.** GAP 4 may still be right for *content* dissemination; **it is the wrong instrument for
detection**, and reading CT is what separates the two.

**Residual privacy cost, stated honestly so it is not discovered later:** a witness learns the
publication cadence of every publisher it witnesses, and a reader that queries a witness discloses
interest to a third party who is *not* already in the read path. That is a real new disclosure, smaller
than CT's and not zero. It is a reason to make witnessing **opt-in per publisher**, which the quorum
model already implies.

---

## §5 The append-only asymmetry — the actual reason rung 4 is closed

This is the deepest structural finding and it re-grounds a register row.

**CT's log is append-only over entries.** That single property is what makes everything else work:

- A **consistency proof** is definable *only* because tree n must contain tree m as a prefix.
- **Monitors** can enumerate: *"when a client has a complete list of entries from 0 up to
  tree_size - 1"* (§2.1.2.1) it can reconstruct and check the root. **The log is the completeness
  claim** — which is precisely what the coverage audit hoped to borrow for rung 4.

**Our published root is a signed snapshot of a mutable set.** `seq` increments; bindings can be
**removed**; root N+1 is under no obligation to contain root N's bindings. Therefore:

1. **A CT-style consistency proof is not merely unimplemented for us — it is not definable**, because
   the content-monotonicity it asserts is not a property our tree has.
2. **Rung 4 cannot be borrowed from CT**, because CT's completeness comes from append-only enumeration,
   and we deliberately do not have append-only.

**Register row L-8 currently reads: *"completeness across authors requires a gatekeeper."*** That is
true but it names the wrong cause and thereby suggests the wrong remedy — it reads as an authority
problem, invitingly solvable by appointing an authority. **The sharper statement:**

> **Completeness requires an append-only structure. A gatekeeper is one way to get one, not the
> reason it is needed.** Every system that achieves completeness pays with an append-only log at some
> scope — CT per log, SSB and Hypercore per feed, ATProto per repo, Radicle per ref. **All of them are
> single-writer at that scope**, which is what makes append-only enforceable. **Cross-author
> completeness is unattained everywhere because there is no scope at which a single writer can be
> compelled to append.**

**This is a confirmation of our design, not a defect in it**, and it is the first argument we have that
reaches *why* rather than *that*. Our set-only-grows model and FEED's set-union mirror (register
**L-10**) are the deliberate trade: we gave up a completeness claim we could only have had by imposing a
single writer per scope, and we bought multi-writer gathering. **`PROPOSAL-APP-CONVENTION-FEED` §4.2's
*"a view is never wrong, only short, and shortness is visible and repairable"* is the correct posture
for a system without append-only**, and CT's own §1.6 is the evidence that the alternative is not on
offer at any price we would pay.

**The open question this raises, and it is a good one:** we *do* have an append-only structure — the
`predecessor` chain over roots, and `supersedes` over registry bindings. It is append-only over
**heads**, not over **content**. **Rung 3 lives entirely on that chain**, and it is exactly the scope
where we *are* single-writer (a publisher is the sole writer of its own root). **So the rung we can
actually close is the one where our structure is append-only, and that is not a coincidence — it is
the same law CT obeys.** That is why §3's witness design works and §5's rung 4 does not.

---

## §6 Scoring our design against what was read

**Fresh eyes, both directions — the operator's ask. Three confirmations, two corrections, one new
obligation.**

**Confirmed:**

1. **The signed-root shape is right** and independently arrived at twice (§2). Ours carries two fields
   CT lacks, and `prefix` is a genuine improvement on the origin line.
2. **`TREE` §3.3a's refusal to claim freshness is right**, and is the same refusal RFC 9162 §1.6 makes.
   We were not being unusually cautious; we were being accurate.
3. **Rung 4's closure is a law, not a gap** (§5). This is a stronger position than the register held.

**Corrected:**

4. **C-2's description of CT is wrong** and is rewritten in the register by this document (§1).
5. **`predecessor` is not a consistency proof** and should not be described as one (§2).

**New obligation — and this is the one that costs something:**

6. **D-45 is no longer "flagged, not designed."** §3 is a design: a witness is a peer, the artifact is
   `published-root`, the path is the invariant pointer, the state is O(1) per publisher, the evidence is
   two roots at one `seq`, and the deployment shape is a **named quorum, not gossip** (§4c). **No new
   type, no new extension, no substrate change.** What it needs is a document and a policy object.
   **It is not written here** — this is an exploration, and L0 rule 3 says normative change is
   proposal-first.

---

## §7 The tree correction — the operator's recollection was right, and the record was unreachable

`[operator, 2026-09-06: "I'm pretty sure we've looked at the type of tree AT proto uses when we did our
tree optimization."]` **Correct, and the coverage audit's `merkle search tree: 0 / prolly: 0` was a true
count that supported a false conclusion.**

**What the audit measured:** the published corpus. **What it concluded:** *"we chose CHAMP and never
compared the order-preserving family."* **False.** `PROPOSAL-TREE-NODE-SHAPE-BOUNDED-FANOUT` went
through five revisions with independent reviews from four seats, and its §3.4 rules on the alternatives
by name — **JMT**, **Verkle / vector-commitment trees** (*"Ethereum walked away"*), and **prolly trees**.

**The reason the count was zero: the document had never been in this repo, in any commit.**
`EXTENSION-TREE` cites it **three times**, including §3.1's bolded normative pointer and, exactly on
point, ***"§3 (why IPLD HashMap, not JMT or other alternatives)."*** It was on the **2026-08-17**
dangling-citation worklist with the `pull-in` disposition and sat for twenty days. **Pulled in this
session** to `docs/proposals/implemented/extensions/`, verbatim, with a provenance note.

> **This is the operator's complaint in its purest measurable form** — sessions keep re-deriving,
> several sessions apart, conclusions this project has already reached. The design work was done, thoroughly, by four
> seats. The record was cited by the spec that depends on it and was **not reachable from the corpus**,
> so a session auditing coverage correctly measured zero and correctly concluded we had a gap. **The
> DESIGN-REGISTER closes this for conclusions reached from now on; it does not reach backwards, and a
> phantom citation is the failure mode it cannot see** — the register indexes what we know, and this was
> a case of knowing something whose evidence had been deleted from under it.

### §7.1 The one thing that is genuinely not covered — and it is a narrow thing

§3.4 rules out the order-preserving family thus:

> *"**Prolly trees:** cross-impl non-determinism in chunker boundaries makes spec-precision hard. Future
> consideration **if range-iteration becomes a requirement**."*

**That objection is correct about prolly trees and does not hold against ATProto's MST.** Per
`atproto.com/specs/repository`, the MST determines a key's layer by *"hash the key … count the number of
leading binary zeros … and divide by two"* — a **fixed function of the key**, not a content-defined
chunker — and the spec states the property normatively:

> *"The overall structure and shape of the MST is deterministic based on the current key/value content,
> **regardless of the history of insertions and deletions** that lead to the current contents."*

**That is the same guarantee our CHAMP canonicalization exists to provide**, obtained a different way,
in an order-preserving tree — so it keeps lexicographic key locality, and with it the prefix-subtree
commitment `EXTENSION-TREE` §3.7.2 records as *"the one cryptographic property genuinely lost"* in the
v4.0 fork.

**So the family was ruled out on a property that its most widely deployed member does not have.** This
is **L20** — *an example set cannot falsify a rule it does not span* — with the example set being one
member of a family.

**How much this matters, honestly: probably not much, and the reason is already written down.** §3.4's
ruling carries an explicit trigger — *if range-iteration becomes a requirement* — and §3.7.2's
four-impl audit found **no consumer in any implementation uses the lost property**, with a documented
four-step reversibility cascade ending in *"pre-1.0; viable."* **The decision stands on its own
evidence.** What is wrong is the *stated reason*, and a wrong reason attached to a right decision is
the class recorded as L8's fourteenth form: nothing downstream breaks, so nothing surfaces it.

**The live thread, named so it is watchable:** §3.7.2's use case 2 is light clients, and the note
concedes *"we ARE actively building light-client-class consumers."* **Light clients are also the
population that most wants compact prefix proofs**, and the trigger §3.4 set is the one that would
fire. **The instrument is `entity-browser-rust`, and this is empirical** — the same conclusion
`HANDOFF-2026-09-06-c` §7 reaches about every rung-2 question.

---

## §8 What would falsify the positions in this document

**Stated as checks, not as confidence** — the operator's *"what will prove it one way or the other."*

| Claim | What would refute it | Cost |
|---|---|---|
| **§3** — a witness needs no new type | Attempt the design against `EXTENSION-ATTESTATION` §5.2 and find a field with no home, or a check a peer cannot run. **The L12 test: name the witness's inputs and how it obtains each** | One session, on paper |
| **§4c** — quorum beats gossip for detection | Find a deployed transparency system where emergent client gossip **did** ship and work. **Sigsum and the Go checksum DB are the places to look, and neither was opened here** | One session |
| **§5** — completeness requires append-only | Find one system with **cross-author** completeness and no append-only scope. Hypercore and Willow are the candidates most likely to embarrass this claim, and **Hypercore is GAP 3 and still unread** | One session |
| **§7.1** — MST's determinism is real at scale | It is normatively stated; the check is whether ATProto's implementations actually achieve byte-identical roots cross-impl, i.e. whether they have our fuzzer | Empirical, low priority |
| **§2** — `prefix` is a real advantage over an origin line | A CT deployment that scopes a log's extent in the signed artifact. **`tlog-mirror` and the Sigsum design are where to check** | Cheap |

**The honest overall position is unchanged from the two prior handoffs and should not be dressed up:**
this arc moved the design record and **no running code**. Every rung-2 and rung-3 question is empirical
and **the client is the instrument**.

---

## §9 Owed, and what the next session should read

**Discharged this session:** GAP 1 (CT, at source) · the tree pull-in (§7) · D-45 designed to the point
where a proposal is writable (§3).

**Owed, unchanged and now more specific:**

1. **Radicle sourcing debt — still open.** `radicle.dev` 403'd twice; the graft/replay CVE analysis in
   `EXPLORATION-FEDERATED-REPOSITORIES` §2 rests on search summaries. **Do not fold S-13/S-14 until the
   disclosure post and the heartwood spec are opened directly.**
2. **Sigsum and the Go checksum database / `tlog-mirror`.** Named by §8 twice as the falsifier for §4c
   and §2. **This is now the highest-value unread thing on this axis**, ahead of the coverage audit's
   remaining GAPs, because it is the only deployed witness ecosystem.
3. **Hypercore / Dat (GAP 3).** Promoted: §5's law is the strongest claim in this document and Hypercore
   is its most likely counterexample.
4. **GAP 4 (gossip/dissemination) is downgraded for the equivocation axis** per §4c, and remains open
   for content dissemination. **The two should not be re-merged.**
5. **ATProto's `rev`, SSB's log, IPNS's validity window** — labelled unopened corroboration in three
   documents, load-bearing for freshness. Unchanged.

**If the next session writes one thing, it is the D-45 proposal** — §3 is the design and §8 row 1 is
its own falsification test.
