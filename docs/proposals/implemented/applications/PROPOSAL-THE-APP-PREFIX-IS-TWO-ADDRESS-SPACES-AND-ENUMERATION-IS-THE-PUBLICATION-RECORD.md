# PROPOSAL — `app/` is two address spaces with one spelling, and enumeration is most of the publication record

**Status:** RATIFIED and FOLDED — 2026-09-16. Two implementations publish under the
affected prefix today; nothing here moves a byte either of them has written.
**Target:** `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §4.1 / §4.2 / §10 (the namespace taxonomy, which
owns the reserved-prefix table) · `GUIDE-APPLICATION-DEVELOPMENT.md` §2 / §3 (the tier standard a
convention author reads) · `GUIDE-ENTITY-WORKBENCH-APP.md` §3 (the claim-by-writing rule `R2` amends) ·
`APP-CONVENTION-FEED.md` §4.2 (the only convention that pins tree paths).
**Tier:** `applications/` — the subject is the application-tier address space. **Not an extension
change**: no handler, no operation and no wire field is touched.
**Scope:** *what the segment `app/` means*; *where a person's own files live*; and *what a reader
holding a peer id can learn about what that peer publishes*. **Not** in scope: the shape of any
entity, any signature or verification rule, and the **shape** of a profile — only the constraint on
where one may live, which is a consequence of `R1` and is stated in §4b.
**Source:** an application-tier audit that measured the collision in a shipping tree, plus the
corpus-side measurement in §2 taken for this proposal.

> **§4a and §4b were added at ratification**, because the filing seat asked for these three to be
> decided **together** and the reason they gave is structural rather than convenient: each of the
> other two pins a third path under the same prefix, and pinning one before `R1` lands is a third
> thing to move if partition-by-root is ever revisited. Deciding them apart is what makes the
> revisit expensive.

---

## §1 The finding in one paragraph

**The segment `app/` names two different address spaces, and the taxonomy that owns the namespace
declares only one of them.** It is an entity **type tag** (`app/feed/entry`), and it is a **tree path**
(`/{peer}/app/…`) — and within the tree path it is used for two opposite purposes: an application's
**private** working state, which is what the taxonomy says it is for, and a convention's **published**
index, which the taxonomy does not mention. The consequence is not aesthetic: **a reader enumerating
`/{peer}/app/` cannot sort what it finds into "published for me" and "private to that peer's
applications", because nothing in the path says which one a segment is.** That is the whole of §5's
problem, and it is why this proposal treats both as one question.

## §2 The measurement

**The taxonomy's own row, quoted** — `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.1's reserved-prefix table:

> `app/{app-id}/` — **Per-application state. Workspace, settings.** Peers running applications. Scoped
> by app ID so multiple apps can coexist.

**Private state, and nothing else.** **§3** of the application-host guide adds the ownership rule that
makes it work: `{app-id}` is *"a per-instance identifier that the app claims by writing to it — there
is no central registry, no reservation mechanism."*

> ⚠ **Citation corrected at ratification, and it is worth one line because of the class it is in.**
> This paragraph, the filing seat's packet, and the receiving ledger row all cited that rule as
> **`GUIDE-ENTITY-WORKBENCH-APP` §5**. It is **§3** (*Path namespace*); §5 is *Selection scope* and
> has nothing to do with it. **The wrong number resolves** — §5 exists — so `spec address` reports
> it clean and `spec sections` finds no duplicate: this is the residue both gates state about
> themselves, arriving on the rule at the centre of the finding. Three documents in two repositories
> carried it and nobody opened the section, on both sides of the channel.

**Now the other use, also normative** — `APP-CONVENTION-FEED` §4.2 pins two paths by hand: the head at
`/{peer}/app/feed/index` and pages at `/{peer}/app/feed/index/{page}`, for the stated reason that *a
reader holding no reference has to start somewhere.*

⇒ **An application that claims the app-id `feed` — which the host guide explicitly permits, with no
mechanism to prevent it — writes its private workspace over a normative public index path.** Neither
document acknowledges the other, and both are correct in isolation.

