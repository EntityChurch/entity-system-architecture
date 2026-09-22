# PROPOSAL — the no-rev-bump carve-out exempts a PROPOSAL, never a VERSION, and the bump ladder has no arm for the change we make most

**Proposes:** a third arm on `SPECIFICATION-FORMAT` §9's version-bump ladder — **Correction** — plus one
governing sentence separating a *process* exemption from a *content* claim, and a corrected description of
the `Spec-Change: cohort-finding` commit trailer so its parenthetical stops reading as a licence.

**And it rules the instrument question underneath it:** the residue the L1 gate declares is **not**
acceptable as declared, the third trigger does **not** belong in `provenance`, and the instrument for the
class **already exists one tier over and was scoped to one directory**.

**Status:** **DRAFT 2026-09-16 · revision 1.**
**Depends:** `SPECIFICATION-FORMAT.md` §5.3, §9, §10.2 · `entity-core-protocol/AGENTS.md` §"Change
discipline" · `entity-system-arch-tools` `spec-tool/provenance.py`, `spec-tool/sdksync.py`
**Kind**: proposal · **Authority**: non-binding until folded · **Governed-by**: `SPECIFICATION-FORMAT.md`
**Audience**: specification authors in both corpora, and any seat that pins a specification document.

**Answers:** `CQ-46`, `CQ-47` (round 8, `entity-system-conformance`).

---

## §0 Summary

| | |
|---|---|
| **Reported** | On one release boundary, two of three core documents changed normative content while their `**Version**:` headers — declared the source of truth for the document — did not move. The conformance corpus was byte-identical on the same boundary. **Both surfaces a consumer is told to trust read as unchanged while a `[MUST]` was added** |
| ⭐ **`CQ-46` half 1 — the carve-out was never about the version** | `Spec-Change: cohort-finding` exempts a **commit** from the proposal-first obligation. The words *"no rev bump"* in its description are a **description of what such a change usually looks like**, not a grant. **Two obligations were collapsed into one trailer** |
| ⛔ **`CQ-46` half 2 — and the root cause is in our own authoring standard** | §9's ladder has **two** arms — *additive* and *breaking*. **The class we make most often is neither**, so an author landing a correction has no arm to follow and reasonably takes *"no rev bump"* from the only sentence that mentions the subject. **A reader consults the right home and it answers wrongly by omission** |
| **Measured, not asserted** | 26 `(commit, spec file)` pairs carry the trailer. **7 bumped a version. 19 did not.** One trailer, one author, both readings — which is the conflation, exactly, with no proxy in it |
| ✅ **The ruling** | **A version header is a claim about content. A process carve-out can exempt a commit from a process obligation; it can never exempt a claim from being true.** Third arm added: **Correction → minor bump.** Cause never decides whether the version moves; **only whether a reader's obligations could differ** does |
| ✅ **What a consumer pins** | **The `**Version**:` header, which becomes true.** All three shapes offered in the ask are declined — each adds a *second* contract surface beside a false first one, and the boundary between two homes is what drifts. ⚠ **Their per-document sha256 manifest is right to exist and answers a different question**, and this proposal does not touch it |
| ⭐ **`CQ-47`** | Residue **not** acceptable. **`provenance` is the wrong home** — its unit is a *commit*, the defect's unit is a *pair of documents*, and the two halves need never move in the same commit. **`sdksync` is the instrument**, built 2026-08-17 for this exact class, scoped to `specs/sdk/`, and never re-read. **Seventh scope-set-once in this toolkit** |
| **Cost** | one §9 subsection · one header-field sentence · one `AGENTS.md` trailer description in each of two repos · one new analyzer reusing a shipped one's machinery. **No normative rule in any specification changes strength, and no peer behaviour changes** |

---

## §1 `CQ-46` — two obligations wearing one trailer

### 1.1 What the trailer actually exempts

`AGENTS.md` names two commit trailers as the only auditable way to claim an exemption:

