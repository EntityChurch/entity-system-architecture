# PROPOSAL — the install set is the missing vocabulary, and *profile* is the wrong word for it

**Proposes:** a **named install set** — *"to have capability C, install set S"* — as a new classification
artifact in `SYSTEM-ARCHITECTURE` §13, with a developer-facing restatement in
`GUIDE-APPLICATION-DEVELOPMENT` and a mechanical check. **Nothing is renamed, no namespace moves, no
existing document loses a section.** The artifact is purely **additive**.

**And it proposes NOT calling it a profile**, which is what the gap has been filed under. §2 is a census
of that word in this corpus and it is the substance of this proposal rather than a footnote to it.

**Status:** **DRAFT 2026-09-14 · revision 1.**
**Depends:** `SYSTEM-ARCHITECTURE.md` §12, §13.1, §13.1b notes 6–9 · `EXTENSION-SUBSTITUTE.md` §9.2 ·
`GUIDE-APPLICATION-DEVELOPMENT.md` §3, §5 · `SPECIFICATION-FORMAT.md` §8.8, §8.9
**Kind**: proposal · **Authority**: non-binding until folded · **Governed-by**: `SPECIFICATION-FORMAT.md`
**Audience**: extension and specification authors; application developers are the artifact's audience but
not this document's.

---

## §0 Summary

| | |
|---|---|
| **The gap** | **`Depends:` answers *what must exist BENEATH me* and never *what must exist BESIDE me*.** Both `SYSTEM-ARCHITECTURE` §12 and §13.1b note 8 already state this, in those words. **Nothing in the corpus is being re-derived here** |
| **The consequence** | *"install this set and you get this capability"* **is unsayable**, and installing one member of a family alone accomplishes nothing |
| **What is new** | one artifact — a named set, its capability sentence, its members, and what is optional within it |
| ⭐ **What this proposal mostly does** | ⛔ **stops the gap being closed under a word that already carries five distinct meanings in this corpus, one of them wire-visible and pinned** — §2 |
| **Recommended noun** | **`INSTALL-SET`**, because it is the phrase the two documents that filed the gap already use for the concept. **Recognition rather than invention** |
| **Cost** | one `SYSTEM-ARCHITECTURE` section · one guide section citing it as authority · one gate widening. **Zero moves, zero renames, reversible** |

---

## §1 The gap, cited rather than re-argued

`SYSTEM-ARCHITECTURE` §13.1b note 8 states it completely:

> *"`Depends` answers what must exist BENEATH me, and never what must exist BESIDE me — so this DAG
> cannot answer the question people bring to it. A functioning content-fallback path needs SUBSTITUTE
> **and** CONTENT **and** TREE **and** a substitute convention **and** the capability system; a
> functioning name-resolution path needs REGISTRY **and** NETWORK **and** at least one backend. None of
> those are `Depends` edges in the inheritance sense, and installing any one member alone does
> nothing."*

**Three things follow and each is independently sufficient motivation:**

1. **The dependency DAG is being read as an answer to a question it cannot answer.** It is a correct
   inheritance graph and a wrong install guide, and nothing in the document says so except note 8.
2. ⛔ **A landed normative ruling presupposes this vocabulary and it does not exist.**
   `EXTENSION-SUBSTITUTE` §9.2 rules that *"required for v1 binds the implementation, not every
   deployment"* — **which is only meaningful if a deployment can be described.** A rule whose scope
   depends on an undefined noun is satisfiable by assertion.
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

**The gap has been filed as `PROFILE` in two places. Before minting the vocabulary, the noun was
counted.** Across `specs/` and `guides/`:

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
| `composition` | 389 | 64 | ⛔ taken, and it names a whole document tier |
| `stack` | 141 | 30 | ⛔ taken |
| `bundle` | 138 | 22 | ⛔ taken |
| `family` | 103 | 35 | ⛔ taken — it already means *a group of related specifications* |
| `suite` | 91 | 20 | ⛔ taken by conformance |
| `variant` | 67 | 23 | ⛔ taken |
| `capability-set` | 0 | 0 | ⚠ **string is free and the MEANING is not** — *capability* is **1589 occurrences in 75 documents** and names the authorization system. *"A capability set"* reads as *a set of grants* |
| `preset` · `kit` · `loadout` · `feature-set` | 0 | 0 | ✅ free, and each is either imprecise or borrowed from a different domain |
| ⭐ **`install-set`** | **2, in 1 document** | 1 | ✅ **free — and both occurrences are the two filings of THIS gap, in the two documents that state it** |

### 2.2 ⇒ Recommendation: `INSTALL-SET`, and the argument is recognition rather than invention

**The two places that identified the gap both call it *"the install-set vocabulary."*** The concept
already has a name in this corpus; what it lacks is an artifact. ⇒ **adopting that phrase costs one
hyphen and zero re-education**, and the corpus's own precedent for this move is explicit: a default
composition ruled elsewhere was accepted on the grounds that **four of seven surveyed consumers already
were it, so it was recognition and not invention.** The same test passes here at 2 of 2.

