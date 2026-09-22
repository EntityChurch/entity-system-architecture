# EXPLORATION — the application tier: four slots, a posture dial, and a carrier that has already converged

**Question.** What *is* an application on this protocol, structurally — and is there a general
carrier payload underneath the specific data models, or are we going to keep writing one schema per
application forever?

**Why now.** The tier has produced five conventions and two independent implementations. That is
enough instances to look for the pattern, and too many to keep adding to without one.

**The short answer, and it inverts the expected one.** The carrier is not a design problem waiting
to be solved. **Measured against the field's three-layer model for this exact question, all three
layers are in place and two of them are the strongest in deployment.** What remains is not a format
to invent but **four narrow corrections**, the sharpest of which is that the compatibility rule as
written is *pairwise* where this substrate needs it *transitive* — because a mirror replays old
entries forever.

**Method.** Structural claims are checked against landed specification text or against
implementation source. Comparative claims were read at the primary source. Where something is
unbuilt or unspecified, this document says so — that boundary is its most useful output.

---

## §1 An application is four slots, and the tree is the only coupling

The pattern was shipped **twice before anyone named it**, which is what makes it a structure rather
than a coincidence.

| Slot | A content site | An app catalog | A feed would be |
|---|---|---|---|
| **tree prefix** | `/{peer}/sites/{site}/…` | `/{peer}/apps/{set}/…` | `/{peer}/app/feed/…` |
| **vocabulary** | manifest · page · asset | catalog · bundle | entry · index-page · mirror |
| **reader** | read the subtree | read the subtree | read the subtree |
| **renderer** | site view | catalog view | feed view |
| **author surface** *(optional, separable)* | the site editor | the ingest path | a composer |

**The coupling rule, from the implementation that discovered it:** an author surface writes the
existing data model at the existing prefix, so anything it creates is picked up by the **frozen**
reader for free — *the tree is the only interface between them.* Additive and disableable: drop the
registration and the reader is untouched.

**The consequence is the argument for the whole tier.** A forum, a chat, a game world and a MUD are
**not four projects**. They are four vocabularies over four slots, **three of which are already
generic**. What differs between applications is the data model; what is shared is everything else.

**Held as an observation, not a lock.** Four is what two instances plus one design produced. A third
application is the test, and the slot list should be allowed to change when it arrives. **The
durable claim is the coupling rule** — *the tree is the only interface* — which is the part that
would still be true if the count moved.

---

## §2 The layer taxonomy is doing work it was never designed for — an open question

The tier is habitually called "L5", and the shorthand has spread far enough to be load-bearing in
documents that never defined it.

**It does not survive contact with the obvious next question: where does a compute program sit?** A
program that runs on the substrate is entity-native, is not a data-model-over-a-tree, and is not
served to a reader. Putting it in the same layer as the presentation vocabularies flattens a real
distinction; giving it the next number up invents a hierarchy nobody has justified and implies a
stack ordering that may not exist.

**Two candidate framings, and this document does not pick one:**

- **A layer number**, which implies ordering and containment. Convenient, and it is what the
  shorthand already suggests, but nothing has established that applications *contain* or *sit above*
  compute rather than beside it.
- **A kind of thing**, orthogonal to depth — *presentation vocabularies* and *execution
  vocabularies* over one substrate, both of which register a prefix and neither of which is above
  the other.

**Flagged, not resolved.** The shorthand is entrenched and mostly harmless in prose. It becomes
harmful the moment it appears in a normative sentence, because a reader will infer an ordering the
corpus has never stated.

---

## §3 Static, hosted and live are one dial

| Posture | Costs | Gains | Loses |
|---|---|---|---|
| **Live peer** | a running process | interactive dispatch, delivery, **request-time authority** | uptime |
| **Hosted publish** | storage + a publish step | no uptime, no operations | request-time authority |
| **Static / dormant** | storage only | approximately free, indefinite | nothing new until revived |

**A static store is already a full participant.** It is found the same way, verified the same way,
followed the same way; nothing in a reader's loop knows or cares whether a process is behind the
bytes. **The same identity, the same tree and the same followers move between rungs without
announcing it** — which dissolves the liveness tax that every deployed peer-to-peer system pays in
some form.

**Hosted is not a fourth posture. It is *who operates* the middle rung**, and the service is small
enough to state completely: accept signed bytes, verify the signature, write them to static
storage. No read path, no per-tenant application, no database, no session, no assembly, no queue.
**It never holds a key, so it never holds authority.** It can refuse, withhold, observe and
correlate; it cannot forge in your name, alter a byte, impersonate you, or take your identity,
followers or history. **The trust surface is availability and privacy — not integrity and not
identity.**

