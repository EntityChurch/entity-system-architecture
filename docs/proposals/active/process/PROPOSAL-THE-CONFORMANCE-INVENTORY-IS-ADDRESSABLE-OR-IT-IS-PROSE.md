# PROPOSAL — a conformance requirement is addressable or it is prose: stable row ids for the extension inventories

**Status:** DRAFT (2026-09-08)
**Target:** `specs/SPECIFICATION-FORMAT.md` §5.1 and §8.5 (normative) — the inventory's shape,
its level vocabulary, and a requirement-row id scheme. Then, **by ratchet and not by sweep**, the
26 extension specifications.
**Depends on nothing.** Blocks the extension half of the conformance check-set work: a check set
specified in neutral language needs something to name, and today a requirement can only be named
by quoting its English.

---

## 1. What was reported, and the half of it that is wrong

The finding as filed: *"the conformance inventory has no declared shape and its rows have no
stable ids."* **The second clause is exactly right and is the expensive one. The first is not
true, and the difference changes what the fix is.**

**The shape is declared.** `SPECIFICATION-FORMAT` §5.1 has prescribed it since the format
standard was written:

```
## N. Conformance
    ### N.1 MUST Implement
    ### N.2 SHOULD Implement
    ### N.3 MAY Implement
    ### N.4 Implementation-Defined
```

**So this is not an undeclared shape. It is a declared shape with no enforcement point**, which
is a different defect with a different remedy: writing the rule again does nothing, and the
corpus is far closer to conformant than the report suggests. **Measured across all 26 extension
specifications, 19 carry that exact content model** — the deviations in those 19 are label
casing (`MUST` / `MUST Implement` / `MUST implement`), section numbering (`9.1` vs `§9.1`), and
one slot's name. That is normalization work, not a per-author parser.

**What is genuinely undeclared is addressability**, and nothing in the corpus has ever said
otherwise: **no requirement row in any specification carries an identifier.** A check can cite
`EXTENSION-HISTORY §9.1` and that section holds **fifteen** separate requirements. The citation
resolves and tells the reader almost nothing.

---

## 2. The census, corrected — 26 specifications, measured section by section

Scored against the **last `##`-level conformance section** in each file. A first pass that matched
the *first* heading containing "conformance" mis-scored at least one specification, whose §1.3 is a
definitions section.

| Shape | Count | Which |
|---|---|---|
| the §5.1 content model, as `###` subsections | **18** | the house shape |
| the §5.1 content model, as **bold-label bullet groups** | **1** | `DURABILITY` — same four groups, `**MUST**` instead of `### N.1 MUST` |
| a **table** with a `Level` column | **2** | `HISTORY`, `REVISION` |
| **numbered conformance levels** | **1** | `QUERY` — Level 1 / Level 2, each with its own MUST and SHOULD |
| **floor + vector list + deferred** | **1** | `ROUTE` |
| **not an inventory at all** | **2** | `ENCRYPTION`, `SUBSTITUTE` |
| **no conformance section at all** | **1** | `ROLE` |

**Two corrections to the number as previously recorded, both from opening the sections rather than
counting headings:**

- **`ENCRYPTION` is not a "floor + vectors + deferred" shape.** Its conformance heading is a
  **landing-status** section — what gates the release, that a finding corrects the spec in place,
  and how the work is scoped against a sibling extension. **The vectors are in a different section
  entirely**, and the conformance heading does not point at them. So it is one of the two with no
  inventory under its conformance heading, not one of the shaped ones.
- **`DURABILITY` is the house shape**, not a third form. It has all four groups, in order, with
  the same content; it writes the labels in bold rather than as subsections. **That is a
  formatting variance and a parser fix, not an authoring divergence**, and counting it as a
  distinct shape overstates the disorder by one.

**The corrected headline: 19 of 26 already follow the declared content model; 4 have a real
alternative shape; 2 have no inventory under the conformance heading; 1 has no section.**

---

## 3. The fourth slot is three different vocabularies, and two of them are not the same kind of thing

Of the 19, the fourth group is variously:

