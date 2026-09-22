# EXPLORATION — the hypertext lineage: Xanadu, Hyper-G, and what the Web declined to solve

**Status:** Exploration (design record). Not a proposal, not normative.

**Operator-requested.** *"Most people say it was a failed project… but a few people have tried to
implement it, and a lot of the same problems we're seeing."* And: *"we don't want to look at Xanadu in
isolation — what was it in its original vision, what happened with implementation, what projects in
the area have or have not taken up the reins, and what failures have they seen."*

**Companion:** `EXPLORATION-THE-CONVERGENT-DESIGNS-…` covers the living systems (Willow, SSB, Usenet,
Matrix, BitTorrent). This one covers the **dead** ones, which are more useful, because a design that
did not ship has a recorded diagnosis and a design that shipped has only survivors' habits.

**Sourcing discipline (D12).** Everything below marked with a URL was opened. Nelson's own technical
statement and the project's own technical pages are the primary sources for what Xanadu *specified*;
Wolf's 1995 reporting and the Abora reimplementation report are the primary sources for what it
*ran*. Claims marked *(our reading)* are this record's inference and are weaker.

---

## §0 The result, stated first

**Every part of Xanadu that was never implemented — in any release, across forty years and three
codebases — is a part that required a globally coordinated authority. Every part that was
implemented is a part that was local.**

| Xanadu requirement | Needs what | Shipped? |
|---|---|---|
| Enfilades (the tree structure) | local computation | **yes** — Green, 1988, ran |
| Transclusion (reference, not copy) | local computation | **yes** — Green, and the 2014 demo |
| Tumbler addressing | a coordinated allocation of a shared address space | **partially** — worked in-system, never across the docuverse |
| *"No redundancy — every text exists only as an original"* | global uniqueness | **no** |
| Royalty / transcopyright at any granularity | global accounting | **no** — absent from Green, Gold **and** OpenXanadu |

**That is a checkable claim, not a moral about perfectionism.** And it maps one-to-one onto this
design, because **we have replaced each of those global requirements with a local one**: a content
hash instead of an allocated address, redundancy-as-the-mechanism instead of no-redundancy, and a
detached signature instead of royalty accounting.

**The second result, and it is the useful one for the next design session:** the reason Xanadu needed
enfilades, I/V transforms and tumbler arithmetic at all is that **its content was mutable**. Ours is
not. **Span-level addressing over an immutable content-addressed entity is arithmetic** — the problem
Xanadu spent thirty years on is, in our substrate, an offset and a length. See §6.3; it is the one
place this read suggests we are leaving something on the table.

---

## §1 What Xanadu actually specified

Read from Nelson's own statement of xanalogical structure and the project's technical pages, not from
summaries — the popular account gets all four of these wrong.

### §1.1 Permanent addresses, and links that are *applicative*

Content receives **permanent universal addresses**. Editing does not invalidate them: an insert
splits a reference pointer into several pointers, but *"the system can show the link to all these
characters in their new positions."*

The critical structural property, in Nelson's words, is that links are **not embedded**:

> *"The xanalogical content link is **applicative** — applying from outside to content which is
> already in place with stable addresses."*

which permits *"thousands of overlapping links on the same body of content, created without
coordination by many users around the world."*

**This is the single most important sentence in the Xanadu corpus for us**, and it is not the one
Xanadu is famous for. Links live outside content. That is what makes third-party annotation possible
without the author's cooperation, and it is what the Web gave up.

### §1.2 Transclusion is an identity claim, not a copy

> *"Transclusions are not copies and they are not instances, but **the same thing knowably and
> visibly in more than one place**."*

Implementation may be *"a live connection, a cache, or an alias"* — what must be maintained is
*"knowable identity; which copies and instances do not [have]."*

**Note what this actually demands: a mechanism for asserting that two byte sequences are the same
thing.** Xanadu supplied that with an assigned address in a coordinated space. **A content hash
supplies it without coordination**, which is why our §1.2 rule (*entries reference, never inline*)
gets Xanadu's property for free — and why the same rule was unreachable for Xanadu at internet scale.

### §1.3 The content list — an EDL, and Hollywood found it independently

> *"A sequential document or version is represented by a **content list**… a list of contents — a
> sequence of reference pointers. These pointers designate spans of characters."*

