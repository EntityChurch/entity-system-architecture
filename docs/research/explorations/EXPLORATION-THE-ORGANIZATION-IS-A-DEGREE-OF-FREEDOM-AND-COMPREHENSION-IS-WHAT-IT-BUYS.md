# EXPLORATION — the organization is a degree of freedom, comprehension is what it buys, and a measured convention is not a law

**Status:** Exploration (design record). Not a proposal, not normative. **Nothing is folded here.**
Informative axis in `SYSTEM-ARCHITECTURE` §13.5b; this is the fuller treatment.

**Why it exists.** The constraint-regime study named three regimes and said of Regime II — the one where
*we* choose — that **coherence and economy** govern. That is incomplete, and the incompleteness is not
academic: it left the largest class of decisions in the corpus without a stated objective, and a decision
with no stated objective gets justified by whatever is nearest to hand. What was nearest to hand, twice
this month, was **necessity** — a convention measured, found consistent, and written down as a law.

**Its predecessor is `EXPLORATION-THE-THREE-CONSTRAINT-REGIMES`, and this document adds one axis to it
rather than replacing anything.** Regime I is governed by mathematics, Regime III by the deployed world,
Regime II by us. **This is about what Regime II is *for*.**

---

## §0 The result

| | |
|---|---|
| **The class** | Decisions underdetermined by the substrate, forced on us anyway, and **optimized for how cheaply another mind can build a correct model of the system** |
| **What is in it** | Document name classes · the habit of naming an extension after a namespace it owns · namespace depth and where a type is homed · tier numbering · family groupings · the extension decomposition · which of several valid homes a rule is stated in |
| **Why it is not decoration** | **The system's claim is verifiability, and verifiability nobody can trace is faith.** The ladder from a convention down to the mathematics has to be walkable by a person, or the central claim is auditable in principle and unauditable in practice |
| **The test that separates structure from ornament** | ⭐ **A grouping earns its cost when it makes the best use of a bounded working set** — when a reader can work at one level and *correctly* hold nothing from the others. *(Amended §2 — the first wording was "licenses ignorance", which is wrong about the mechanism.)* |
| **The aesthetic dimension** | ⭐⭐ **Not separable from the above, and omitting it was itself a tell.** *Beauty here is understanding compressed into a form pristine enough to transfer directly from one mind to another* — which is why accumulated design guidance (proportion, symmetry, the rule of three) predates any cognitive account of why it works, and is the right model for what a style guide in this domain IS |
| **The trap** | **There is no machine-side signal.** Rename everything to UUIDs and every hash, proof and dispatch still resolves — so it cannot be validated the way the rest of the corpus is. ⚠ **But the second half of this row was WRONG and is amended (§4.2): the constraint is not human-only.** A bounded context window is the same constraint, and hierarchical navigation is how it is managed |
| **The named failure** | ⛔ **Manufacturing necessity for a choice** — measuring a convention, finding it consistent, and promoting it to an invariant, because a law needs no justification while a choice does |
| **The disposition** | **Converge deliberately; mark the convergence as chosen.** Neither "leave it open" nor "decree it" |

---

## §0a Amendment `[2026-09-14]` — three corrections, and each changes a conclusion

**Recorded as an amendment rather than edited away, because the *shape* of all three errors is the same
one this document is about: reasoning from the author's own conditions and then generalizing.**

| | What the first version said | The correction |
|---|---|---|
| **1** | *"A grouping earns its cost when it licenses IGNORANCE."* | ⛔ **Wrong about the mechanism, and the word imports a value judgment.** The reader is not being licensed to not-know; **they are allocating a bounded working set.** ⇒ ***a grouping earns its cost when it maximizes the utility of constrained working memory and accessibility*** — which names the actual resource and makes the test sharper, because *bounded* is measurable in a way *ignorance* is not |
| **2** | *"The machine is indifferent… an author indifferent to naming will under-weight what naming is for."* | ⛔ **True of the conditions this was written under and false in general.** A frontier model with a very large context window is nearly indifferent; **a constrained local model is not, and for it every context token is critical.** Hierarchy is exactly how a bounded context is navigated and dynamically managed. ⇒ **the same organizational structure that serves a human working set serves a small context window, for the same reason** — and the document had generalized from one deployment's abundance |
| **3** | *"`NETWORK` owns `system/peer/*` — therefore the naming convention is not a rule."* | ⛔ **Scored a misalignment as a legitimate exception.** Measured properly (§5a), that namespace is the **one place in the corpus where a reader holding a path cannot find the specifying document** — 10 of 17 such paths, one namespace, four specifying documents. **It is not evidence against the convention; it is the convention's strongest catch, and the document filed it as an excuse** |

