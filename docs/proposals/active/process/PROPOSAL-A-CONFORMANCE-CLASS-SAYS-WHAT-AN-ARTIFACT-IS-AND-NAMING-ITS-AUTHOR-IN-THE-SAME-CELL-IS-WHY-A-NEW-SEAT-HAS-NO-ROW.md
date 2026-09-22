# PROPOSAL — a conformance class says what an artifact IS; naming its author in the same cell is why a new seat has no row

**Proposes:** splitting `GUIDE-CONFORMANCE` §7.0's class table into **what the artifact is and what it can
prove** (the class) and **who authors it today** (a dated, roadmap-level statement). **No class is
removed and no artifact is reclassified.** The `Authored by` column moves; it does not disappear.

**And it rules the question that produced this**, which turns out to have two halves with different
answers: **the class declaration `[MUST]` binds conformance *items*, not *requirements*** — so one half
of the reported blocker is inapplicable rather than unsatisfiable — while **the second half is real and
is a defect in the taxonomy.**

**Status:** **DRAFT 2026-09-15 · revision 1.**
**Depends:** `SPECIFICATION-FORMAT.md` §8.5, §8.5a · `GUIDE-CONFORMANCE.md` §7.0, §5.2b.1, §7a, §7d ·
`PROPOSAL-CONFORMANCE-ORACLE-CONTRACT.md` §5a
**Kind**: proposal · **Authority**: non-binding until folded · **Governed-by**: `SPECIFICATION-FORMAT.md`
**Audience**: specification authors, check-set authors, and any seat producing a conformance artifact.

---

## §0 Summary

| | |
|---|---|
| **Reported** | §8.5 makes declaring a conformance class a `[MUST]`; §7.0 enumerates four classes; **a seat authoring a neutral requirement corpus finds none of the four describes what it produces**, and reports fifty files unable to satisfy a `MUST` |
| ⭐ **Half 1 — the `MUST` does not bind those files** | §8.5's obligation is on a **conformance item**. §8.5a already separates the two objects **in those words** — *"`R` distinguishes a requirement from a conformance item"* — so a requirement inventory has no class to declare and is not in breach. **Inapplicable, not unsatisfiable.** Fifty files unblock with no change to either document |
| ⛔ **Half 2 — the taxonomy defect is real, and it is the second clause of the report** | **Each class row names an author repository inside the class definition.** A class is a statement about an artifact's nature; an author is a statement about the current division of labour. **Bundling them means the taxonomy cannot absorb a seat that did not exist when it was written** — and one did not |
| ⭐⭐ **Why it is more than filing** | As written, the behavioral row routes every behavioral check to **one** author *by rule*. **That contradicts a position this corpus has already taken** — that a single oracle cannot measure itself, and that the target is independent parallel implementations of the check set. A taxonomy cannot make that direction unsayable |
| **The fix** | one column moves out of the class table into a dated statement beside it. **No fifth class is minted** |
| **Cost** | one guide section restructured · no spec text changes · reversible |

---

## §1 The two halves, because conflating them is what made this look unsolvable

The report reads: *"§7.0's four conformance classes have no row for what this seat produces, and §8.5
makes declaring your class a `[MUST]`. Add a fifth class, or stop naming one repository as author of the
first."* **Both clauses are correct observations. They are about different objects and only the second
needs a change.**

### 1.1 ✅ Half 1 — the `[MUST]` is on items, and a requirement is not one

`SPECIFICATION-FORMAT` §8.5: *"A conformance **item** MUST declare its class."* §8.5a then defines the
two objects and distinguishes them by id form:

> *"**`R` distinguishes a requirement from a conformance item.** `ROUTE-R3` is a requirement;
> `ROUTE-EXACT-1` is an item that may exercise it."*

⇒ **A requirement inventory row is not a conformance item and has no class to declare.** Its shape
obligation is §8.5a's — a stable `<PREFIX>-R<n>` id, one independently failable obligation per row, a
`Level` from the closed six-value vocabulary, a section reference — and a class field is **not** among
them. A corpus of files named `<PREFIX>-R<n>` is declaring, by the corpus's own id convention, that it is
the first object.

**So the fifty files are conformant on this axis today and the `MUST` was never pointed at them.** No
edit to either document is required to unblock them.