**Its weaknesses, stated rather than hidden:** it is a compound where the corpus prefers single nouns for
extensions *(this is not an extension — §4.1)*; and *install* implies a deployment-time act, which is
**correct** but does make it read as operational rather than architectural. **If a single noun is wanted,
`PRESET` and `KIT` are both measurably free**; neither is more precise and both are less literal.

⚠ **This is a recorded decision with its reason and not a derivation.** No mechanism fails if the word
is different — organizational choices have no such mechanism, and a census of past choices describes the
past and cannot constrain the future. **The census is evidence about which words are already spoken for;
it is not an argument that any particular replacement is necessary.**

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

### 3.3 Where it lives, and why not in its own document

| | |
|---|---|
| **normative home** | ⭐ **`SYSTEM-ARCHITECTURE` §13, beside the tier classification and the dependency DAG.** The DAG is what a reader holding this question already opens, note 8 already sends them here, and **the roster gate already reads this document** — so the check in §5 is a widening rather than a new instrument |
| **developer-facing home** | **`GUIDE-APPLICATION-DEVELOPMENT`**, whose audience is the party this artifact exists for — **as a restatement that NAMES §13 as its authority**, so the next consistency sweep is a grep rather than a reading |
| ⛔ **not its own document** | an install set is a **roster**, not a composition. The composition tier is for *properties no single extension owns*; a list of extensions is not a property. **A new top-level document would put the roster one lookup further from the DAG it corrects** |
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
same objects**, which is why the set lives beside the DAG rather than inside it.

### 4.4 Not a conformance artifact

**An install set says nothing about what a peer must implement to be conformant.** It describes a
deployment. Conformance profiles already exist, already mean something else, and this must not be read as
touching them — **which is the second independent reason not to call this a profile.**

---

## §5 The enforcement point, because a vocabulary without one is a paragraph

**Two mechanical checks, both available by widening an instrument that already reads
`SYSTEM-ARCHITECTURE`:**

| check | why it is not cosmetic |
|---|---|
| **every member of every set resolves to a specification that exists** | the corpus already expresses part of its roadmap as citations to **nine extensions that have no document**, the most-cited reached by five references. **A set naming one of those would read as installable** |
| **every extension a specification calls *required for* a capability appears in at least one set** | this is the gap's own shape turned into a query. Without it the vocabulary lands, two sets exist, and the eleventh capability nobody wrote a set for is indistinguishable from a capability needing no set |

⚠ **Both are reader-level first, not gates.** A first run with a large backlog teaches people to skip the
check, and the honest state on day one is *two sets written, an unknown number owed.* **The count that
ratchets is the second check's backlog.**

---

## §6 Open questions

| | |
|---|---|
| **1** | ⛔ **The noun.** §2.2 recommends `INSTALL-SET` on a recognition argument. **`PRESET` and `KIT` are measurably free.** This is a recorded decision, not a derivation, and it is the one thing in this proposal that wants an explicit choice rather than a review |
| **2** | **Does a set carry a version?** Arguments both ways: a set is a statement about a capability and capabilities do not version, but its **membership** changes as extensions land. **Leaning no** — a set that versions becomes a second roster to keep in sync with the specification headers, and the corpus has already been bitten four times by exactly that shape |
| **3** | **Who may mint a set?** If an application convention may declare one, the vocabulary spreads to the tier least able to keep it current. **Leaning: sets live in the one normative home and a convention cites one** |
| **4** | ⚠ **Does the locator mechanism get a set of its own, or is it a `Choose one` inside `content-fallback`?** ***Deliberately unanswered here*** — that mechanism's own placement is open pending two implementations building against it, and answering it from this side would be deciding it by the back door |
| **5** | **Is there a minimum set — what one installs to have a working peer at all?** Probably, and it is a different kind of object: every other set is optional by construction and that one would not be. **Not proposed here** |

---

## §7 What this revision establishes, and what it does not

✅ **Establishes:** that the gap is already stated twice in normative text and needs no re-derivation
(§1) · that a landed ruling presupposes the vocabulary (§1.2) · ⭐ **that *profile* is unavailable, with
the census behind it** (§2) · that the candidate field was measured rather than guessed (§2.1) · the
artifact's shape with a falsifiable membership test (§3.1) · two worked sets lifted from existing prose
(§3.2) · the homes and the reason against a new document (§3.3) · four exclusions (§4) · two mechanical
checks (§5).

⛔ **Does NOT establish:** the noun (§6.1, and it is the one thing owed a decision) · any set beyond the
two already written in prose · whether a set versions · who may mint one · a minimum set.

⚠ **And it does not claim the vocabulary is necessary.** It claims the **gap** is real and measured, and
that closing it is cheap and additive. **An organizational artifact has no mechanism that fails when it
is absent** — what fails is a reader, which is why the motivation in §1 is three consequences a reader
hits and not an appeal to consistency.
