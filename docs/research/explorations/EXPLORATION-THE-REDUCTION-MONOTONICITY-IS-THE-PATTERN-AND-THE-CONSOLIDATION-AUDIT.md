# EXPLORATION — the reduction: the communication pattern is monotone accumulation, and what that costs

**Status:** exploration. Nothing ruled; no normative text is proposed here.

**The question.** How reduced a primitive is the publish-and-converge pattern this system uses, and how
far can it be separated from the content being exchanged?

**The short answer.** It reduces to **monotone accumulation of independently-signed claims over
coordinates any party can compute**, it is content-independent for exactly that reason, and the places
where it must know about content are the same places where it must coordinate.

---

## §1 The frame: CALM

Hellerstein & Alvaro, *Keeping CALM: When Distributed Consistency is Easy* (arXiv:1901.01930).

> **Definition 1.** *A program P is monotonic if for any input sets S, T where S ⊆ T, P(S) ⊆ P(T).*

> **Theorem 1 (CALM).** *A program has a consistent, coordination-free distributed implementation if
> and only if it is monotonic.*

> *"Intuitively, monotonic programs are 'safe' in the face of missing information, and can proceed
> without coordination. Non-monotonic programs, by contrast, must be concerned that truth of a property
> could change in the face of new information. Therefore they cannot proceed until they know all
> information has arrived, requiring them to coordinate."*

**The negative example is the one that matters here — distributed garbage collection:**

> *"a decision based on incomplete information … can be invalidated by the subsequent arrival of new
> information that demonstrates reachability… The output does not grow monotonically with the input:
> previous 'answers' may need to be retracted! To avoid this, a machine must ensure that it has heard
> **everything there is to hear** … The only way to know it has heard everything is to coordinate with
> all the other machines."*

Two further clauses bear directly: **Observation 1**, *"coordination-freeness is equivalent to
availability under partition"*; and on deletion, *"deletes are not monotonic"*, with tombstones as the
standard repair.

**This frame is not new to this project.** It was applied while the versioning and merge design was
being built, and the earlier statement of it is sharper than the one reconstructed below — see §5.

---

## §2 The reduction

> **Monotone accumulation of independently-signed claims over coordinates any party can compute.**

**Four nouns and one law:**

| | |
|---|---|
| **entity** | a signed, content-addressed artifact |
| **namespace** | the one place its author has authority |
| **coordinate** | a hash over a preimage any two parties compute independently |
| **claim** | an entity that names a coordinate |
| **the law** | **claim sets merge by union** — associative, commutative, idempotent |

**There is one verb: *publish*.** Reading is not a protocol operation — it is fetching bytes and
verifying them. Merging is not one either; it is a property of the type.

**A number of separately-derived results are consequences of the law rather than independent findings:**

| Stated as | Is | 
|---|---|
| claim sets merge by union, so *"which one is right"* is never asked | the law |
| a participant claim is a pointer, never evidence | monotone — a false claim adds nothing that survives verification |
| the base case needs no reverse index | forward edges are already claims |
| a citation carries a locator for what it cites | a locator is a claim about a coordinate |
| a gathered set and a build-attestation set are one object | both are claim sets; only the predicate differs |
| a gatherer's output must be the same type as its input | closure under the law |
| a published artifact is not a service | Observation 1 |

**That last row is the load-bearing one.** Publishers that are offline most of the time are a
**permanent partition**, and CALM says a system that must remain available under permanent partition
has no choice but to be monotone. **The design's central constraint and its central technique are the
same fact** — the static-publication model is not a tolerated limitation, it is what monotonicity buys.

---

## §3 Content-independence

**The pattern is blind to what is inside a claim, and the reason is the theorem rather than taste.** A
monotone program's *"output depends only on the content of their input, not the order in which it
arrives"*, and union does not inspect what it unions. Signature verification, content addressing,
coordinate computation, forward traversal and merge are all content-blind.

**Content re-enters at exactly three sites, and they are the non-monotone ones:**

