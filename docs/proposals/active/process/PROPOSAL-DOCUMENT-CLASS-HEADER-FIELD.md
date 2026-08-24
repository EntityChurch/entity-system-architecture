# PROPOSAL — a document cannot say what kind of document it is, and three analyzers guessed

**Status:** DRAFT — filed 2026-08-17.
**Target:** `specs/SPECIFICATION-FORMAT.md` §5.3 (header fields) + §10 (relationship to architectural
documents), with a consuming change in `entity-system-arch-tools` `config.default.toml`.
**Class of change:** an authoring-standard addition. No wire surface, no extension, no core touch.
**Read at:** this corpus `04c7235` · arch-tools `f629145`.

---

## §1 The defect, and the three times it fired

`specs/` holds three kinds of document. §10 of `SPECIFICATION-FORMAT` already knows this — it draws
the normative-spec vs architectural-document line in a five-row table, and §11.3 leans on that line to
decide citation legitimacy (*"another normative spec"* · *"an architectural / guide document"* ·
*"anything else"*).

**What no document has is a way to say which one it is.** §5.3's header table is `Version` · `Status` ·
`Depends` · `Encoding`. There is no class field. So the distinction §10 defines and §11.3 depends on is
carried nowhere a reader or a tool can find it.

The consequences are measured, not hypothetical:

1. **Two files invented their own field.** `ARCHITECTURE-IDENTITY-INFRASTRUCTURE` and
   `SYSTEM-IDENTITY-COMPOSITION` both declare `**Authoritative scope:** Informative…`. Two documents,
   one need, no standard, so each authored a private convention. They are the only two in 40.

2. **The toolkit grew a class map with no normative anchor.** `config.default.toml` `[classes]` assigns
   a class by **filename glob** — `GUIDE-*.md` → guide, `ARCHITECTURE-*.md` → arch-doc, plus six
   hand-listed exceptions (`ENTITY-SYSTEM-REFERENCE.md`, `CHARTER.md`, `SPECIFICATION-FORMAT.md`,
   `STYLE-NAMING-CONVENTIONS.md`, `SYSTEM-ARCHITECTURE.md`, `SYSTEM-IDENTITY-COMPOSITION.md`). Every
   one of those six is a document whose *name* does not reveal its *kind*, so the map had to be told.
   **A new document that matches no glob is silently classed `canonical-spec`** — the default is a
   guess, and it is the guess that fails closed in the wrong direction.

3. **Two analyzers then disagreed about the same corpus.** `spec coverage` did not read the class map
   at all; it grew a second, independent detector over the invented `Authoritative scope:` field. Its
   first run reported **five specs with no guide and three with no design record** — a list containing
   a rulebook, a condensed working reference and a domain charter, none of which is a spec, each
   already classed non-spec in the map the other analyzer was reading. Corrected at arch-tools
   `f629145`; the worklist fell to two and one.

**The pattern across all three is one thing:** a fact the corpus depends on lives only in a tool's
config, keyed by a filename pattern, and is re-derived — differently — by whoever needs it next.

## §2 What this does NOT claim

**Not that `SPECIFICATION-FORMAT` defines no document classes.** It was recorded that way in a prior
handoff and in the `coverage` docstring, and it is wrong: §11.3 defines three citation-target kinds and
§11.4 their dispositions, and §10 tabulates the spec/arch-doc split. **The concept is landed.** What is
missing is narrower and purely mechanical — **a declaration site**. This proposal adds one; it does not
introduce the taxonomy, and it must not be read as doing so.

## §3 The change

### §3.1 A `Class` header field (`SPECIFICATION-FORMAT` §5.3)

Add one row to the §5.3 table:

| Field | Required | Description |
|-------|----------|-------------|
| Class | Yes | `canonical-spec`, `guide`, or `arch-doc` — what kind of document this is (§10, §11.3) |

And a short subsection defining the vocabulary against §10's existing table:

- **`canonical-spec`** — binding normative text. Answers *what must an implementation do?* Versioned;
  breaking changes tracked. Held to the full authoring standard, and to §11.3's citation restriction.
- **`guide`** — a rulebook or how-to. Teaches a rule or a practice rather than stating a wire
  obligation. `SPECIFICATION-FORMAT`, `STYLE-NAMING-CONVENTIONS`, and everything in `guides/`.
