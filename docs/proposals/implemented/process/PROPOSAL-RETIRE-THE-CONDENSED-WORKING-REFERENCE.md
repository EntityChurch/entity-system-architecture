# PROPOSAL — retire the condensed working reference; a derived document is generated or it is not kept

**Status:** **IMPLEMENTED (2026-09-08)** — ruled and folded in the same session.
**Target:** `specs/ENTITY-SYSTEM-REFERENCE.md` → **RETIRED**, moved to `docs/archive/`; undeclared
from `CANONICAL-DOCS.toml`; a section→source map carried as the archived document's banner.
**Scope:** one document in this repository. **The cross-repository re-pointing worklist §5 first
proposed is NOT part of it** — see the ruling immediately below.
**Precedent:** the core machine specification, retired 2026-08-31 on this exact reasoning.

---

## 0. The ruling, and it narrows this proposal rather than accepting it

**Ruled 2026-09-08 — retire it, and the citation count does not change the disposition.**
The reasoning, which is worth stating because this proposal had it backwards:

- **A document that was never supposed to be cited does not acquire authority by being cited.**
  §8.4.3 already said it is not a definition of anything. **49 citations to a non-authority are
  49 wrong citations, not 49 obligations on the seat that retires it.**
- **A stale citation's fix is to cite the source.** Not a fresher copy, not a re-sync, not a
  scheduled migration — the source. That is the whole content of the answer to every seat that
  reports drift against this document, and it is why re-pointing is *cheap* wherever it is done.
- **Nothing is lost and nothing breaks.** The document remains readable in the archive and its
  full history is in git. A reader arriving from an old citation lands on a banner that tells
  them what happened and where the content lives.

**So this proposal's §5 was the error in it.** It sized a retirement by the downstream work it
imagined creating, and then treated that imagined work as a reason the call was hard. **The call
was not hard.** What arch owes is the retirement, the map, and telling the seats plainly what
this document is; **what each seat does about a citation it should not have made is that seat's
ordinary hygiene, on nobody's schedule.**

### What was actually done, in this repository

| | |
|---|---|
| the document | moved to `docs/archive/ENTITY-SYSTEM-REFERENCE.md`, banner + section→source map prepended, body unedited below it |
| `CANONICAL-DOCS.toml` | entry removed, replaced by a comment saying why and *"do not re-declare; do not author a successor"* |
| `docs/archive/INDEX.md` | created — the breadcrumb the tree-hygiene rule requires, listing this and the retirement precedent |
| the two **live spec** citations | re-pointed to the real owners: `EXTENSION-SIGNALING` §6.5 (the four-axis `scope_subset` relation → `ENTITY-CORE-PROTOCOL.md` **§5.6 Attenuation Rules**, which is where `scope_subset` is defined) and `SYSTEM-ARCHITECTURE` §7.1's repository map (line removed) |
| `docs/STATUS.md` | dropped from the system-model list |
| the other 23 in-tree mentions | **left alone** — status docs, folded proposals, explorations and the archive. They are historical record and are correct as history |

**The `EXTENSION-SIGNALING` one is the finding this retirement was worth doing for**, and it was
found by opening every live citation rather than counting them: a **normative `MUST` in a
published extension spec** named the condensed reference as the authority for the relation the
whole rule turns on. The relation is core's, in core's §5.6, and always was.

---

## 1. The rule already exists and this document is the thing it names

`SPECIFICATION-FORMAT` §8.4.3 is normative and landed:

> **A generated, condensed, or summarizing document is downstream of its sources and MUST NOT be
> cited as the definition of anything.** … **A derived document is therefore not repaid by
> re-syncing it: the maintenance burden is unbounded and the gate does not exist, so the
> disposition is retire. Do not author a new condensed, "implementation", or "machine" edition of
> a spec in this corpus.**

**`ENTITY-SYSTEM-REFERENCE.md` is titled *"Condensed Working Reference."*** It is the document
class §8.4.3 forbids authoring, it predates the rule, and it was never reconciled against it.

**Its header already concedes the status without drawing the conclusion:** *"Not for: First-time
implementation — see ENTITY-CORE-PROTOCOL.md for the full normative spec. … Everything here is
grounded in V7; section references (§) point to the normative spec."* A document whose own header
says every claim in it is owned elsewhere is a derived document by its own account.

---

## 2. The precedent, and where this case is worse

The machine specification was retired with four findings. **Three hold here identically and the
fourth is inverted, which makes this case more urgent rather than less:**

| The machine spec's finding | Here |
|---|---|
| a derived document restating the real specs | **same** — 669 lines, 14 sections, all restatement |
| drifted unchecked because **nothing gates spec-against-spec** | **same** — no analyzer compares a summary to its source, and none can |
| found **2-for-2 wrong in the only two sections ever examined** | **same rate** — the capability-pattern table was examined once, this session, and was wrong |
| **nothing generates from it; nothing vendored it** | **INVERTED — 49 files across 6 repositories cite it** |