Nelson notes the convergent discovery outright: *"Hollywood discovered this method separately. This
is how movies are now edited"* — what film calls an **EDL (Edit Decision List)**, Xanadu calls a
content list. And the delivery model follows:

> *"The delivery of a document is in two logical phases: the content list, then the fulfilment of the
> contents."*

**That is our `mirror` and our `collection`, exactly** — a document that is a list of references,
delivered separately from the things it references. Three independent derivations (Xanadu, film
editing, this corpus) of one structure is about as strong a signal as design gets.

### §1.4 Tumblers and enfilades — the machinery

**Tumblers** are multipart transfinite numbers (`0.zzz.yyy.xxx…`) naming node, account, work, address
and element type, drawing *"a little from the Dewey decimal system… and a little from the tumblers in
locking mechanisms."* A span is a start tumbler plus a difference tumbler. Nelson calls the xu88
scheme **rootless**, and its purpose was explicit: because *"the distributed nature of the project
made central control impossible,"* they needed a consistent way to mint and use unique addresses
*wherever they might lie in the docuverse.*

**Read that carefully — the goal was decentralization and the mechanism was a shared numbering
scheme.** Those are in tension, and the tension is the whole story. A rootless multipart number is
still a *name someone assigns*; two parties who never meet cannot both mint one and be sure. **A
content hash is the resolution of that tension and it did not exist as an engineering option in 1979.**

**Enfilades** are the tree family (WIDative: values propagate up; DSPative: properties impose down)
that made spans indexable — a B-tree with displacement arithmetic. Green used three: the
**Granfilade** (content), the **POOMfilade** (address transformation) and the **Spanfilade**
(document-to-document mapping). Gold replaced them with Drexler's **Ent**.

They were **trade secrets until 1999** — *mentioned but never explained in publication*. That fact is
worth more than it looks (§3.3).

### §1.5 The seventeen rules, and the two that ended it

The design requirements include unique secure identification of servers/users/documents; any data
type; visible links with transclusion; *"royalty mechanism at any desired degree of granularity"*;
access without knowledge of physical storage location; **automatic redundancy**; and an open
published protocol.

Two of them are the global-authority requirements: **rule 9 (royalty at any granularity)** and the
no-redundancy invariant Wolf reports as *"there could be no redundancy in the grand Xanadu library.
Every text could exist only as an original."*

**Those two never shipped, in any codebase, ever.** (Note also that "automatic redundancy" as a
requirement and "every text exists only as an original" as an invariant are in visible tension — that
tension is itself evidence the model was never reduced to practice.)

---

## §2 What actually ran — the honest timeline

| Year | Artifact | What it achieved |
|---|---|---|
| 1972 | Cal Daniels' demo | first demonstration version; the 1972 mockup used **clear plastic overlays and a cardboard frame** to simulate terminal graphics — four years after Engelbart's NLS demo showed real ones |
| 1979 | tumblers | Gregory + Miller design the addressing scheme at Swarthmore |
| 1981–88 | **Xanadu 88.1 / "Green"** | designed by Gregory and Miller, **essentially completed 1988**, C backend + Python (`Pyxi`) frontend. **Ran, at small scale.** Sample dataset: instructions and the Declaration of Independence |
| 1983–92 | Autodesk period | ~$5M spent. C version *"unsatisfactory"*; Smalltalk rewrite created internal factions; divested 1992 |
| 1992 | **Xanadu 92.1 / "Gold"** | Miller/Tribble/Pandya, wrapping the Ent. Kernel Ent routines *"have worked well since about 1990"* — **the full system was never usable.** Development stopped |
| 1999-08-23 | **Udanax** | Green and Gold open-sourced under MIT/X-11. First publication of Enfiladics and the Ent |
| 2007 | XanaduSpace 1.0 | a 3-D **viewer** over ZigZag structures |
| 2014 | **OpenXanadu** | a **browser demo of one document** with visible connections. Not a server, not a network, nothing to author into |

**The reimplementers' verdict on Gold is the most useful sentence in this section.** The Abora project
(a Smalltalk reimplementation of a Udanax-Gold subset) reports the barriers as *"the size of the code,
the generality of the design, the incompleteness of the release, the absence of a running system, and
the lack of documentation"* — and adds that studying Nelson's writings *without any representative
running system* left one short of understanding how the features would work day to day.

**Forty years, three codebases, and the largest working artifact is a browser demo of a single
document.** That is the fact any sympathetic reading has to survive.