⚠ **With one condition, which is the other half of the same report and is the filing seat's own
observation:** where such a file also carries **scored arms** — a thing that is *exercised* and yields a
verdict — that file is **both objects at once**, and §8.5a's split applies to it. The requirement half
keeps the `R` id; the exercising half becomes an item, gets a class, and **names the requirement ids it
drives** (§8.5a's `[SHOULD]`). **That is a split the filing seat has already identified and offered to
perform, and it is the right move** — performed now it costs a schema change across a corpus that is
still growing; performed later it costs the same change plus every row added in between.

### 1.2 ⛔ Half 2 — the class table bundles an artifact's nature with its current author

§7.0's table has four rows and **an `Authored by` column**, whose cells name specific repositories, with
the behavioral row adding a routing rule: a spec revision needing behavioral proof files the check
against the oracle author, *"not whichever seat found the defect."*

**The routing rule is good and this proposal does not touch it.** It was earned — it exists because
several documents asked the *scorer* to write the exam, and *"asking the scorer to write the exam"* is
exactly the right objection. **What is wrong is where it lives.**

| | |
|---|---|
| **a class** | a statement about **what an artifact is and what it can prove**. Stable. Changes only when a genuinely new kind of artifact appears |
| **an author** | a statement about **the current division of labour**. Dated. Changes whenever the estate changes — and it changed |

**Bundled, the table answers *"what is this thing?"* with *"whoever currently writes those."*** A seat
producing a neutral requirement corpus then reads four rows, finds its own name in none of the author
cells, and correctly concludes it has no row — **when in fact the artifact-nature question had a clean
answer all along and only the author question did not.**

⇒ **This is the general defect of putting a build-state claim inside a durable classification, and this
corpus already holds the rule against it one tier over** — a specification does not record who argued
what, because a dated observation inside a durable document is re-read as a rule long after it stops
being true, and correcting it in place only resets the clock. **§7.0 is the same failure in a guide**: the
table was written when the estate had one check-set author, and nothing re-read it when the estate grew a
seat.

---

## §2 ⭐⭐ Why this is not a filing preference: the taxonomy contradicts a position already taken

The corpus's standing position on measurement is that **a single oracle cannot distinguish *"the peer is
wrong"* from *"the oracle is wrong"*** — every check it runs is scored by the judgement that wrote it, so
its own errors are invisible by construction. The stated target is **independent parallel implementations
of the check set**, and a draft proposal already owns that contract.

**A class table that names one repository as *the* author of behavioral checks makes that target
unsayable by rule.** A second, independent behavioral check set is precisely *"a behavioral check not
authored by the oracle author"* — which the table, read as written, classifies as nothing.

⇒ **The decoupling is not tidiness. It is what lets the taxonomy describe the state the corpus is
deliberately moving toward**, rather than freezing the state it had on the day the section was written.

---

## §3 The proposed shape

### 3.1 `GUIDE-CONFORMANCE` §7.0 — the class table, author column removed

| Say | What it is | What it can prove | Lives in |
|---|---|---|---|
| **behavioral check** (*`validate-peer` check*) | an assertion driven **over the wire against a running peer**, grouped into a category | interop under live conditions; the only class that can falsify a cross-peer claim | the check set's own tree |
| **fixture-corpus vector** | static byte-level data — source notation plus canonical encoding | byte-level agreement, offline, with no peer running | the core protocol's test-vector tree |
| **host-seam check** | an assertion about a peer's **in-process API**, driven by the peer's own harness and asserted over the wire | that a surface a wire protocol cannot reach behaves as specified (§7d) | harness per peer; assertion in the check set |
| **impl-internal test** | unit / property / fuzz, not cross-impl | correctness inside one implementation. **Not conformance** | each implementation's own tests |

**Unchanged:** the four classes, their definitions, and the rule that a spec pinning a conformance item
names which. **Removed from this table:** the author cells.

### 3.2 New — §7.0a, *Who authors each class today*

A short, **dated** subsection carrying what the column carried, stated as current division of labour
rather than as definition — **including the routing rule verbatim**, which is the part worth preserving:
a spec revision needing behavioral proof files the check against the check set's author rather than
against whichever seat found the defect, and the scorer's re-census is listed as a **consequence, never
an ask**.

> ⭐ **And one sentence the current text does not carry:** *the author of a class is a fact about the
> estate on a date, not a property of the class. A second independent implementation of any class is
> expected and is the direction of travel (§5a of the oracle-contract proposal); it does not need a new
> class to exist in.*

### 3.3 New — §7.0b, *A requirement is not a conformance item*

One paragraph restating §8.5a's split **in the guide where the class question is asked**, and naming
§8.5a as its authority so the next consistency sweep is a grep rather than a reading:

> A **requirement** (`<PREFIX>-R<n>`) states an obligation. A **conformance item** exercises one or more
> requirements and yields a verdict. **The class declaration `[MUST]` in `SPECIFICATION-FORMAT` §8.5 binds
> items.** A requirement inventory declares no class. **An artifact that both states an obligation and
> scores it is two objects and is split**, the requirement keeping its `R` id and the item naming the
> requirement ids it drives.

---

## §4 What this deliberately does NOT do

1. ⛔ **It does not mint a fifth class.** Asked for, and it would be wrong: the reported artifact is a
   **requirement inventory**, which §8.5a already governs. **A fifth class would create a second home for
   an object that has one**, and the boundary between two homes is exactly what drifts.
2. **It does not change the routing rule**, which is earned and correct.
3. **It does not reclassify any existing artifact.** Every item classified today keeps its class.
4. **It does not touch `SPECIFICATION-FORMAT`.** §8.5 and §8.5a are correct as written; the guide they
   point at is what needs the edit. *(This is worth stating because the reported symptom appeared in §8.5
   and the defect is not there — a `MUST` reported as unsatisfiable when its referent was misread is a
   different repair from a `MUST` that is wrong.)*

---

## §5 The enforcement point

**A class table with no author column is checkable by the rule this corpus already applies to
specifications:** a durable classification that names a repository is the same finding as process
narrative in normative text, and there is an existing narrative rule for seat names. ⇒ **extend that
rule's scope to `GUIDE-CONFORMANCE` §7.0's table region**, so a future author cannot re-add an author
cell without the gate saying so. The dated subsection in §7.0a is exempt **by being declared** — which is
the same shape as every other auditable exemption here: an exemption nobody can audit is not an
exemption.

⚠ **Honest limit:** this catches a repository *name*. It does not catch a class definition that is
implicitly single-author in prose. **That is a limit to know, not a defect to fix now.**

---

## §6 Open questions

| | |
|---|---|
| **1** | **Should §7.0a live in the guide at all, or in the roadmap?** A dated division-of-labour statement is roadmap-shaped. **Leaning: keep it in the guide, adjacent** — a reader asking *"which class is mine"* is one lookup from *"who writes those today"*, and splitting across documents is how the two drifted into one cell in the first place |
| **2** | **Does the host-seam class survive the decoupling intact?** It is the one class defined partly *by* its three-party authorship. **Leaning yes** — its nature is *"an assertion about a surface the wire cannot reach"*, and the three parties are how it is currently produced, not what it is |
| **3** | ⚠ **Where does a second, independent behavioral check set record that it is second?** Not a class question, and not answered here. It matters for how a run reports assurance, since two independent sets agreeing is a different claim from one set run twice |

---

## §7 What this revision establishes, and what it does not

✅ **Establishes:** that the reported blocker has two halves with different answers (§1) · ⭐ **that the
class `[MUST]` binds items and not requirements, on the corpus's own id convention, so a requirement
inventory is inapplicable rather than in breach** (§1.1) · that an artifact both stating and scoring an
obligation is two objects and splits (§1.1) · that the class table bundles a durable classification with
a dated fact and is therefore unable to absorb a new seat (§1.2) · ⭐⭐ **that as written it makes an
already-adopted direction unsayable by rule** (§2) · the restructured table, the dated author subsection
and the requirement/item restatement (§3) · four exclusions including why a fifth class is the wrong
answer (§4) · one enforcement point with its stated limit (§5).

⛔ **Does NOT establish:** whether the author statement belongs in the guide or the roadmap (§6.1) ·
how a second independent check set declares its independence (§6.3) · any change to
`SPECIFICATION-FORMAT`, which is correct as written.
