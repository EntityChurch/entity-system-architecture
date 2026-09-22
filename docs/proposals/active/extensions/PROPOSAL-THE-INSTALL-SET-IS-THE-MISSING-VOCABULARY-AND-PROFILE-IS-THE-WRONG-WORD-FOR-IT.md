# PROPOSAL — the install set is the missing vocabulary, *profile* is the wrong word for it, and the family it joins is peer composition

**Proposes:** a **named install set** — *"to have capability C, install set S"* — as an additive artifact
in the **peer-composition family**, with a developer-facing restatement and a mechanical check.
**Nothing is renamed, no namespace moves, no existing document loses a section.**

**And it proposes NOT calling it a profile**, which is what the gap has been filed under. §2 is a census
of that word in this corpus and it is the strongest single thing in this document.

**Status:** **DRAFT 2026-09-15 · revision 2.**
**Depends:** `guides/GUIDE-PEER-COMPOSITIONS.md` §1, §5, §10, §11 · `SYSTEM-ARCHITECTURE.md` §12, §13.1,
§13.1b notes 6–9 · `EXTENSION-SUBSTITUTE.md` §9.2 · `GUIDE-APPLICATION-DEVELOPMENT.md` §3, §5 ·
`SPECIFICATION-FORMAT.md` §8.8, §8.9
**Kind**: proposal · **Authority**: non-binding until folded · **Governed-by**: `SPECIFICATION-FORMAT.md`
**Audience**: specification authors and peer operators; application developers are the artifact's
audience but not this document's.

---

> ## ⛔⭐⭐⭐ REVISION 2 — what changed, and two of revision 1's arguments are WITHDRAWN
>
> Revision 1 was reviewed against the corpus rather than against itself, and **two of its load-bearing
> claims did not survive.** Both are withdrawn here rather than quietly repaired, because one of them was
> this document's entire case for its own recommended noun.
>
> | | |
> |---|---|
> | ⛔ **WITHDRAWN — the *"recognition, 2 of 2"* argument** | Revision 1 recommended `INSTALL-SET` because *"the two documents that filed the gap already use the phrase."* **They are not two documents.** Both filings — §12's entry and §13.1b note 8 — are **two sections of ONE file, introduced in ONE commit, hours before revision 1 was written.** The precedent it invoked was *four of seven **independent** consumers already were it*; here the surveyed population is **the author, twice.** ⇒ **the recognition argument is circular and is gone. `INSTALL-SET` now has no evidentiary advantage over the other free candidates** (§2.2) |
> | ⛔ **WITHDRAWN — the placement, and the reason is worse than being wrong** | Revision 1 put the artifact in the architecture document's classification section and ruled out `composition` as *"taken — it names a whole document tier."* **That measured the STRING and missed that the CONCEPT is taken by this exact idea**: a landed guide defines a peer as a **configuration** whose facets include *installed extensions*, defines a **composition** as a family of configurations, ships a **catalog of seven named ones** — and **states this document's gap, in its own opening, as an explicit scope exclusion** (§1) |
> | ⚠ **What SURVIVES unchanged** | **the `profile` census (§2), which was always the substantive half** — and the gap itself, which is now supported by **three** independent statements instead of one |
>
> ⭐ **The transferable lesson, and it is recorded because this document committed it while arguing about
> naming:** *a census of a NOUN is not a search for a CONCEPT.* Revision 1 counted nine candidate words
> across the corpus, found `composition` at 389 occurrences, and concluded *taken* — **which is the
> correct output of the wrong query.** ⇒ **before proposing an artifact, search for the FAMILY it would
> join, never for the artifact's own name.**

---

## §0 Summary

| | |
|---|---|
| **The gap** | **`Depends:` answers *what must exist BENEATH me* and never *what must exist BESIDE me*.** Stated in the architecture document twice and — **newly, revision 2** — a third time in a landed guide, as that guide's own scope exclusion. **Nothing is re-derived here** |
| **The consequence** | *"install this set and you get this capability"* **is unsayable**, and installing one member of a family alone accomplishes nothing |
| **What is new** | one artifact — a named set, its capability sentence, its members, and what is optional within it |
| ⭐ **Home (CHANGED in r2)** | **the peer-composition family.** That guide already defines a peer as a configuration *including installed extensions* and already carries a named catalog. **The install set is the single-peer half of its own frame** |
| ⭐ **What this proposal mostly does** | ⛔ **stops the gap being closed under a word that already carries five distinct meanings, one of them wire-visible and pinned** — §2 |
| **Recommended noun** | **`INSTALL-SET`, on taste rather than evidence** — the recognition argument is withdrawn. `PRESET` and `KIT` are equally free. **§6.1 is the one thing owed a decision** |
| **Cost** | one guide section · one developer-facing restatement citing it as authority · one new reader. **Zero moves, zero renames, reversible** |