> ⭐⭐ **And error 2 is the interesting one, because it was not carelessness — it was the author measuring
> a cost against its own current abundance.** *The section on why an author cannot judge this was itself
> written from the position it warns about.* **The transferable form: state which deployment's constraints
> a cost claim was measured under**, the same discipline the corpus already applies to a build-state claim.

## §0b The aesthetic dimension, which the first version omitted entirely

**The omission is worth naming before the content, because it is diagnostic:** a document arguing that
organization is for other minds managed not to mention beauty once. **That is what an argument written from
the machine side looks like even when its conclusions are right.**

> ⭐⭐ **The working definition, and it is load-bearing rather than decorative: beauty here is understanding
> compressed into a form pristine enough to transfer directly from one mind to another.** On that reading,
> aesthetic quality and comprehensibility are not two criteria that happen to correlate — **the aesthetic
> response IS the felt signal of a successful compression**, which is why it arrives before the
> explanation and why people trust it.

**Three consequences that make this operational:**

1. **Accumulated design guidance is the right model for what this document is producing.** Proportion,
   symmetry and asymmetry, the rule of three, the platonic solids, the spiral: **abstracted rules,
   derived from practice, that long predate any cognitive account of why they work** — and that later
   receive partial formal explanations in terms of processing efficiency. ⇒ **a style guide in this domain
   is that kind of artifact: a rule of thumb with accumulated authority and acknowledged exceptions, not
   a theorem and not a preference.** Which is exactly the register a linter occupies.
2. **The mapping is medium-specific and the underlying account may be general.** Visual composition,
   musical structure and the organization of a technical corpus are different media; if what they share
   is efficiency of mind, then borrowing *between* them is legitimate at the level of the principle and
   illegitimate at the level of the rule.
3. ⚠ **And clarity is not the only aesthetic objective — it is ours.** Complexity, difficulty and
   disorientation are legitimate artistic ends and produce a different response deliberately.
   **We are not exploring art for its own sake: we are guiding a reader toward comprehension of a system
   built from mathematical principles and intentionally designed to be understood.** So the preference for
   pristine transmission is a *scoped* choice, and saying so is what keeps it from being smuggled in as a
   universal.

---

## §1 Coherence is not the objective, and the gap between them is where the work is

**Coherence is a property of an artifact.** It asks: do these parts contradict each other? It is
checkable against the artifact alone, which is why it was the criterion that got written down.

**Comprehensibility is a property of an artifact relative to a reader.** It asks: how much must be held
at once before any of it is usable? It is not checkable against the artifact alone, and it has no
fixture.

**They come apart, in both directions, and both are live here:**

| | |
|---|---|
| **Coherent and hard to learn** | The most economical decomposition is often the least learnable. Twenty-six orthogonal extensions with no redundancy is a perfectly coherent design in which a newcomer has no entry point — **every door leads to twenty-five others.** "Fewest documents" and "fastest correct model" are different objectives and the corpus has been optimizing the first |
| **Learnable and slightly incoherent** | A restatement that gives a reader one place to start is a duplicate, and duplicates drift. **The SDK tier is the corpus's own instance:** three `SDK-*` specs restate extension schemas *so an SDK author has one document to read*, which is a deliberate comprehensibility purchase paid for with 51 blocks that can go stale — and it needed a gate of its own to make the payment safe |

⇒ **The interesting decisions are exactly the trades between them**, and a criterion that names only one
side cannot see a trade at all. **That is the practical cost of leaving the objective unstated: the trade
gets made anyway, silently, in whichever direction the author's own load happens to point.**

---

## §2 Why comprehension is a functional requirement here specifically

**This is the part that distinguishes the claim from a preference for tidiness**, and it is specific to a
system of this kind rather than generic good practice.

The system's proposition is that integrity, authorship, authority and currency are **verifiable** rather
than asserted. A reader is asked to rely on content addressing, detached signatures at a constructable
pointer, capability chains, and a signed root's sequence. **Reliance without understanding is faith —
which is precisely the posture the whole design exists to remove.** A cryptographic guarantee that its
user cannot trace has been converted back into trust in the author.