**Three further measurements, because the shape of the problem is not obvious from the collision
alone:**

| # | measured | consequence |
|---|---|---|
| 1 | **`FEED` is the only application convention that pins tree paths.** The content-site convention registers a **URL projection** and explicitly not a tree-storage rule | the collision is not a general convention habit — it is one convention's pin landing in a prefix reserved for something else |
| 2 | **§4.2 *Content Domains* already permits open top-level paths** — *"Beyond the reserved prefixes, peers can use whatever top-level paths make sense… These are open. Applications define their own content domains"* | a published sibling namespace at the top level (`/{peer}/sites/…`, `/{peer}/apps/…`) is **conformant today.** An implementation using one is not squatting and owes no apology |
| 3 | ⭐ **A static analyzer written by the party that owns this vocabulary has confused a tree path for a type tag, or the reverse, four times in four distinct shapes** — a prefix with a trailing slash; a complete pinned path with none; a parametric type family; and a comment *explaining the confusion*, parsed as an emission | **this is the empirical core of the proposal.** The two spaces are not merely similar, they are **the same bytes**, so a heuristic is the only available discriminator and it fails on exactly the case the convention pins |

## §3 The distinction to state — two axes, and the taxonomy carries neither

1. **Type-tag space versus tree-path space.** `app/feed/entry` is a value in the `type` field of an
   entity. `/{peer}/app/feed/index` is a location in a tree. They share a spelling and an intuition
   and they are not the same namespace. **Nothing in the corpus states this**, which is why §2's
   row 3 happened.
2. **Published versus private, inside the tree-path space.** A convention's index is *addressed to a
   stranger*. An application's workspace is *addressed to nobody* and is deliberately excluded from
   publication. Today both are `app/…`, and every implementation that publishes correctly does so by
   maintaining a hand-written allow-list of what may be projected — **a rule enforced in each
   implementation's source rather than by anything a reader can see in the path.**

**A rule that lives only in each publisher's allow-list is a rule no consumer can apply**, and a
consumer is the party that needs it.

## §4 The proposal — partition by rule, not by root

**Adopt the cheaper of the two partitions**, and state the distinction §3 names:

- **`R1` [MUST]** A published application-convention namespace under `app/` is **declared by the
  convention that owns it** and is drawn from a **closed, enumerable set**. A convention that pins a
  tree path declares it in its own specification and it is listed in the taxonomy.
- **`R2` [MUST NOT]** An application-chosen `{app-id}` MUST NOT be equal to a declared convention
  namespace under `R1`. The host guide's *claim-by-writing* rule is otherwise unchanged: the set of
  forbidden names is small, closed and published, so a claim can still be made without a registry.
- **`R3`** The taxonomy states the **type-tag versus tree-path** distinction once, in §4.1, and the
  authoring standard requires a convention pinning a tree path to say which space it is pinning in.

**And the alternative is recorded with its cost, because it is the better design in the abstract.**
*Partition by root* — a reserved top-level segment for published convention data, leaving
`app/{app-id}/` unambiguously private — is cleaner, and it makes §5 answerable by looking rather than
by rule. **It is rejected on cost, not on merit:** it moves both of `FEED` §4.2's pinned paths, which
two shipping implementations already publish and read, for a partition that `R1`/`R2` deliver without
moving a byte. ⚠ **If the set under `R1` ever grows past a handful, revisit this** — the rule scales
by enumeration and the root scales by construction.

## §4a A person's files are not application state, and the taxonomy has asked this question of itself since it was written

**The ask.** A location for a person's own files that every host agrees on. Today one host keeps them
at `/{peer}/app/entity-browser/files/…`, which makes a person's documents *one application's data* —
and the same person's save file, on a second host, would be somewhere else by construction.

**The taxonomy's own §10 open question 1 is this question, verbatim:** *"Should there be a container
for user content on session-archetype peers, or is the peer's tree implicitly 'the user's space'?"*
It has been open since the guide was written and nothing has been decided against it.

