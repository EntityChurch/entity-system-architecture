# PROPOSAL — one matcher, two input domains: `name_constraints` never sees a `/`

**Status:** IMPLEMENTED — folded at `EXTENSION-REGISTRY` **v1.16**. D1–D4 verified against the spec 2026-09-06: row 3 is now the old row 4 (`a[b`) and the `/`-crossing absence is normative; *"One matcher, two input domains `[MUST, v1.16]`"* is §6a.9.1's paragraph.
**Tier:** extensions — `EXTENSION-REGISTRY` §6a.9.1, §11.1
**Answers:** `entity-core-go` spec-issue `2026-08-19-b` (GAP 1 of their consolidated packet)
**Read at:** arch `84cf677` · go `921accc` · rust `39082d8` · py `7a71fd4`

---

## §0 Summary

**`REG-NAME-CONSTRAINTS-GRAMMAR-1` row 3 is unsatisfiable on the wire, and it is my defect from
yesterday.** The row reads:

> 3. `name_constraints: "x*z"` → **`x/y/z`** admitted — `*` crosses `/`, the row that fails against
>    every path-glob.

**No conformant registry can admit `x/y/z`.** §6a's name-path safety is normative and categorical:

> **Name-path safety (normative):** identical to §6.3 — no `/`, no C0/DEL control chars, NFC at
> issue time.

So a `register-request` for `x/y/z` is refused `400 bind_invalid_name` **before** the policy's
`name_constraints` is ever consulted. The row asserts an admission that the section three
subsections up forbids. **A vector that no conformant implementation can pass is worse than a
missing vector**: it reports a conformant peer as failing, and the only way to "pass" it is to
delete the name-path check that exists for path-injection safety.

**Verified in all three engines — every one enforces §6.3/§6a name-path safety before admission:**

| Seat | Site | Commit |
|---|---|---|
| `entity-core-rust` | `extensions/registry/src/registration.rs` → `bind_invalid_name` on the **register-request** path | `39082d8` |
| `entity-core-go` | `ext/registry/…` `normalizeName` step 1; `RegistryErrBindInvalidName` (`core/types/registry_ext.go`) | `921accc` |
| `entity-core-py` | `entity_handlers/registry.py` → `_error(400, "bind_invalid_name", …)` | `7a71fd4` |

---

## §1 The real rule: one matcher, two input domains

v1.15 ruled **one name matcher per registry**, and that ruling stands — it is about the *grammar*,
and the grammar is one. What v1.15 did not state, and what produced this defect, is that the one
matcher is applied at **two sites whose input domains are not the same**:

| Site | Input | Can the input contain `/`? |
|---|---|---|
| `name_format_dispatch[].pattern` (§4) | the **raw argument to `meta_resolve(name)`** — an arbitrary string arriving from a URL, a link, a user paste | **Yes.** Nothing normalizes or rejects it before §4.1 step 2; dispatch runs *before* any backend is consulted |
| `name_constraints` (§6a.9.1) | a name that has **already passed §6a's name-path safety** at issue time | **No.** `/` is refused `400` upstream |

**So `*`-crosses-`/` is a real and testable property of the matcher — but it is observable only
through dispatch.** Through `name_constraints` it is unreachable by construction.

**And it is already tested, in the right place.** `REG-DISPATCH-GRAMMAR-1` (§4) carries the
identical row against the domain that admits it:

> pattern `x*z` matches `x/y/z` — **`*` crosses `/`**, which is the row that fails against every
> path-glob implementation.

Row 3 of `REG-NAME-CONSTRAINTS-GRAMMAR-1` is therefore **both unsatisfiable and redundant**. The
property is proven once, where it can be.

### §1.1 How it got written, because the shape is L12's

The v1.15 ruling built `REG-NAME-CONSTRAINTS-GRAMMAR-1` by **transplanting the discriminating rows
from `REG-DISPATCH-GRAMMAR-1`** — correctly for rows 1, 2 and 4 (`a?c`, `a[bc]d`, `a[b`, all of
which are legal names) and incorrectly for row 3, the only row whose input is illegal at the
second site. The matcher was verified; **the path by which the input reaches the matcher was
not.**

That is exactly **L12** — *a ruling names a mechanism; check that the actor can reach its input* —
firing on arch's own text one day after L12 was written, and in the proposal that unified the
matcher. **Unifying two matchers does not unify their callers**, and the input domain is a property
of the caller, not of the grammar.

---

## §2 Deltas

| # | File | Section | Change |
|---|---|---|---|
| D1 | `specs/extensions/EXTENSION-REGISTRY.md` | §11.1 `REG-NAME-CONSTRAINTS-GRAMMAR-1` | **Delete row 3** (`x*z` / `x/y/z`); renumber old row 4 → row 3 |
| D2 | `specs/extensions/EXTENSION-REGISTRY.md` | §6a.9.1 (`name_constraints` block) | Add the **input-domain** paragraph: one matcher, two input domains; `name_constraints` is applied **after** §6a name-path safety, so no legal input contains `/`, and `*`-crosses-`/` is asserted only by `REG-DISPATCH-GRAMMAR-1` |
| D3 | `specs/extensions/EXTENSION-REGISTRY.md` | §6a.9.1 | Cross-reference §6a's name-path safety by name from the `name_constraints` block, so a reader arriving at the admission gate sees the legality rule that runs before it |
| D4 | `specs/extensions/EXTENSION-REGISTRY.md` | header | `1.15` → `1.16` |

**Not done, deliberately:** go asked for a **name-legality table** (*backend × legal character-set ×
normalization*). Declined as unearned — the rule is already stated once in §6.3 and pointed at from
§6a (*"identical to §6.3"*), which is one canonical home plus a cross-reference, the shape this
corpus wants. What was missing was **not a third statement of the rule** but a pointer from the
*admission gate* to it, which is D3. Adding a table would create a second normative home for a rule
that already has one.

---

## §3 Cohort impact

| Seat | Owed |
|---|---|
| `entity-core-go` | **Filing seat.** Nothing to build — go is already conformant (it refuses `x/y/z`). Row 3 was the thing that could not be satisfied; it is gone. Re-check any fixture asserting row 3 |
| `entity-core-rust` | Same — conformant, drop the row from any local vector table |
| `entity-core-py` | Same |
| app tier | None — `name_constraints` is an issuer-side gate; no app-tier seat runs a registry issuer |

**No implementation changes behavior.** This deletes an unpassable assertion and states why.