So there is a ladder, and it has to be climbable:

```
an application convention   — "a feed entry is signed by its author"
      ↓ depends on
an extension                — the signature's home, the tree's shape, the miss path
      ↓ depends on
the core protocol           — hash = f(type, data) · path → hash · the invariant pointer
      ↓ depends on
the mathematics             — collision resistance, Merkle structure, public-key signatures
```

**Every rung of that ladder is a document someone has to find, open and connect to the one below.**
Which document, what it is called, what else it drags in, and whether it says what it depends on — **all
Regime II, all ours, and all of them decide whether the ladder is climbable or merely present.**

⭐ **Three consequences worth stating, because they are what makes this operational rather than a
sentiment:**

1. **The audience is at least three kinds of mind.** A person learning the system; an implementer under
   deadline looking for one obligation; and an agent asked to change one thing without breaking another.
   **They fail differently** — the first fails at entry points, the second at where-is-this-stated, the
   third at what-else-does-this-touch — and a single organizational choice can serve one and hurt
   another.
2. **A community's competence is downstream of this.** A reader who can trace an extension to the core
   and the core to the mathematics can argue with the design, find its defects, and advocate for it.
   One who cannot can only adopt or decline. **The second kind of reader is not a weaker advocate; they
   are not an advocate at all.**
3. **And the failure is silent.** Nobody files a bug saying *"I could not build a model of your
   system."* They leave, and the absence of the complaint reads exactly like its absence.

---

## §3 The test — a grouping earns its cost by making the best use of a bounded working set

**Hierarchy's function is to let a mind spend a fixed budget well.** `[amended §0a]` The first version of
this section called that *"permission to ignore"*, which names the symptom and gets the mechanism wrong: the
reader is not being permitted to not-know, **they are allocating a working set they did not choose the size
of.** The resource is bounded attention and retrieval cost, and hierarchy is the instrument for navigating
it — **which is why the same structure serves a person and a constrained context window (§4.2).**

> **Before adding a tier, a prefix, a family, a document or a namespace level, ask: *what can a reader now
> hold NOTHING of, and still be correct?* If the answer is "nothing", the grouping is ornament, whatever
> else is true of it.**

⭐ **Two properties of the resource make the test usable rather than vague.** **Bounded** — the budget is
finite and roughly known, so a claim about it is arguable. **Navigable** — the cost of *retrieving* what
was put down is part of the price, so a structure that lets a reader drop a subject but not find it again
has moved the cost rather than removed it. *(That second half is where an index earns its keep, and it is
the half a taxonomy alone does not supply.)*

**Worked, in both directions, on things this corpus already has:**

| | Licenses ignorance? |
|---|---|
| **Quarantining connectivity establishment in its own extension** | ✅ **Yes, maximally** — the rest of the system can ignore ICE/STUN/TURN entirely. The reader who is not doing NAT traversal never opens it. *This is the strongest instance in the corpus and it was made for a different reason (keeping accumulated-technology complexity out of the substrate) — the two criteria agree here* |
| **Transport profiles as entities rather than dispatcher branches** | ✅ **Yes** — a reader of the dispatcher can ignore how many transports exist. Six today, and the dispatcher reads the same with sixty |
| **The five-tier classification** | ⚠ **Partly.** It licenses *"I am building a single-peer thing, so Tier 2 is not mine"* — real and useful. It does not license ignoring anything *within* a tier, and Tier 1 has twelve members |
| **The `EXTENSION-` / `SYSTEM-` / `GUIDE-` name classes** | ✅ **Yes, cheaply** — a reader knows from the filename whether a document obliges an implementation, describes an emergent property, or teaches. That is a real reduction for one character of prefix |
| **Naming an extension after a namespace it owns** | ⚠ **Weakly, and honestly so.** It buys a guess — seeing `system/relay/...` you know which document to open. It does not license ignoring anything, and it is contradicted often enough (§5) that a reader cannot rely on it. **A useful mnemonic; not a structural device** |
| **Numbering the sub-tiers 2a / 2b / 2c** | ⛔ **No.** The letters carry no reader-facing meaning that the group names do not already carry. Harmless, and not structure |