| Site | Content-aware? | Monotone? |
|---|---|---|
| union of claim sets | no | **yes** |
| coordinate computation | no | **yes** |
| signature / hash verification | no | **yes** |
| **merge strategy for a convergent asset** | **yes, necessarily** | needs a version DAG |
| **acceptance policy** (K-of-N, weights, exclusions) | usually | reader-local, so free |
| **revocation / removal / deletion** | yes | **no** |

> **The places where the pattern must know about content are the places where it must coordinate. One
> boundary, two symptoms.**

Acceptance policy is the exception that proves it: content-aware and free, because one reader applying
policy to data it already holds is not a distributed computation at all.

**The architecture confines the content-dependence to one declared slot.** `EXTENSION-REVISION` §5.1
makes the merge strategy a per-path pluggable cascade, and §5.4 makes CRDT *"a computational artifact
during merge, not a storage format."* **The single content-aware step sits behind a declared
interface** — which is what the design does when the theorem is known, and it was.

### §3.1 The corpus's honest limits are one boundary named several ways

| Recorded as | The non-monotone operation |
|---|---|
| cross-author completeness is impossible without a single writer at some scope | *"heard everything there is to hear"* — CALM's garbage-collection case |
| the absent-versus-withheld collapse | concluding **non-existence** from a partial view |
| publish the gathered set anchored at a coordinate you own | ownership manufactures a **single writer** — the smallest scope where a closed world legitimately exists |
| a walk of a signed root makes omission visible (`EXTENSION-REGISTRY` §6a.3a) | coordination obtained by **authorship** rather than by messaging |
| removal protects future content only; fetched bytes cannot be recalled | **deletion is not monotone** |
| the version DAG | coordination for the one non-monotone asset |

**These are not separate compromises. They are one boundary encountered repeatedly.**

**The rule that follows is cheap and mechanical:**

> **Ask whether a mechanism is monotone. If it is, it needs no coordination and the design is *publish
> and union*. If it is not, do not look for a clever protocol — find the smallest scope at which a
> single writer legitimately exists, and put the closed world there.**

Ownership, a signed root and a declared tracked prefix are three instances of that one technique.

**Sequencing follows too**, and CALM states it directly: keep coordination off the critical path. The
version system's independent nested prefixes and the group-encryption snapshot-level rekey are both
that advice already taken.

---

## §4 The boundary, stated as the sharper earlier form

The formulation this project reached while designing the versioning layer is a layer boundary rather
than a list of sites, and it is better:

> **Content-store sync is monotonic — coordination-free. Tree sync is non-monotonic — it requires merge
> strategies, conflict resolution and a version DAG. That is where all the complexity lives, and it is
> contained to the tree layer.**

Entities are immutable and content-addressed, so exchanging them can only add; **bindings are mutable,
so a path can be rebound and two peers can disagree about what a name points at.** One layer is a
join-semilattice by construction; the other is where every hard problem in the system lives. **That is
the whole architecture in two sentences**, and it explains why the merge framework is a pluggable
cascade rather than a fixed algorithm: it is the designated coordination point.

---

## §5 Status of this document

**This is a consolidation, not a discovery.** The CALM frame was applied during the design of the
versioning and merge layers, and the §4 boundary is the earlier and stronger statement of what §3
reconstructs. What this document adds is a single place where the frame, the reduction and the
resulting list of honest limits sit together.

**What remains genuinely unread:** Bloom / Dedalus — the programming language built on CALM — which
matters only if monotonicity is ever to be *checked* mechanically rather than argued. Merkle-CRDTs,
delta-state CRDTs and Matrix state resolution have all been surveyed previously and are re-reads rather
than new reads.

**Open:** whether every mechanism claimed monotone here actually is. §3.1 is an argument, not a proof,
and CALM itself notes *"a number of clever exceptions to these rules that still achieve semantic
monotonicity."* A per-mechanism audit is well-posed work and has not been done.