**The one thing static genuinely cannot do**, and it should never be blurred, is **evaluate a grant
at fetch time**. Audience control on the static path is therefore **cryptographic rather than
capability-gated**, which is why confidential entries matter and why the two mechanisms are not
interchangeable.

### §3.1 Will the distinctions dissolve? Partly — and the static rung has reasons to outlive them

As peer-to-peer integration improves, the *operational* differences between these rungs should
narrow, and some of the current gap is an artifact of immaturity rather than of design: there is not
yet a community, a hosting ecosystem, or shared infrastructure to lean on.

**But three properties of the static rung are structural and do not erode:**

1. **Cost is predictable and scales with distribution, not with demand.** No bottleneck under load,
   no capacity planning, no bill that tracks popularity.
2. **There is nothing to crash.** The failure mode of a running service — it stops, and your
   presence stops with it — has no analogue.
3. **It is the cheapest possible on-ramp**, and the on-ramp is the point (§5).

**What it trades is latency and request-time behaviour.** For a large class of publishing that is
simply the right trade, and it will stay the right trade after the live path is excellent.

---

## §4 The carrier payload — the convergence has already happened, and it is measurable

**This is the section that changes the plan.**

The intuition to check: *the specific structured types this tier keeps producing should reduce to a
general carrier the communication layer moves, whose content varies by application.* The instinct
behind it is right — this is another data layer over another network layer, a problem the field has
been solving since the 1970s, and inventing a novel answer would be a mistake.

**Two problems are routinely conflated here and they have different answers.**

**The encoding problem** — *my parser meets a field it does not know* — was settled thirty years
ago, and **this substrate is in an unusually strong position on it.** Entities are open maps with
string keys, and the type is **itself a content-addressed entity that can be fetched**. That is
close to a schema-resolution model with the registry problem dissolved: a content hash is a schema
reference that needs no registry and cannot drift.

**The vocabulary problem** — *your `post` and my `post` are different things* — is solved by nobody,
and it is where decentralized systems bleed, because it fails as **mutual unintelligibility with no
error**. The field's mitigation comes in exactly three layers:

| Layer | The field's best | **Here** |
|---|---|---|
| **1. A normative rule** — one way to do each thing; no in-place breaking change | criteria for new message kinds; schema evolution rules | ✅ **landed as three project-tier `[MUST]`s** — and *stronger* than the tier-local discipline that was asked for, because it binds every specification rather than one family |
| **2. Registry / collision prevention** | opaque integer kinds; DNS-rooted namespaced ids | ✅ **and it is the strongest of the three** — types are content-addressed **resolvable** entities: fetchable and self-describing, which an opaque integer is not |
| **3. Runtime fallback** | an alt-text convention; handler advertisement | ✅ **and the strongest** — a **mandatory non-empty fallback**, a degradation ladder, and *unknown format drops clean rather than showing source text* |

**So all three layers are in place, and the position is better than the field's.** *(A first pass of
this document said layer 1 was "drafted and unlanded". That was true when the underlying schema study
was written and had **already stopped being true**: the rule was promoted from a proposed
family-local discipline to a project-tier obligation in the same week. **A build-state claim, stale
inside a fortnight, in the document written to survey the state** — which is the recurring shape
worth noticing more than the error.)*

### §4.0 What the rule actually says

**The compatibility contract.** A published type vocabulary is a commitment: **all data valid under a
previous version must remain valid under the current one, and data produced under the current one
must remain valid under the previous.** Concretely — *new fields are optional · a field's type never
changes · a field is never renamed · a tag is never repurposed · a breaking change takes a new type
tag.* A spec that must break compatibility **mints a new name and leaves the old one meaning what it
meant.**

**Why it is an obligation and not advice, and this is the sharpest sentence in the area:** a naming
divergence **has no discovery path**. A type-filtered query using the wrong tag returns a *correct,
complete, empty* answer. It never surfaces as a byte mismatch, because the two sides never hold each
other's data at all — so the failure is invisible from both ends, indefinitely. **Its enforcement
point is that pinned tags are greppable and a shape change fails cross-implementation comparison**;
what a specification owes is that the case is *named* in its conformance inventory, so there is a
comparison to run.

**The anti-explosion gate.** A new entity type is minted **only when a conformant consumer must
behave differently**, and whoever mints one must name that behaviour; otherwise new products go to a
body type, a handler, or a renderer. **The test is behavioural, not taxonomic** — two things a
consumer treats identically are one type however differently a person would describe them.