---

## §1 The gap, cited rather than re-argued — and it is stated three times, not once

`SYSTEM-ARCHITECTURE` §13.1b note 8 states the mechanics:

> *"`Depends` answers what must exist BENEATH me, and never what must exist BESIDE me — so this DAG
> cannot answer the question people bring to it. A functioning content-fallback path needs SUBSTITUTE
> **and** CONTENT **and** TREE **and** a substitute convention **and** the capability system; a
> functioning name-resolution path needs REGISTRY **and** NETWORK **and** at least one backend. None of
> those are `Depends` edges in the inheritance sense, and installing any one member alone does
> nothing."*

⭐ **And `GUIDE-PEER-COMPOSITIONS` states the same gap independently, in its opening scope note, as the
thing it deliberately does not answer** *(found in revision 2; revision 1 cited this document zero
times)*:

> *"It is not the application-development question. **'What do I install to build a thing, which
> conventions apply, when do I write a handler instead of an extension, and what can I build on top?'**
> is a different question with a different audience, and this guide does not answer it. **Nothing here
> tells an application developer what to install.** … note that **the gap between those two and this one
> is real and is not yet closed by any document.**"*

⇒ **That is not a supporting citation. It is the gap, named by the family that owns the surrounding
vocabulary, with an explicit statement that no document closes it.** It is also the reason §3.3's
placement changed.

**Three consequences follow, each independently sufficient:**

1. **The dependency DAG is being read as an answer to a question it cannot answer.** A correct
   inheritance graph and a wrong install guide, and nothing says so except note 8.
2. ⛔ **A landed normative ruling presupposes this vocabulary and it does not exist.**
   `EXTENSION-SUBSTITUTE` §9.2 rules that *"required for v1 binds the implementation, not every
   deployment"*, and adds a **`[MUST]`** that a peer offered for conformance measurement have a named
   convention installed — **both of which are only meaningful if a deployment can be described.** ⚠
   **This is the oldest and most independent witness in the document**, predating the gap's filing by
   weeks, and revision 1 underused it in favour of the two filings that turned out to be one.
3. ⭐ **The application developer's question is answered by no grouping of documents.** The corpus's
   organizing axes answer *what mechanism is this* and *who is this about*; **neither answers *what do I
   need in order to do X*.** That question does not want a directory, a tier or a namespace — **it wants
   a list.**

⚠ **And the cheap version must be ruled out explicitly, because it is what everyone reaches for first:
this is not solved by grouping the specification documents into families.** A distribution that ships a
build-essential metapackage **does not relocate the compiler into a `devel/` directory to do it.** The
package namespace stays flat and global; the group is an aggregation **layered over** it, explicitly
non-exclusive, and anyone wanting a different combination names one. **The flat list and the group are
compatible by construction — that is the whole design.** Grouping the documents delivers a smaller
benefit at a far higher price and **forecloses combinations rather than describing them.**

---

## §2 ⛔⭐⭐⭐ *Profile* is the wrong word, measured — and it already has five senses

**The gap has been filed as `PROFILE`. Before minting the vocabulary, the noun was counted.** Across
`specs/` and `guides/`:

| | |
|---|---|
| occurrences of *profile* | **238 in 24 documents** *(321 counting possessives and hyphenated forms)* |
| the single largest holder | **`EXTENSION-NETWORK` — 177 occurrences in one document** |

**And it is not one meaning used 238 times. It is at least five:**