```
Spec-Change: hygiene          wording-only; no normative change
Spec-Change: cohort-finding   an impl finding fixed in place, no rev bump
```

**The subject of both lines is `spec provenance`.** That gate's entire question is *"was there a proposal
for this normative edit?"* — the L1 discipline, and nothing else. `hygiene` says *there is no normative
edit here, so L1 does not apply*. `cohort-finding` says *there is one, and it was discovered by an
implementation building rather than by an author arguing, so it lands without a proposal* — which is
`L26`'s position stated as a carve-out: **the cohort discovers by building, and arch's failure mode is not
folding what they built.**

**Neither line is about the version header, and neither gate reads one.** The clause `no rev bump` is
describing the class — an in-place fix does not manufacture a release — and it has been read as granting
one.

### 1.2 ⛔ The conflation is measurable, and the measurement needs no proxy

Every commit in both corpora carrying `Spec-Change: cohort-finding`, joined against the `specs/**.md`
files it touched, and whether that commit wrote a new `**Version**:` line for that file:

| corpus | `(commit, file)` pairs | wrote a version line | did not |
|---|---|---|---|
| `entity-system-architecture` | 23 | 5 | 18 |
| `entity-core-protocol` | 3 | 2 | 1 |
| **total** | **26** | **7** | **19** |

**Seven of twenty-six bumped while carrying the trailer that says not to.** That is one author, one
trailer, and both readings live in the record — which is the whole finding, and it is exact.

⚠ **A second, weaker cut is reported as weak, deliberately.** Of the 19 that did not bump, **14 touched at
least one diff line containing a normative keyword.** That is a **candidate population and not a count of
false headers**: the filter counts lines that *contain* `MUST`/`SHOULD`, which a reflow or a pointer fix
also does. **Hand-audited, n=1:** `64c58ae`'s `EXTENSION-CONTENT` edit scores 6 on that filter and is a
**stale-citation correction** — `V7 §1.1.4` → `V7 §1.6`, with the cited obligation unchanged — where **not
bumping is correct** under §1.4 below. **No full audit of the 14 has been run and this proposal does not
claim one.** The 7-vs-19 split carries the argument without it.

### 1.3 ⭐⭐ The root cause is in `SPECIFICATION-FORMAT` §9, and it is an enumeration that omits a member

§9 states the bump rule in full, and in two arms:

> - **Minor** (X.Y → X.Y+1): Additive changes (new optional fields, new MAY requirements)
> - **Major** (X.Y → X+1.0): Breaking changes (new required fields, changed algorithms, removed types)

**A correction is neither.** Fixing a rule that was stated wrong adds no field and breaks no consumer who
implemented what the document *meant*; it is a third thing, and it is the change this corpus makes most
often — 26 commits carry the trailer for it in six weeks.

⇒ **An author landing a correction opens the one home that owns the bump rule, finds no arm covering the
change in front of them, and falls back to the only sentence in the estate that mentions rev bumps at all
— the trailer description.** The trailer did not overreach into versioning; **§9 left a vacancy and the
trailer was the nearest text.**

⭐ **This is `L23`'s ENUMERATION SHAPE, third instance in four days**, and its polarity is the expensive
one: **the reader consults the RIGHT home and it answers WRONGLY by omission**, which manufactures
confidence rather than doubt. The other two are `ENTITY-CORE-PROTOCOL` §4.11's cause table omitting the
root-hash row (`0.8.2.28`, reported by a peer who read the table) and the mood axis of `0.8.2.21`. **The
first two were found in specifications. This one is in the standard that governs them.**

### 1.4 ✅ The ruling — cause never decides; consequence does

> **A document's `**Version**:` header is a claim about that document's content. A process carve-out can
> exempt a commit from a process obligation. It can never exempt a claim from being true.**

And the operative test, which replaces *"was this an impl finding?"* with a question a reader can answer:

> **The version moves when a conformant implementation of the previous text could be non-conformant under
> the new text.** Whether the change was proposed, found by a peer, or corrected in place is a fact about
> **how the work reached the document** and decides only which `Spec-Change:` trailer applies.

That resolves `64c58ae` cleanly in both directions: the frame-budget obligation is identical before and
after, only its citation moved, so **no implementation's conformance could differ and the header
correctly stayed put** — while the two documents on the reported boundary each changed what an
implementation must sign, so **both were owed a bump**.

### 1.5 ✅ What a downstream consumer pins — and why no new field

**The `**Version**:` header, as their own `AGENTS.md` already declares.** The ask offered three shapes — a
sha256 in a published index, a `Content-Rev:` header beside `**Version**:`, a fourth component on the
version line — and argued for none. **All three are declined, on one ground:** each one leaves the
declared source of truth able to be false and puts a second, truer surface beside it. **A fact with two
homes drifts at the boundary between them**, which is this corpus's own repeated finding, holding at
three scales already — `SDK-*` restatements against their sources, the discipline charter against its
always-in-context summary, and four version rosters against the headers they copy. **Adding a fourth
instance to answer a question caused by the first three is the wrong direction.** The cheaper fix is to
stop falsifying the surface that exists.

⚠ **And their manifest is not the thing being declined.** `spec-data/…/v0.8.2.26/MANIFEST.md` pins a
sha256 per document and carries a `⚠` on the two unbumped cells. **A version answers *did this document's
obligations move*; a digest answers *are these the same bytes*. Those are different questions and a
snapshot pin needs the second one.** Their workaround is evidence for half the ask and a correct artifact
in its own right; **nothing here asks them to remove it**, and it is not the contract surface either.

⚠ **Stated honestly against ourselves: a minor bump is a coarse signal.** It says obligations moved, not
which ones or how far, and two corrections landing in one release reach the consumer as one increment.
**That is accepted, not overlooked** — the finer signal is the `## Document History` entry and the
proposal it cites, both of which already exist, and neither of which a machine needs in order to answer
the only question a pin asks.

### 1.6 ⚠ The reported instance has since self-corrected, and that is a reason to land the rule, not to drop it

Both documents named in the ask now carry moved versions: `ENTITY-CBOR-ENCODING` `1.7 → 1.8` at
`89ded4e` and `ENTITY-NATIVE-TYPE-SYSTEM` `4.2.1 → 4.3` at `5a64678`. **Neither bump was a correction of
the omission.** Each is a *later, unrelated* fold that happened to touch the same file and bump it on its
way past.

⇒ **The evidence for the ask expired while the ask was in flight, and nothing was fixed** — `L9` on the
axis of a finding rather than a deferral. Left alone, the record now reads as though the headers were
never false, so **the next seat to hit this re-derives it from a boundary we have already been handed.**

---

## §2 `CQ-47` — the residue is real, and the third arm is not in `provenance`

### 2.1 What was measured

`ENTITY-NATIVE-TYPE-SYSTEM` §10.2 changed normative meaning in `232a9ed` — *the target entity's content
hash **digest** bytes* → *the target entity's **full `content_hash`** — format code ‖ digest* — and fires
**neither** `provenance` trigger. The version did not change; the **count** of normative tokens did not
change, because §10.2 is **indicative prose restating a `[MUST]` whose normative home is another
document**. The gate's own docstring anticipates a `MUST`-for-`MUST` swap. This is not that: it is an
unmarked restatement drifting **by one field** from the authority **named in its own sentence**.

**Not acceptable as declared.** Two reasons, and the second is the one that generalizes:

1. **We have a controlled measurement of this exact class, filed before the ask arrived.** Register row
   `EN-4`: two restatements of the same registry, one naming its authority and one not. The one that
   named it **stayed correct**; the one that named nothing **drifted on three axes at once**, invisibly
   from both ends. That is `L23`'s restatement clause with an experiment behind it rather than an
   assertion.