**The container rule.** A property governing what a consumer may *do* with an entity — republish,
warn over, cache — **lives on the entity, never on a collection, index or mirror containing it.**
*Containers do not travel; entities do.*

### §4.1 The falsification, which is the strongest evidence here

**Four real objects from four deployed systems** — a post from each of two federated protocols, an
event from a relay-based one, and a message from a room-based one — were each read **from the
primary schema** and mapped onto this tier's entry shape. **All four map. Not one demanded a fifth
shape.**

**And the incidental finding is worth more than the confirmation:** all four carry *both* "this
exact thing" and "whatever is at this place now", and **every one expresses that difference as a
different shape or name — not one makes it an optional field on a single shape.** Four independent
arrivals at the same encoding decision.

### §4.2 So what is actually open

Not *"design a carrier"*, and no longer *"land the rule"*. **Four narrower things, each checked
against the landed text rather than against the proposal that asked for it:**

1. ⚠ **The contract is PAIRWISE and this substrate needs it TRANSITIVE.** As landed it binds *"a
   previous version"* — singular, i.e. each version against the one before it. The field's name for
   the stronger form is *transitive* compatibility, and it exists because teams that replay
   long-retention data get bitten by the weak one. **This system replays long-retention data by
   design: a mirror re-serves entries indefinitely, and a reader walking an old signed root is
   reading a vocabulary several versions back.** So the chain-of-pairwise-steps guarantee is not the
   one we need. **Small correction to landed text, real consequence.**
2. **No experimental namespace convention exists** — searched, and there is none at any tier. The
   field reserves a range or a marked segment precisely so that *"this is not yet a commitment"* is
   sayable. Without one, every experiment is either a permanent commitment on first publication or
   an undeclared breach of the contract above, and **the contract's own strictness is what creates
   the need**.
3. **No freeze trigger is stated.** Nothing says when a vocabulary stops being editable. The field's
   sharpest answer is behavioural rather than procedural — *a third-party implementation, even one
   built without permission* — which fits this ecosystem well, since publication is unilateral and
   nobody grants permission to implement.
4. **Defaults, and the asymmetry is peculiar to content addressing.** The field says defaults are
   what make adding and removing fields compatible. Here there is optionality and a
   hash-distinguishing absent/null rule, which is stricter and may make defaults *impossible* rather
   than merely missing: **a default a consumer materializes changes no bytes; a default a producer
   materializes changes the hash.** That has no paragraph anywhere.

**And then the design question:** how much structure the envelope carries. Too much and every
application inherits a schema it did not want; too little and nothing generic can index, check
freshness, or render a fallback. **The scorecard narrows it considerably** — layers 2 and 3 already
supply most of what an envelope would have been invented to provide, so what remains is a much
smaller decision than it looked before this pass.

### §4.3 A consequence worth stating plainly

**If layer 1 lands and the envelope question is answered, the applications built so far become
reference examples rather than the pattern.** They were built to find the shape; a later generation
would be written against the abstraction instead of alongside it. **That is a healthy outcome and
should be planned for rather than resisted** — but it is a reason not to over-invest in the current
vocabularies before the governing rule is in place.

---

## §5 The on-ramp is the point, and it is the gate

**The tier's claim is not technical:** *a person publishes under a name they chose, to storage they
can pay for or be given, other people read them in an ordinary client, and if whoever hosts them
disappears they lose nothing but uptime.*

**Why that claim is the product.** People do not publish to their own domains today because the path
is hard — the domain, the hosting, the operations, the maintenance. They use a large platform
instead, and accept the terms, because the alternative costs more than they want to spend. **If
publishing your own photographs and writing were genuinely as cheap and as easy, the reason to
accept those terms mostly evaporates.** The architecture above is what makes that plausible; the
remaining friction is operational, and it is exactly the friction the hosted rung removes.

**What gates it today, measured, is the on-ramp and not the vocabulary.** In the shipped
implementation the publish path runs on **a fresh ephemeral peer seeded with a bundled demo set** —
its own module header calls it *"a demo / SSG generator, honestly so"* — rather than on the user's
live profile. **A composer that writes a perfect entry therefore produces something nobody can
see.** This blocks the *claim*; it does not block the vocabulary work, and the vocabulary should not
wait for it.