⚠ **And the diagnosis this test returns on the corpus as a whole is uncomfortable: there is a great deal
of true dependency and very little licensed ignorance.** A cross-spec reference sweep finds **228
cross-spec edges among 42 documents**, with the most-depended-upon document cited by 18 of them. **That
is the measurement behind "it is getting hard to isolate one piece and refine it"** — isolation is not a
discipline problem, it is the absence of any layer whose job is to let a reader put things down.

⇒ **The most valuable Regime II work available is not renaming anything. It is creating licensed
ignorance** — and the `PROFILE` gap is the clearest instance already named: *"install this set to get
this capability"* is precisely a statement that lets a reader ignore everything outside the set.

---

## §4 The asymmetry, the trap, and the failure it produces

### 4.1 There is no machine-side signal, at all — and that part stands

**Rename every extension, handler, type and path to a UUID. Every hash still resolves, every signature
still verifies, every dispatch still lands, and the conformance suite stays green.** Organization is the
one part of this corpus with **zero** machine-checkable content.

⇒ **It cannot be validated the way everything else here is validated**, and — the sharper half — **the
absence of a failing test is not evidence that it is working.** Every other defect class in this corpus
eventually announces itself. This one never does.

### 4.2 ⚠ AMENDED — the constraint is not human-only, and the first version generalized from its own abundance

**What the first version said:** *an agent holds thousands of arbitrary identifiers at no cost, so the load
hierarchy exists to reduce is a load it does not feel.*

⛔ **That is true of the conditions it was written under and false as a general claim.** A frontier model
running with a very large context is nearly indifferent to naming. **A constrained local model is not: every
context token is contested, and the ability to navigate hierarchically — to pull in one branch, work, and
release it — is exactly how a bounded context is managed.** ⇒ ***the structure that serves a human working
set serves a small context window, for the same reason and to the same end.*** Organization is not a
concession to human limits; **it is the interface to a bounded working set, whatever kind of mind holds it.**

⭐ **What survives, and it is narrower and more useful:** **an author operating under abundance cannot feel
the cost it is imposing.** The failure is not *machine versus human* — it is *abundant versus constrained*,
and it applies to a well-resourced human author reading a corpus they already know by heart just as much as
to a large-context agent. ⇒ **the discipline is to state which conditions a cost claim was measured under**,
the same way a build-state claim is pinned to a tree.

> ⚠ **And the instance is this section.** *The passage explaining why an author cannot judge this was itself
> written from the position it warns about* — it measured the cost of naming against its own abundance and
> reported zero. **Which is the document's own thesis arriving as a defect rather than as an argument.**

### 4.3 ⛔ The failure that follows has a shape: manufacturing necessity for a choice

> **Measure a convention. Find it consistent. Promote it to an invariant.**

**Why this specific move and not some other error:** a rule is cheaper to carry than a judgment. A law
needs no justification, applies without thought, and cannot be argued with. A choice has to be re-argued
every time it is touched. **So an author under load will convert the second into the first whenever the
data permits — and a consistent pattern always permits it.**

**The worked instance, from this month, in the document that now carries the correction:**

| | |
|---|---|
| **The measurement** | `grep -c "system/x"` across the 26 extension specs → **26 of 26** |
| **The claim written** | *"The single-word extension name IS the namespace, and it is an invariant rather than a style choice,"* with three consequences: renaming the document renames the namespace · a hyphenated name implies a namespace no extension has · **grouping therefore CANNOT live in extension names** |
| **What the measurement actually measured** | **the presence of a string.** Not ownership, not exclusivity, not that anything else was absent |
| **What the corpus already contained** | ⛔ three counter-examples (§5) |
| **The cost had it stood** | a landed *"cannot"* in the architecture document, foreclosing a design axis by citing a grep |

⭐ **The generalizable tell, and it is worth more than the instance:** ***a rule derived from a census of
our own past choices cannot constrain our future ones — it can only describe the past.*** The census was
true. The inference was a category error: **consistency is evidence of a convention; it is never evidence
of necessity.** To claim necessity you need a mechanism that *fails* when the convention is broken, and
here the mechanism is absent by §4.1 — nothing fails.

**And the standing rule this earns is a recording rule, not a new prohibition:** ***a Regime II decision
is written down AS a decision, with its reason, so the next reader can disagree with it.*** A convention
with its reasons attached is usable and arguable. **A convention re-described as a law is unarguable —
which is the worst of both kinds, since it has neither the authority of mathematics nor the revisability
of taste.**

---

## §5 The counter-evidence — and one row of it was misfiled