---

## §3 Three diagnoses, and which survives contact with the evidence

### §3.1 The managerial account (Wolf, *Wired* 3.06, 1995)

Perfectionism, "six months away" forever, rabid prototyping, recurring penury. It is well-reported
and it is true as far as it goes. **It is also unfalsifiable and it does not explain the pattern**:
if the failure were purely Nelson's temperament, others would have built it without him. Several
tried. None finished either.

Nelson's rebuttal is on the record and is worth registering: he holds that the Web *"trivialises"*
the model with one-way, ever-breaking links and no version or content management.

### §3.2 The hardware account

Wolf reports the team building a universal library on machines with **128 KB of RAM, later doubled to
256 KB**, and one programmer describing *"20 lines of very, very hairy C++"* just to extract a piece
of text from the back end. Real, and dated: it explains 1980 and explains nothing about 1999 or 2014,
by which time the code was open and the hardware was not the constraint.

### §3.3 The structural account — and it is the one that holds

**Three findings, each independently sourced, that point the same way.**

**(a) The unimplemented parts are exactly the globally-coordinated parts.** §0's table. Royalty
accounting at arbitrary granularity requires a settlement authority over the whole docuverse.
"No redundancy, every text exists only as an original" requires global uniqueness — which is, in our
own vocabulary, one of the **two genuine walls** (`BEARINGS-2026-09-04` §5b) and not a gradient. **The
project's own requirements list contained an item that is not achievable in a decentralized system,
and it was load-bearing for the economics.**

**(b) The addressing scheme needed the primitive that did not exist yet.** Tumblers are assigned
names in a shared space, adopted *because* central control was impossible — a scheme aimed at
decentralization that still requires everyone to agree on an allocation. Content addressing dissolves
this and postdates the design by decades. *(our reading)*

**(c) The trade-secret decision.** Enfilades were secret until 1999 — *"mentioned but not explained"*
in twenty years of publication. **A system whose central data structure cannot be reviewed cannot
recruit implementers, cannot be criticized into correctness, and cannot be reimplemented when its
authors stop.** Rule 17 of the seventeen asks for an *open, published protocol encouraging
third-party development*; the project violated its own rule for two decades and the disclosure came
seven years after development stopped. **This is the cheapest transferable lesson in the document and
it costs us nothing, because our corpus is public by construction.**

---

## §4 Hyper-G — the control experiment, and the more instructive failure

**Xanadu is the famous failure. Hyper-G is the useful one, because Hyper-G shipped.**

Graz University of Technology, from 1989. Links were stored **outside documents**, in a separate link
database (the DBServer) beside an object store and a full-text index. Consequences, all real and all
delivered:

- **all links bidirectional** — you could see what linked to what, and follow backwards
- **third-party annotation** — users could attach links to read-only documents (Nelson's applicative
  property, §1.1, actually implemented)
- **link integrity guaranteed** — remove a document and links to it vanish from all HyperWave
  servers; restore it and they **reappear**
- **`p-flood`** — a scalable, probabilistic, prioritizable server-to-server protocol for distributing
  update information across thousands of servers, explicitly designed for referential integrity at
  scale, and noted at the time as addable to WWW and Gopher in principle

They also correctly diagnosed the Web: *first-generation*, links one-way and embedded, no way to know
what refers to a document, dangling links, *"lost in hyperspace."* Every criticism was accurate.

**It lost anyway, and the recorded reasons are the ones that matter to us:**

1. **The guarantees held only inside its own world.** Distributed integrity required *every* remote
   site to run a HyperWave server. A system whose properties evaporate at the boundary of its own
   deployment is competing on the boundary, not on the properties.
2. **The session model was a bottleneck by design.** A Hyper-G client talked to **one** server for a
   whole session, which proxied everything else — where Web clients contacted many servers directly.
3. **Interoperating meant conceding.** The surviving server had to *"bend backwards"*: parse HTML
   though it preferred HTF, speak HTTP though it preferred HG-CSP.
4. **The feature advantage eroded.** By 2000 the differentiators — indexing, search, link checking,
   log analysis, dynamic content, load balancing — were available as Apache modules, and Apache ran
   more than half the Web. **Monolith versus components, and components won.**

**The transferable finding, and it is aimed squarely at us:** *p-flood was an elegant solution to a
problem the Web won by declining to solve.* **Before building a mechanism, ask whether the property
it guarantees is one the competing design simply lives without** — and whether it survives at the
edge of your own deployment.

