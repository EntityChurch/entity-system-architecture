# PROPOSAL — §2.2a's `resource` column is three-valued, and §11 already computes which

**Status:** DRAFT (2026-09-16)
**Target:** `specs/extensions/EXTENSION-TREE.md` §2.2a — replace the two-valued `resource` column with the
three-valued one the operation set actually has; cite §11's `map_operation` as the mechanical test. **v4.11 → v4.12.**
**Provenance:** reported independently by three implementations within a day of the v4.11 fold, each from
building against it rather than from reading it.
**Scope:** **corrective, and it removes an obligation rather than adding one.** No implementation had adopted
the rows being withdrawn. No core-protocol text moves, and no document outside `EXTENSION-TREE` changes.

---

## 0. The defect, stated once

§2.2a (v4.11, landed 2026-09-14) declares a `resource` requirement for all eight `system/tree` operations.
**Three of its eight rows — `diff`, `create`, `destroy` — say `required`, and are contradicted by four
normative homes in the same document.**

| row | §2.2a says | the document says |
|---|---|---|
| `diff` (§4.2) | `required` | §4.2, verbatim: *"Diff operates on stored snapshots — no path-level authorization. **The `resource` field is optional**; when omitted, handler-scope authorization (§11) suffices."* |
| `create` (§7.2) | `required` | params are `system/tree/config` — a `tree_id`. No path subject appears in the operation |
| `destroy` (§7.3) | `required` | params are `primitive/string` — a `tree_id`. Same |
| all three | `required` | **§11's table:** `diff` — *"Operates on stored snapshots; no path-level check"*; `create`/`destroy` — *"Handler scope only"* |
| all three | `required` | ⛔ **§11's normative pseudocode, which names all three, twice:** `map_operation` returns `null` for them, and `tree_handler_path_permission` reads *"`if base_permission is null: return ALLOW ; No path-level check needed (diff, create, destroy)"*` |

`400 path_required` instructs a caller to **supply a resource**. For these three there is no resource to
supply. As landed, a conformant peer MUST refuse `diff(base, target)` — **the only call shape §4.2
documents** — and the `snapshot`→`diff` composition breaks at both ground-up seats.

**Nobody implemented it.** Every implementation that read the contradiction held the narrower reading, said
so at the code with both citations, and reported it rather than picking. That is the correct disposition for a
table whose rows contradict the sections they cite, and it is why this correction costs nothing to adopt.

---

## 1. The finding is not "three rows are wrong." It is that the column cannot express the set.

`ENTITY-CORE-PROTOCOL` §3.3 carries the test, and it is a **three-way** test:

> *"**Which** operations require one is stated by each operation's own specification … and an operation that
> **targets no entity binding** carries no requirement: `system/quorum:verify` takes both operands in `params`
> and binds nothing, so a `resource` would have nothing to name. (The test is whether the operation targets a
> binding, not whether it is configuration-shaped: an operation that writes or reads at a path derived from
> `EXECUTE.resource.targets[0]` requires a resource however administrative it looks.)"*

So an operation is in exactly one of three states:

1. **requires** a `resource` — the path is the subject (`put`, `merge`)
2. **resource-optional** — the `resource` selects or narrows, and §3.3 (0.8.2.25) obliges a **BROAD-RESULT /
   OPTIONAL-FILTER** declaration (`get`, `snapshot`, `extract`)
3. **targets no entity binding** — §3.3's carve-out. The `resource` names nothing for this operation, so
   neither the requirement nor the declaration obligation arises (`diff`, `create`, `destroy`)

**v4.11's column had two values and forced state 3 into state 1.** Writing `optional` instead would be the
opposite error: it would drag `diff`/`create`/`destroy` into §3.3's declaration obligation and demand a
BROAD-or-FILTER answer for a field that selects nothing — and there is no honest answer, because the result of
`diff(base, target)` is identical whether a `resource` is present or not.

⭐ **The cheapest reading of §4.2 is the one that misleads.** *"The `resource` field is optional"* is a sentence
about **authorization** — it says handler scope suffices — and it reads as a sentence about **subject
selection**, which is the axis §2.2a is classifying. Those are different questions and the same word answers
both. That is the whole mechanism of this defect.

---

## 2. The fix

### D1 — §2.2a's table, three-valued

