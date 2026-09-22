# EXPLORATION — there is no single reader: the audience model was already on disk, and the one party it never names

**Status:** Exploration (design record). Not a proposal, not normative. **Nothing is folded here.**
Companion to `EXPLORATION-THE-ORGANIZATION-IS-A-DEGREE-OF-FREEDOM-AND-COMPREHENSION-IS-WHAT-IT-BUYS`,
which established *what* organizational decisions optimize for. **This one answers the question that
study's own architecture section is titled for and never answers: whose understanding?**

**Why it exists.** The comprehensibility axis is stated in terms of *a reader* — "another mind", "a
person", "a newcomer". Singular and undifferentiated. That is not a wording slip: it is the reason the
axis cannot settle an argument. Two organizational choices that serve different readers look equally
defensible until you say which reader the corpus is for, and **the corpus had never said.** The gap was
already recorded as an open prerequisite under every namespace question; this document is the work of
closing it, and the work turned out to be mostly *measurement*, because the model was already written
down in 27 places and had simply never been collected.

---

## §0 The result

| | |
|---|---|
| **The model already exists, distributed** | 27 documents open with an `**Audience:**` declaration naming who they are for. **Nothing prescribes that field** — it is in no format standard, has no gate and no vocabulary. It was adopted by convention, and it is consistent enough to collate |
| **Adoption is inverted against stakes** | **23 of 36 guides** declare an audience. **0 of 26 extension specs.** 0 of 5 application conventions, 0 of 1 domain spec, 1 of 3 SDK specs. The teaching layer knows who it is talking to; **the normative layer does not** |
| ⛔⭐ **The party the corpus is about is the party no document addresses** | The word *user* appears in **358 lines** of the published surface, is defined in **no glossary**, and appears in **zero** of the 27 audience declarations. Every declared audience is someone who builds, runs, reviews or reads the system |
| **Why the model was missing, mechanically** | ⭐ **The corpus's reflex is to name a machine role where a human role is the subject.** The reserved-prefix table's "who uses it" column answers with *peers* — "every peer", "substrate-aware peers", "the runtime". The vocabulary had no slot for a person, so no person got defined |
| **Six classes, and they navigate differently** | §3. Two navigate by **mechanism**, two by **subject**, two by **the ladder**. That is not a taxonomy for its own sake — **it is the same two axes the namespace turned out to have** |
| ⭐⭐ **The tiebreak, and it is not a preference ranking** | §4. When two classes' organizational interests conflict, rank them by **which comprehension failure is recoverable.** An author's failure costs time and they recover. A deployment operator's failure is a **misauthorization** — not recoverable, and it lands on the party who read nothing |
| **What it settles** | §5. The subject axis serves whoever runs a deployment; the topic axis serves the extension author. **Ranking the readers ranks the axes**, which is the open namespace question reached from the other end — and the capability namespace already resolved it this way on its own |
| **What it separates** | §6. Three decisions have been arriving as one. **Runtime namespace paths · specification directory layout · install-set vocabulary.** They differ in blast radius by two orders of magnitude and only the first is expensive |

---

## §1 The claim that was true and is now measurable

The open prerequisite was recorded as: *there is no stakeholder model — who is a "user", who runs a
deployment, who authors an extension, who calls an operation? — and any future rule about the `system/`
namespace needs the model first.* It was filed after a normative rule was found to have been built, and
to have survived eight revisions, on the undefined word *user*.

**That record is correct and this document does not overturn it. What it adds is that the raw material
was never missing** — it was distributed across document headers where nothing could collect it, which
is the same shape as every other finding in this corpus about a fact with no canonical home.

---

## §2 The measurement

### 2.1 The convention nobody wrote down

27 of 78 documents on the published surface open with:

```
**Audience:** <who this is for>, <what it assumes they have read>
```

**The format standard does not mention it.** There is no rule requiring it, no gate checking it, and no
controlled vocabulary for the parties it names. It is a pure convention — which makes it a clean instance
of the thing the companion study warns about in the opposite direction: *a consistent pattern is evidence
of a convention and never of a necessity*, and here the convention is genuinely useful and has simply
never been ratified, indexed or completed.

### 2.2 Adoption, and it is inverted