---

## §5 The Web's counter-design, in its author's framing

Berners-Lee's account is explicit and is not a retrofit. Asked whether existing hypertext systems
could be made worldwide, designers said no, **citing the need for a clearinghouse** to maintain
referential integrity. His conclusion: the dangling-link problem *had to be accepted* — someone in
Tokyo must be able to remove a document without informing everyone in the world who points at it.

In his own 2000 words, the Web *"had to throw away the ideal of total consistency,"* ushering in
`404` but allowing *"unchecked exponential uncontrolled growth."* The general principle he draws:
**to be universal, a system must be minimally constraining — requiring only what is essential for
everything to work.**

He did not treat rot as acceptable forever; he treated it as **social rather than architectural**,
which is what *Cool URIs Don't Change* (1998) is.

**Score it honestly.** He was right about scale — nothing with a clearinghouse has ever reached global
scale. He was wrong about the residue: link rot is a real and permanent tax, and the archival
community has spent thirty years paying it. **Both halves are true and the second does not undo the
first.**

---

## §6 The map — Xanadu's five properties against ours, one by one

This is the section the operator asked for. Each row is *their* requirement, *our* answer, and
whether we are actually better off or merely different.

### §6.1 Stable permanent addressing — **we are strictly better, and it is not close**

Xanadu: a tumbler, a multipart number assigned in a shared space, whose uniqueness rests on everyone
honouring the allocation.
Us: a **content hash**. Self-verifying, mintable by anyone, collision-resistant without coordination,
and it authenticates the bytes as well as naming them — which a tumbler never did.

**A content hash is the primitive Xanadu needed and could not have had.** This is the single largest
gap between the two designs and it is entirely in our favour. It is also *why* the four decades of
enfilade engineering were necessary: **absent self-verifying names, you need a machine to maintain the
mapping — and that machine is what never finished.**

### §6.2 Transclusion — **we have it, at coarser granularity, and the rule is already written**

Xanadu: *the same thing knowably in more than one place*, at arbitrary span granularity.
Us: `PROPOSAL-APP-CONVENTION-FEED` §1.2 — *"an entry MUST NOT contain another entry's bytes"*; what it
quotes, it references. §1.2's renderer note is transclusion in one sentence: *"where a renderer wants
to show quoted text, it resolves the reference and renders what it resolved."*

**Same property, different unit: theirs is a character span, ours is an entity.** And note our reason
for adopting it was not Nelson's — the FEED derivation reached it from cost (*a reply costs the size
of the reply*) and from dedup semantics (*content addressing dedups nothing by itself; what it buys
is a shared name*). Two derivations, one rule.

### §6.3 Span addressing — **we do not have it, and this is the finding**

Our reference atom is `{peer, hash, path?}` (FEED §2.2). Entity granularity. **There is no way to say
"the third paragraph of that post."**

**The Hypothes.is comparison makes the point sharply.** The W3C Web Annotation model needs
`TextQuoteSelector` (the quote plus ~32 characters of prefix and suffix) *and* `TextPositionSelector`
*and* a fuzzy last-resort match through diff-match-patch/Bitap, because **the target document
mutates** — positions drift even without edits, since a page can inject different advertising text
between two loads. They accept drift, score candidates by confidence, and keep *orphaned* annotations
in a separate view rather than deleting them (deletion would let a target author erase criticism).
Microsoft Research's 2001 robust-positioning work is the lineage.

**None of that applies to us.** An entry is immutable and content-addressed. A span over immutable
bytes is `{reference, offset, length}` and it is **exact, permanent, and verifiable by arithmetic** —
no selectors, no fuzzy match, no orphan state, no confidence score.

> **The observation for the design record: the hardest problem in the entire hypertext lineage —
> making a sub-document address survive — is trivial in this substrate, and we have not taken it.**
> Xanadu built enfilades for it. Hypothes.is built fuzzy anchoring for it. We would need one optional
> pair of integers on an existing atom.

**Not proposed here** (L1 — this is an exploration, and a normative change is proposal-first). Filed
as an open question in §8, with the caveat that a span into a *rendered* body is not the same as a
span into *encoded bytes*, and which one a quote wants is genuinely unobvious.

### §6.4 Bidirectional links — **we correctly do not have them, and Hyper-G is the evidence**