**These are the facts the withdrawn rule ran into. Three of them are genuine exceptions with reasons
recorded, which is what a healthy convention looks like. `[amended §0a]` The fourth was not an exception at
all, and calling it one is how a comprehensibility defect got scored as evidence that comprehensibility
does not have rules.**

| | |
|---|---|
| ✅ **depth is ordinary and load-bearing** | 122 distinct three-segment namespace paths in live use: `system/type/constraint/*`, `system/substitute/sources/{hex}`, `system/capability/revocations/{hash}`. **Nothing is flat and nothing should be** |
| ✅ **a name can be deliberately outside any prefix** | `ENCRYPTION` keeps flat `system/encrypted` / `system/encryption-pubkey` — **ruled**, because a rename would rebind the hash of every encrypted entity at rest and change every `recipient_key`, which *is* a content hash of a public key |
| ✅ **placement is a judgment the team exercises, with stated conditions** | `INBOX` moved `system/protocol/inbox/delivery` → `system/inbox/delivery` as a ratified decision weighing two conditions, one of which is exactly what stopped the encryption rename. **Two extensions, one question, opposite answers, both correct** |
| ⛔ **~~`NETWORK` owns `system/peer/*`, so the convention is not a rule~~** | **MISFILED. It is the convention's strongest catch — see §5a.** The corpus is *not aligned here*, and the misalignment has a measurable reader cost. **A grouping guideline that cannot be broken is not a guideline; one whose breakage is scored as proof that it does not matter is not being used** |

> ⭐ **The third row remains the strongest argument that this is a convention and not a law.** The corpus
> has asked *"should this name move?"* twice and answered it **differently**, each time on the merits.
> **A naming law would have produced one answer and one of them would have been wrong.**

## §5a `[2026-09-14]` The audit, run — and the measurement the convention is actually FOR

**The right question is not *does the name match the namespace*. It is the reader's question:** ***holding a
path, can I find the document that specifies it?*** That is `OR-3`'s bounded-working-set test applied to the
namespace, and it is mechanical.

**Measured across `specs/` — 185 declared type paths:**

| | | |
|---|---|---|
| **168** | the first segment **names a document in the corpus** | ✅ the reader navigates by name, with nothing held |
| **17** | the first segment is a **core-owned namespace** | ⚠ the reader must already know which extension specifies it |
| **0** | neither | — |

⇒ ⭐⭐ **The convention is not decoration: it is the corpus's lookup index, and it works for 168 of 185
paths.** *That* is the argument for keeping it — not consistency, which proves nothing (§4.3), but that a
reader with a path in hand reaches the right document without an index, a search or a prior model.

**And the 17 exceptions concentrate, which is the finding:**

| namespace | paths | specified by |
|---|---|---|
| ⛔ **`system/peer/*`** | **10** | **four documents** — the core protocol (`self`, `alias`), `NETWORK` (`status`, `session`, six `transport/*` profiles), `RELAY` (`inbox-relay`), `ROLE`; and `TREE` binds `published-root` there |
| `system/protocol/*` · `system/capability/*` · `system/envelope` · `system/hash` · `system/signature` · `system/connection` | 7 | the **core protocol** — legitimately, and a reader who knows "core owns these" is done |

> ⭐⭐⭐ **So there is exactly one place in the corpus where a reader holding a path cannot get to the
> specifying document, and it is `system/peer/*`.** Ten paths, four specifying documents, no signpost
> anywhere in the namespace saying who owns what. **The candidate realignment is the obvious one** —
> connectivity-owned members move under the connectivity document's namespace, and `system/peer/*` keeps
> what the core genuinely owns (identity, alias, self) — **and it would take the exception count from 17 to
> 7, all of them core, all of them explicable in one sentence.**

⚠ **And then the honest half, which is why this is a finding and not a patch.** The rename is **not** moving
folders:

| path | refs | across | note |
|---|---|---|---|
| `system/peer/transport` | **85** | **14 documents** + core | ⛔ `profile-id` is **pinned as the final path segment** and feeds a `(priority asc, profile-id lex)` selection ordering that is **three-way cross-impl green**. The path is load-bearing in a *deterministic* rule |
| `system/peer/status` | 69 | 8 documents + core | transition-written, `MUST`-level |
| `system/peer/session` | 25 | 5 documents | holds the held-capability across reconnect |
| `system/peer/published-root` | 22 | 6 documents | the object the entire verification ladder rests on |