| Surface | Declares an audience |
|---|---|
| Guides | **23 of 36** |
| Architecture / composition specs | 3 of 7 |
| SDK specs | 1 of 3 |
| **Extension specs** | ⛔ **0 of 26** |
| Application conventions | 0 of 5 |
| Domain specs | 0 of 1 |

⇒ **The documents that teach declare who they teach. The documents that bind declare nothing.** That is
backwards on every axis that matters: the extension specs are the widest-read surface, the one
independent implementations are built from, and the one a reader is most likely to arrive at without a
guide in hand.

### 2.3 The party vocabulary, as actually used

Counted across the 27 declarations:

| Party named | Declarations |
|---|---|
| application developer | 14 |
| deployment / peer operators | 8 |
| peer implementer | 8 (+3 "implementation team") |
| extension or spec author | 6 |
| architecture / reviewer | 5 / 4 |
| SDK implementer or builder | 4 |
| new reader, newcomer | 3 |
| contributor | 1 |
| ⛔ **user** | **0** |

### 2.4 ⛔ The absence is the finding

**The word *user* occurs in 358 lines of `specs/` and `guides/`.** It is the corpus's most-used party
noun. It is **defined nowhere** — no glossary, no terminology section, no format standard — and it names
the audience of **none** of the 27 documents that declare one.

**So the party the system is ultimately built for is the only party with no document, no definition and
no declared entry point.** Every other named party is someone who *builds or runs* the system.

That is not an oversight to be embarrassed about — it is arguably correct that no specification is
addressed to an end user. **But it means that when a design argument invokes "the user", the word is
doing work that nothing in the corpus can check**, which is precisely how an undefined party ended up
carrying a normative rule for eight revisions.

---

## §3 The classes, and how each one navigates

Collapsing the measured vocabulary gives six classes. **The column that matters for organization is the
second one** — not who they are, but *what shape of question they arrive with*.

| Class | Arrives asking | Navigates by | Entry point |
|---|---|---|---|
| **Deployment operator** | *"who is this peer, what does it hold, what did I grant it, can I reach it?"* | ⭐ **SUBJECT** | a running peer's tree |
| **End user** | *"did my thing work?"* | — | the application; **reads none of this** |
| **Application developer** | *"what do I need to do X?"* | **capability** — outcome first, mechanism later | a guide, then the SDK specs |
| **Extension / spec author** | *"what mechanism is this, what does it own, what may I assume?"* | ⭐ **TOPIC** | the extension spec |
| **Peer implementer** | *"what exactly must I produce?"* | the **normative surface, exhaustively** | the core protocol + conformance |
| **Reviewer, auditor, newcomer** | *"is this claim true, and can I check it myself?"* | **the ladder** — convention → extension → protocol → mathematics | wherever they landed |

**Three observations, and each one does work later:**

1. ⭐⭐ **Deployment and authoring navigate by the two axes the namespace turned out to
   have.** That is not a coincidence and it is the central result of this document. The corpus's two
   organizing axes — *what mechanism is this* and *who is this about* — are not a taxonomic curiosity;
   **they are two readers, and each axis is one of them made structural.**
2. **The implementer is nearly indifferent to grouping and acutely sensitive to completeness.** An
   implementer reads everything anyway. Reorganizing for them buys almost nothing; omitting something
   costs them a conformance failure. ⇒ **they are not a vote in an organization argument**, and treating
   them as one has inflated the apparent cost of every proposed move.
3. **The developer wants neither axis — they want an install set.** *"What do I need to do X"* is not
   answered by any grouping of documents; it is answered by a named set of components that travel
   together. That is a different artifact entirely (§6).

---

## §4 ⭐⭐ The tiebreak: rank by which failure is recoverable

Most of the time the classes do not conflict, and where they do not, the question is not interesting.
**Where they do conflict, a ranking is needed, and "comprehensibility" alone cannot supply one** — both
sides of every such argument are comprehensibility arguments.

**The ranking is: deployment operators, and through them end users, take priority over the reader working in
the internals.** Stated as a preference that would be arbitrary. It is not arbitrary, because the
classes differ in something observable:

| Class | What a comprehension failure costs them | Recoverable? |
|---|---|---|
| Extension author, implementer | **time** — a wrong model, found and corrected while building | ✅ yes, and the build is what corrects it |
| Application developer | **time**, plus a design they revise | ✅ yes |
| ⛔ **Deployment operator** | **a wrong authorization** — a capability granted to the wrong peer, a scope wider than intended, a reachability record trusted that should not have been | ❌ **no.** An exercised capability cannot be un-exercised, and content already fetched cannot be recalled |
| ⛔ **End user** | the consequence of the above, **having read nothing and chosen nothing** | ❌ no, and they had no opportunity to prevent it |

> ⇒ **The tiebreak is not "who matters more." It is that one class's comprehension failure is a
> security outcome borne by a third party, and the others' is a delay borne by themselves.**

**Two things keep this honest and both matter:**

- ⚠ **It applies only where the interests genuinely conflict**, which is rarer than it sounds. Where a deployment
  has no exposure — an authoring convention, a document's internal section order, a tier
  label — the author's convenience is the only signal present and wins by default. **A priority is not a
  licence to degrade the author's surface in the name of a reader who never opens it.**
- ⚠ **It is a lean, not a law.** It says which way to fall when the arguments are otherwise balanced. It
  does not license an expensive move on a thin deployment-side benefit, and **the cost side of the ledger is
  unchanged by it.** Promoting it to an unarguable rule would be exactly the failure the companion study
  documents.

---

## §5 What this settles: ranking the readers ranks the axes

The open namespace question is: **which facts belong on which axis, and is duplication across axes
acceptable?**

**The reader model answers the first half without a single further namespace measurement**, because each
axis has exactly one primary class:

| Axis | The reader it serves | Priority |
|---|---|---|
| **by SUBJECT** — *who is this about* | **deployment operators** | ⭐ higher, per §4 |
| **by TOPIC** — *what mechanism is this* | the **extension author** | lower, where they conflict |

⇒ **A fact needed while looking at one peer belongs on the subject axis.** A fact that only
makes sense as part of a mechanism's definition belongs on the topic axis. **Where a fact is both, it is
carried on both** — and the second half of the question is already answered *yes* by landed text and by
practice: one transport type is defined once and located twice, locally by protocol and remotely by peer,
deliberately.

### 5.1 ⭐ The capability namespace already did this, on its own

**The strongest evidence is not in the transport pair, which was already known — it is at the security
surface, where §4 predicts the axis split should matter most.**

The capability namespace carries **both axes side by side**, and the split falls exactly where the
priority ranking says it should:

| Path | Keyed by | Serves |
|---|---|---|
| `system/capability/policy/{peer_pattern}` | ⭐ **a peer** | the **deployment-side** question — *what does peer P get from me?* |
| `system/capability/grants/{pattern}` | a resource pattern | the **mechanism's** question — *what authority backs this handler?* |

**Nobody derived that from a reader model, because there was no reader model.** The deployment-facing
question was put on the subject axis and the mechanism-facing one on the topic axis, by authors following
the grain of each question. **That is the model being confirmed by a case it did not produce**, which is
the strongest form of support available for a claim of this kind — and it is a fifth independent piece of
evidence for the two-axis reading, at the surface where getting it wrong is most expensive.

### 5.2 And it independently confirms the withdrawal

A proposal to regroup the peer-subject namespace by mechanism was drafted and withdrawn when the second
axis was found. **The reader model reaches the same conclusion from the opposite direction and for a
better reason:** that regrouping would have taken the deployment-side namespace — *everything known about one
peer under one prefix* — and rearranged it to suit the class whose comprehension failures are the ones
that get corrected by building. **Two independent derivations landing on one answer is the only kind of
confidence available on an axis with no machine-side signal.**

---

## §6 Three decisions have been arriving as one

Much of the difficulty in this arc has been that *"should the extensions be organized into families?"* is
three different questions wearing one sentence. **They differ in blast radius by two orders of magnitude,
and only the first is expensive.**

| | What actually moves | Cost | Reversible? |
|---|---|---|---|
| **1. Runtime namespace** — `system/*` path segments | **entity paths.** Hashed, signed, pinned inside deterministic orderings two parties compute independently | 21–107 references per candidate; the peer-transport segment is **85 references across 14 documents**, with a path segment pinned inside a cross-implementation-green selection ordering | ⛔ no — a rebind changes hashes |
| **2. Specification directory layout** — which folder a spec file sits in | **document filenames.** No entity path, no hash, no wire byte, no type tag | §6.1 | ✅ yes, mechanically |
| **3. Install-set vocabulary** — naming which components travel together | **nothing.** A new declarative artifact beside the existing ones | purely additive | ✅ yes |