- **`arch-doc`** — synthesis, navigation, or orientation. Answers *why does the system work this way?*
  or *where do I find things?* Informational per §10's Authority row. `ARCHITECTURE-*`,
  `ENTITY-SYSTEM-REFERENCE`, a domain `CHARTER`.

**`Class` is REQUIRED, and that is the load-bearing half.** An optional field that a new document may
omit leaves the tool defaulting to `canonical-spec` on silence, which is the exact failure this
proposal exists to remove. A missing `Class` must be a finding, not a default.

The existing `Status` field is unchanged and orthogonal — `Class` says what kind of document it is,
`Status` says where it is in its lifecycle. A `guide` can be Draft; a `canonical-spec` can be
Superseded.

### §3.2 Retire the invented field

`ARCHITECTURE-IDENTITY-INFRASTRUCTURE` and `SYSTEM-IDENTITY-COMPOSITION` replace
`**Authoritative scope:** Informative…` with `**Class**: arch-doc`. Their prose scope note may stay as
prose; it is the *field* that was private convention.

### §3.3 Tooling (arch-tools, consuming — not part of the spec change)

`config.doc_class()` reads the declared field first and falls back to the glob map, so the map becomes
a **compatibility shim for undeclared documents rather than the authority**. The six hand-listed
exceptions retire as their documents declare themselves. `coverage` and `address` already route through
`doc_class()` as of `f629145`, so both inherit the fix without further change.

## §4 Enforcement point

*A discipline with no enforcement point is theater*, so this proposal names one before it lands.

A new `standards` rule — **`header-class-missing`** — beside the existing `header-version-missing` and
`header-status-missing`, which are already **errors**. Same mechanism, same severity tier, same
baseline treatment: existing undeclared documents go into `.spec-baseline.json` as known debt so the
rule does not fail on contact, and anything new gates. `--update-baseline` only ever lowers, so the
backlog can only burn down.

That rule is the whole reason to prefer a header field over any other design. A convention nothing
checks decays to two files inventing `Authoritative scope:` — which is precisely the observed
starting state.

## §5 Why not the alternatives

| Alternative | Why not |
|---|---|
| **Directory placement decides class** | Already false: `specs/` holds all three kinds today, and moving them would break every citation in the corpus and every downstream consumer's links. §11.1 makes the file the citation host — relocating hosts to encode a class is the expensive way to store one word. |
| **Keep it in tool config only** | The status quo. Its failure mode is measured above: keyed by filename pattern, invisible to authors, re-derived differently by each analyzer, and silently defaulting for anything new. |
| **Infer from `Status: Informative`** | Overloads a lifecycle field with a kind field, and `Informative` is not in `canonical_status`. It also cannot express the guide/arch-doc split, which is the split §11.3 actually needs. |
| **Leave it; judge case by case** | This is what produced a worklist telling us to write guides for a rulebook and a charter. Case-by-case judgment is exactly what the class field encodes once, at the source, by the author who knows. |

## §6 Scope boundary

**This proposal touches an authoring standard, not the protocol.** No wire format, no entity type, no
operation, no capability. `SPECIFICATION-FORMAT` is classed `guide` by the very map under discussion,
and the change is additive: adding a header row invalidates no existing document's content, only its
completeness against the new rule — which is what the baseline is for.

**Out of scope, deliberately:** whether the three-value vocabulary is complete. `intent` exists as a
fourth class in the tool (proposals, explorations, reviews) but never appears in `specs/`, so no
document in scope would ever declare it. If the workspace layers later want a declared class, that is
its own proposal against its own evidence — adding a value nothing uses is how a vocabulary earns
members it cannot justify.

## §7 What ratification requires

1. Fold §3.1 into `SPECIFICATION-FORMAT` §5.3 + a §5.3a vocabulary subsection, cross-referenced from
   §10 and §11.3. Version bump on that file (it is a `guide`, and this is a real authoring rule).
2. Declare `Class` on all 40 documents in `specs/` (mechanical; the glob map supplies the starting
   assignment and the six exceptions are already adjudicated).
3. §3.2's two retirements.
4. `header-class-missing` added to `[analyzer.standards.rules]` as `error`, with the undeclared set
   captured into `.spec-baseline.json` in the same commit.
5. `config.doc_class()` reads the declaration first; glob map demoted to fallback.

Steps 1–3 are this repo. Steps 4–5 are `entity-system-arch-tools`, same session as the fold, per the
standing rule that a gate defect or gate addition is fixed in the tool repo with its own commit and its
own tests.
