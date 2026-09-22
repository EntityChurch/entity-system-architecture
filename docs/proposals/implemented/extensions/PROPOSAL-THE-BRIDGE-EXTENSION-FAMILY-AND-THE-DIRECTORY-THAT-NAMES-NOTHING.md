# PROPOSAL — `guides/GUIDE-BRIDGE-EXTENSION-DEVELOPMENT.md` + the `EXTENSION-BRIDGE-*` family: recover the shape that was already decided, and replace `specs/domains/` with `specs/bridge-extensions/`

**Status:** **RATIFIED AND FOLDED (2026-09-14).** Opened the same day as a scoping draft; D1–D5 are
landed, D2 in a form that reverses this proposal's own first draft, and D6 is new from the audit that
preceded the fold. **The deliberately-unruled items in §5–§7 stay unruled.**
**Target (all landed):** a new `guides/GUIDE-BRIDGE-EXTENSION-DEVELOPMENT.md`; a new
`specs/bridge-extensions/` directory replacing `specs/domains/`; `specs/SYSTEM-ARCHITECTURE.md` §2 layer
diagram, §7.1 directory map, §13.1 classification and §13.5b name-class inventory; the reserved-prefix
table in `guides/GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §4.1; `ROADMAP-EXTENSIONS.md`; the forward
citations in `GUIDE-EXTENSION-DEVELOPMENT.md`, `GUIDE-CROSS-PEER-MESSAGING.md`, `EXTENSION-SUBSTITUTE.md`,
`EXTENSION-NETWORK.md` and `EXTENSION-REVISION.md`. **No wire change, no renumber, no core change, no
normative rule added or removed.**
**Provenance:** the organization arc's third study,
`docs/research/explorations/EXPLORATION-THE-BRIDGE-LAYER-IS-ALREADY-DESIGNED-AND-DOMAIN-IS-FOUR-DIFFERENT-WORDS.md`,
plus the pre-split archive's transport-family/bridge-shape study and its bridges-framing review, which
decided most of this and did not survive the repository split.
**Scope:** **additive and organizational.** Nothing here changes a protocol behaviour. Two items touch
shipped identifiers and are called out as such (§6).

---

## §0 The result

| | |
|---|---|
| **The family shape was decided and is recoverable** | Three layers — protocol / transport / **adapter**. Bridges are the adapter layer. **No umbrella bridge extension**: *bridge is a development-time category, not a runtime surface* |
| **The naming was decided too** | `EXTENSION-BRIDGE-<TECHNOLOGY>` per bridge, plus one non-normative `GUIDE-BRIDGE-EXTENSION-DEVELOPMENT` — **the same shape the rest of the corpus already uses** |
| ⭐ **A bridge IS an extension, and it sits at L5** | These are not competing answers. **Artifact class and layer are different axes**, and conflating them is what made the placement question feel unanswerable |
| ⛔ **NINE citations across SEVEN documents name a bridge specification that does not exist** | Not two. **Five of the seven are landed specs.** Two technologies are named — a web bridge and a mail bridge — and neither resolves. §1a |
| ⛔ **Bridge material is already IN the landed corpus, and the study that opened this said it was not** | A version-control bridge mapping and handler sketch is **a whole section of a landed Tier-1 extension spec**; a bridge is one of seven named peer compositions; a future auth bridge is named in a guide. §1a |
| ⛔ **THREE prefixes are in play for bridge state, and the reserved one is the only empty one** | `bridge/` reserved, zero occupants · `system/bridge/` in the web-bridge forward references · `app/bridge/` in the version-control sketch. **The first bridge authored locks this**, and today it would lock it by accident. §4/D6 |
| **`specs/domains/` is replaced by `specs/bridge-extensions/`** | One member, a circular definition, and **the founding study explicitly rejected the word** — *"bridge is a neighborhood, not a domain"*. ⭐ **A sibling directory, NOT a merge into `specs/extensions/` — see D2, which reverses this proposal's first draft** |
| ⚠ **The seven cross-bridge questions already exist** | And they were extracted **from the one document that was in `specs/domains/`** — which is the strongest argument that it is a bridge and was always treated as one |
| ⚠ **The document name is NOT the cheap half of the rename, measured** | Namespace **438 files** · document name **119, of which 67 are source code** · directory path **7**. **Three identifiers, two orders of magnitude apart** — and only the path is free. §6.1 |
| ⇒ **So the correction to the misnamed member is ADDITIVE, and the rename is dropped** | A successor bridge for the general filesystem case, beside the old one, which deprecates once consumers have somewhere to go. **Nothing is renamed and no seat pays until it chooses to move.** §6.2 |

---

## §1 The problem, stated so it is not mistaken for a naming preference

**There are no bridge specifications.** The category is designed, reserved in a top-level path prefix,
and cited by two landed documents, and it has zero members. Meanwhile a single document sits in a
directory whose name is defined once, circularly, in the architecture map.

**Three consequences, all live:**

1. **Two dangling normative pointers.** `GUIDE-EXTENSION-DEVELOPMENT` §3.7 tells an author that HTTP as
   foreign substrate *"lives in `EXTENSION-BRIDGE-HTTP.md`"*, and an extension spec's related-documents
   line names the same file. **Neither resolves.** An author following the guidance arrives nowhere.
2. **The first bridge written sets the convention**, whether or not anyone decides it does — and the
   convention was already chosen once, in a document that did not cross the split.
3. **The existing member is inconsistent with the family it belongs to.** It is named for a directory
   rather than for what it is, and the cross-bridge discipline the family needs was *derived from it*.

## §1a ⛔ The audit before the fold — what the opening study got wrong, and it was wrong the same way twice

**This section exists because the study that opened this proposal ran a census by filename and reported
it as a census of the subject — and its own §5 carries a correction for having done exactly that, one
paragraph earlier.** Third instance in one arc. The corrected measurements:

### 1a.1 The dangling citations are nine across seven documents, not two

| Document | Citations | Names |
|---|---|---|
| `EXTENSION-SUBSTITUTE.md` | 2 | web bridge |
| `EXTENSION-NETWORK.md` | 1 | web bridge |
| `EXTENSION-RELAY.md` | 1 | web bridge |
| `EXTENSION-TREE.md` | 1 | web bridge |
| `GUIDE-EXTENSION-DEVELOPMENT.md` | 1 | web bridge |
| `RUNBOOK-CDN-BROWSER-DEPLOYMENT.md` | 1 | web bridge |
| `GUIDE-CROSS-PEER-MESSAGING.md` | 1 | **mail bridge — a second technology the study did not surface** |

**Five of the seven are landed specifications**, and the addressing standard's own rule is that *an
unmarked forward citation is an error*, permitted only when marked `(planned)` and confined to a
non-normative note. **Most of these were unmarked.**

> ⛔ **The instrument had already found them, and the finding was never read.** The addressing analyzer
> reports every one of these in its `dangling` class. Its human-readable output prints **five examples
> and then `... +170 more`** — so a run that was executed, passed over and summarized as a count never
> surfaced a single bridge citation. **A truncated reader is not a measurement, and the truncation is
> invisible in the summary line.** *(Fixed in the toolkit this session: the analyzer gains the `--owed`
> flag every other gate in the set already had.)*

### 1a.2 Bridge material is in the landed corpus — the census said zero

| Where | What |
|---|---|
| `EXTENSION-REVISION.md` **§10, "External System Bridge"** | ⭐ A **complete version-control bridge mapping** — entity-concept ↔ foreign-concept table, import walk, export walk — plus §10.2's handler-operation sketch. **In a landed Tier-1 extension spec** |
| `GUIDE-PEER-COMPOSITIONS.md` §5.6 | **Bridge as one of seven named peer compositions**, with its capability discipline and a flag that it carries the highest cycle risk of the seven |
| `GUIDE-SERVING-MODE.md` | Names a **future auth bridge** as a member of the web-bridge family, explicitly conditional on a driver |
| `GUIDE-EXTENSION-DEVELOPMENT.md` §3.7 | The transport-versus-foreign-substrate discipline — the one piece the study did find |

⇒ **The claim *"no bridge but the web one is even named"* was false when written: version control is
named, specified at the mapping level, and sketched down to its operations.**

### 1a.3 ⛔ Three prefixes, and the reserved one is the empty one

| Spelling | Where | Occupants |
|---|---|---|
| `bridge/` | **reserved** for external system connections in the namespace guide §4.1, and named as reserved in `SDK-OPERATIONS.md` | **zero** |
| `system/bridge/...` | what the web bridge's forward references actually spell — both the entity type and the capability | the forward references |
| `app/bridge/...` | what the version-control sketch spells | the sketch |

**This is the same defect the opening study measured for `host/` versus `local/`, one level down and
not previously seen: a reserved prefix has no occupants because everything actually written went
somewhere else.** Reserving a prefix and specifying one are different acts, and only the second leaves a
document. ⇒ **D6.**

## §2 What is already decided, recovered

### 2.1 Three layers

| Layer | Holds | Pluggable |
|---|---|---|
| **Protocol** | envelope · signature · capability chain · identity · entities | **No — frozen across every transport** |
| **Transport** | how bytes get from A to B: live · async-queued · polling-static · mesh-pull · one-way-authenticated · routed-circuit · physical | Yes — per-peer transport profiles |
| **Adapter** | **bridges** — translating a foreign system's idioms into protocol calls | Yes — one extension per bridge |

### 2.2 No umbrella extension, one extension per bridge

**"Bridge" is a development-time category, not a runtime surface.** There is no `EXTENSION-BRIDGE`.
Unification at the all-bridges level was examined and rejected on four fracturing signals: **different
trust models, different idempotency, different capability flows, different failure surfaces.**

**What is shared is knowledge, not runtime** — hence one non-normative guide beside the per-bridge specs,
exactly parallel to the extension-development guide that already exists.

### 2.3 Direction is a mode, not a second extension

**Egress and ingress flip the threat model, the capability-flow direction, the idempotency properties and
the failure surface — and this does not force two extensions.** Each bridge is **one extension with
install-time `egress-only` / `ingress-only` / `both` modes**, the same mode-axis shape an existing
extension already uses.

⇒ **A conformance section MUST cover all three configurations**, including the negative: an
ingress-only deployment rejecting egress operations with a clear capability error.

⚠ **This is the answer to *"is an extension a unit of installation?"* — and the honest answer is no.**
An extension is a coherent **specification** surface; what a deployment installs is that surface *in a
declared mode*. **Half a bridge is a conformant deployment, not a partial install**, and saying so
explicitly is what stops the mode axis from being read as a loophole.

### 2.4 The seven cross-bridge questions

Every bridge specification answers all seven. **They were extracted from the existing local-filesystem
specification**, which is the prior art the whole family framing was derived from:

1. **Authority assignment** — who may act on the foreign system, and on whose behalf.
2. **Capability translation, both directions** — protocol capability → foreign permission, and back.
3. **Namespace convention** — where the foreign system's objects land in the tree.
4. **Sync-state visibility** — bridges introduce a **third** state beyond have / do-not-have:
   *fetching*, *stale*, *remote-unreachable*, and it must be observable rather than inferred.
5. **Cache invalidation** — when a held copy stops being an answer.
6. **Hash-identity translation** — a foreign identifier is not a content hash; state the mapping.
7. **Idempotency semantics on egress** — what a repeated outbound operation means.

### 2.5 The transferable hazards

- **External path resolution is adversarial input.** The symlink-escape class generalizes: version-control
  refs, HTTP redirects and package-store path resolution all surface it.
- **Time-of-check/time-of-use under external mutation** — the foreign system changes without telling us.
- **Boundary capability discipline must be explicit**, never implicit-by-reference.

## §3 ⭐ A bridge is an extension, and it sits at L5 — both, without conflict

**The placement question felt unanswerable because two different axes were being asked as one.**

| Axis | Question | Answer for a bridge |
|---|---|---|
| **Artifact class** | *what kind of document/surface is this?* | **An extension.** It contributes entity types, handler operations, storage conventions and a conformance section — which is the definition |
| **Layer** | *how far from the mathematics does it sit?* | **L5.** It speaks the entity language inward and a foreign protocol outward |

**Why the layer answer felt wrong at first:** L5 is where applications sit, and a bridge is not an
application — it does not depend on one and nothing built on it needs one. ⇒ **L5 is not one direction.
Applications build UP on the system; bridges reach OUT of it.** Same distance from the mathematics,
different bearing. **A bridge is an orthogonal dimension at the same radius**, which is why it kept
reading as *"nearly the application tier, maybe past it, not really either."*

> ⇒ **Neither answer excludes the other, and the directory map's annotations are what implied they did**
> — three artifact-class directories annotated with layer numbers, which works only because those three
> happen to be layer-aligned. **A fourth that is not layer-aligned then has nothing to write in the
> column**, and the circular annotation is the symptom.

## §4 Proposed dispositions

### D1 — Adopt `EXTENSION-BRIDGE-<TECHNOLOGY>` as the family naming `[LANDED — the decision that was free today]`

`EXTENSION-BRIDGE-HTTP`, `EXTENSION-BRIDGE-GIT`, `EXTENSION-BRIDGE-NIX`. **This is not a new choice —
it is the one already made, and it is the spelling all nine dangling citations already use**, so
adopting it resolves them rather than adding a tenth spelling.

⚠ **The directory says `bridge-extensions` and the documents say `EXTENSION-BRIDGE-…`, and the word
order differs on purpose.** The **name class leads with what the artifact IS** — every specification in
this corpus that contributes types and operations is an `EXTENSION-`, and a bridge is not an exception
to that. The **directory leads with what the group is FOR**, because that is the question a reader
browsing directories is asking. Aligning them would mean either inventing a `BRIDGE-EXTENSION-` name
class that breaks nine landed citations, or naming the directory for the artifact class it shares with
its sibling — which is the distinction the directory exists to draw.

### D2 — `specs/domains/` → `specs/bridge-extensions/`, a SIBLING of `specs/extensions/` `[RULED — this reverses the draft]`

**LANDED.** `specs/domains/` is retired and `specs/bridge-extensions/` replaces it; the filesystem
bridge moved there unchanged.

⛔ **This proposal's first draft argued the opposite — that a bridge is an extension (§3), therefore it
belongs in `specs/extensions/`, therefore the directory question dissolves.** That reasoning is
**valid and its conclusion was not adopted**, which is worth recording precisely because the argument
still reads well:

> **Being the same *kind* of artifact is not an argument for being in the same *place*.** The
> directories group by what a specification is **about**, which §2 of the opening study established and
> this proposal then failed to apply to its own conclusion. `specs/extensions/` is the **entity system's
> own surface** — what a peer is, what a deployment needs. **A bridge is how the system reaches a
> technology that is not it.** A reader opening `extensions/` and finding `EXTENSION-BRIDGE-NIX` beside
> `EXTENSION-TREE` learns something false about what the system is: that a package manager is part of
> it. **The family is a spread of unrelated foreign technologies, and a reader should be able to see
> that spread as a spread.**

⇒ **Two things are true at once and the directory encodes the second: a bridge extension IS an
extension by artifact class (§3), AND it is categorically not a native one.** `bridge-extensions` is
the name because it says both — the noun is *extension*, the qualifier is what distinguishes it.

**What this dissolves.** The open question *"subdirectory inside `specs/extensions/`, or flat?"* is
gone: the family has its own directory and is flat inside it. **Do not add a further level until there
is a reason that is not aesthetic.**

**What it costs, measured.** Seven files across the independent implementation trees reference the
`specs/domains/` path; none is source code and none breaks. **The path is the cheap identifier in this area — §6 is about the two
expensive ones.**

### D3 — Write `guides/GUIDE-BRIDGE-EXTENSION-DEVELOPMENT.md` `[LANDED]`

**Written and declared canonical.** Non-normative, parallel to the extension-development guide, carrying
§2.2–§2.5: the three layers, the no-umbrella ruling, the mode axis, the seven questions and the hazards
— plus the naming and placement rules (D1/D2), the prefix collision (D6), the *name-for-the-technology-
not-the-deployment* rule (§6.2) and the unsettled items (§5). **This is where the cross-bridge knowledge
lives, because the runtime is deliberately not shared.**

### D4 — Resolve the dangling citations `[LANDED, for the five that read as instructions]`

Marked `(planned — not yet authored)` per the addressing standard §11.4 in the five places a reader is
told a specification is the home for something: the extension-development guide's mechanism split, the
cross-peer messaging guide's mail bridge, the two in the substitute extension, and the one in the network
extension. **A pointer to a non-existent file is worse than an acknowledged gap — it reads as an
instruction the reader has failed to follow.**

⚠ **The four remaining are a different class and are B-8**: they cite *proposals* rather than planned
specifications, which the addressing standard treats as an internal-artifact leak, and the population is
corpus-wide rather than bridge-specific. **Sweeping them inside this fold would have hidden a corpus
problem inside a bridge one.**

### D5 — Give the reserved prefixes a stated rule `[LANDED]`

Three top-level prefixes are reserved for the world outside the protocol, **two of them claim the
filesystem, and nothing says which wins.** The distinction they encode is real and unrecorded:

> **What we are standing on is not what we are reaching out to.** A filesystem or process bridge is
> assumed present — the system runs *on* it — and is substrate as well as adapter. A version-control or
> package-store bridge is an optional integration reached across a boundary.

⇒ **State that rule in the prefix table**, whichever words survive §6.

### D6 — ⛔ Record the three-prefix collision, and rule that no precedent binds the first bridge `[NEW — from §1a.3]`

**LANDED as a statement of the state, not as a choice of prefix.** The namespace guide's §4.1a now
carries the standing-on / reaching-out rule (D5), the fact that the reserved `bridge/` has no
occupants, and the fact that the corpus's two bridge sketches spell two different other prefixes. The
bridge guide's §6.1 says the same thing to the audience that will actually trip on it, and the
version-control sketch in `EXTENSION-REVISION.md` §10.2 now says outright that its own prefix is that
sketch's and is not settled.

**Why this is a disposition and not an open item.** The collision was invisible from every side: each
of the three spellings is locally reasonable, no document compares them, and **the first bridge
specification authored would have silently made one of them the convention** — by precedent, not by
decision, which is precisely how the directory this proposal is retiring came to exist. **Naming the
collision costs nothing and converts an accident into a decision somebody has to make.**

⚠ **It is deliberately NOT ruled here.** A prefix ruling wants the first real bridge specification in
front of it, and this proposal does not specify one.

## §5 What this proposal does NOT decide

- **It does not specify any bridge.** Each needs its own document answering the seven questions.
- **It does not settle whether the web-protocol bridge is single-extension or stratified.** The archive
  flags it as the one member that *might* want a substrate-plus-conventions shape rather than one
  extension, and says to re-examine when the real proposal hardens. **That flag stands.**
- ⚠ **It does not resolve resource contention between a bridge and a transport sharing a protocol.**
  A peer running both a web-protocol transport and a web-protocol bridge server may want the same
  listening port. **They are two different protocols' worth of semantics over one substrate, and the
  contention is real and unaddressed.** It belongs in the per-bridge specification, and it is named here
  so it is not discovered late.
- **It does not propose a rename of any shipped identifier.** §6 does that, separately, on purpose.

## §6 ⛔ The shipped-identifier question, separated deliberately

**One existing namespace is named for where it was first prototyped rather than for what it is**, and it
is the family's only member. **Renaming it is a coordinated change across five independent
implementations and several hundred files.**

**It is separated from D1–D5 because those are free and this one is not** — and because bundling a
priced change with free ones is how the free ones stop happening.

**The argument for the rename is not tidiness, and it is worth stating because it is a correctness
argument:** the current prefix asserts **locality**, and locality is the one property the underlying
resource does not guarantee. A network-mounted filesystem is reached through the same interface and is
not local. **A prefix naming *whose machine* rather than *how near the bytes are* stays true under that
case; the current one does not.** A second reserved prefix with exactly that meaning already exists and
has no occupants.

**The argument against is cost, and cost alone is not a reason** — it is an input.

### 6.1 ⚠ The cost was measured on ONE of the two identifiers, and the other is nearly as large

**This proposal's draft priced the shipped *namespace* at 312 files and treated the *document name* as
the cheap half. It is not.**

**Surface of the measurement, stated because the number is meaningless without it:** eight independent
trees — the three reference implementations, the conformance anchor, the two application-tier peers, the
generation repo and the core specification — **excluding build output and vendored copies of one repo
inside another.** Files, not occurrences.

| Identifier | Reach | Kind of reference |
|---|---|---|
| the `local/files` **namespace** | **438 files** | paths, handler registrations, tests — the shipped surface |
| the **document name** | **119 files**, of which **67 are source code, not documentation** | section citations inside implementing code (`§4.3`, `§5.5`) |
| the `specs/domains/` **path** | **7 files** | prose references only; nothing breaks when it moves |

> ⚠ **A vendored copy is not a second seat.** A first pass of this returned **214** for the document
> name, and **95 of the difference was one repository's pinned copy of another** — the same
> count-the-clones error the routing-inbound gate was built to stop, arriving in a hand count of a
> different thing. **Exclude vendored trees and build output, or the widest-reaching number will be a
> repository counted twice.**

⇒ **There are three identifiers here, not two, and they price two orders of magnitude apart.** The path
was free and is done (D2). **The document name and the namespace are both coordinated cross-implementation changes**,
because implementers cite the specification *by name and section* in the source that implements it —
which is a sign of a healthy corpus and is exactly what makes its name expensive.

### 6.2 ⇒ The disposition: additive successor, not rename

**RULED. The correction is a new bridge alongside the old one, never a rename of the old one.**

1. **Nothing is renamed.** The document name and the `local/` namespace stay exactly as they are. They
   are a **holdover**, named for the deployment shape they were first prototyped against, and that is
   recorded plainly in the specification's own header rather than quietly tolerated.
2. **The general filesystem case gets its own bridge extension** — one that covers a network mount and
   a userspace filesystem as naturally as a local disk, which is the case the current name asserts
   falsely.
3. **The old member is deprecated only once consumers have somewhere to go**, and archived only well
   after that. **Deprecated-with-a-successor is a different artifact from a rename**: it costs no seat
   anything until that seat chooses to move, and it leaves the history legible to a reader who finds
   the old name in an old tree.

> ⭐ **Why this is not the "no backward compatibility" rule being bent.** That rule is about *the
> protocol* — no legacy wire paths, no migration windows, no dual-kind acceptance. **This is a
> specification document and a tree-path convention, neither of which is on the wire**, and the thing
> being preserved is not compatibility but *the ability to read a five-year-old comment and find what
> it refers to.* A superseded bridge that no new deployment installs costs the protocol nothing.

> ⭐ **And the deeper reason the first name was wrong is not carelessness — it is that the theory was
> not yet paid for.** The filesystem bridge is what *taught* this project what a bridge is: the seven
> questions in §2.4 were extracted from it. **A first implementation names itself for the case it was
> built against, because the general case is not visible until the specific one exists.** That is the
> normal cost of learning by building, and the guide now carries the rule the cost bought —
> *name a bridge for the technology, not for the deployment shape you built against first.*

⚠ **Sequencing.** The draft said *rule the prefix, then add the member.* **§6.2 supersedes that**: the
successor is additive, so it does not deepen the debt and does not wait on a prefix ruling. **What it
DOES wait on is D6** — a new filesystem bridge must not pick a prefix by accident, and a successor to
`local/files` is exactly the document that would.

## §7 Open items

**This proposal is filed as implemented and the open rows below do not contradict that.** Its scope was
the family's shape, naming and placement, and every disposition in §4 landed. **The rows marked OPEN are
each owed by a document this proposal does not write** — a bridge specification, or the first author to
need a prefix — **and each one is now recorded where that author will meet it** rather than only here.
That is the difference between a partial fold and a complete one whose subject has a future.

| # | | State |
|---|---|---|
| **B-1** | Which bridges are worth specifying first, and in what order | **OPEN.** Version control and package store are the named candidates; both are structurally better teachers than the web protocol, which is simultaneously infrastructure. ⭐ **Version control now has a head start nobody had counted: `EXTENSION-REVISION.md` §10 already carries its concept mapping and an operation sketch** (§1a.2) — a bridge specification would be *lifting and completing*, not starting |
| **B-2** | Subdirectory inside `specs/extensions/` or flat? | **DISSOLVED by D2** — the family has its own directory and is flat inside it |
| **B-3** | Single-extension or stratified shape for the web-protocol bridge | **OPEN, and recorded where an author will meet it** — `GUIDE-BRIDGE-EXTENSION-DEVELOPMENT.md` §7 |
| **B-4** | Port/resource contention between a bridge and a transport over one protocol | **OPEN, recorded** — guide §7. No general answer; it belongs in the per-bridge specification |
| **B-5** | The shipped-identifier question | **RULED (§6.2) — additive successor, no rename.** The sequencing constraint changes with it: the successor no longer waits on a prefix ruling, but it MUST NOT pick a prefix by accident (D6) |
| **B-6** | Does the tier classification gain a row for the family? | **RULED — no.** `SYSTEM-ARCHITECTURE.md` §13.1 now carries a **bridge-extensions note beside the five tiers**, stating that the family takes no row in them and that this is not an omission. The five tiers classify the system's own extensions; a bridge answers a different question |
| **B-7** | ⛔ **Which namespace prefix do bridges use?** | **OPEN and now VISIBLE (D6).** Three spellings in play, the reserved one empty. **The first bridge specification authored decides it — deliberately, not by precedent** |
| **B-8** | The six `PROPOSAL-*` citations in landed specs that name non-existent proposals | **OUT OF SCOPE, recorded.** Distinct class from D4 — a citation to an internal artifact rather than to a planned specification. The addressing standard §11.3 governs it and the population is corpus-wide, not bridge-specific |
