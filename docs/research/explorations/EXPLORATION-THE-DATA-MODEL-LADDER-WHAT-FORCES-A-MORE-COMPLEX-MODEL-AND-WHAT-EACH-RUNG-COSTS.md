# EXPLORATION — the data-model ladder: what forces a more complex model, and what each rung costs

**Status:** Exploration (design record). Not a proposal, not normative.

**The question, in the operator's words:** *"we do want to look at the larger field… a little bit more
depth to figure out when we would want to move to more complex data models, and when or why."*

**Why now.** `EXPLORATION-THE-FALSIFICATION-TEST-…` closed by naming its own limit: four
microblog-lineage objects, with the wiki / marketplace / collaborative-document class untested, *"and
that is where a genuine falsification would come from."* This is that class — approached not as
products but as **data models**, because the products differ in a hundred ways and the models differ
in about four.

---

## §0 The result — and TWO CORRECTIONS, both of them to this document's own first version

**There is a clean five-rung ladder with a forcing condition and a price per rung** (§3), and a
three-question test that tells you which rung a use case needs (§4). **That part stands.**

**Everything this document originally said about our own corpus was wrong, twice, in opposite
directions, and both errors have the same cause: reasoning about two large extensions from `grep`
output instead of reading them.** Corrected 2026-09-05 on the operator's challenge. The first version
is not preserved; what it claimed is stated below so the correction is checkable.

> **Error 1 — the CRDT claim (was "C-5"). WITHDRAWN ENTIRELY; there is no defect.**
> The first version read `EXTENSION-REVISION`'s *"no persistent CRDT metadata"* out of the overview and
> concluded that *"the framework can host merge strategies and cannot host a CRDT, despite naming
> one."* **§5.4 is titled *CRDT as Merge Strategy* and states the principle it follows by name — the
> Eg-walker principle: *CRDT is a computational artifact during merge, not a storage format.*** The
> causal information a CRDT needs **is** persisted — as the **version DAG's parent pointers** — and
> replayed at merge time, which is precisely Eg-walker's result. **The 2018 survey the objection was
> derived from predates it.** §7.2 then proves convergence outright: *"concurrent merges of the same
> inputs produce the exact same version entry hash"*, *"the number of distinct heads across the cluster
> is monotonically non-increasing"*, **convergence guaranteed** — with a `deterministic` merge-ordering
> setting specified for exactly the p2p case where asymmetric strategies would otherwise diverge.
> **The operator's summary is the accurate one: revision already is a convergent replicated structure,
> the settings are specific and have to be right, and anything further can be built on top.**

> **Error 2 — the multi-device reframing. RETRACTED; the ORIGINAL C-2 was correct.**
> The first version claimed that under the identity model *"the same Alice"* is N peer-IDs, so there is
> no single namespace that is Alice's feed and following her is N follows. **That is not what the
> identity model says.** `EXTENSION-IDENTITY` §6 is explicit: *"the controller's pubkey **IS** the
> user's published handle. Contacts cache it."* Agents are **per-device daemon keys with authority to
> act on behalf of the identity** — they are how Alice operates, not what she is. **Alice publishes
> under one public peer-id, which is the whole point of "stable cross-peer recognition."**
> **So C-2's original statement was right and the reframing moved it away from the real problem.**
> Two writers under one published namespace, both advancing one signed root, is the race — and it is
> the shape `entity-browser-rust` independently reported from a deployment they already ship. §6.

**What survives the second correction is narrower, sharper, and actually checkable** — and it exists
because REVISION *does* specify contention handling, for a different pointer. §6.

---

## §1 This is not the taxonomy question, and conflating them is the trap

The taxonomy floor asks: **are these two things different *types*?** Its test is *must a conformant
consumer behave differently*. It operates **inside** one data model and it is settled — four content
shapes, externally tested.

**This asks a question one level down: is the model itself still the right one?** *"An immutable,
content-addressed, signed value in exactly one author's namespace"* is not a neutral container. It is a
strong set of commitments, and the question is what makes those commitments stop paying.