| Spelling | Count | What it actually is |
|---|---|---|
| `Implementation-Defined` | 10 | **the absence of a requirement** — the spec declines to constrain |
| `MUST NOT` | 3 | **a requirement level** — a prohibition, as binding as a MUST |
| `Compatibility` | 2 | neither — a mixed bag of migration and interop notes |
| *(absent)* | 4 | only three groups |

**`MUST NOT` in the fourth slot is the one that matters.** A prohibition is a requirement and must
be checkable; filing it in the slot the format standard names *Implementation-Defined* puts a
binding rule in the position a reader has been taught means *unconstrained*. **Two seats reading
the same document can reasonably reach opposite conclusions about whether the section binds them.**

The correction is not to add a fifth heading. It is that **level is a property of the row, not of
the heading it sits under** — which the table shape already gets right and the subsection shape
structurally cannot.

---

## 4. What this proposes

### 4.1 The inventory is a table, and each row is a requirement

```
## N. Conformance

**Requirement id prefix:** `HIST`

### N.1 Requirements

| id | Requirement | Level | § |
|---|---|---|---|
| `HIST-R1` | Store transition entities in the content store | MUST | §3.1 |
| `HIST-R2` | Store head pointers at `system/history/head/{path}` | MUST | §3.2 |
| `HIST-R7` | Support `max_depth` retention | SHOULD | §3.3 |
| `HIST-R14` | Never record the local peer's own `system/history/*` writes | MUST NOT | §3.2 |
```

**One row is one requirement.** A row that needs the word "and" between two independently
failable obligations is two rows — that is the whole test, and it is the one that makes the
inventory worth having.

### 4.2 `Level` is a closed vocabulary

`MUST` · `MUST NOT` · `SHOULD` · `SHOULD NOT` · `MAY` · `IMPL-DEFINED`

Six values, no others. **`IMPL-DEFINED` is a row like any other** — it is a deliberate statement
that the specification declines to constrain a named surface, which is information a peer author
needs and a check set must not test. `Compatibility` is not a level: its contents are notes,
which belong in prose beside the table, or are requirements, which get a level.

### 4.3 The id: `<PREFIX>-R<n>`, allocated once and never reused

- **The prefix is declared by the specification**, in one line above the table. It is not
  inferred from the filename, because a reader must not have to guess.
- **The prefix is already in use.** Seven specifications carry conformance-vector identifiers
  under a short spec prefix — `ENC`, `REG`, `REV`, `HIST`, `NET`, `ROUTE`, `TREE`. **This scheme
  adopts the prefix that exists rather than minting a second one**, so one specification has one
  identifier namespace covering both its requirements and its vectors.
- **`n` is allocated sequentially and is never reused and never renumbered.** Rows may be added,
  reordered, or **retired** — a retired row keeps its number and is struck, exactly as an error
  code would be. **An id that can be renumbered is not an id**, and the only property this scheme
  has to deliver is that a citation written today still names the same requirement in a year.
- **`R` distinguishes a requirement from a vector.** `ROUTE-R3` is a requirement; `ROUTE-EXACT-1`
  is a vector that may exercise it. They are different kinds of thing with different authors, and
  the existing `SUBJECT-CONDITION-N` vector form is unchanged by this proposal.

### 4.4 A vector says which requirements it drives

Where a specification lists conformance items, each item names the requirement ids it exercises:

```
- `ROUTE-EXACT-1` — exact `match` → forward to `via`. **Drives:** `ROUTE-R2`, `ROUTE-R3`.
```

**This is the join, and it is the reason the ids are worth the trouble.** With it, two questions
become mechanical that are today a careful read by someone who knows both documents:

1. **Which requirements does no item drive?** A `MUST` with no check is a rule the ecosystem
   believes it enforces and does not.
2. **Which items drive nothing declared?** An item testing behaviour no requirement states is
   either a missing row or a check asserting a private opinion — and a check asserting the wrong
   thing passes just as green as one asserting the right thing.

**Neither question can be asked today at the extension tier at all**, because the only handle on a
requirement is the section it lives in and a section routinely holds a dozen.

---

## 5. What this does NOT propose

- **Not a 26-specification sweep.** See §6. A conversion campaign across a whole corpus is how
  this kind of change acquires a schedule, an owner, and then neither.
- **Not a change to any requirement.** Converting an inventory to a table is a shape change and
  nothing else. **Where converting one surfaces a requirement that is really two, or one nobody
  can check, that is a finding to file — not a thing to fix silently inside a formatting edit.**
- **Not a new vector id form.** `SUBJECT-CONDITION-N` stands, has years of use behind it, and is
  not what is missing.
- **Not a change to `Types Installed` / `Handler Registered`.** Those are a manifest, not
  requirements — they stay as their own subsections, and they are already machine-readable.
- **Not a claim that the numbered-level shape is wrong.** `QUERY`'s Level 1 / Level 2 is a real
  distinction the flat model cannot express. **It composes rather than conflicts:** the level
  becomes a column, and the rows carry both.

---

## 6. How it lands — the ratchet, because the sweep is what fails

**Three things, in this order, and only the first two are scheduled:**

1. **`SPECIFICATION-FORMAT` §5.1 and §8.5 gain the shape, the closed level vocabulary, and the id
   rule.** One edit, and it makes every later conversion a lookup instead of a decision.
2. **One worked reference converts in the same change** — the shape argues for itself far better
   from a real inventory than from a schematic, and the reference is what a later author copies.
3. **Every specification touched for any other reason converts its inventory in that same edit.**
   Not on a schedule, not as a project. The set closes as a by-product of ordinary work.

**The third step needs an enforcement point or it is a wish**, and this corpus has learned that
the expensive way: a rule that is correct, canonical and enforced by nothing changes no
behaviour at all.

**The enforcement point — a `spec inventory` analyzer:**

- **Reader by default, exits 0.** A first run reporting 20-odd reds teaches people to skip the
  gate, and a gate people skip is worse than none.
- **The ratcheted number is the count of conformant inventories**, held in the same
  only-ever-improves form the corpus already uses for two other backlogs. It may rise; it may not
  fall.
- **The findings it reports, in increasing order of how much they matter:** an inventory not in
  the declared shape · a `Level` outside the vocabulary · a duplicate or missing id · **an id that
  moved between revisions**, which is the one silent failure this whole scheme exists to prevent ·
  a requirement no item drives · an item driving no declared requirement.
- **The last two are reported as counts, not as errors.** An undriven `MUST` is a real gap and
  usually not the authoring seat's to close in that session.

---

## 7. `ROLE` has no conformance section, and that is a gap rather than a shape question

One specification of the 26 has **no conformance section at all** — it ends with security
considerations and examples. It is not a small document, it defines a handler, five type
families, a grant-derivation algorithm and a three-layer exclusion model, and **nothing in it says
what an implementation must do to claim it.**

**This is not part of the shape work and should not be folded into it.** Writing that inventory
means deciding which of that document's behaviours bind, which is authoring, not formatting. It is
named here so it stops being invisible behind a census that reports it as a formatting outlier.

---

## 8. Open items

1. **Do the core protocol's own conformance requirements get ids under the same scheme?** They
   have the same problem and a larger blast radius, and the core specification is versioned by a
   different authority with a deliberately low change rate. **Recommendation: not in this change.**
   Prove the scheme on the extension tier first — the extension tier is where the check sets are
   actually missing, and a scheme that has never been used is a bad thing to put in the core.
2. **What happens to a requirement that is retired?** The row stays with its id and is marked
   retired. **Open: whether a retired row keeps its text.** Keeping it makes the table grow
   forever; dropping it makes an old citation resolve to a blank. Leaning to keeping the row with
   the text struck, on the grounds that a citation resolving to *"this was a requirement and no
   longer is"* is strictly more useful than one resolving to nothing.
3. **Does an application-tier or SDK-tier document use this scheme?** They have conformance
   surfaces too, and the same argument applies. Deliberately out of scope here until the extension
   tier has run with it.
