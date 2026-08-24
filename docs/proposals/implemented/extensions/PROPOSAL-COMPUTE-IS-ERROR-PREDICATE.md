# PROPOSAL — define `is_error()`, the predicate §4.1 uses 39 times and never defines

**Status:** **RULED (B) — propagate. FOLDED at `EXTENSION-COMPUTE` v3.23.** Ruled in the pass after
the one that found it, with §5's open questions answered from the corpus first — see §5a.
**Routed to the cohort as `ROUTING-2026-08-16-a`.**
**Domain:** core / `system/compute` — `EXTENSION-COMPUTE` §4.1 (evaluator pseudocode), §2.3 N1,
§2.4, SA-1 (§424), the short-circuit MUST (§2094-region).
**Source of the finding:** `entity-core-go`, routed 2026-08-16 as
`UPDATE-FOR-ARCH-2026-08-16` §1 + spec-issue
`2026-08-15-e-materialized-error-in-construct-embed-vs-propagate.md`. **Read from go's own
documents** (L2), at go `71071c6` — a verified ancestor of their HEAD `3369f55`, which carries only
that update.
**Blocks:** `PROPOSAL-COMPUTE-ERROR-MATERIALIZATION-DETERMINISM` (§2.4), gated since 2026-07-23.

---

## 0. Why this is a separate proposal and not a section of the one it blocks

The blocked proposal asks *how a materialized `compute/error` is hashed* (code-only). **It
presupposes that materialization happens.** The divergence go measured shows that presupposition is
the open question — so answering it inside that proposal would settle the premise in a document
whose subject is the consequence.

More importantly, the defect is **not specific to `compute/construct`**. The undefined predicate
governs **every** short-circuit site in the evaluator. `AGENTS.md`'s equivalence-collapse meta-rule
applies verbatim: **state the invariant once, generally, and never patch instances.**

## 1. What go measured, and it is not what the routing framed

Go authored the gate vector `ROUTING-2026-08-15-e` §1 asked for and ran it three-way. **It does not
lock.** One `compute/error` literal as a `compute/construct` field value:

| Impl | Pin | Outcome |
|---|---|---|
| `entity-core-go` | `71071c6` | **embeds + materializes code-only** → entity boundary `ecf-sha256:8321beb0…` |
| `entity-core-rust` | `462f2c2` | **propagates** — error-kind outcome; the construct never materializes |
| `entity-core-py` | `f33526f` | **embeds, does not materialize** → `CBOREncodeTypeError`, handler crashes |

**Control:** a non-error construct produces byte-identical `4fc00038…` in go and rust, and
**330/331 compute-corpus vectors still lock three-way.** The divergence is specific to an error
value in a construct.

> **Build state re-taken (L4), because go's peer pins were stale.** rust `462f2c2`→**`55cc507`**,
> py `f33526f`→**`c55c631`**. **Neither moved its compute surface** — `git diff --name-only` across
> both ranges returns no compute path (signaling, continuation and the methodology overlay only).
> **Go's three-way reading stands at the current HEADs**, and this note is why that is an
> observation rather than an assumption.

## 2. The finding — all three implemented our spec, from different lines of one block

Go's framing is that §2.3 N1 *"describes a value placed into data; it does not name a `compute/error`
that arrives at a construct field by evaluation."* That is close, and the fork is **one level
further down.**

`EXTENSION-COMPUTE` §4.1, the `compute/construct` branch, contains **both behaviours, in sequence**:

```
value = evaluate(value_target, scope, budget, ctx)
if is_error(value): return value                                   ; ← propagate      (rust)
...
if kind_of(value) in {entity, compute/closure, compute/error}:
    result_fields[name] = ctx.content_store.put(materialize(value, ctx))   ; ← embed  (go, py)
```

**The guard makes `compute/error` in the materialization set unreachable — unless `is_error()` is
narrower than "kind is `compute/error`."** Which it is depends entirely on a predicate the spec
never defines.