Xanadu and Hyper-G both required them. Hyper-G **delivered** them and the guarantee evaporated at its
own deployment boundary (§4).

Our references are one-way by design. **What replaces the backlink is that a reply is itself a
published entry in the replier's own tree, and a mirror is a published view.** So a backlink exists
exactly when somebody published one — discoverable by walking the reference graph from anyone you
already know, never guaranteed. **That is weaker than Hyper-G's guarantee and stronger than Hyper-G's
guarantee actually was in practice**, because ours does not require the other party to run our
software.

Nelson's applicative-link property (§1.1) is the half we do partially recover: **our "links" live in
the referring party's tree, not in the referenced content**, so third-party annotation of content
whose author never cooperated is native. We get the applicative property; we decline the
bidirectional index.

### §6.5 Versioning with intact history — **comparable, and ours is cheaper**

Xanadu: I-to-V and V-to-I transforms over enfilades, mapping an address in the permanent space to its
current position and back — the machinery that made *"the link still points at the right characters
after an edit"* work.

Us: entries are immutable, so nothing moves; a revision is a *new* entry referencing the old; the
signed root chain carries the history. **We do not need an address-transform layer because we never
transform an address.** Immutability buys the whole of §1.1's hardest guarantee.

**And the deletion result is where we beat both.** The trie is canonical *regardless of insertion or
deletion history*, so a tree with an entry removed is byte-identical to one that never held it, while
the past stays signed in the root chain. Xanadu could not delete at all (permanence was the point);
Hyper-G could delete and made the links vanish network-wide, which required p-flood.

### §6.6 Attribution surviving re-use — **we solve it; Xanadu never did, in any release**

Xanadu's mechanism was **transcopyright**: rule 9, royalty at any degree of granularity, so that a
transcluded fragment paid its author. **It is absent from Green, absent from Gold, and absent from
OpenXanadu.** Forty years, three codebases, zero implementations of the economic mechanism the design
rested on.

Our mechanism is the **detached `system/signature` at the core's invariant pointer path**, per entry.
A mirror republishes bytes it cannot alter, carrying signatures it cannot forge. **Attribution
survives re-use without any accounting authority, because the proof travels with the bytes instead of
being settled centrally.**

**And we found the failure mode by measurement, which is the part worth keeping.** Entities carry no
embedded signature, so `{peer, hash}` alone gave a mirror **integrity without authorship** — exactly
the property Xanadu's whole royalty apparatus existed to prevent, discovered here by building a mirror
rather than by review (`BEARINGS-2026-09-04` §3 result 2). FEED-9 is the vector that pins it.

---

## §7 What we take, what we decline, and what we should reconsider

**Take:**

1. **The applicative-link framing (§1.1).** Nelson's sentence is a better statement of why our
   references live in the referring tree than anything currently in our corpus. It is the reason
   third-party annotation is native here and impossible on the Web.
2. **The EDL/content-list framing (§1.3).** *"Delivery is in two logical phases: the content list,
   then the fulfilment of the contents"* is precisely our mirror and collection, and it is a better
   explanation to an outsider than ours.
3. **The trade-secret lesson (§3.3c).** Free, and we already comply.
4. **Hyper-G's question (§4).** *Before building a mechanism, ask whether the property is one the
   competing design lives without, and whether it survives at the edge of your deployment.* This is
   the sharpest transferable rule in the document.

**Decline, with the reason now evidenced rather than asserted:**

5. **Bidirectional link integrity.** Hyper-G shipped it and it did not save them; Berners-Lee
   declined it and reached global scale. Our one-way references are the right call and we can now say
   why from measurement rather than taste.
6. **Any global-authority requirement.** §0's table is the general warning. **A requirement that
   needs a settlement or uniqueness authority does not get built, and it takes the design with it.**
   This is a direct restatement of our own two-walls result, arrived at by a different route.

**Reconsider:**

7. **Span addressing (§6.3).** The one place where the lineage's hardest problem is our easiest, and
   we have not taken it. It is not obviously *wanted* — nobody has asked — but the cost is small and
   the asymmetry is unusual enough to be worth a decision rather than a default.

---

## §8 Open questions this read produced