⇒ **This is the `ENCRYPTION` calculus again, at ten times the size, and the same two conditions decide it:
is a path segment inside something signed or hashed, and is it a value two parties must independently
derive?** For `transport/{profile-id}` both are arguably yes.

> ⛔⛔ **WITHDRAWN LATER THE SAME DAY, and the withdrawal is the most instructive thing in this document.**
>
> **There are TWO organizing axes in this corpus and the measurement above knew only one.** A namespace can
> group **by topic** (*what mechanism is this*) or **by subject** (*who is this about*). **`system/peer/*` is
> a subject index** — `status/{peer_id}`, `session/{peer}`, `transport/{peer_id}/{protocol}`,
> `published-root/{peer}`, `identity/{peer_id}` — **with `self` as the slot for *me* in an index otherwise
> keyed by others, which is what proves the axis rather than being an oddity.** And the core protocol states
> the pairing outright: *"Stored at `system/transport/{protocol}` for local transports. Remote peer
> transports at `system/peer/transport/{peer_id}/{protocol}` (same type, reused for peer discovery
> data)."* **One type, two locations, one per axis, deliberately.**
>
> ⇒ **So grouping those members by mechanism would BREAK the property that everything known about one
> remote peer sits under one prefix** — itself a comprehensibility property, and plausibly the stronger
> one, because *"what do I know about peer P"* is a commoner reader question than *"what does the
> connectivity extension define."*
>
> ⭐⭐ **What survives is a better question than the recommendation was:** ***which facts belong on which
> axis, and is duplication across axes acceptable?*** — and the corpus already answers the second one
> **yes**, for transports, and it works.
>
> ⛔⭐ **And the instrument was what misled.** The advisory checker built alongside this document measured
> the topic axis and reported every subject-indexed path as the corpus's single worst lookup failure. **That
> is a new failure mode and worse than the three before it: a purpose-built instrument reporting a finding
> that is an artifact of its own model, on its way to driving a fourteen-document rename.** Fixed — subject
> paths get their own bucket, the detector keys on the subject *key* so a second subject axis is found
> without editing the module, and reporting is at **member** granularity because
> `system/capability/policy/{peer}` is subject-keyed while `grants/{pattern}` beside it is not.
>
> ⇒ ***Before publishing a count, state what the instrument can and cannot see.*** Four measurements in
> this arc each produced a confident wrong conclusion — string-presence read as ownership, citation read as
> declaration, heading-match read as definition, and one axis read as the whole space. **All four were
> measurements whose shape was invisible in their result.**


## §6a `[2026-09-14]` ⭐⭐⭐ The radial model — and it is the same coordinate the other two axes were measuring

**The tiers, the constraint regimes and the divergence bands have been three separate tables. They are one
coordinate: distance from the mathematics.**

```
            ·  anyone's applications, built on all of it
         ·  application conventions
      ·  the outer families — connectivity · data exchange · data location
   ·  the core extensions — flat, orthogonal, minimally interconnected
·  the core protocol
◦  the mathematics
```

**Read outward, four things move together, and that co-movement is the claim:**

| Moving outward | |
|---|---|
| **degrees of freedom** | rise — from none, to a bounded set, to a wide landscape, to taste |
| **the constraint regime** | shifts I → II → II-with-III-quarantined |
| **the review method** | correctness → coherence-with-the-choice → landscape fit → cross-impl contract only |
| ⭐ **the ladder back to the mathematics** | **lengthens** — and it is the reader's climb, so **every ring outward is a ring the reader must descend to justify trusting anything** |

⭐⭐ **That last row is what the other two axes could not say.** Axis A tells you *what governs* a decision;
Axis B tells you *how many answers exist*. **Neither says that the cost of being far out is paid by the
READER, in rungs.** The radial picture says it in one move: **the outer rings are where organization matters
most, because that is where the climb is longest and the freedom is widest** — and it is exactly where the
corpus has been flattest.

**Two structural observations it produces immediately:**

1. ⭐ **The core extensions are flat because they are nearly orthogonal, and that is a property rather than a
   style.** They sit one ring out from the protocol, each exercising a small bundle of primitives, with
   little to say to one another — `CONTENT`, `COMPUTE`, `HISTORY` and `CLOCK` depend on nothing but the
   protocol; the real interconnection is one cluster (`INBOX → SUBSCRIPTION → NETWORK`, with `CONTINUATION`
   beside it). **A flat namespace is the honest shape for a ring whose members are independent.** ⇒ *the
   flatness is not the problem and should not be "fixed."*