**The distinction matters because the two questions have opposite failure modes.** Minting a
type you did not need costs one redundant tag. **Staying on a data model that cannot express your
problem costs correctness**, and it shows up as an application quietly implementing consensus badly.

---

## §2 The axis is values versus operations, and git is the diagnostic case

**Our model stores values.** An entity *is* a value. A tree maps a name to a value. A revision is a new
value. Nothing anywhere stores *"what changed"* — the change is derived by comparing two values.

**A more complex data model is one that must store operations**, because the unit of change has become
smaller than the unit of addressing. If the thing you must express is *"insert this character at
position 400"* rather than *"here is the document now"*, the entity is the wrong granularity.

**Git is the case that proves the axis discriminates**, and it explains a verdict already in the corpus.
Git *looks* like an operation system — everyone thinks in commits and diffs — and it is not: blobs,
trees and commits are immutable content-addressed **values**, and the diff is computed on demand.
**That is exactly why the interaction landscape rated code forges as a partial fit** (*"repositories are
content-addressed DAGs already"*). **A system that feels operational but stores values sits on rung 0
and costs nothing.** Ask what is *persisted*, never what the UI presents.

---

## §3 The five rungs

| # | Rung | What it is | Forcing condition — you are here when… | The price |
|---|---|---|---|---|
| **0** | **Single-writer values** | one author, own namespace, immutable signed entities | always, by default | none — conflict is *structurally impossible*, which is what makes everything else cheap |
| **1** | **Assembled union** | many writers, each on rung 0; a reader gathers | several parties write *about* one subject and **the union is the answer** | **still no new model** — it is rung 0 plus a reader. Completeness is not claimed, and that is the honest part |
| **2** | **Multi-writer with a merge rule** | one logical object, several writers, a deterministic rule (or a surfaced conflict) decides | the writes **interact** — union is wrong because they overwrite, contradict or must be reconciled | **you must trust the rule's input.** For last-writer-wins that input is a clock (§7) |
| **3** | **Convergent operations (CRDTs)** | operations designed to commute; no rule needed | the object is **continuously co-edited** and a merge rule would lose intent | metadata that grows with history; the object stops being a value you can hash and be done with (§8) |
| **4** | **Totally ordered operations** | a log with an arbiter | there is an **invariant spanning writers** — a balance, a capacity, a uniqueness | **an authority.** Which is the thing the whole design exists not to have |

**Rung 1 does far more work than anyone expects, and the falsification test is the evidence.** Four
foreign social objects, four systems, and every one of them absorbed at rung 0/1. A forum thread, a
leaderboard, an RSVP list and a comment section are all *unions*, and a union across single-writer
namespaces needs no coordination, no clock and no arbiter. **Most things people assume are shared
mutable state are unions wearing a shared-mutable-state interface.**

**The jump that costs is 1→2**, and it is where every 🟡 row in the interaction landscape sits.

---

## §4 The forcing test — three questions, in order

> **1. Do two parties write to one *logical* object?**
> No → **rung 0.** Stop. This is most things, and the corpus keeps rediscovering that.
>
> **2. Is the union of their writes the answer?**
> Yes → **rung 1.** Still no new model — assemble it in a reader and claim no completeness.
>
> **3. How is the disagreement resolved?**
> · By a **rule over verifiable data** → **rung 2.**
> · **By construction**, because the operations commute → **rung 3.**
> · Only by an **invariant someone must enforce** → **rung 4**, which needs an authority.

**Question 3 is the one that does the work, and its wording is deliberate: *over verifiable data*.** A
resolution rule is only as good as the input it reads. **A rule keyed on a field the reader cannot
verify is not a resolution rule; it is an attack surface** — whoever writes the field wins. That
sentence is the whole of §7 and it is where our substrate has a specific, structural constraint.

---

## §5 The corpus against the ladder — and this is the part that was wrong

| Rung | Status here | Where it lives |
|---|---|---|
| **0** | **Built, and it is the substrate** | the core protocol; `APP-CONVENTION-FEED` |
| **1** | **Built and externally validated** | `FEED`'s mirror; confirmed absorbing four foreign objects |
| **2** | **BUILT, and this document originally claimed otherwise** | **`EXTENSION-REVISION` v3.13** — version DAG, divergence detection, per-path merge framework, three-way default, **conflicts as entities so the document never disappears**, and **§8.1's concurrent-commit contention rule** with a stated no-orphan invariant and an explicit non-conformance clause |
| **3** | **BUILT — and it is rung 2's mechanism, not a separate one** | `EXTENSION-REVISION` **§5.4** (*CRDT as Merge Strategy*, following the **Eg-walker** principle — the causal record is the version DAG, replayed at merge, not per-element metadata) and **§7.2** (*convergence is guaranteed*; deterministic merge ordering for p2p). **A further CRDT can be built on top; nothing excludes it** |
| **4** | **Declined, deliberately, and already written down** | the interaction landscape §5.1 — global uniqueness and scarcity are *genuinely closed*; the live rung is where a hard capacity goes |

**Recording both errors, because the cause is one thing and it is reusable.**

**The first** — *"multi-device publishing is undesigned; nothing in the corpus describes it"* — is a
**negative**, published without the exhaustive named search a negative requires. One `grep` for
`multi-device` returns `EXTENSION-IDENTITY` in the first three hits.

**The second is worse, and it is the one to learn from: the correction was made with `grep` too.**
Having found that IDENTITY and REVISION exist, this document then **reasoned about what they say from
section titles and matched lines** — and got both wrong in the confident direction. `EXTENSION-REVISION`
is ~3,800 lines and **only its first ~130 were opened**; the CRDT objection was derived from one
sentence in the overview while **§5.4, §7.2 and §8.1 — the three sections that answer it — sat 3,000
lines further down.** `EXTENSION-IDENTITY` §6's *"the controller's pubkey IS the user's published
handle"* was never read, so the agent-key model was reconstructed from a deployment note.

> **A `grep` is a search instrument and not a reading instrument, and a document's own §10 declaring
> *"only the opening sections were opened"* does not make a claim derived from them safe** — it makes
> it unpublished work wearing a disclaimer. **The rule is not "search harder before claiming an
> absence"; it is "a claim about what a document says is made by reading the document."** That is L4,
> which this corpus ratified long ago and which the first version of this file cited in its own sources
> list while violating.

---

## §6 What actually survives: the published root has no stated contention rule

**Alice publishes under one peer-id.** `EXTENSION-IDENTITY` §6: *"the controller's pubkey **IS** the
user's published handle. Contacts cache it."* Agents are per-device daemon keys with **authority to act
on behalf of** that identity — Alice's laptop and her phone both operate *her* namespace. That is the
point of the three-key default, and it is why *"stable cross-peer recognition"* is listed as one of the
three properties it buys.

**So C-2 is exactly what it always said: two writers, one published namespace, one root sequence.** And
`entity-browser-rust` reports the same shape from a configuration they already ship — *"Tori plus a
browser profile on one peer is a configuration we already support and the on-ramp makes normal, and two
writers advancing one seq is exactly the undesigned race."*

**The precise, checkable form — and it exists because the corpus already solved the analogous problem
one pointer over.**

| Pointer | Contention rule | Where |
|---|---|---|
| **the revision head** (`system/revision/{H}/head`) | **specified** — implementations **MUST** use CAS+retry, single-writer serialization per prefix, or equivalent, to satisfy a **no-orphan invariant**; *"implementations that allow concurrent head advances to overwrite each other… are non-conformant"* | `EXTENSION-REVISION` §8.1 |
| **the published root** (`system/peer/published-root`) | **`seq` MUST increase monotonically**, `predecessor` MUST carry the prior root's hash, and a consumer **MUST reject `seq < N`** — a **rollback** defense | `EXTENSION-NETWORK` §6.5.6 |

> **The gap is the diagonal cell.** Monotonicity and the predecessor chain defend against a root going
> *backwards*. **Neither defends against two roots at the same `seq`.** Two devices under one identity
> both republish at `seq = N+1`, both citing the `seq = N` root as `predecessor`, both **correctly
> signed by an authorized agent** — and a consumer holding one and then the other sees **equal** `seq`,
> so the rollback check does not fire, and nothing tells it which is current or that it forked.

**That is the whole of C-2, and stated this way it is small.** It is not a consensus problem and not a
data-model problem — **it is one pointer missing the discipline its sibling already has.** The candidate
answers are the ones REVISION §8.1 already enumerates (CAS+retry, single-writer serialization) plus one
this substrate makes available and REVISION does not need: **detect and surface the fork**, since two
signed roots at one `seq` are self-evidently a fork and both are verifiable.

**The right next step is a measurement, not a design**, and browser-rust has already volunteered the
better version of it: *"I'd rather reproduce it deliberately with a gate than find it in someone's
profile."* **Reproducing it decides whether it needs a MUST or only a stated invariant** — and it is a
cheaper way to be right than another arch document about it.

## §7 The ordering primitive — what we have, what we do not, and why we cannot copy the neighbours

**Rung 2 needs a resolution rule, and the most common one in the field is last-writer-wins. We cannot
use it, and the reason is structural rather than a gap to fill.**

**The literature is explicit about what LWW requires.** A last-writer-wins semantics needs *"a total
order among updates… that approximates wall-clock time"*, built by combining a physical clock with a
site identifier — and the same source notes that because of clock skew these timestamps *"do not
necessarily respect the happens-before relation"*, which is why hybrid logical clocks exist.

**Our timestamps cannot carry that.** `FEED` §2.3.1 is unambiguous: `created_at` is *"the author's own
clock, it is unverifiable, and a peer may set it to anything"*, and a reader **MUST NOT** rely on it for
correctness. **So an LWW rule over `created_at` is not a resolution rule — it is a race won by whoever
writes the largest integer.** That is question 3's *over verifiable data* clause, biting.

**What we do have is a verifiable order, and its scope is exactly one namespace.** The signed root
carries a `sequence`, and it is signed — so *"this peer published A before B"* is checkable by anyone.
**What does not exist, and cannot without an arbiter, is an order across namespaces.**

| | Within one namespace | Across namespaces |
|---|---|---|
| **Verifiable order** | **yes** — the signed root sequence | **no**, and building one needs rung 4 |
| **So rung 2 is** | **available** — this is `EXTENSION-REVISION`'s home ground | **available only by surfacing, never by deciding** |

**Which makes the corpus's existing choice the right one, and it was made before this argument
existed.** `EXTENSION-REVISION` resolves what it can and **stores what it cannot as a conflict entity,
leaving the document in place**. That is the *surface, do not decide* answer, and it is the only honest
rung-2 policy for a substrate with no cross-namespace clock. **The literature's name for the same move
is the multi-value register — keep all concurrently written values and let the reader see the set.**

> **And the same missing primitive explains an item filed separately.** C-3 (*time-scoped grants are
> absent*) guessed that *"the honest reason may be that our timestamps are presentational, making a
> time-scoped grant a scope over a field we do not trust."* **That guess is correct and it is the same
> root cause as this one.** C-2, C-3 and the inability to adopt the nearest neighbour's LWW answer are
> **three symptoms of one property**: there is no verifiable order across namespaces, by design.
> **That belongs in the corpus as a stated property, not rediscovered a fourth time.**

---

## §8 Which CRDT family — and the corpus had already chosen, correctly

`EXPLORATION-THE-INTERACTION-LANDSCAPE-…` §7 item 2 names this exactly: *"the CRDT recommendation is a
pointer, not a design. It does not say which family, and that choice is consequential."*

**It had already been chosen — `EXTENSION-REVISION` §5.4 — and this section is why that choice is
forced here rather than optional. Read it as corroboration, not as an answer (§8.1).**

**It is answerable from our delivery model alone, and the answer is state-based, not operation-based.**

| | Operation-based (CmRDT) | State-based (CvRDT) |
|---|---|---|
| **Requires** | every operation **reliably delivered to all replicas**, and *"most operation-based CRDT designs require causal delivery"* | states form a **join-semilattice**; operations are **inflations**; merge is the **join** |
| **Tolerates** | neither loss nor duplication; reordering only if *all* effectors commute | **loss, duplication and reordering** — merge is idempotent, commutative and associative, so *"as long as the synchronization graph is connected, every update will eventually propagate"* |
| **Fits us?** | **no** | **yes** |

**The reason is not preference, it is that we structurally cannot supply op-based's precondition.**
Replication here is **pull-based, partial and lossy by design** — there is no broadcast, no membership,
no delivery guarantee, and a reader's view is explicitly bounded by coverage. *"Reliable causal
delivery to all replicas"* is a thing this substrate has declined to build, at every layer, on purpose.
**A state-based merge asks for exactly what we already have: two parties in contact, exchanging state,
in any order, possibly twice.**

**Delta-state is the practical form, and content addressing makes it unusually cheap here.** Delta-state
CRDTs propagate only **delta-mutators** — the changes since last contact — rather than whole states, and
*"the first time a replica communicates… the full state needs to be propagated."* **Map that onto our
substrate and it is free:** each delta is an immutable content-addressed entity, so **dedup is
automatic**, a delta already seen costs nothing, and the merge is a fold over whatever deltas you hold —
which is the same partial-view assembly a mirror already does. **And the "full state on first contact"
step is the published snapshot — the entry.** *The seam really is publication*, and it turns out to be
the literature's own bootstrap step rather than a convenience.

### §8.1 ~~The hook as written cannot host a CRDT~~ — WITHDRAWN, and the corpus was ahead of the analysis

**The first version of this section claimed a defect: that `EXTENSION-REVISION`'s *"no persistent CRDT
metadata"* contradicts convergence, because an add-wins set cannot distinguish concurrent add+remove
from remove-after-add without causal metadata. That claim is withdrawn in full.**

**The reasoning was sound against the 2018 survey and the survey is not what REVISION implements.**
§5.4 names its principle outright — **Eg-walker**: *"CRDT is a computational artifact during merge, not
a storage format."* The causal record a CRDT needs is not absent; **it is the version DAG**, whose
entries carry `root` and sorted `parents` and are therefore a content-addressed causal history. A merge
handler receives base/local/remote, replays operations against a CRDT instance, and returns a plain
entity — **the metadata is reconstructed from the graph rather than carried per element**, which is the
entire point of the result and is newer than the framing this document brought to it.

**And convergence is not asserted, it is argued, in §7.2:** structural version entries mean *"concurrent
merges of the same inputs produce the exact same version entry hash"*; *"the number of distinct heads
across the cluster is monotonically non-increasing"*; content addressing gives `O(1)` convergence
detection. **The one case where it does not hold is named rather than hidden** — asymmetric strategies
(`source-wins` / `target-wins`) under `caller-perspective` ordering — with the remedy specified
(`deterministic` merge ordering) and oscillation detection as a backstop. *That is a spec that has
thought about p2p convergence more carefully than the objection did.*

> **So the honest statement of §8's whole finding is a demotion: state-based/delta-state is the right
> family for this substrate, the corpus already chose a form of it, and the derivation above is
> corroboration for a decision that was made — not an answer to an open question.** The interaction
> landscape's *"which family?"* is answered by `EXTENSION-REVISION` §5.4, and the value of §8 is that it
> says **why** that choice is forced here rather than optional: op-based's reliable-causal-delivery
> precondition is one this substrate has declined at every layer.

## §9 What this changes on the board

1. **C-2 stands as originally written, and gains a precise form.** Two writers, one published namespace,
   one `seq`. **The gap is that `EXTENSION-REVISION` §8.1 specifies contention handling for the revision
   head and nothing specifies it for the published root** — where `seq` monotonicity and `predecessor`
   defend against rollback but not against **two roots at the same `seq`.** §6.
2. **C-5 is withdrawn. There is no CRDT defect.** §8.1.
3. **C-3's reason is confirmed** — no verifiable order across namespaces — and that property is worth
   stating once, durably. §7. **This is the one item unaffected by either correction.**
4. **The next step on C-2 is a measurement, and the seat that would hit it has offered to take it.**
   Reproducing the fork with a gate decides whether it needs a MUST or a stated invariant, and it is
   cheaper than another arch document.
5. **Nothing here proposes a new content type**, which remains a mild corroboration of the taxonomy
   floor.

---

## §10 What this does not establish

1. **The ladder is a frame, not a survey.** Rungs 2–4 are named from the distributed-systems literature
   and from systems already read; **no wiki, marketplace or collaborative editor was read from a
   specification for this document.** That is the same limit the interaction landscape declared and it
   is still not discharged — **and it is now the main thing this document owes**, because the two
   findings it did produce about our own corpus were both wrong.
2. **§6's gap is derived from two specs, not from a run.** The claim that two signed roots at one `seq`
   pass the rollback check follows from `EXTENSION-NETWORK` §6.5.6's text; **no implementation has been
   made to do it**, and a peer may already serialize publishes in a way that makes it unreachable.
3. **The corrections were prompted, not self-found.** Both errors in §0 were caught by the operator
   challenging the conclusions, not by this document's own review — and one of them had already been
   committed and pushed. **A document that states its limits in §10 and then reasons past them is not
   protected by having stated them.**
4. **No implementation seat has reviewed the ladder itself.** The rung framing and the forcing test are
   arch reasoning about mechanisms it does not build.

## §11 Sources

**Primary:** Preguiça, Baquero & Shapiro, *Conflict-free Replicated Data Types* (arXiv:1805.06358) —
**and the limit of it, which this document learned the hard way: it is a 2018 survey, it predates
Eg-walker, and applying its framing to a spec that follows Eg-walker produced a false defect (§8.1)** —
the CmRDT/CvRDT synchronization models, op-based's reliable-and-causal delivery requirement, the
state-based join-semilattice/inflation/join conditions, delta-state CRDTs, the multi-value register,
and LWW's requirement of a wall-clock-approximating total order.

**In-corpus:** **`EXTENSION-REVISION` v3.13 — §§1–3, and critically §5.4 (*CRDT as Merge Strategy*, the
Eg-walker principle), §7.2 (the convergence argument and the `deterministic` ordering remedy) and §8.1
(concurrent-commit contention, the no-orphan invariant). The first version of this document read only
§§1–3 and was wrong twice for that reason** ·
**`EXTENSION-IDENTITY` §6 (*"the controller's pubkey IS the user's published handle"* — the sentence
that refutes this document's first §6) and §§11.2–11.6** (three-key default; agents as per-device
daemon keys with authority to act on behalf of the identity) · `ARCHITECTURE-IDENTITY-INFRASTRUCTURE`
(the peer-graph model and the fall-back-to-V7 principle) · `PROPOSAL-APP-CONVENTION-FEED` §1.1 (the
authorship rule §6 collides with), §2.3.1 (**timestamps are not an ordering authority** — §7's
constraint), §2.4 (`follow.subject` is one namespace) ·
`EXPLORATION-THE-INTERACTION-LANDSCAPE-…` §1 (owned-vs-contended, which this ladder refines rather than
replaces), §3 (the 🟡 rows), §4.1 (the CRDT pointer), §5.1 (rung 4 declined), §7 item 2 (**the question §8 explains rather than answers**) · `EXPLORATION-THE-FALSIFICATION-TEST-…` §9 item 1 (**the limit that prompted this**) ·
`EXPLORATION-WILLOW-…` (the nearest neighbour's multi-writer namespaces and its LWW-on-a-wall-clock
answer, which §7 explains we cannot copy) · `EXTENSION-NETWORK` §6.5.6 (`seq` monotonicity as a
**rollback** defense, `predecessor` chaining, and the republish MUST — the three rules §6 shows do not
cover a same-`seq` fork) · `entity-browser-rust` `62c6d62` (the deployment shape that makes C-2 concrete,
and the offer to reproduce it with a gate).