**`R4` [MUST] — a person's own content is a top-level content domain, not an application namespace,
and the domain is `files/`.** `/{peer}/files/…`, with entities addressed under it.

**The derivation, and §3's distinction is what makes it sharp rather than a preference:**

1. **`app/{app-id}/` is defined as *per-application* state** — the taxonomy's own row, and §3's
   claim-by-writing rule is what scopes it. A person's documents are not an application's working
   state, and they outlive any particular application. Putting them there is a category error the
   taxonomy already names; it only looked acceptable because there was nowhere else declared.
2. **It cannot be `local/`.** That is the host-disk mount with its own handler and its own config
   namespace — *the machine's* files, reached through an adapter. A person's files in the entity
   tree are entities the peer is the authority for. Two different things, and the filing seat ruled
   this out correctly before asking.
3. **It is conformant today with no permission owed** — §4.2 *Content Domains* says a peer may use
   whatever top-level paths make sense. ⭐ **So `R4` buys exactly one thing, and it is the thing that
   was asked for: not permission, AGREEMENT.** A content domain every host uses is a convention; a
   content domain one host chose is that host's directory layout.
4. **Top-level is what makes it enumerable** (§5). A person's files under `app/{some-app}/` are
   invisible to a consumer that does not already know which application wrote them.
5. **`storage/{identity}/` is not a competitor and does not move.** It scopes *whose* data on a
   **shared** peer. On such a peer the domain nests inside it (`storage/{identity}/files/…`); on a
   personal peer the person is the peer and the domain is top-level. One rule, two deployments.

**`R5` [MUST NOT] — a user-content domain is not published by default.** A host MUST NOT project
`files/` into the published root as a consequence of the person putting a file there; publication of
anything under it is a separate explicit act.

⚠ **`R5` is deliberately a rule about PUBLICATION and not about the grant table, and the distinction
is the one place this could have gone wrong.** The filing seat asked whether *the default connection
grant excludes it*. **A grant posture is a deployment's choice and is not a convention's to mandate**
— that is already ruled, and a convention that legislates one would be reaching past format into
deployment. What a convention *can* say, and what the ask actually needs, is what the domain **means**:
this space is addressed to nobody until its owner addresses it to someone. A deployment that wants the
exclusion expresses it in its own grant table, and now has a declared name to express it against —
which is the half that did not exist.

> ⛔ **Stated because building on it otherwise would be building on a floor that is not there:
> `R5` is not privacy and cannot be, while `system/content:get` resolves bytes without regard to
> namespace.** A grant can withhold a file's *name* and not its *bytes*. That is a kernel item,
> filed by the same seat, owned elsewhere, and **not closed by anything in this proposal.** A host
> MUST NOT describe a `files/` entry as private on the strength of `R5`.

**`R6` — an imported subgraph lands in a staging domain, `imports/`, and never directly in `files/`.**
**The reason is provenance and not tidiness:** an import merges a subgraph a stranger authored, and
once it is indistinguishable from the person's own content there is no later moment at which the
difference can be recovered. Staging keeps *I made this* and *this arrived* separable until a person
acts. `/{peer}/imports/{stamp}/` is adopted as proposed.

⚠ **`R6` answers one of the four archive asks and none of the other three.** Whether
`system/tree:extract` gains a content-closure option, whether `.entities` is a declared exchange
format, and whether an archive is signed are **open and not ruled here** — they are a mechanism
question, a format question and a trust question respectively, and only the placement one rides the
address space.

## §4b The profile does not need a third namespace — that was the whole of the collision

**The ask that rides this** is the well-known path for a profile: the filing seat holds the type
unwired and publishes no bytes specifically so that arch can decide the path before a baseline sets.

**Under `R1` the path question dissolves and is no longer a blocker.** `APP-CONVENTION-FEED` declares
`app/feed/` as its namespace; a well-known profile path for the feed convention lives **inside the
namespace that convention already declares**. It is not a third top-level pin, it does not enlarge
`R1`'s set, and it is not a thing to move under any future revisit that `app/feed/index` is not
already a thing to move.