2. ⭐⭐ **The outer ring is where families are real, and it has two of them.** Connectivity and data —
   `SYSTEM-DATA-EXCHANGE`, with location as its owed *source* chapter — are genuinely webs: many hubs, heavy
   interconnection, members that compose rather than stand alone. **That is the ring where a composition
   document earns its place**, and where the flat shape stops being honest. ⚠ **And the application
   conventions are not under `system/` at all — which is not an oversight, it is the radius showing: they
   are one ring further out, and the namespace already says so.**

## §6b `[2026-09-14]` Flat was the right first answer, and the reason it is changing now is not that it was wrong

**Premature taxonomy is a well-known failure**, and it is worse than the disorder it replaces: a taxonomy
asserted before the boundaries are known encodes a wrong model *as structure*, where it is expensive to
remove and is inherited by everyone downstream. **A flat namespace defers the choice by not making it** —
which is a real and correct move when the organizing principles are not yet apparent.

> ⭐ **So the question is not "was flat wrong" but "has the deferral expired?"** — and it expires when the
> system's boundaries, mechanisms and style guidance become apparent, because at that point the deferral is
> no longer buying option value, it is only withholding an index from the reader.

**Three signals that it is expiring now**, offered as evidence rather than as a decision: the outer families
have stabilized enough to be named; a composition tier already exists and is in use; and **the audit in §5a
returns a concrete, bounded, priced misalignment rather than a vague unease** — which is what having the
principles looks like. ⚠ **And the counter-signal, kept because it is real:** the largest single realignment
the audit finds is also the most expensive one to execute (85 references, a pinned path segment, a three-way
green ordering). **Deferral has not expired everywhere at once.**

## §7 What this does NOT claim

- **Not that organization outranks correctness.** Regime I is not negotiable for comprehensibility, and a
  clearer name that makes a wire claim false is simply wrong.
- **Not that the current organization is bad.** §3's worked table finds several groupings paying for
  themselves. The finding is that **the criterion was never stated**, so the good ones were luck or
  instinct rather than judgment — and instinct does not transfer to the next reader or the next session.
- **Not a discipline, and not a rung on the promotion ladder.** One synthesis, one incident, and the
  incident is our own. **It is a lens, and it must not be cited as though it were ratified.**
- **Not a claim about what a human finds comprehensible.** That is an empirical question about readers we
  do not have yet. **§2's ladder argument does not depend on the answer** — it says the ladder must be
  climbable, not how wide the rungs should be — and §3's test is about licensed ignorance, which is
  structural rather than a matter of taste.
- **Not settled on where grouping should live.** Namespace depth, family documents, document classes and
  tier grouping are all available and are not exclusive. **The argument is that the choice is ours, has a
  criterion, and should be recorded — not that any particular answer is correct.**

## §8 Sources

`EXPLORATION-THE-THREE-CONSTRAINT-REGIMES` §1 (Regime II, *"coherence governs; we choose"*), §2, §6 ·
`SYSTEM-ARCHITECTURE` §13.1, §13.1a, §13.1b, §13.2 (the three reasons to spec, reason 3 being
anti-fragmentation), §13.5a (both axes) · `SYSTEM-DATA-EXCHANGE` header (the composition tier's
self-description) · `SPECIFICATION-FORMAT` (document classes; the encryption and inbox naming rulings) ·
`STYLE-NAMING-CONVENTIONS` (the kebab/snake split, and the constraint-family rename) ·
`EXTENSION-NETWORK` (declared types) · the SDK tier's restatement discipline and its pin gate.

**Measured for this document, invocations quoted so the numbers are falsifiable:**

```bash
# types declared per extension, and how many sit outside the namesake prefix
grep -oE 'type: "system/[a-z0-9/_-]+"' specs/extensions/EXTENSION-*.md | sort -u
# three-segment namespace paths in live use across the published surface
grep -rhoE '`system/[a-z0-9-]+/[a-z0-9-]+/[a-z0-9{}_-]+`' specs guides | sort -u | wc -l   # 122
# the cross-spec dependency graph
spec topology specs        # 42 specs, 228 cross-spec edges
```