| sense | what it means | weight |
|---|---|---|
| ⛔ **transport profile** | a named transport binding — `profile-id`, `profile_ref`, the per-protocol entries | **the dominant sense; 46 bare + 11 `profile-id` + 9 `profile_ref`, and it is WIRE-VISIBLE and PINNED inside a deterministic ordering three implementations agree on** |
| **conformance profile** | a named subset of checks a run is graded against | `GUIDE-CONFORMANCE`, 35 occurrences |
| **durable profile** | a durability posture | 8 |
| **validation / operational / performance profile** | three further unrelated senses | 7 |
| **default profile** | ⭐ **a default composition of DECISIONS inside one mechanism** — a settled ruling elsewhere in the corpus, and the closest sense to this proposal's subject **while still being a different thing**: it composes choices within a mechanism, not extensions across a deployment | — |

⇒ ⭐⭐⭐ **Minting *profile* for an install set makes it a SIXTH sense, in a corpus where the dominant
sense is wire-visible and pinned.** A reader meeting *"the content-fallback profile"* has no way to know
whether that is a transport binding, a conformance subset, a durability posture or a list of extensions.

> ⛔ **And this corpus has just paid for exactly this.** A neighbouring family was retired specifically
> because its noun *had become four different words*, and the lesson recorded from it was that a term
> carrying several senses is not a naming preference — **it is a lookup failure, because a reader holding
> the word cannot reach the thing.** ⇒ **minting a sixth sense of an already-quintuple-overloaded word,
> in the same cycle that retired a quadruple-overloaded one, would be that finding committed inside its
> own correction.** That is the specific reason this proposal leads with the noun.

### 2.1 The candidates, counted

**Every candidate noun was measured against the same surface before one was recommended**, because a name
that *sounds* free and is not is the failure above with an extra step:

| candidate | occurrences | documents | verdict |
|---|---|---|---|
| `profile` | 238 | 24 | ⛔ **five senses, one of them pinned** |
| `group` | 1398 | 39 | ⛔ taken — and it is also a landed extension |
| `set` (bare) | 817 | 69 | ⛔ far too general to be a noun |
| `tier` | 577 | 47 | ⛔ taken, and **this is deliberately not a tier** — §4.1 |
| `composition` | 389 | 64 | ⛔ **taken — and revision 2 corrects WHY.** Not merely *"it names a document tier"*: it names the **family this artifact belongs to** (§3.3), and it is **already in active use for a second, narrower scope elsewhere in the ecosystem** — §6.4 |
| `stack` | 141 | 30 | ⛔ taken |
| `bundle` | 138 | 22 | ⛔ taken |
| `family` | 103 | 35 | ⛔ taken — it already means *a group of related specifications* |
| `suite` | 91 | 20 | ⛔ taken by conformance |
| `variant` | 67 | 23 | ⛔ taken |
| `capability-set` | 0 | 0 | ⚠ **string is free and the MEANING is not** — *capability* is **1589 occurrences in 75 documents** and names the authorization system. *"A capability set"* reads as *a set of grants* |
| `install-set` · `preset` · `kit` · `loadout` · `feature-set` | ~0 | — | ✅ **all free.** `install-set`'s two occurrences are the two filings of this gap — **which revision 1 counted as recognition and which are one author, one commit, one document** |

### 2.2 ⇒ Recommendation: `INSTALL-SET`, **on taste, and the document says so**

⛔ **The recognition argument is withdrawn (see the revision-2 banner).** With it gone, `INSTALL-SET`,
`PRESET` and `KIT` are three free strings and the choice is judgement.

**`INSTALL-SET` is still the recommendation, for two reasons stated as preferences rather than
evidence:** it is **literal** — it says what the set is a set *of* — and it **names the act**, which is
correct because membership is a deployment-time fact rather than an architectural property.

**Its weaknesses, stated rather than hidden:** it is a compound where the corpus prefers single nouns for
extensions *(this is not an extension — §4.1)*; and *install* makes it read as operational rather than
architectural, **which is accurate and may still be unwanted.**

⚠ **This is a recorded decision with its reason and not a derivation.** No mechanism fails if the word is
different — organizational choices have none — and **a census says which words are spoken for, never that
a particular replacement is necessary.** ⇒ **§6.1 is the one item in this document owed an explicit
choice rather than a review.**

---

## §3 The artifact

### 3.1 What one install set declares

```
INSTALL-SET <name>

  Capability      one sentence, in the application developer's words, of what
                  becomes possible. Never a list of mechanisms.
  Required        the extensions that MUST all be present. Installing a strict
                  subset delivers nothing, and that is the membership test.
  Choose one      where the set needs a member of a class but not a specific one
                  (a substitute convention; a registry backend; a transport).
  Optional        members that widen the capability without being required for it.
  Not included    named exclusions where a reader would reasonably expect one.
  Authority       the specification that owns each member's own contract.
```