| Operation | `resource` | Absent-case answer | `targets:[P] exclude:[P]` |
|---|---|---|---|
| `get` (§2.2) | optional | **BROAD** — the root listing | **400 `path_required`** |
| `snapshot` (§3.2) | optional | **BROAD** — prefix `""`, the **whole tree** | **400 `path_required`** |
| `extract` (§6) | optional | **BROAD** — an envelope of **every bound entity** under the prefix | **400 `path_required`** |
| `put` (§2.2) | **required** | — | 400 `path_required` (both empties, §3.3 unchanged) |
| `merge` (§5.2) | **required** | — | 400 `path_required` (both empties) |
| `diff` (§4.2) | **no path subject** | — | not read; proceeds on handler scope |
| `create` (§7.2) | **no path subject** | — | not read; proceeds on handler scope |
| `destroy` (§7.3) | **no path subject** | — | not read; proceeds on handler scope |

### D2 — the sentence that makes the third value mechanical rather than editorial

> **`no path subject` is `ENTITY-CORE-PROTOCOL` §3.3's carve-out — the operation targets no entity binding, so
> a `resource` would have nothing to name.** Neither the `path_required` requirement nor §3.3's
> BROAD/OPTIONAL-FILTER declaration obligation arises for these rows. **The test is not editorial: §11's
> `map_operation` already computes it** — an operation for which `map_operation` returns `null` has no path
> subject, and §11's `tree_handler_path_permission` returns `ALLOW` on exactly that branch. The two statements
> are one fact and §11's block is the authority. **The handler MUST NOT refuse one of these three on
> `resource` grounds `[MUST]`.** Dispatch-level `check_permission` (`ENTITY-CORE-PROTOCOL` §5.2) is unchanged
> and unaffected — whatever it does with a `resource` it was handed, it has already done before the handler
> runs.

### D3 — `merge` is re-ordered above `diff`, and `put`/`merge` stay `required`

Not contested by any seat. §5.2's EXECUTE line carries `resource: {targets: [...]}`; `put` writes a binding at
a path. Both target a binding and both keep `required`. The row order now groups the three states.

---

## 3. What this does NOT decide, named so nobody re-derives it

- **Whether §3.3's two-empties rule reaches a no-subject operation at all.** §3.3's rule is written for an
  operation whose result depends on `resource`; for a no-subject operation there is no absent-case behaviour to
  withhold, so the rule has no work to do. **This proposal states the disposition for these three operations
  and does not rule the general question**, which travels with the per-operation declaration denominator:
  `grep -l 'BROAD' specs/extensions/*.md` → **1 file of 26**, and `CORE-RESOURCE-TWO-EMPTIES-1` arm (c)
  (OPTIONAL-FILTER) has **no subject anywhere in the corpus**, so that arm is undrivable rather than merely
  unimplemented.
- **Anything about the BROAD rows.** They are correct, they are implemented, and they have been driven green
  across a socket. §2.2a's value is the BROAD column and it is untouched.

---

## 4. Why it happened — a pseudocode block is a normative home, and this one already computed the answer

A code block in a normative section states its rule in a form that a search for the rule's *words* cannot
find, which is why sweeping them before a fold is a standing step here. §11's `map_operation` is such a block.
It decides this exact question, by name, for these exact three operations — and the v4.11 sweep did not open
it.

⛔ **This instance is one step sharper than that, and it is the part worth carrying.** The usual failure is a
block that *states* a rule a prose sweep misses. Here the block **already computes the classification the new
table was introducing** — so §2.2a was never a missing declaration at all. It was a **second, divergent copy
of a value §11 derives.** An unmarked restatement of a canonical result is invisible from the authority side,
and when the authority is a code block it is invisible from both.

⇒ **D2 is written to close the class rather than the instance:** the table now **cites `map_operation` as the
authority** instead of restating its result, so the next edit to §11 cannot leave §2.2a behind.

**A second cheap tell, worth recording:** §2.2a was added to satisfy a core obligation (declare BROAD or
FILTER), and the declaration it owed was for the **three optional** rows. The five other rows were filled in to
complete the table — **the defect is entirely in the cells nobody needed to write.** A table completed for
symmetry asserts as loudly as one completed for a reason.

---

## 5. Conformance

No new requirement row. **The BROAD half is driven from core** — `CORE-RESOURCE-TWO-EMPTIES-1`, three arms,
exercised across a socket by three independent implementations.

⚠ **The `required` column was driven by nothing, and that is why a contradiction against four homes survived a
fold.** `EXTENSION-TREE` §12 is a legacy bullet inventory carrying no stable `<PREFIX>-R<n>` ids, so there was
no row to have failed. **Not fixed here** — giving this document an addressable conformance inventory is its
own work, tracked separately. Recorded so the absence is a known one rather than an assumed clean.

---

## 6. Adoption

`diff`/`create`/`destroy` behaviour in every implementation that reported this is **already what D1 says**.
There is nothing to build; this is a withdrawal notice, and adopting it means dropping a hold rather than
changing code.
