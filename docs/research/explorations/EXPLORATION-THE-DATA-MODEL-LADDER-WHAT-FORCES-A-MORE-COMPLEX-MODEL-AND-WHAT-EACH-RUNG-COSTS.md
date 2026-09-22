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

> **Error 1 — the CRDT claim (was "C-5"). The filed defect is WITHDRAWN; the analysis that replaces it
> is §8.1 and it finds two narrower things.**
> The first version read `EXTENSION-REVISION`'s *"no persistent CRDT metadata"* out of the overview and
> concluded the framework *"cannot host a CRDT despite naming one."* **§5.4 cites the Eg-walker result
> (Gentle & Kleppmann, EuroSys '25) and cites it correctly** — that paper's CRDT structure is explicitly
> *"not persisted or replicated and is discarded when the algorithm finishes."* The 2018 survey the
> objection came from is not a law; it is a design point that result supersedes.
> **The second version then withdrew the claim on the operator's say-so plus a skim, and that was also
> not analysis.** §8.1 now does it from the spec and the paper, and lands in three parts: **(a) yes**,
> the version DAG plus deterministic merge is genuinely convergent and §7.2's argument holds; **(b)
> under three preconditions, of which §7.2 names two** — the third, *both peers hold the same merge
> config*, is peer-local tree state that neither party can observe in the other; **(c) it is a state
> DAG, not an event graph**, so a fine-grained text CRDT built on it must bring its own operation log,
> because a diff recovers *a* plausible edit script rather than the one that happened.

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

## §6 C-2 is a KEY CUSTODY question, and the `seq` race is downstream of it

**Corrected a second time, and this is the operator's framing, verified in `EXTENSION-IDENTITY` §7.1.**
The previous version of this section said *"two writers, one published namespace, one root sequence"* and
went straight to the contention rule. **That skipped the question that decides whether the race exists at
all: which key signs a published root, and how is it held on two devices?**

**Alice would use one public key.** She *may* publish under as many peer-ids as she likes and nothing
prevents it — **it just fractures the identity, which is the thing nobody wants.** So the target shape is
one stable public face, and in the three-key default that face **is the controller** (*"the controller's
pubkey IS the user's published handle"*).

**§7.1 assigns every key a job, and publishing is not one of them:**

| Key | Custody | What it signs |
|---|---|---|
| **Quorum constituents** | **cold** — paper, secondary device, hardware token, trusted holder | recovery, quorum updates, new controller certs |
| **Controller** | **hot, encrypted at rest** | *"internal-management entities — peer-config writes, role-assignment records, agent certs"* |
| **Agents** | **hot, one per device daemon** | *"cross-peer caps (V7 standard)"* |

> **Nothing in the identity stack says who signs a `published-root`, and no key's stated job covers it.**
> The controller signs *internal management*; agents sign *capability tokens*. **Publishing authority is
> unassigned**, and §7.1 describes the controller key in the singular — hot, encrypted at rest — with no
> statement about replicating it to a second device, which is exactly the operator's *"we haven't fully
> figured out how that would work in terms of private key, secure key management across devices."*

**So there are three branches and the corpus commits to none:**

1. **Copy the publishing key to every device.** Gives one stable face and **makes the `seq` race real**
   (§6.1). Custody is undesigned — this is where a keystore-with-unlock or a distributed-key scheme
   would go.
2. **Publish through one device or an intermediary service.** Serializes by construction, so no race —
   and it introduces a component nobody has specified, plus a single point of failure the rest of the
   design avoids.
3. **Each device publishes under its own id.** No shared key, no race, **fractured identity.** The
   identity stack's *"concurrent multi-controller… where each device has its own controller"* variant
   (§7.3) is this branch — and in the **three-key** default, where the controller *is* the handle, that
   means a different public handle per device. **The four-key shape is the closest existing answer**,
   since a stable `identifier` peer survives controllers rotating underneath it — **but publishing is
   not mapped onto that structure anywhere.**

**What the identity stack does give is operational, and the distinction is the operator's:** devices can
act and form peer-to-peer relationships as one identity, because they identify against the same cert
chain. **That is not the same as being able to sign as the public face**, which still needs that key.

### §6.1 The `seq` race — real, but only in branch 1

**If the key is shared, the corpus has the analogous problem solved one pointer over and not here:**

| Pointer | Contention rule |
|---|---|
| **revision head** | **specified** — `EXTENSION-REVISION` §8.1: CAS+retry or single-writer serialization, a **no-orphan invariant**, and *"implementations that allow concurrent head advances to overwrite each other… are non-conformant"* |
| **published root** | **`seq` monotonic + `predecessor` chained** — `EXTENSION-NETWORK` §6.5.6. **A rollback defense** |

**Neither rule covers two correctly-signed roots at the same `seq`.** Both cite the `seq = N` root as
`predecessor`, both verify, and a consumer sees **equal** `seq` — so the rollback check does not fire and
nothing says which is current or that it forked.

> **The ordering is what changed: custody first, contention second.** `entity-browser-rust` reports the
> race from a config they already ship and has offered to reproduce it with a gate — **that measurement
> is still the right next step**, and it also answers which branch their deployment is actually in.

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

### §8.1 Do we have CRDTs? — the analysis, done properly, landing in three parts

**This section has now been wrong twice in opposite directions and neither error was the analysis.**
First it claimed a defect from one overview sentence checked against a 2018 survey. Then it withdrew the
claim on the operator's say-so plus a skim — *"don't take my word for it, do the analysis"* is the
correct response to that, and this is the analysis. **Sources are `EXTENSION-REVISION` §5.1, §5.4, §7.2
and §1.1, read; and Gentle & Kleppmann, *Collaborative Text Editing with Eg-walker* (EuroSys '25,
arXiv:2409.14252), read.**

#### (a) Yes — the version DAG plus deterministic merge is a convergent replicated structure

**This is the operator's read and it holds.** §7.2's argument is real and checkable: version entries are
**structural** — `{root, sorted parents}`, no author, no timestamp, no message — so *"concurrent merges
of the same inputs produce the exact same version entry hash"*, each merge is a descendant of all its
inputs, and *"the number of distinct heads across the cluster is monotonically non-increasing."*
Idempotence is free (equal hash ⇒ converged, `O(1)` to detect). **That is a join-semilattice in the
shape that matters, and calling it a CRDT is fair.**

**And the Eg-walker citation in §5.4 is accurate, not decorative.** The paper's replica state is exactly
three parts — event graph (persisted), document state (plain, no metadata), and a **temporary CRDT
structure that is not persisted or replicated and is discarded when the algorithm finishes.** So *"CRDT
is a computational artifact during merge, not a storage format"* is the paper's actual result, correctly
cited. **The 2018 survey's per-element-metadata requirement is not a law; it is one design point that
this result supersedes.**

#### (b) Conditionally — and the third condition is not stated in the guarantee

**Convergence holds under three preconditions. §7.2 names two.**

| # | Precondition | Named? |
|---|---|---|
| 1 | **`deterministic` merge ordering**, not `caller-perspective` | **yes** — §7.2's asymmetric-strategy caveat, with the remedy and an oscillation-detection backstop |
| 2 | strategies that are **symmetric or deterministic** — `source-wins`/`target-wins` return `remote_hash`/`local_hash`, which are **per-peer roles** | **yes**, same caveat |
| 3 | **both peers hold the same merge configuration** | **no** |

**Condition 3 is the finding, and it follows from §5.1 alone.** Merge strategy is resolved from
`system/revision/config/merge/type/{...}` and `system/revision/config/merge/path/{...}` — **entries in
the peer's own tree.** They are local state: not exchanged during sync, not part of the version entry,
not committed to by any hash a counterpart can check. **So two peers with different merge configs
produce different trie roots from identical `(base, local, remote)`, hence different `V_m` hashes, hence
divergence** — and §7.2's *"same merge inputs produce the same version hash"* is true only if *inputs*
is read to include the config, which the sentence does not say and a reader would not assume.

> **This is not the asymmetric-strategy caveat.** That one is about *ordering* within one peer's merge
> and is fixed by a setting both peers can independently choose correctly. **Config divergence cannot be
> fixed by either peer alone**, because neither can see the other's config. *A convergence guarantee
> whose precondition is unobservable to both parties is a guarantee neither can verify it is meeting.*
> **Whether that is a defect or the intended git-like "merge policy is local" stance is a real question
> and it is the spec's to answer — but §7.2 currently claims convergence without the caveat.**

#### (c) And it is a state DAG, not an event graph — which bounds what can be built on it

**This is the precise limit, and it is the one that matters for the collaborative-editing case.**

**Eg-walker persists an *event graph*** — the original operations, with their causal parents — and
replays them, guaranteeing *"the same final document state regardless of which [topological sort] is
chosen."* **`EXTENSION-REVISION` persists a *state DAG*:** each version entry is `{root, parents}`, a
**trie root hash** — an endpoint, not an operation. §5.4's merge handler *derives* operations by
diffing base→local and base→remote.

> **A diff recovers *a* plausible edit script, not *the* one that happened.** For coarse-grained entity
> merges that distinction is immaterial and the framework is right. **For fine-grained collaborative
> text it is the whole ballgame** — delete-and-retype looks like a no-op, and concurrent interleaving is
> reconstructed rather than replayed, so Eg-walker's guarantee does not transfer.
>
> **The correct statement, and it is a scoping note rather than a defect: a text CRDT hosted in this
> framework must bring its own operation log, because the version DAG is not one.** **REVISION already
> scopes real-time collaborative editing out** (§1.1: *"application-level, uses merge framework"*), so
> nothing is broken — but *"CRDT as merge strategy"* invites the reading that the DAG supplies what a
> CRDT needs, and it supplies the ancestor, not the operations.

#### (d) One foot-gun the spec documents itself, and it is the serious one

§5.1's v7.70 Amendment 1 is worth quoting because it is the strongest self-audit in the corpus: a
wildcard config with a conflict-suppressing strategy *"silently rewires conflict resolution for every
prefix and every future merge on the peer, including prefixes that do not exist yet"*, and **"there is
no audit signal today"** — a config-resolved conflict produces a merge result **byte-identical** to a
genuinely conflict-free merge, so *"a subscriber / sync chain / downstream verifier cannot tell a clean
merge from a config-suppressed one."* Tracked (W2), unspecified.

**Read beside (b), the two compound: config is invisible to a counterpart, AND its effects are invisible
in the result.** That is not an argument against the design — it is the precise statement of what a
downstream verifier can and cannot conclude from a merged root, and anyone building verification on top
of revision needs it.

> **So the answer to *"do we have CRDTs?"* is: yes, and the settings are specific, exactly as stated —
> plus one unstated precondition (b) and one boundary (c).** No retraction is owed to the corpus here;
> what is owed is that §7.2's guarantee name its third precondition.

## §9 What this changes on the board

1. **C-2 is a key-custody question first.** Which key signs a `published-root`, and how is it held across
   devices — **§7.1 assigns publishing to no key.** Three branches (§6), corpus commits to none, and the
   `seq` race exists only in the shared-key branch. **The four-key `identifier` shape is the closest
   existing structure and publishing is not mapped onto it.**
2. **C-5 is withdrawn as filed and replaced by two narrower items** (§8.1): **(b)** §7.2's convergence
   guarantee has an **unstated third precondition** — both peers holding the same merge config, which is
   peer-local state neither can observe in the other; **(c)** the version DAG is a **state DAG, not an
   event graph**, so a text CRDT built on the framework brings its own operation log. **Neither is a
   defect in the design; both are sentences the spec does not currently say.**
3. **C-3's reason is confirmed** — no verifiable order across namespaces — and unaffected by any of the
   corrections. §7.
4. **The next step on C-2 is browser-rust's gate**, which also settles which branch their deployment is
   in.

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