2. **The failure is silent and the corpus is the only detector.** `entity-core-protocol/AGENTS.md`
   retired a whole document over this class, in these words: *"a hand-maintained restatement ages on every
   edit to its source, **nothing in the toolkit gates spec-against-spec**, and the drift is reachable only
   by a human reading both documents side by side."* **§10.2 is that artifact at one-sentence scale,
   inside a live spec.**

### 2.2 ⛔ Why a third `provenance` trigger cannot work — a unit mismatch, not a tuning problem

**`provenance`'s unit is a commit.** It asks *what did this commit do to this file, and was it proposed?*

**The defect's unit is a pair of documents.** A restatement and its authority drift apart when **either
side moves** — and the side that moves is usually **the authority**, in a commit that does not touch the
restating document at all. The reported instance is the easy half; the hard half is §7.3 being tightened
next month with §10.2 untouched, where **no commit-scoped gate has both halves in front of it, ever.**

⇒ **A third trigger would catch the subset where both sides move together and report clean on the rest** —
which is `could-not-look wearing a verdict's clothes` for the seventh time in this toolkit, built
deliberately this time. **Declined.**

The declared residue therefore stands **as a true statement about `provenance`** and stops being an
excuse: the gate is correctly scoped and the class needs a differently-shaped instrument.

### 2.3 ⭐⭐ The instrument exists, and it was scoped to one directory

**`spec sdksync`**, shipped 2026-08-17. Its docstring states the class in the general:

> *"An `SDK-*` spec is a **restatement**. It copies a schema out of an extension spec … and from that
> moment the copy and the original are two artifacts with one meaning and no link between them. Nothing
> in the toolkit compared them, because nothing could: `coherence` is same-file by design, `address`
> checks that a citation *resolves* and not that it is still *true*, and `standards` reads shape. A copy
> that silently stops matching its source is invisible to all three."*

**Every sentence of that is true of `ENTITY-NATIVE-TYPE-SYSTEM` §10.2.** The mechanism — a pin file
recording, per restated block, the source span it was written against and a digest of that span at pin
time — transfers unchanged. What does not transfer is two constants:

```python
PIN_RELPATH   = "specs/sdk/.sdk-source-map.json"
DEFAULT_ROOTS = [Path("specs/sdk")]
```

⇒ **The analyzer that answers `CQ-47` has been in the toolkit for a month, pointed at three files.**
**Seventh instance of a scope set once and never re-read** — after `address`'s `--namespace-root`,
`provenance`'s `--proposal-root`, `coverage`'s class map, `pins`' composition filter, `inbound`'s peer
root, `check`'s corpus default, and `deps`' resolver premise. **The generalization has been written down
six times and did not fire**, which is itself the finding: *an instrument's scope is a premise, and a
premise is invisible in a clean run.*

### 2.4 ✅ The mechanism — pin the pointer to its authority

**A restatement that names its authority is machine-pinnable; one that does not is not.** That is not a
limitation to apologise for — it is `L23`'s cheap half doing the work it was ratified for, and `EN-4`
measured that naming the authority is also what keeps the restatement *correct*. The two halves of the
rule turn out to be one mechanism.

**The idiom is already the corpus's own and is bounded:** `<DOC>.md §<N> is the normative home` occurs
**19 times across both corpora** — 13 in `entity-core-protocol` (6 `ENTITY-CORE-PROTOCOL`, 4
`ENTITY-CBOR-ENCODING`, 2 `ENTITY-NATIVE-TYPE-SYSTEM`, 1 in the ECF changelog) and 6 in this one (3
`EXTENSION-TREE`, 2 `EXTENSION-NETWORK`, 1 `GUIDE-CAPABILITIES`). **Bounded and completable**, the same
shape as the `~40` pseudocode blocks the `L23` sweep already runs against.