**The last row is the finding.** The machine spec could be deleted because nobody was reading it.
**This one is being read, and it is being read as an authority**, which is the specific harm
§8.4.3 exists to prevent.

---

## 3. Measured — it is cited as a definition, and it is already known to be stale

**Citations, counted per repository** *(recursive search rooted at each repository; a first
attempt rooted at the shared parent returned zero for a control term and was a could-not-look, not
a measurement)*:

| Repository | Files citing it |
|---|---|
| this repository | 25 |
| the browser application repository | 12 |
| the Rust reference implementation | 5 |
| the Go reference implementation | 3 |
| a second application repository | 3 |
| the core protocol repository | 1 |
| **total** | **49 across 6 repositories** |

**Cited as the definition of a type**, which §8.4.3 forbids in terms — from implementation source
comments:

> `system/tree/path` (`ENTITY-SYSTEM-REFERENCE` §4, one of the six bootstrap meta-types)
> The naming-space address type is `system/tree/path` (`ENTITY-SYSTEM-REFERENCE` §4, and its own row in the §8 table)

**And peers have already filed drift against it, in quantity, without anyone acting on the
document itself:**

> `ENTITY-SYSTEM-REFERENCE` ×10 — all qualified or generalized upstream
> `ENTITY-SYSTEM-REFERENCE` | ten sites (:39, :43, :99, :105, :135, :568, :602, :603, …)
> `ENTITY-SYSTEM-REFERENCE.md:75/601`, **which still carries the stale form**

**Two seats independently reported the same two lines as stale and each worked around it locally.**
That is §8.4.3's predicted failure verbatim — *"an auditor comparing a spec against it will find
real differences and report them as real defects"* — and it has now cost at least three separate
reports.

**This session added one more, found while sweeping for something else:** the capability-pattern
table gave the peer wildcard as `*/system/type/*` and labelled it *"Peer wildcard"* — the exact
spelling the core protocol's `canonicalize` rejects by name. **That is in the table a reader
consults to learn the spelling.** It was corrected in place this session; the correction is not
the fix, because the next drift has the same undetectable path in.

---

## 4. What replaces it

**Not a regenerated edition.** §8.4.3 forbids authoring one, and the governing test stated
forward is the same: *a redundant restatement whose source is elsewhere may be useful, but then it
is **generated output**, not maintained source.* **Nothing in this corpus generates it
today, and building a generator for a condensed prose summary is a larger project than the
document is worth.**

**What is kept is the routing.** A section→source map, published as the archived document's
header, so that every existing citation can be re-pointed rather than merely broken:

| Reference § | Subject | Real owner |
|---|---|---|
| 1–2 | what the system is, entity / tree / identity | core protocol §1 |
| 3 | making a request | core protocol §3 |
| 4 | tree operations, the bootstrap meta-types | the native type system spec; tree extension §1 |
| 5 | handlers | core protocol §6 |
| 6 | security model, capability verification | core protocol §5 |
| 7 | connection establishment | core protocol §4 |
| 8 | tree extension operations | tree extension |
| 9 | extensions summary | the extension roadmap |
| 10–11 | type system quick reference, key type paths | the native type system spec |
| 12 | wire format quick reference | the CBOR encoding spec |
| 13–14 | common patterns, gotchas | the extension development guide |

**Every row lands on a document that is already canonical, already gated, and already the thing
the citation should have named.** No content is lost; one un-gated copy of it is.

**The guides are explicitly not in scope and are not this class.** A guide is a different medium —
it teaches, it does not restate a normative table as its own — and it is the intended home for
rows 13–14.

---

## 5. ~~The worklist, and it spans repositories~~ — **OVERTURNED by §0, kept for the record**

> **This section was wrong and §0 says why.** It is left in place rather than rewritten because
> the error is the useful part: it sized a retirement by imagined downstream work, and then read
> that imagined work back as evidence the decision was expensive. Struck text below.

~~**Each citing repository owns re-pointing its own citations** — 24 outside this repository.~~

**What is true:** arch owns the retirement, the archive move, the undeclaration, and the map.
**There is no worklist for anyone else.** A seat that finds a stale line in a document it should
not have been citing fixes it by citing the source, whenever it next touches that code — the same
as any other wrong citation, with no packet, no schedule and no tracking row against it.

**The two lines already reported stale are not a debt either.** They are the evidence for the
retirement, and the retirement is the answer to them.

---

## 6. What this does not do

- **Does not delete anything.** Archive with a breadcrumb, per the tree-hygiene rule.
- **Does not touch the guides.** Different medium, different rule.
- **Does not propose a generator.** If a condensed view is wanted later it is generated output
  with a source-of-truth pin, and that is a separate proposal with a build behind it.
- **Does not claim the content is wrong.** Most of it is correct today. The claim is that
  *nothing can tell you when it stops being*, which is the same claim §8.4.3 already ratified.