**`is_error` is used 39 times in `EXTENSION-COMPUTE` and defined nowhere in it.** The corpus's only
definition is in **another extension** — `EXTENSION-CONTINUATION` §(`is_error = status >= 400`) — an
**HTTP-status** predicate with no bearing on compute values. An implementer who greps the corpus for
the definition finds the wrong one.

**And SA-1 is what creates the ambiguous category.** §424: *"Evaluating a value-type entity returns
it unchanged (SA-1)"* — explicitly including `compute/error`. So a `compute/error` literal
**evaluates successfully, to itself**. Is that value "an error" for short-circuit purposes?

- **Read 1 — `is_error(v) ≡ kind_of(v) == compute/error`.** SA-1's returned value trips the guard →
  **propagate (B)**. The `compute/error` entry in the materialization set is dead code.
- **Read 2 — `is_error(v) ≡ evaluation raised/failed`.** An SA-1 literal evaluated *successfully*, so
  it is not an error-in-flight → falls through → **embed + materialize (A)**, matching N1.

**Both readings are internally consistent with the landed text.** This is not two teams being wrong;
it is three teams implementing the same block and the block not deciding.

> **The defect class is `undefined-wire-referent`** — the family `coherence.py` exists for
> (`CONTINUATION` §3.6's `generate_internal_deliver_token`). **It is invisible to that gate by
> design:** the rule fires only where a helper's return value is assigned to a *field*, and
> `is_error(x)` flows into control flow. **The gate's own docstring already names this residue** —
> *"a cross-peer-observable value that is not a wire field is still a contract; nothing checks
> those."* This is that residue, cashed.

## 3. Recommendation — **(B) propagate**, and it is a recommendation, not a ruling

**Four landed statements point at propagate; one points at embed.**

1. **The short-circuit MUST** — *"Once a `compute/error` enters a value position, all expression
   types that consume values MUST short-circuit and propagate the error"* — and it **names
   `compute/construct` explicitly**, alongside arithmetic, compare, logic, field, apply and if.
2. **F10 / §1.5** — *"an evaluated `compute/error` is a value (errors propagate like NaN)."* The NaN
   model is the stated one, and **NaN propagates through an operation; it is not boxed into the
   operation's result.**
3. **§7.2's cascade model** — *"errors propagate through the dependency graph … the same model as
   NaN propagation in IEEE 754."*
4. **Order inside the block** — the guard is evaluated **before** the materialization branch.

Against: **§2.3 N1** names `compute/error` among the kinds referenced-by-hash at `compute/construct`
fields, and the materialization set repeats it.

**Under (B), N1's `compute/construct` mention for `compute/error` is wrong and must be corrected in
the same pass** — the remaining legitimate materialization sites are the `result_path` write (§7.2)
and `store` (SA-9), which is precisely what the blocked §2.4 proposal is about. That is a coherent
end state: **errors materialize where they are *written*, never where they are *consumed*.**

## 4. What tightening requires, if (B) — L6's "tighten §§3–9 in the same pass"

**Answering A-or-B alone would leave the defect.** The pass must:

1. **Define `is_error()` normatively in `EXTENSION-COMPUTE`**, once, adjacent to §4.1 — and state
   that it is **kind-based, not outcome-based**, so SA-1's returned value is an error for
   short-circuit purposes. **This is the actual pin.**
2. **Disambiguate from `EXTENSION-CONTINUATION`'s `is_error`** (status ≥ 400) — different extension,
   different meaning, same token. Either rename here or state the scoping explicitly.
3. **Remove `compute/error` from §4.1's construct materialization set**, or state the reachable case
   that keeps it there. Dead pseudocode is how this divergence happened.
4. **Correct §2.3 N1** to stop naming `compute/construct` fields as a `compute/error` placement
   site, and name the surviving sites (§7.2 `result_path`, SA-9 `store`).
5. **A worked example** nailing the error-in-construct-field case, since prose review did not catch
   this and will not catch the next one.

## 5. Open questions — not to be decided alone

1. **Is there a reachable case for a deliberately-embedded `compute/error`?** An error-handling
   combinator that takes an error as data would need one. If so, (B) needs an explicit escape and
   §2094's blanket MUST needs a carve-out. **Nothing in the corpus exercises it, and the compute
   corpus does not cover `result_path` error writes either.**
2. **Does the ruling reach `compute/apply` args and `compute/scope` bindings**, the other two N1
   sites? §2094 says apply short-circuits. If so, N1's error mention is wrong at all three sites,
   not one.
3. **What does the gate vector become?** Under (B) it expects rust's outcome, and go flips. Under
   (A) it expects go's. **The vector is not neutral and cannot ratify §2.4 until this lands.**

## 5a. The ruling, and how §5's open questions closed `[2026-08-16]`

**Ruled (B) — propagate.** `is_error(v)` is **kind-based**: true iff `kind_of(v)` is `compute/error`,
**not** "evaluation failed". SA-1 governs what `evaluate` *returns*; the predicate governs what the
consumer *does with it*, and the guard sits between. **The two are compatible, which is why both
readings survived review.**

**Q1 — is there a reachable case for a deliberately-embedded `compute/error`? NO, and the corpus
says so three ways.** There is **no error-handling combinator in `EXTENSION-COMPUTE`** — no
`catch`, `try`, `recover`, or error predicate exposed to expressions; an exhaustive named search
returns nothing. The short-circuit `[MUST]` says expressions short-circuit *"rather than attempting
to read its fields."* And §7.2 states where inspection **does** happen: the error written to
`result_path` *"is an entity like any other — content-addressed, storable, inspectable."*
**Errors are inspected by reading them from the tree, never by consuming them in an expression.**
So (B) needs **no escape hatch**, and the blanket MUST needs no carve-out.

**Q2 — does it reach `compute/apply` args and `compute/scope` bindings? YES, all three sites.** The
short-circuit MUST already names `compute/apply` in both modes; §4.1's `compute/let` branch carries
the same guard on every binding value. **A second, independent confirmation for scope:** the
`system/compute/scope-binding` schema is a two-variant union — `{kind: "entity"}` or
`{kind: "value"}` — and **has no error variant at all**, so an error was never representable there.
**N1 was therefore wrong at all three sites, not one**, and is corrected as a whole rather than
patched at the construct.

**Q3 — the gate vector is not neutral, and now resolves.** Under (B) it expects **rust's** outcome.
**Go flips from (A)**, which they pre-committed to. The vector can then ratify §2.4 — narrowed, per
§0, to the `result_path` / `store` sites.

### What landed (five edits, one pass — L6's tighten-in-the-same-pass)

1. **`is_error(v)` defined normatively** in §4.1, kind-based, with why-it-is-pinned and the
   cross-peer split it caused.
2. **Disambiguated from `EXTENSION-CONTINUATION`'s `is_error`** (`status >= 400`) in the definition
   itself — *"a reader grepping the corpus for one will find the other."*
3. **`compute/error` removed from §4.1's construct materialization set**, with the absence marked
   normative in the pseudocode comment, because unreachable text is what produced the divergence.
4. **§2.3 N1 corrected** — `compute/error` dropped from the placement list at all three sites, with
   the surviving materialization sites named.
5. **Worked example** at the short-circuit MUST, naming **both** wrong answers so they are
   recognizable, since prose review did not catch this one.

## 6. Independent of the ruling

**py's `CBOREncodeTypeError` is a defect under either answer** — an unmaterialized `Entity` must
never reach the CBOR encoder, and today the failure surfaces as a dropped connection rather than a
diagnostic. Go named this and it is not contingent on the pin.

## 7. Process record

- **Routed, not voted** — go cited `GUIDE-CONFORMANCE` §4 (all-three-differ ⇒ tighten the spec) and
  routed rather than adopting a majority. **This is L6 honored in a peer's tree**, and it is the
  first time arch has received the clean case.
- **L4b fired a third time in two days.** The item arrived framed as *"the case is under-pinned"*;
  **our own corpus carries an explicit MUST that names `compute/construct`.** The gap was real, but
  it was one level down from where the routing put it, and only opening our own spec found that.
- **L1 honored:** this is the proposal, written **before** any spec edit. No `specs/` file is touched
  by the commit that lands it.
