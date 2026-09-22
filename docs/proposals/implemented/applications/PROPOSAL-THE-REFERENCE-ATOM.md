# PROPOSAL — the reference atom: one shape for "this points at that", and a string form for the half that is typed by a human

**Status:** IMPLEMENTED (2026-09-08) — **FOLDED as `specs/applications/APP-CONVENTION-REFERENCE.md` v0.1**,
the domain's fourth member. All eight deltas landed; **two were adjudicated differently from the text
below and two defects were found in the fold — see §12, which is the authority on what actually
shipped.** The convention is **authored and NOT ratifiable**: the §7 vectors are specified and not
shipped (charter #5), and they are carried on the spec's own §6.2.
**Target:** a new `specs/applications/APP-CONVENTION-REFERENCE.md` (the atom and its string projection,
imported by the tier) · `specs/applications/APP-CONVENTION-SEMANTIC-CONTENT-SITE.md` §4 (`nav-node.target`
gets a grammar instead of *"the renderer's classifier"*) · `specs/applications/APP-CONVENTION-EMBED.md` §3
and §7 (the pointer and image-source slots adopt the atom) · `specs/applications/APP-CONVENTION-SHARE.md`
§2.2 (`blob-target` / `prefix-target`) · `PROPOSAL-APP-CONVENTION-FEED` §2.1–§2.2 (re-cut before it lands,
which is the whole reason this is sequenced now) · `specs/applications/CHARTER.md` (Members).
**Provenance:** three design passes, 2026-09-06, against two questions — *why does a peer-plus-path link
not work, and what does that tell us about the system?*, then *build several designs, weigh them, and
converge on the one that makes the most sense*. The scheme name was ruled 2026-09-08.
**Derivation, not repeated here:** `EXPLORATION-THE-LINK-AND-THE-WALK-WHAT-A-REFERENCE-CARRIES-AND-WHAT-A-CLIENT-DOES-WITH-IT`
(the census: nine shapes, five discriminators, two peer terms) ·
`EXPLORATION-THE-REFERENCE-PRIMITIVE-SIX-SYSTEMS-AGAINST-ONE-TUPLE` (the reduction) ·
`EXPLORATION-WHY-PEER-PLUS-PATH-IS-NOT-A-LINK-ADDRESSES-NAMES-AND-THE-TWO-HOPS` (why an address is not a
name) · `EXPLORATION-THE-REFERENCE-ATOM-FOUR-DESIGNS-SCORED-AND-THE-CONVERGENCE` (four designs, nine
pre-fixed criteria, Design C chosen).
**Workstream:** the social tier. Sequenced ahead of the feed convention because the feed carries the two
atoms this proposal generalizes, and changing an atom before two implementations ship is the entire cost
of the decision.

---

## §0 The one-sentence version

**The app tier points at things nine different ways, discriminated five different ways, and only two of
those nine carry the term that makes a reference reachable — the publisher.** This proposal lands one
tagged atom carrying *who · what · optionally which part · optionally where-to-look-first*, and a
canonical string projection of it, because a nav target, a prose link, a pasted address and a scanned code
are all strings and the corpus specifies none of them.

**The new material is §1, and it inverts the framing the derivation was written under:** the string layer
is not unbuilt. **Two independent implementations have each built a link classifier, they agree with each
other, and they resolve the dispatch scheme as a hyperlink.** This proposal is therefore a *convergence on
built behaviour with one correction*, not a design landing on empty ground.

---

## §1 What was measured — the string layer already exists, twice

**Every prior document in this arc listed "open the two application-tier trees" as unexecuted, three times
in a row. It was executed for this proposal and it changes the shape of the work.**

### §1.1 Two implementations, one classifier, built independently and deliberately aligned

Both application-tier implementations carry a link classifier with **the same four outcomes**:

| Outcome | Meaning |
|---|---|
| **in-site** | a page within the current site |
| **cross-site** | a page in a different site on the same peer |
| **cross-peer** | a page in a site on another peer |
| **external** | a link that leaves the entity system |

One implementation's classifier carries a comment stating that it mirrors the other's, function for
function. **This is not two teams arriving somewhere by luck; it is one design, cross-checked.** The
corpus has been describing this surface as undefined while two conformant implementations were already
interoperating on it.

**What they built, stated as the design it is:**

- **A scheme for same-peer cross-site links, in opaque form:** `site:{site-id}/{page}`. No authority
  component, no `//`.
- **A scheme for cross-peer links, in hierarchical form**, reusing the dispatch scheme:
  `entity://{peer}/…/sites/{site}/pages/{page}`.
- **A base-URI rule for bare strings.** A link with no scheme is in-site and resolves **directory-relative
  to the current page**, with a leading `/` meaning site-root-absolute, and a trailing `.md` / `.markdown`
  stripped. One implementation documents the convention explicitly: authored body links are
  directory-relative; every application-generated link (nav, breadcrumbs, generated index pages) is
  authored root-absolute so it resolves identically from any page.
- **A refusal rather than a guess.** An unparseable cross-peer URI is classified **external** — *"treat as
  external rather than guess."*

**Three of those four are exactly what the corpus said it lacked.** The convention's `nav-node.target` is
specified as a `tstr` resolved by *"the renderer's classifier"*, and `classifier` occurs exactly once in
all of `specs/applications/` — in that comment. **The classifier it names was built twice and specified
never.**

### §1.2 The one thing they built that this proposal has to correct

**Both classifiers parse the dispatch scheme as a hyperlink**, and the parse is a **tolerant scan** rather
than a grammar: it splits on `/`, takes segment 0 as the peer, then *searches the remaining segments* for
the literal `sites` and again for `pages`, joining whatever follows. It is explicitly written to keep a
superseded path layout resolving.

Two things follow, and they point in opposite directions:

1. **The layer confusion is real and is now deployed.** `entity://` is the **EXECUTE dispatch URI**, with a
   normative home: *"Wire URIs use the `entity://` scheme… The `entity://` scheme maps to RFC 3986: the
   peer_id is the authority, the rest is the path."* Clicking a menu item is not executing a handler.
   Under the deployed classifiers, one string form means *navigate here* in a document body and *dispatch
   to this handler* on the wire, and nothing but position disambiguates them.
2. **A tolerant scan is not a parser, and its failure mode is silent.** Searching for a literal `sites`
   segment anywhere in the path means a page whose slug happens to be `sites` re-anchors the parse. There
   is no error; there is a different, well-formed, wrong location. **Neither implementation can detect
   this and neither is at fault** — a scan is the correct engineering response to a string with no
   grammar, and the absent grammar is the corpus's.

> **This is the argument for the proposal, made by the implementations rather than by arch.** The seats did
> the discovery by building, they converged, and what they converged on is *the best available reading of a
> specification that does not specify this*. What is owed to them is the grammar, not a correction.

---

## §2 The scheme — what was ruled, what follows from it, and what is still open

### §2.1 The ruling

**The scheme is `entity+ref://`, the canonical long form** *(ruled 2026-09-08)*. `ent:` was rejected as
too ambiguous standing alone; `entity-ref` was acceptable; **`+` was taken as the established convention
for a scheme that is a variant of another** — `git+ssh`, `svn+http`. **A short alias may be added later and
the ruling is explicitly open to one**, so nothing in this proposal may be written in a way that forecloses
an alias.

### §2.2 What the ruling settles that the derivation left open — and the evidence is §1

The derivation recorded two sub-questions as unsettled by the name: whether the form is **hierarchical**
(`scheme://authority/path`) or **opaque** (`scheme:name`), and whether a reference has one entry point or
several.

**On the first, this proposal takes a position and states its reasoning rather than leaving it to be
inferred:**

- **The ruled spelling carries `//`.** In RFC 3986 §3 the `//` is not decoration — it is the delimiter that
  introduces an **authority** component. A scheme spelled with it and used without it is a scheme whose
  own name lies.
- **The scheme it is a variant of already has this mapping, normatively:** the peer id **is** the RFC 3986
  authority and the remainder is the path. A sibling scheme that mapped its authority differently would be
  a variant in name and not in structure.
- **A reference's authority component is exactly the term the census found missing** — the publisher. Nine
  shapes, and only two carry a peer term; **a reference is routable if and only if it does.** Putting the
  peer in the authority slot is not a formatting choice, it is the finding.
- **And the deployed system already uses both forms for the two jobs**, which is the strongest evidence
  available: `site:` is opaque and names something *within an implied authority*; the cross-peer form is
  hierarchical and names the authority explicitly.

**So: hierarchical, with the peer id as authority. Recorded as a derived reading of the ruling, not as the
ruling itself** — if the ruling intends otherwise, this section is the part to correct, and the atom in §3
is unaffected either way.

**On the second — one entry point or several — this proposal does not close it and does not need to.** §3's
tag is what makes a third intent addable without disturbing the first two.

### §2.3 What happens to the existing `entity://` links

**Nothing, for a period the seats set, and this proposal does not mint a flag day.** The two classifiers
resolve documents already published. The rule proposed is:

- **`entity+ref://` is the form a conformant producer emits.**
- **A consumer SHOULD continue to resolve `entity://` in a link position** as it does today, and **SHOULD
  surface that it did** — the same *name your own provenance* rule the tier already applies to a live
  reference whose hash moved.
- **A producer MUST NOT emit `entity://` in a link position** once it emits the atom at all.

**This is deliberately a SHOULD on the consumer side and a MUST on the producer side.** The asymmetry is
the whole mechanism: it stops the population of ambiguous strings growing while never breaking a document
that already exists.

---

## §3 The atom

### §3.1 Shape

```cddl
; ---- imported, per the charter's encoding rule --------------------------------
content-hash = bstr                    ; self-describing (format_code, digest).
                                       ; NEVER fixed-width — the charter's rule 6.
peer-id      = tstr                    ; the peer identity
tree-path    = tstr                    ; absolute or peer-relative

; ---- the atom -----------------------------------------------------------------
entity-ref = pinned-ref / live-ref     ; TAGGED. There is no untagged form.

pinned-ref = {
  tag:   "pin",
  peer:  peer-id,                      ; WHO published it — the authority
  hash:  content-hash,                 ; WHAT it is — identity and expectation coincide
  ? at:  anchor,                       ; which PART. Absent = the whole entity
  ? via: [* hint]                      ; ADVISORY. Ordered, descending confidence
}

live-ref = {
  tag:    "live",
  peer:   peer-id,                     ; WHO publishes there
  path:   tree-path,                   ; the address of record — authoritative
  ? seen: content-hash,                ; what the linker saw — an EXPECTATION only
  ? at:   anchor,
  ? via:  [* hint]
}

anchor = { field: [* tstr] }           ; a field path within the entity. See §3.4
hint   = { tag: "origin" / "mirror" / "peer", value: tstr }
```

### §3.2 The two intents are preserved exactly, and only the discriminator changes

**The pinned/live split is not this proposal's to revisit.** It is derived, corroborated against six
systems, and its refutation of a single shape with an optional hash stands: under an optional hash, a
reference arriving without one is indistinguishable between *the author wants the live version*, *the
author's implementation did not populate it*, and *the author only ever had a URL* — one intent and two
bugs, with no way to tell them apart.

**What changes is how a reader tells them apart: a tag, not a field-name exclusive-or.**

| | field-name discrimination (today) | tag (proposed) |
|---|---|---|
| A reader tells them apart by | observing which of `hash` / `seen` is present, and rejecting an atom with both | reading one field |
| A malformed atom is | detectable only by a reader that implements the xor rule | a tag mismatch, at the first field |
| A third intent later | needs a third field-presence rule, interacting with the first two | is a third tag |
| Consistent with the tier | **no** — it is the only member that discriminates this way | **yes** — the other two members both tag |

**The counter-argument this has to answer, because the existing rule is well argued:** field-name
discrimination was chosen deliberately, and its argument is that *a reader can always tell*. **A tag
satisfies that requirement more directly than a presence rule does** — it makes the discriminator a value
rather than an inference — and the argument was made against an **optional field**, not against a tag.
Nothing in it is retracted; it is satisfied by a different mechanism that the tier already uses twice.

### §3.3 `via` — advisory, ordered, and droppable or it is not a hint

```
? via: [* hint]      hint = { tag: "origin"/"mirror"/"peer", value: tstr }
```

**The normative rule is one sentence, and it is what bounds the rot surface:**

> **[MUST]** A reader that ignores every `via` hint MUST reach the same answer as one that uses them, or
> fail. A hint MUST NOT be the only path to a correct resolution, MUST NOT be signed, and MUST NOT be
> treated as authority for anything.

**Why the slot exists at all:** the network layer defers peer-to-transport discovery, so a reader holding a
peer id it cannot resolve has no normative next step. Three deployed systems — a federated chat protocol,
a relay-based social protocol, and a peer-to-peer file transport — each added exactly this slot,
independently, for exactly this reason, and one of them states the problem in terms: *"room IDs are not
routable on their own as there is no reliable domain to send requests to."*

**And why it must be droppable:** hint decay is measured and severe in the surveyed systems. A hint that
becomes load-bearing is a link that rots. The rule above is what keeps a stale `via` a slow path rather
than a broken one.

**This slot and a publisher's own published resolver view are complementary, not alternatives** — the
link-borne hint is the bootstrap, the published record is the steady state, and they fail in opposite
directions. One surveyed system ships both, deliberately. The published-record half is a separate,
independent piece of work and is not proposed here.

### §3.4 `at` — an anchor slot, declared now, at zero cost

`? at: anchor` where `anchor = { field: [* tstr] }` — a path of field names into the referenced entity.
**Absent means the whole entity, so nothing changes for anyone who does not use it.**

It is declared now because the alternative is minting a second atom later for *"the same thing, but a part
of it"*, and because the durable-reference ladder's first rung is exactly a field path — **the hash
changes, the field path does not.**

**What it deliberately does not do:** it does not name a location *inside a rendered body*. That would need
the embed convention to mint a name slot first, and **no consumer has asked for one.** The tag structure
means that rung can be added as another `anchor` variant without disturbing this one.

> **Checked, and flagged as a genuine open point (§10.1):** the core type system names four address
> primitives — content hash, tree path, type name, peer id — and a *field path within an entity* is not
> among them. The scope grammars that exist (`targets`, `exclude`, path scopes) are **tree-path** patterns
> with a trailing `*`, not intra-entity field paths. **So `anchor` is new vocabulary, not a reuse**, and it
> is the one part of this atom with no existing grammar behind it.

---

## §4 The string projection

### §4.1 The grammar

```abnf
entity-ref-uri = "entity+ref://" peer-id path-abempty [ "?" query ] [ "#" fragment ]
```

Mapped onto RFC 3986 §3 exactly as the dispatch scheme already is:

| RFC 3986 component | Carries |
|---|---|
| **scheme** | `entity+ref` — a valid scheme name per §3.1 (`ALPHA *( ALPHA / DIGIT / "+" / "-" / "." )`) |
| **authority** | the **peer id**, and nothing else. No userinfo, no port |
| **path** | for a `live-ref`, the tree path. For a `pinned-ref`, empty |
| **query** | the non-identity terms: `hash`, `seen`, `via` |
| **fragment** | the `at` anchor |

```
live:   entity+ref://{peer}/{tree-path}[?seen={hash}][&via=…][#{anchor}]
pinned: entity+ref://{peer}/?hash={hash}[&via=…][#{anchor}]
```

**Which component carries what is a design decision, not a formatting one, and the rule is: the identity
terms are in the address, the advisory terms are in the query.** A `pinned-ref`'s identity is its hash, so
its path is empty and the hash is a required query parameter — this reads oddly and it is the honest
encoding, because a pinned reference genuinely has no path-shaped identity. **The alternative considered
and rejected** was putting the hash in the path, which makes a pinned and a live reference structurally
indistinguishable to any generic URI parser and hands the discriminator back to a scan.

**The tag is recoverable from the string without a lookup table**: `hash` present ⇒ `pin`; `path`
non-empty ⇒ `live`. **A string with both, or neither, is malformed** and MUST be refused rather than
resolved — the string form inherits the structured form's rejection rule rather than softening it.

### §4.2 Round-tripping is normative, and it is the vector that matters

> **[MUST]** The string projection is **lossless in both directions** for every atom expressible in §3.1.
> Parsing a conformant string and re-serializing it MUST produce a byte-identical string; serializing an
> atom and re-parsing it MUST produce an equal atom.

**This is the tier's existing precedent applied to a new surface** — the embed convention already requires
that a directive string and the child entity it projects are *the same thing*, and it is a conformance
cost there for the same reason it is one here.

### §4.3 Normalization, stated rather than waved at

Percent-encoding, case and normalization are where string forms fail in practice, so:

- **Scheme is case-insensitive on parse and MUST be emitted lowercase** (RFC 3986 §3.1 / §6.2.2.1).
- **The authority is a peer id and is CASE-SENSITIVE.** It is a Base58 identity, **not a DNS name**, so the
  host-normalization rule of RFC 3986 §6.2.2.1 **MUST NOT** be applied to it. *This is the single most
  likely implementation error*, because every general-purpose URL library lowercases the host by default.
- **Path segments are compared after percent-decoding** (§6.2.2.2); an implementation MUST NOT compare raw.
- **No dot-segment removal** (§6.2.2.3 is not applied). A tree path is not a filesystem path and `..` is
  not meaningful in it; a path containing a `.` or `..` segment is **malformed**, not resolved.
- **Reserved characters in a tree path segment MUST be percent-encoded** (§2.1), and `/` is the segment
  delimiter and is never encoded when it is one.
- **No empty-vs-absent conflation:** an absent query parameter and one present with an empty value are
  different, and the second is malformed.

### §4.4 The relative form, which is what the seats actually use most

**The grammar above is the absolute form. The overwhelmingly common case is a bare string in a document
body, and the built implementations already agree on how to resolve it.** This proposal adopts what they
built, as the normative base-URI rule:

> **The base is the referring entity's own location.** A link with no scheme is resolved against it:
> a leading `/` is **root-absolute within the current site**; anything else is **directory-relative to the
> current page**. Producers of application-generated links (navigation, breadcrumbs, generated index pages)
> **SHOULD** emit root-absolute form so a link resolves identically from any page.

**Adopting the built rule rather than a cleaner one is the point.** It is deployed in two implementations,
it matches what a document author expects from every other markup system, and the alternative is a
correction with no defect behind it.

**The `site:` scheme is likewise adopted as-is** — opaque, same-peer, cross-site — because it works, it is
deployed, and it is the natural short form for the case where the authority is implied. **It is not a
competitor to `entity+ref://`; it is the relative form of it one rung up**, and stating that relationship
is what this proposal adds to it.

---

## §5 What this costs the two implementations

**Deliberately measured against what they have built rather than against the specification**, because the
specification is the thing that was missing:

| Change | Size |
|---|---|
| Emit and accept `entity+ref://` beside the existing form | small — one prefix branch in a function that already has four |
| Replace the tolerant `sites`/`pages` scan with the §4.1 grammar | **small, and it removes code** — a grammar is shorter than a scan and its failures are detectable |
| Keep resolving `entity://` in link position, and surface that it did | small; §2.3 |
| Adopt the tag when reading atoms | **not yet shipped anywhere** — the feed convention that carries the atoms has not landed, which is why this is sequenced now |
| The `at` anchor | **zero** until something uses it |
| Round-trip conformance vectors | real, and it is the charter's rule 5 |

**The four-outcome classification is unchanged and is not being redesigned.** in-site / cross-site /
cross-peer / external survives verbatim; this proposal gives three of the four a grammar and leaves the
fourth alone.

---

## §6 Charter compliance, stated explicitly because a new member has to earn it

| Charter rule | How this satisfies it |
|---|---|
| **1 · defines FORMAT only** | the atom is a CDDL shape; the string form is a serialization of it. Both are byte-level cross-impl contracts |
| **2 · no kernel or SDK machinery** | no operation, no handler, no required SDK surface. A classifier is application code and stays there |
| **3 · improvises no protocol** | it mints a URI scheme in the application tier and **touches no wire form**. The one thing it needs from below — peer-to-transport resolution — it does **not** invent: `via` is advisory by construction and a reader that ignores it reaches the same answer |
| **4 · valid floor** | an atom with no `via` and no `at` is the whole floor. A peer that resolves only the pinned form and only within its own peer is a valid participant |
| **5 · ships vectors** | §7, and the round-trip pair is the load-bearing one |
| **6 · never locks a hash width** | `content-hash = bstr`, self-describing, no fixed-width form anywhere |

---

## §7 Conformance vectors — required before ratification

**Class: application-tier format vectors — example entities plus expected bytes**, per the charter's rule
5. These are not wire-oracle checks and not host-seam checks; they are static fixtures the tier already
ships for its other members.

| # | Vector | What fails without it |
|---|---|---|
| **R-1** | a `pinned-ref` round-trips atom → string → atom, byte-identical both ways | the round-trip MUST in §4.2 is unenforced, and a re-serializer silently rewrites references |
| **R-2** | a `live-ref` with `seen`, `via` and `at` round-trips | the query-component encoding is the part most likely to diverge, and it carries three different kinds of term |
| **R-3** | **a mixed-case peer id survives parse+serialize unchanged** | **the highest-value vector here.** Every general URL library lowercases the authority; this one MUST NOT. A peer id that round-trips through a lowercasing parser names a different peer, and the failure is a clean `404` at a well-formed address |
| **R-4** | a string with both `hash` and a non-empty path is **refused** | the discriminator degrades from a rule to a convention, which is the exact failure the two-shape design exists to prevent |
| **R-5** | a string with neither is **refused** | as R-4, the other arm |
| **R-6** | a path segment containing a reserved character round-trips percent-encoded | the most common real-world content case — a page slug with a space or a `#` |
| **R-7** | a reference with an **unknown `via` hint tag** resolves identically to the same reference with `via` absent | this is the droppability MUST measured directly, and it is the one that keeps hints from becoming load-bearing |
| **R-8** | a path containing a `.` or `..` segment is **refused**, not resolved | §4.3; a resolver that borrows filesystem semantics produces a different, well-formed, wrong address |

**R-7 and R-3 are the two that would not be written by someone implementing from the prose**, which is the
argument for authoring them here rather than leaving them to the seats.

---

## §8 Delta table

| # | File | Section | Change |
|---|---|---|---|
| **D1** | `specs/applications/APP-CONVENTION-REFERENCE.md` | **new** | the atom (§3), the string projection (§4), the normalization rules, the vectors (§7) |
| **D2** | `specs/applications/CHARTER.md` | Members | the new member, its role, and that it is **foundational** — it is imported by the other three |
| **D3** | `specs/applications/APP-CONVENTION-SEMANTIC-CONTENT-SITE.md` | §4 | `nav-node.target` gets §4's grammar. **Delete *"resolved by the renderer's classifier"*** — it is the sentence that made this a per-implementation decision |
| **D4** | `specs/applications/APP-CONVENTION-SEMANTIC-CONTENT-SITE.md` | §3.1 | the *"adds no prose vocabulary"* sentence is scoped: a **link** in a body has an entity-native meaning, given by §4.4. It remains true that the convention adds no other prose vocabulary |
| **D5** | `specs/applications/APP-CONVENTION-EMBED.md` | §3, §7 | the pointer/child payload and the image-source slot take `entity-ref`. **The existing tagged discipline is the model, not a casualty** |
| **D6** | `specs/applications/APP-CONVENTION-SHARE.md` | §2.2 | `blob-target` / `prefix-target` take `entity-ref` |
| **D7** | `PROPOSAL-APP-CONVENTION-FEED` | §2.1, §2.2 | the two atoms are re-cut as the tagged form and **imported** rather than redefined. §2.2.3's argument is preserved and restated against the tag (§3.2) |
| **D8** | `PROPOSAL-APP-CONVENTION-FEED` | §2.2.4 | row 3's *"fetched from anywhere"* is scoped to what the substrate actually offers (§10.3) |

**Homes enumerated by subject, not by token.** A grep for `reference` finds D5–D7 and misses D3 and D4 —
those are about a **link**, which is this rule's subject under a different word, and D4 shares none of the
atom's vocabulary at all. The three landed members each define `content-hash` and `peer-id` locally
(measured); D1 becomes the single home and the three import it.

---

## §9 Impact on the implementations

| Consumer | Delta |
|---|---|
| The two application-tier implementations | §5. Both have a built classifier; the work is a grammar replacing a scan, one new scheme branch, and the vectors |
| The feed convention, unlanded | re-cut before it lands — **this is the reason for the sequencing and the only reason it is cheap** |
| The three landed conventions | adopt the atom in their pointer slots (D5, D6) and stop defining the shared atoms locally |
| Core and extension tiers | **none.** No wire form, no entity type, no operation, no capability |

---

## §10 Open items

**Carried deliberately. None of these blocks the atom, and each is named so it is not rediscovered.**

1. **`anchor`'s field-path grammar is new vocabulary** (§3.4). The core type system's four address
   primitives do not include an intra-entity field path, and the existing scope grammars are tree-path
   patterns. **Either it earns a grammar of its own here, or the slot ships with `field` as an opaque
   sequence of names and the grammar lands when rung 2 does.** Leaning: the second — a slot with no
   consumer should not mint a grammar.
2. **Whether a reference has one entry point or several** — carried from the ruling, and the tag is what
   keeps it open at no cost.
3. **The short alias.** The ruling explicitly leaves room for one. Nothing here forecloses it; the
   canonical long form is what a producer emits and an alias would be an additional accepted input.
4. **The publishable resolver view** — the steady-state half that makes `via` droppable in practice rather
   than only in principle. Independent, unblocked, and not proposed here.
5. **The deprecation horizon for `entity://` in link position.** §2.3 sets producer and consumer rules but
   deliberately no date. **That is the implementations' call**, not arch's, because they hold the corpus of
   already-published documents.

---

## §11 What was not read, named so nobody assumes coverage

- **The rendered-body anchor case** (rung 2) is out of scope by choice and no consumer has asked for it.
- **Percent-encoding behaviour of the deployed link detectors** — §4.1 notes that plus-schemes are
  handled unevenly by real-world link autodetection, and this was **not** re-measured for this proposal.
  It was an argument *against* the ruled name and the name is ruled, so it is now a **rendering** concern
  for the implementations rather than a design input.
- **The pre-split design archive was searched for prior work on a reference scheme — by title index across
  all 1,068 documents, and by full text — and the two on-point documents were opened**, not merely
  matched: the URI/type/domain clarifications of the second core revision and the URI-centric refactor plan
  of the fourth. **What the archive holds is the *addressing* model** — *"URI provides location/identity;
  Type provides classification/routing"*, `URI = entity://peer_id/domain/path` — **which is the same
  finding this proposal builds on from the other direction: that form is an address, and a reference is a
  name.** There is no prior design for a reference or hyperlink scheme in it. The four addressing
  explorations in this arc are the study.
- **No implementation's link-rendering surface was read** — only the classifiers. A claim about how either
  one *displays* a reference is not made anywhere in this document.

---

## §12 The fold record — what shipped, and where it differs from the text above

**Folded to `specs/applications/APP-CONVENTION-REFERENCE.md` v0.1.** The atom (§3), the string
projection (§4), the normalization rules and the vector set all landed substantially as written. **This
section is the authority where it and the sections above disagree**, and it exists because two of the
eight deltas were ruled differently and two defects were found by reading the target documents.

### §12.1 Two deltas adjudicated differently — D5 narrowed, D6 declined

**D5 and D6 said the pointer slots "take `entity-ref`". Applied literally that is wrong, and the
delta table's own framing is what obscured it: it enumerated slots by their role in the census rather
than by whether they can cross a peer boundary.**

| Slot | Ruled | Why |
|---|---|---|
| `EMBED` `child-payload.ref` | **converted** to `entity-ref` | It was `(path / content-hash)` — an **untagged** union discriminated by CBOR major type, the one place EMBED's own tagged discipline was not applied. It is also the only payload that can name something on another peer |
| `EMBED` `pointer-payload.hash` | **unchanged** | names a blob in the resolving peer's own content store |
| `EMBED` `img-src` | **unchanged** | an output is rendered where it was resolved; no authority is left to name |
| `SHARE` `blob-target` / `prefix-target` | **unchanged** | **already the pinned/live split, tagged, minus the authority** — because a share is over the sharer's own content on the sharer's own peer |

**The general rule this fold states instead, and it is in the spec at §3.4:** the same-peer slots are
the **implied-authority form** of the atom, exactly as `site:` is to `entity+ref://`. **Adding a
required `peer` to them would restate a term that is already known and give it somewhere to be
wrong** — and in `SHARE` specifically that is not hypothetical, since §1.1 of that document exists
because the audience has twice been put in the wrong slot.

### §12.2 Two defects found in the fold

1. **`peer-id` was typed incompatibly across the tier, and nothing recorded it.**
   `APP-CONVENTION-SEMANTIC-CONTENT-SITE` defined `peer-id = bstr` citing core §1.2 — **the CONTENT
   HASH section** — while `APP-CONVENTION-SHARE` and the feed draft define it `tstr` citing §1.5.
   **Core §1.5 settles it: `PeerID := Base58(...)`, so the encoding is part of the definition**, and
   §1.4 uses it as a tree-path segment. Both deployed link classifiers carry it as a string.
   **Ruled `tstr`; the site convention is corrected.** The field that used it has **no implementation
   in any repository**, so the correction costs nothing — but a proposal making one document the
   single home for an atom **must check that the members agree about it**, and this one asserted they
   each defined it locally without noticing that two of them disagreed.
2. **`via` could not express the thing it was said to replace.** §8's D7 folds the pinned form's
   optional `path` into `via` — *"one field, one job"* — but the hint vocabulary as proposed was
   `"origin" / "mirror" / "peer"`, none of which carries a tree path. **A `"path"` hint tag is added**,
   with the *"a 404 here proves nothing"* obligation stated normatively (`REF-R21`, `REF-V11`).

### §12.3 One ambiguity in the proposal, resolved in the spec

**§4.3's dot-segment refusal and §4.4's directory-relative base rule contradict each other as
written** — `..` cannot be both malformed and the mechanism of relative resolution. **The spec scopes
the refusal to the absolute form** and states that relative resolution consumes the dot segments
before producing an atom. `REF-V9` is the arm that catches over-application, because a reader that
applies the refusal to an unresolved relative string rejects ordinary correct links.

### §12.4 What is owed, and it is not owed by this proposal

- **The ten vectors** (spec §6.2). The convention is not ratifiable until they ship — the same state
  the share convention is in, and for the same charter rule.
- **`anchor`'s field-path grammar** stays an opaque sequence of names, per §10 item 1's leaning: a
  slot with no consumer does not mint a grammar.
- **The deprecation horizon for the dispatch scheme in link position** is deliberately undated (§2.3),
  and that is a decision for whoever holds the corpus of already-published documents.