⭐ **`Required` has a falsifiable membership test and it is the whole discipline: remove any one member
and the capability must stop working.** A member whose removal leaves the capability intact is
`Optional`, and a set whose members are individually useful is not a set — it is a list of extensions
with a heading.

### 3.2 The two sets that already exist in prose, as worked examples

**Neither is invented here. Both are lifted verbatim from §13.1b note 8**, which is what makes them the
right first two:

| | `content-fallback` | `name-resolution` |
|---|---|---|
| **Capability** | *"when my content store misses, try somewhere else"* | *"turn a name somebody gave me into a peer I can reach"* |
| **Required** | `SUBSTITUTE` · `CONTENT` · `TREE` · the capability system | `REGISTRY` · `NETWORK` |
| **Choose one** | a substitute convention | a registry backend |
| **Not included** | any locator mechanism — **the fallback set works with a pre-configured list and the locator is what turns a bare hash into candidates.** Stating the exclusion is what keeps the locator optional | — |

⚠ **Two is the right number to land with.** A roster of a dozen sets written before anyone has consumed
one is a taxonomy invented in advance, and the corpus's standing position is that a set is pinned after
the thing has been exercised, never before.

### 3.3 ⭐⭐ Where it lives — **CHANGED IN REVISION 2**

| | |
|---|---|
| ⭐ **normative home** | **`GUIDE-PEER-COMPOSITIONS`, as a new section beside its catalog.** That document **already defines a peer as a *configuration* whose facets include *installed extensions***, already defines a **composition** as a family of configurations, and already carries a **named catalog of seven**. ⇒ **an install set is the single-peer half of that guide's own frame**: the guide names sets of **peers**; this names sets of **extensions** inside one. **The tell that this is right: adding it makes the guide's own scope-exclusion paragraph (§1) unnecessary** — a document that has to disclaim a question is one section short of answering it |
| **developer-facing home** | **`GUIDE-APPLICATION-DEVELOPMENT`**, whose audience is the party this artifact exists for — as a restatement that **NAMES the composition guide as its authority**, so the next consistency sweep is a grep rather than a reading |
| ⛔ **NOT the architecture document's classification section** | revision 1's answer. It is a **freeze-sequencing taxonomy written for specification authors**, and the reader who asks *what do I install* is an application developer or a peer operator. Note 8 stays exactly where it is and **gains a forward pointer**, which is the whole of that document's change |
| ⛔ **not its own document** | an install set is a **roster**, not a composition **document**. A new top-level file would put it one lookup further from the family whose vocabulary it uses |
| ⛔ **not a tier row** | §4.1 |

---

## §4 What this deliberately is NOT

### 4.1 Not a tier, and not a sixth tier either

The existing classification answers **what a peer is · what a production deployment needs · what the
community will expect.** An install set answers **what one capability requires.** ⇒ **they are different
questions and a set takes no row in the tiers**, the same way the bridge family takes none. **A set may
draw members from any tier, and it must be able to** — every worked example above already spans two.

### 4.2 Not exclusive, and not a partition

**A given extension belongs to as many sets as it serves.** `TREE` will be in nearly all of them. **A set
that tried to own its members would reintroduce the foreclosure the flat-list argument in §1 rejects.**

### 4.3 Not a dependency graph, and it must not be mistaken for one

`Depends` stays exactly what it is. **A set does not create, imply, weaken or override a `Depends` edge**,
and a member's own contract remains its specification's. ⇒ **the two answer different questions about the
same objects.**

### 4.4 Not a conformance artifact

**An install set says nothing about what a peer must implement to be conformant.** It describes a
deployment. Conformance profiles already exist, already mean something else, and this must not be read as
touching them — **which is the second independent reason not to call this a profile.**

### 4.5 ⭐ Not a composition, and the distinction is the reason the two can share a home

**A composition is a set of PEERS wired by grants and coupling, yielding a property no single peer can
yield alone. An install set is a set of EXTENSIONS inside ONE peer, yielding a capability no single
extension yields alone.** Same shape of claim, one scope apart — which is why they belong in one document
and why **the two words must not be used interchangeably in it.**