### 6.1 What a directory regrouping actually costs, measured

**Inside this corpus it is nearly free**, and the reason is a property of the house citation style:

- **869** references to extension specs in `specs/` and `guides/` are **bare filenames** — `EXTENSION-X.md`
  with no directory. A folder move does not touch one of them.
- Only **4** citations on the published surface carry the directory path.
- 126 more carry it in the internal workspace, a third of which are in frozen archive material.

**Outside this corpus the cost is real and it is concentrated in one place.** 74 files across nine
sibling repositories cite an extension spec by path; 47 are prose. **The load-bearing remainder is the
provenance layer of the conformance and generation pipeline** — vendored-copy manifests recording which
source file a fixture set was cut from, one drift-checking tool, and one machine-read track table — plus
fourteen source files carrying the path in a doc comment.

> ⚠ **That is the finding that changes the sequencing, not the total.** The manifests exist to answer
> *what was this generated from*, so a move that invalidates them during a generation round destroys the
> provenance of that round. **This decision is cheap before or after a generation wave and expensive
> during one.**

⚠ **A correction worth carrying, because it is the third instance of this exact error in this toolkit:**
the first count of the sibling surface was **191 files in one repository**. 175 of them were build
output, five vendored snapshots of the same documents, and worktree copies — **not one was a distinct
citation.** A tree walk that does not exclude generated and vendored copies is not a census. The true
figure there is 10.

### 6.2 ⭐⭐ The package-manager analogy argues for the third decision and against the first

The intuition that extensions should come in families is usually expressed through package management:
*"install the development group and you get the compiler and the build tooling."* **Taken seriously, that
analogy argues against moving anything.**

A distribution that ships a `build-essential` meta-package **does not relocate the compiler into a
`devel/` directory to do it.** The package namespace stays flat and global; the grouping is an
**aggregation layered over it**, it is explicitly non-exclusive, and a user who wants a different
combination simply names one. **The flat list and the group are compatible by construction — that is the
whole design, and it is why the analogy is a good one.**

⇒ **The benefit the analogy promises is delivered entirely by decision 3, which costs nothing and moves
nothing.** Decision 1 delivers a different and much smaller benefit at a much higher price, and it
forecloses combinations rather than describing them.

**This is not a new conclusion so much as a newly-motivated one.** The corpus's own measurement of its
organizational debt found the same thing and ranked it plainly: with 228 cross-document dependency edges
among 42 documents, **the most valuable organizational work available is not renaming anything — it is
creating a layer that lets a reader put things down**, and the install-set vocabulary is the clearest
instance already named. **The reader model says who that layer is for: the application developer, whose
question is the one no grouping of documents answers.**

---

## §7 What this does NOT claim

- **It does not claim the audience convention is correct as it stands.** 27 declarations, no controlled
  vocabulary, and the parties are named in a dozen spellings. It is raw material, not a model.
- **It does not claim the end user should become an audience of any specification.** The measured absence
  is offered as the reason to stop using the word loosely in design arguments, not as a gap to fill with
  documents.
- **It does not rank the classes outside a genuine conflict**, and §4 says so twice on purpose.
- **It does not settle any individual namespace move.** The two landed tests — *is the segment inside
  something signed or hashed*, and *is it a value two parties independently derive* — are unaffected by
  anything here, and they decide the cost side. **This document decides only which way to lean when the
  costs are comparable.**
- **It does not claim the reader classes are exhaustive.** Six is what the measured declarations support.
  A seventh would be evidence, not a problem.

---

## §8 Sources

- The 27 `**Audience:**` declarations across `specs/` and `guides/` — the whole of the distributed model.
- The reserved-prefix table's *who uses it* column, whose answers are peers rather than people.
- The capability namespace's two keyings, `policy/{peer_pattern}` and `grants/{pattern}`.
- The transport pairing — one type, two locations, one per axis.
- The companion exploration on organization as a degree of freedom, and the architecture axis it feeds.
- The organizational-debt topology measurement: 228 cross-document edges among 42 documents.