**`R7`** — a convention pinning a **further** well-known path pins it **within its own declared
namespace** under `R1`. A convention MUST NOT pin a second top-level namespace for a second
well-known path.

⚠ **What is ruled here is the CONSTRAINT and not the profile.** Whether a profile is the feed
convention's existing collection type live-addressed, what it carries, and its exact spelling are a
convention change with its own fold. **The only thing that was blocked on the address space is
unblocked**, and holding the type unwired is no longer the right call for that reason — it may still
be the right call for the shape one.

## §5 *What does this peer publish?* — and the answer is mostly already built

**The question.** A reader holding a peer id wants generic, convention-level dispatch: enumerate what
is there, recognize *this is a site* and *this is a feed*, and hand each to the viewer that knows that
convention. **The consumer should know conventions, not applications.**

**What exists, and it is more than it looks.** The published-root front door is landed: a consumer
resolves a peer's published root at an origin and the head pointer has a pinned path. The root
**commits to a key set**, and walking a published root is a shipped operation. So *enumerate the top
level and dispatch on what is there* is **mechanically available today.**

**What blocks it is §1, not a missing record.** A consumer walking `/{peer}/` finds `sites/`
(unambiguous), a published sibling domain under §4.2 (legitimate but undiscoverable by anyone who does
not already know that publisher), and `app/` — **which is a convention's public index and an
application's private state with no rule to sort them.**

⇒ ***`R1` and `R2` make enumeration answer the question.*** A declared, closed set of published
convention namespaces is exactly the table a generic consumer dispatches on, and it is the same table
`R2` forbids colliding with. **This is one fix, not two.**

**What a separate publication record would still buy, stated so it can be judged on the residue rather
than adopted by default:**

- **Cost.** Enumeration is one walk per peer; a record is one fetch. The walk is bounded and the
  difference matters only at a scale nobody has built yet.
- **Things enumeration cannot say** — a convention's *version*, or a declaration that a namespace is
  present but empty rather than absent.

**If it is adopted it MUST be the peer's own signed statement about its own tree.** A third party's
durable claim about somebody else's publication is unfalsifiable at the reader, which is why this
belongs nowhere near a directory of other peers.

⚠ **Two adjacent designs exist and this is neither of them.** A **published walk** answers *who
contributed to a subject nobody owns* — a claim about many writers and one subject; this is a claim by
one peer about its own tree. A **cited-locator convention** answers *how do I reach this peer at all* —
already ruled, over the existing directory namespace. **Reaching a peer, learning what it publishes,
and gathering a subject across peers are three questions**, and the residue in this section is only
the third of them.

## §6 Conformance

| id | level | requirement |
|---|---|---|
| `NS-1` | `[MUST]` | A convention pinning a tree path under `app/` declares that namespace in its own specification |
| `NS-2` | `[MUST NOT]` | An application-chosen `{app-id}` equal to a declared convention namespace |
| `NS-3` | `[MUST]` | A publisher's projection excludes application-private state; the exclusion is expressible from the declared set rather than from an implementation's internal list |
| `NS-4` | `[SHOULD]` | A consumer performing generic dispatch enumerates the published root's top level and dispatches on the declared set, rather than probing per convention |
| `NS-5` | `[MUST]` | A person's own content is addressed under the `files/` content domain, not under an application namespace |
| `NS-6` | `[MUST NOT]` | A host projects a user-content domain into the published root as a side effect of a write to it |
| `NS-7` | `[MUST]` | An imported subgraph lands under `imports/`, distinguishable from authored content until the owner places it |
| `NS-8` | `[MUST NOT]` | A convention pins a second top-level namespace for a further well-known path, rather than pinning within the one it declares |