**But the gap is a wiring gap, not a missing capability, and that is the most useful thing this
survey re-measured.** The seam is already named in the source, with the assertion that *swapping the
"fresh + seed demo" body for "load peer dir → read its real sites" leaves every emitter downstream
unchanged* — and **the native backend already loads the durable store and the key together**, in one
record carrying both the keypair and a concrete on-disk store path. **The two halves exist on the
same side of the wall and are not connected.** So this is a smaller and much better-understood piece
of work than *"publishing is unimplemented"* suggests.

**And it is the structural reason the native build is the natural publishing host** (§4): the two
things the publish seam needs are already resolved together there and nowhere else. Any other host
would have to re-implement the half that already exists.

**The pipeline underneath it — domains, storage providers, cache behaviour, transport semantics — is
deliberately unspecified for now.** It is the least protocol-native surface in the system, the one
where these guarantees stop applying and a hosting provider's begin, and it is still being explored
in practice. **The constraint worth holding meanwhile is already derived and is checkable in review: a
hosted service must not be able to produce any entity a self-hoster could not produce identically.**

---

## §6 What is built, what is specified, what is neither

| Capability | State |
|---|---|
| Resolve name or key → peer → origin | 🟢 **built** |
| Walk any peer-relative tree key at any origin, **verified** | 🟢 **built**, generic |
| Bounded lazy traversal | 🟢 **built** — the navigation primitive the design notes list as missing |
| Currency and freshness for foreign bytes | 🟢 **built** — one chokepoint, with a lint making it the only legal path |
| Retry policy that never retries a terminal error | 🟢 **built** |
| Foreign content stored at the **author's** path | 🟢 **built** — universal namespace |
| Multi-peer projection | 🟢 **built** — a root claims only its own |
| Static and live indistinguishable to a consumer | 🟢 **built** — the consumer never asks |
| Byte fidelity when re-serving a foreign encoding | 🟢 **built and measured**, falsified in both directions |
| The shared reference atom | 🟡 **specified**, not yet exercised |
| Feed / share / embed / site vocabularies | 🟡 **specified**; exercise belongs to the implementations |
| **The compatibility rule (layer 1)** | 🟢 **landed**, as three project-tier obligations — but **pairwise where it needs to be transitive**, §4.2 |
| **Follow set, cursor, refresh loop** | 🔴 **neither** — three shipped consumers want it, none has it |
| **Publish from a live profile** | 🟡 **both halves built, not wired** — the seam is named and the durable store and key already load together on the native side; the headless publish binary does not go through that path (§5) |
| **The hosted-publishing handoff** | 🔴 **specified nowhere** |

**One correction this table encodes:** several capabilities recorded elsewhere as missing are built.
**Read the implementation before scoping work against it.**

---

## §7 Sequencing — by dependency and relative scale, not by date

**No calendar estimates.** Sizes are relative and are anchored to whether a template exists in
shipped code.

| # | Work | Scale | Rests on | Note |
|---|---|---|---|---|
| 1 | **Make the compatibility contract transitive**, and add an experimental namespace + a freeze trigger | **S** | nothing | The rule is landed; these are corrections to it. Cheapest high-value work on the board |
| 2 | **The refresh loop as substrate** | **M–L** | nothing | **Do it before any feed work.** Three shipped consumers already want it, none has a cursor, and it carries the most design risk. A loop falsified against three real callers beats one falsified against a hypothetical |
| 3 | **Name the application ABI** (§1), including the emit slot | **S** | — | Prevents a third application growing its own arm |
| 4 | **Feed vocabulary + reader + composer** | **S/M each** | 2, 3 | Every piece has an in-tree template |
| 5 | **Publish from a durable store** | **M** *(was sized L)* | — | **Not a missing capability — a wiring gap.** The seam is named, its downstream is asserted unchanged, and the store and key already load together on the native side. **Re-sized after a source read; the L came from a survey** |
| 6 | **The hosted handoff, and the live rung beside it** | **?** | the pipeline settling | Gates the §5 claim. Not sized until the shape stops moving |
| 7 | **The envelope question** (§4.2) | **M** | 1, and enough vocabularies to generalize from | Arguably reachable now, and smaller than it looked |

**The ordering that matters most is 2 before 4**, and it inverts the obvious one.

---

## §8 What this document is not

**A synthesis, not an authority.** Every claim has a home that owns it — the conventions in the
applications tier, the posture and hosted analysis in the convergence work, the three-layer
scorecard and the four-object falsification in the schema-evolution and falsification explorations.
**Cite those, never this**; a survey that becomes a competing home for a rule is the failure mode
the design register exists to prevent.

**And it will go stale.** It is edited in place when the picture changes, and the picture changes
whenever an implementation ships something.