**Proposed: `spec pointers`** — a new analyzer reusing `sdksync`'s `normalize`/`digest`/`span_for_anchor`
verbatim, differing only in its unit and its scope.

| | |
|---|---|
| **unit** | a **declared pointer** — a passage naming another document's section as its normative home |
| **scope** | `specs/` **and** `guides/`, in **both** corpora, cross-repo-resolved |
| `pointer-source-moved` | **error** — the authority's span changed since this pointer was written against it. **The pointer is not necessarily wrong; it is unreviewed** |
| `pointer-source-missing` | **error** — the pin names a document or anchor that no longer resolves |
| `pointer-unresolved` | ⚠ **never `missing`** — the authority is in a corpus this run was not pointed at. **Could-not-look is reported as itself.** The run prints the roots it searched |
| `pointer-unpinned` | **warn, ratcheting** — a declared pointer with no pin. First run is the backlog; a gate whose first run is 19 reds is a gate the next contributor skips |

⛔ **What it cannot do, stated in the tool rather than discovered later:** it finds drift in restatements
that **declare themselves**. An *undeclared* restatement — the shape §10.2 had **before** `0.8.2.26` fixed
it — is invisible to it, and to everything else. **That residue is real and is not closed by this
proposal.** What changes is that declaring a pointer now buys enforcement, which is the incentive the rule
lacked.

---

## §3 The edits

| # | file | change |
|---|---|---|
| **E1** | `specs/SPECIFICATION-FORMAT.md` §9 | third bump arm — **Correction** — plus the non-bumping class named explicitly, and the operative test stated as the consumer's question |
| **E2** | `specs/SPECIFICATION-FORMAT.md` §5.3 | one sentence on the `Version` row: the header is a **claim about content**, and what may and may not exempt it |
| **E3** | `AGENTS.md` (this repo) | the trailer block's `cohort-finding` description loses *"no rev bump"* and gains what it actually exempts |
| **E4** | `entity-core-protocol/AGENTS.md` §"Change discipline" | the per-document version obligation, citing `SPECIFICATION-FORMAT` §9 as authority rather than restating it |
| **E5** | `entity-system-arch-tools` | `spec pointers`, per §2.4, with a self-test asserting each silence case |

**E1 and E2 are a normative change to an authoring standard, so `SPECIFICATION-FORMAT` bumps its own
version under the rule it is adding** — `1.4 → 1.5`. That is the first application of the arm and it is
deliberately the document's own.

**E3/E4 are guidance, not specification**, and are the reason this stays one change: the trailer
description is where the vacancy in §9 got filled, so closing §9 without correcting the trailer leaves the
misreading's source intact.

---

## §4 What this does NOT rule

1. ⛔ **Not a version-triggered provenance gate.** Measured and rejected on evidence; nothing here
   revisits that, and the ask explicitly did not request it.
2. ⛔ **Not removal of either carve-out.** Both stay, both keep their gate, and `cohort-finding` keeps
   meaning exactly what it meant about proposals.
3. ⛔ **Not a retroactive sweep of the 19.** The rule binds forward. A sweep of the 14 candidates is
   **owed and unrun**, and is recorded as owed rather than implied — two instances is not a swept corpus.
4. ⛔ **Not a change to any conformance obligation.** No specification's normative content moves in this
   fold; no implementation's behaviour changes; no conformance vector is affected.
5. ⛔ **Not a ruling on the release boundary itself.** Whether public `master`'s re-authored history
   should carry per-document digests is a publication question owned elsewhere.

---

## §5 Open

| | |
|---|---|
| **The undeclared restatement** | §2.4's stated residue. Nothing detects one, here or anywhere. **Unclosed, named** |
| **The 14-candidate audit** | §1.2. Owed, unrun, and not claimed either way |
| **Whether `pointers` should gate or read on first landing** | proposed as warn-with-ratchet on the same reasoning as every other backlog gate in this toolkit; revisit when the unpinned count reaches zero |