1. **Do we want sub-entity references?** `{reference, offset, length}` over immutable bytes is exact
   and cheap. Blocked on: is the offset into the **encoded entity** or the **rendered body**? These
   diverge under `APP-CONVENTION-EMBED`'s ladder, and the second is not stable across renderers.
   **Nobody has asked for this** — it is a capability observation, not a demand, and it should not be
   built ahead of one (L26 cuts the other way here: the seats discover by building, and no seat has
   hit this).
2. **Is there a case for publishing backlink views as a first-class shape?** A mirror already is one.
   The question is whether *"who referenced this"* deserves a tag distinct from `app/feed/mirror`, and
   the taxonomy floor's test (*must a conformant consumer behave differently?*) probably says no.
3. **Does the applicative-link framing belong in the charter?** It is the clearest one-sentence
   statement of why the corpus puts references in the referring party's tree.

---

## §9 Sources opened

**Primary — Xanadu:**
[Nelson, *Xanalogical Structure*](https://xanadu.com.au/ted/XUsurvey/xuDation.html) ·
[xanadu.com/tech — enfilades, tumblers, Green/Gold](https://xanadu.com/tech/) ·
[Udanax open-source release + license](http://udanax.xanadu.com/license.html) ·
[Udanax Green distribution](http://www.open.xanadu.com/green/index.html) ·
[Abora — technical report on reimplementing Udanax Gold](https://abora.dgjones.info/dolphin-demo/tech.html) ·
[Enfilade (structure)](https://en.wikipedia.org/wiki/Enfilade_(Xanadu)) ·
[Project Xanadu — timeline, the seventeen rules](https://en.wikipedia.org/wiki/Project_Xanadu)

**Primary — the failure reporting:**
[Gary Wolf, *The Curse of Xanadu*, Wired 3.06, June 1995](https://www.thing.de/hartmoderne/text/xanadu.html)
([PDF mirror](https://club1.fr/media/books/the-curse-of-xanadu_wired.pdf)) ·
[*The lessons of Xanadu*](https://www.lesswrong.com/posts/wr9dH2GjztvCz6pYX/the-lessons-of-xanadu)

**Hyper-G / Hyperwave:**
[*The Hyper-G Network Information System*](https://www.researchgate.net/publication/220349551_The_Hyper-G_Network_Information_System) ·
[Vision and Reality of Hypertext and GUIs — Hyper-G/HyperWave](https://mprove.de/visionreality/text/2.1.15_hyperg.html) ·
[*Why Hyperwave?*](https://jaschke.net/hyperwave.html) ·
[Hyper-G overview, Helsinki](https://www.cs.helsinki.fi/research/rati/HyperG.html) ·
[*Hypertext Link Integrity*](https://www.researchgate.net/publication/220566171_Hypertext_Link_Integrity) ·
[prior-art-dept.: the hierarchical hypermedia world of Hyper-G](http://oldvcr.blogspot.com/2025/05/prior-art-dept-hierarchical-hypermedia.html)

**The Web's counter-design:**
[W3C Design Issues (Berners-Lee's own notes)](https://www.w3.org/DesignIssues/) ·
[Berners-Lee, Semantic Web talk, 2000](https://www.w3.org/2000/Talks/0906-xmlweb-tbl/text.htm)

**Span anchoring:**
[Hypothes.is — Fuzzy Anchoring](https://web.hypothes.is/blog/fuzzy-anchoring/) ·
[W3C Web Annotation Vocabulary](https://www.w3.org/TR/annotation-vocab/) ·
[Brush, Bargeron, Gupta, Cadiz — *Robustly Anchoring Annotations Using Keywords*, MSR TR-2001-107](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-2001-107.pdf)

## §10 What is unread, named so nobody assumes coverage

- **`Literary Machines` (1981/1987) in full.** Version 87.1 is effectively the system specification
  and only excerpts were read here. If any claim in §1 is contested, that is the arbiter.
- **The FeBe (front-end/back-end) protocol document.** The client-server wire format, in tumbler
  terms. It is the closest thing Xanadu has to our `ENTITY-CORE-PROTOCOL` and it was not opened.
- **The Udanax Green source itself.** Mirrors exist. Nobody here has read a line of it, and every
  claim above about what Green *did* is from prose about Green.
- **Engelbart's NLS/Augment**, the other 1968 lineage, and **Microcosm**'s Distributed Link Service,
  the third link-database design named in the Hyper-G literature.
- **OpenXanadu 2014 specifically.** The one detailed claim available about its royalty mechanism came
  from an AI-generated reference site with no editorial review and is **not** relied on above.