---

## §5 The enforcement point, because a vocabulary without one is a paragraph

**Three mechanical checks. The first two were named in revision 1; the third is new and is the one that
would have caught this document's own placement error.**

| check | why it is not cosmetic |
|---|---|
| **every member of every set resolves to a specification that exists** | the corpus already expresses part of its roadmap as citations to **nine extensions that have no document**, the most-cited reached by five references. **A set naming one of those would read as installable** |
| **every extension a specification calls *required for* a capability appears in at least one set** | the gap's own shape turned into a query. Without it the vocabulary lands, two sets exist, and the eleventh capability nobody wrote a set for is indistinguishable from a capability needing no set |
| ⭐ **every set's members are reachable as a dependency closure, and the closure's roots are declared** | **this is the analyzer the artifact actually needs**: given a set, walk each member's `Depends` and report what the set pulls in *transitively* but does not *name*. A set whose closure silently requires a twelfth extension is the `Depends`-vs-beside confusion **reappearing inside the artifact built to fix it** |

⚠ **All three are reader-level first, not gates.** A first run with a large backlog teaches people to skip
the check, and the honest state on day one is *two sets written, an unknown number owed.*

---

## §6 Open questions

| | |
|---|---|
| **1** | ⛔ **The noun.** §2.2 recommends `INSTALL-SET` **on taste, the recognition argument having been withdrawn.** `PRESET` and `KIT` are equally free. **This is the one thing in this proposal that wants an explicit choice rather than a review** |
| **2** | **Does a set carry a version?** **Leaning no** — a set that versions becomes a second roster to keep in sync with the specification headers, and the corpus has already been bitten four times by exactly that shape |
| **3** | **Who may mint a set?** If an application convention may declare one, the vocabulary spreads to the tier least able to keep it current. **Leaning: sets live in the one normative home and a convention cites one** |
| **4** | ⚠ **Does the locator mechanism get a set of its own, or is it a `Choose one` inside `content-fallback`?** ***Deliberately unanswered*** — that mechanism's placement is open pending two implementations building against it, and answering it from this side would be deciding it by the back door |
| **5** | **Is there a minimum set — what one installs to have a working peer at all?** Probably, and it is a different kind of object: every other set is optional by construction and that one would not be. **Not proposed here** |
| **6** | ⭐⭐ **NEW in r2 — `composition` is in active use at two different scopes in this ecosystem and nothing reconciles them.** The landed guide uses it for *a family of peer configurations wired together*; **at least one other party uses it for *one peer plus a declared extension set*, which is precisely this proposal's subject.** ⇒ **that is one word for the two scopes §4.5 exists to separate, and it must be resolved before either is pinned.** ⚠ **Not resolved here, deliberately — it is not a question this document may answer alone**, and it is the reason §2.1's `composition` verdict is now *taken by the family* rather than *taken by a document tier* |

---

## §7 What this revision establishes, and what it does not

✅ **Establishes:** that the gap is stated **three** times in landed text — twice in the architecture
document and **once as a landed guide's own scope exclusion, with that guide saying no document closes
it** (§1) · that a landed ruling and a landed `[MUST]` both presuppose the vocabulary (§1.2) · ⭐ **that
*profile* is unavailable, with the census behind it** (§2) · that the candidate field was measured (§2.1)
· the artifact's shape with a falsifiable membership test (§3.1) · two worked sets lifted from existing
prose (§3.2) · ⭐ **the home, corrected to the peer-composition family, with the guide's own disclaimer as
the argument** (§3.3) · five exclusions including the composition/install-set scope split (§4) · three
checks, the third being the dependency-closure analyzer (§5).

⛔ **WITHDRAWN from revision 1:** the *"recognition, 2 of 2"* argument for the noun · the architecture
document as normative home.

⛔ **Does NOT establish:** the noun (§6.1, **the one thing owed a decision**) · any set beyond the two
already in prose · whether a set versions · who may mint one · a minimum set · ⭐ **the two-scope
`composition` collision (§6.6), which is new, real, and not this document's to settle.**

⚠ **And it does not claim the vocabulary is necessary.** It claims the **gap** is real, measured and
stated three times by parties who each declined to close it, and that closing it is cheap and additive.
**An organizational artifact has no mechanism that fails when it is absent** — what fails is a reader.