**Driven by:** `NS-1`, `NS-2` and `NS-8` are checkable against the corpus by the existing
namespace-resolution analyzer once the declared set exists — the same instrument whose four
misclassifications are §2's evidence, which is the point. `NS-4` is a consumer behaviour and is
exercised by a cross-implementation read, not by a static check.

⛔ **`NS-5`, `NS-6` and `NS-7` are host behaviours and NOTHING IN THIS ESTATE CAN OBSERVE THEM.** They
are satisfied or broken inside an implementation's tree layout, which no corpus analyzer reads and no
wire byte reflects — the same structural blindness that lets two app seats diverge on vocabulary while
both stay wire-conformant. **They are named here as owed to a check, not as covered by one**, and the
honest state of all three is *authored, unexercised*. **A second host reading the first host's `files/`
is what actually tests them**, and that is a cross-seat read, not a gate.

## §7 Open questions

1. ✅ **RESOLVED at ratification — §4b.** The well-known path for a profile does not need a namespace
   of its own: under `R1` it lives inside the declaring convention's already-declared namespace. The
   profile's **shape** remains open and is not this proposal's.
2. ✅ **RESOLVED — the taxonomy.** `R1`'s set lives in `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.1a,
   with the tier standard carrying only the obligation to declare. **The reason is the one this
   corpus keeps paying for:** a set and an obligation in one document is one home for two facts that
   move on different schedules — the set changes when a convention lands, the obligation changes
   almost never.
3. **`NS-3`'s expressibility** is the clause most likely to be wrong in practice, because every
   publisher shipping today implements the exclusion as source. It is stated as an outcome rather than
   a mechanism deliberately.
4. **`R4`'s interaction with a multi-user peer is stated and not exercised.** `storage/{identity}/files/…`
   is derived, not built; no implementation runs a shared peer with per-identity user content today.

## §8 What this does not touch

No entity shape, no signature rule, no verification behaviour, no operation, no wire field. Both
pinned feed paths keep their spelling. **A conforming implementation that publishes correctly today
continues to publish correctly, byte for byte, after this lands.**

⚠ **`R4` and `R6` name paths no implementation currently writes, so they are the one part of this
that costs a shipping seat a move.** That cost is a host's tree layout and not its wire output: no
peer reading it sees a different byte, and nothing published under `app/feed/` moves at all.

---

## §9 Fold record

| Target | What landed | Version |
|---|---|---|
| `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.1 | the type-tag / tree-path distinction (`R3a`); `app/{app-id}/` row split into private-application and declared-convention rows | — |
| `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.1a | **the declared set** (`R1`), and `R2`'s MUST NOT | — |
| `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.2 | the user-content domain `files/` and the staging domain `imports/` (`R4`, `R5`, `R6`) | — |
| `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §10 | open question 1 **retired**, answered | — |
| `GUIDE-APPLICATION-DEVELOPMENT` §2.3 + §3 | a convention pinning a tree path declares its namespace and says which space it pins in (`R3b`, `R7`) | 2.0 → **2.1** |
| `GUIDE-ENTITY-WORKBENCH-APP` §3 | claim-by-writing amended by `R2`: the forbidden set is small, closed and published | — |
| `APP-CONVENTION-FEED` §4.2 | declares `app/feed/` as its namespace under `R1` | **no bump** |

**Why `APP-CONVENTION-FEED` does not bump, stated rather than assumed** (`SPECIFICATION-FORMAT` §9.1,
§9.2). `R1`'s obligation is on the **convention document** — declare the namespace you pin in. It
places no obligation on an implementation of `FEED`: nothing it must emit, accept, refuse or compute
changes, and the §9.1 test is *could a conformant implementation of the previous text be non-conformant
under the new text.* It could not. The obligation `R2` creates lands on the **host**, through
`GUIDE-ENTITY-WORKBENCH-APP` §3, which is where app-ids are claimed.
**This is the editorial arm, and it is the arm most often claimed wrongly** — so the argument is
written down where a reader can disagree with it rather than left as a silent omission.

**The two guides carry no version header of their own beyond `Status: Active`**, which is a gap this
fold does not close and does not create.
