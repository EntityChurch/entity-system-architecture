# PROPOSAL — one name matcher per registry, and a `<glob>` in a schema block is an undefined referent

**Status:** IMPLEMENTED — folded at `EXTENSION-REGISTRY` **v1.16**. D1–D3 verified 2026-09-06: *"the registry's name matcher"* is §4's sentence, the `500`-arm removal is row 3 of `REG-NAME-CONSTRAINTS-GRAMMAR-1`, and that check carries the discriminating rows a `*.lab` example cannot reach.
**Tier:** extensions — `EXTENSION-REGISTRY` §4, §6a.9.1, §11.1
**Answers:** `entity-core-go` spec-issue `2026-08-18-e` · `entity-core-py` `SA-PY-14` — **filed independently, same finding, and the two seats already diverge in code**
**Read at:** arch `a587ce0` · go `6ae71c4` · py `23e77ff`

---

## §0 Summary

`EXTENSION-REGISTRY` defines **two** glob fields over the **same domain** — the user-facing name
string, in the same handler, in the same registry:

| Field | Section | Grammar |
|---|---|---|
| `name_format_dispatch[].pattern` | §4 | **closed at v1.13** — `*` only, every other byte literal, `*` crosses `/`, no pattern invalid |
| `name_constraints` | §6a.9.1 | `<glob \| null>`, example `*.lab`, **defined nowhere** |

**And v1.13's own scoping sentence reads the second field out of the ruling that landed beside
it:** *"it is therefore a **registry-local matcher**, scoped to this field."*

**The two seats that implement it have already split**, which is what makes this a ruling rather
than a cleanup:

| | `name_format_dispatch` | `name_constraints` |
|---|---|---|
| `entity-core-go` (`6ae71c4`) | `matchDispatchName` (ruled) | **`path.Match`** — `?`/`[…]` are wildcards, `*` stops at `/`, and a malformed pattern is **`500 internal_error`** |
| `entity-core-py` (`23e77ff`) | `name_glob_match` (ruled) | **`name_glob_match`** — interim, reported not converged |

**Ruled: `name_constraints` uses §4's grammar. There is one name matcher per registry.**

---

## §1 Why one matcher, not two

**Same domain, same string, same handler.** Both fields match a user-facing name — a flat string
with no segment structure and no peer-id head. Nothing distinguishes the domains, so nothing
justifies distinguishing the matchers. Two matchers for one job is the silently-diverging shape this
corpus has now met three times, and it is the shape §4's own MUST NOT-delegate rule exists to
prevent.

**`*.lab` — the spec's own example — is grammar-identical under both candidate readings**, which is
precisely why the divergence survived review: the one example given cannot discriminate.

**It is an admission gate, not a routing hint.** `name_constraints` decides whether a binding is
**issued at all** — `403 not_entitled` versus a signed, published binding. Two registries running
the *same operator policy* would admit different names. That is cross-impl-observable on the
strongest surface this extension has.

**The `500` arm is the sharp end and it disappears.** `path.Match` returns `ErrBadPattern`, so
`name_constraints: "a[b"` makes **every** register attempt against that policy return `500` — a
policy the operator installed, silently un-registerable, with no diagnostic naming the pattern.
Under §4's grammar **no pattern is invalid**, so the arm is unreachable and the failure mode is
gone rather than handled.

**Capability loss, stated rather than discovered.** `?` and character classes stop being expressible
in `name_constraints`. Both seats reported this and neither asked for it back: they are expressible
under no other reading this corpus sanctions, and an issuer policy needing richer matching layers it
in the issuer, exactly as §4 says a deployment needing richer dispatch layers it in the backend.

---

## §2 The class this is the third instance of

`entity-core-py` names it, and it is worth adopting verbatim: **a `<glob>` in a schema block is an
undefined referent unless a grammar is cited at that field.**

Three instances in two days, all found by implementers rather than by review:

| Instance | Field | Found by |
|---|---|---|
| `EXTENSION-REVISION` §2.4 | `glob_match` undefined; `exclude_types` matched two ways | py `SA-PY-11` |
| `EXTENSION-REVISION` §2.3 | the merge-config call site | py `SA-PY-12` |
| `EXTENSION-REGISTRY` §6a.9.1 | `name_constraints: <glob>` | go `2026-08-18-e`, py `SA-PY-14` |

**This is the D10 class** (*a normative construct may not name a referent the corpus does not
define*) **in its quietest form** — the same quiet form as the `pinned`/`did-key` tokens: a schema
block reads as data, so a reviewer checks that the field exists and an implementer has to invent
what it means. It fails nothing until two implementers invent differently.

**Mechanically checkable, and filed as a gate ask:** `spec` can flag a schema-block type annotation
naming a matcher/grammar (`<glob>`, `<pattern>`, `<regex>`) with no grammar citation at the field.
**Filed, not built** — recorded so it is not mistaken for enforced.

---

## §3 Spec deltas (executable form)

| # | File | Section | Edit |
|---|---|---|---|
| **D1** | `EXTENSION-REGISTRY.md` | §4 pattern grammar | *"scoped to this field"* → **the registry's name matcher**, naming both fields that use it. This sentence is the defect: it was written to fence the matcher off from `ENTITY-CORE-PROTOCOL` §5.4 and it fenced off §6a.9.1 as collateral. |
| **D2** | `EXTENSION-REGISTRY.md` | §6a.9.1 schema | `name_constraints: <glob \| null>` → cite §4's grammar at the field. State the `500`-arm removal: no pattern is invalid, so a registry MUST NOT refuse a policy for a malformed `name_constraints` and MUST NOT fail a registration on one. |
| **D3** | `EXTENSION-REGISTRY.md` | §11.1 | **`REG-NAME-CONSTRAINTS-GRAMMAR-1`** — the discriminating rows a `*.lab` example cannot reach. |

**Not in scope:** the §4 grammar itself (settled v1.13) · richer matching in either field · any other
`<glob>` site outside this spec (§2 routes those to their own owners).

---

## §4 Conformance — `REG-NAME-CONSTRAINTS-GRAMMAR-1`

Against an issuer policy in a live mode, four rows, each discriminating against `path.Match` /
`fnmatch`:

1. `name_constraints: "a?c"` — a register-request for name **`a?c`** is admitted; one for **`abc`**
   is refused `403 not_entitled`. *(Inverted under both stdlib matchers.)*
2. `name_constraints: "a[bc]d"` — **`a[bc]d`** admitted, **`abd`** refused.
3. `name_constraints: "x*z"` — **`x/y/z`** admitted. *(Fails against every path-glob: `*` stops at
   `/`.)*
4. `name_constraints: "a[b"` — the policy is **accepted** by `set-issuer-policy`, and a register for
   the literal name `a[b` is **admitted**; no `500` on any path. *(This row is the one that fails
   against a `path.Match` implementation by **crashing** rather than by answering wrongly, which is
   why it is asserted separately from row 2.)*

**Control:** `name_constraints: "*.lab"` — `alice.lab` admitted, `alice.dev` refused. The example
the spec already carries, which passes under **both** readings and therefore proves nothing on its
own; it is included so a failure of rows 1–4 cannot be mistaken for the constraint being ignored
entirely.

---

## §5 Cohort impact

| Seat | Owed |
|---|---|
| **`entity-core-go`** | Point `name_constraints` at `matchDispatchName` (`ext/registry/peerissued/register.go`, `applyAdmission` — the divergence is already named in a comment there). **Delete the `500 internal_error` arm**; it is unreachable under the ruled grammar. Your `2026-08-18-e` is **confirmed on its "most likely" reading**, and routing rather than converging was the right call. |
| **`entity-core-py`** | **Your interim is the ruling** — `name_glob_match` at `name_constraints` stands as-is. `SA-PY-14` closes. The class you named in it (§2) is adopted into the proposal record. |
| **`entity-core-rust`** | Check your `name_constraints` site against §4's matcher; neither filing seat measured it, so **no claim is made here about which matcher you use** — it was not read. |
| **all three** | `REG-NAME-CONSTRAINTS-GRAMMAR-1` (§4). Row 4 is the one that will crash rather than fail a `path.Match` implementation. |

---

## §6 What this proposal does NOT claim

- **Not** that either filing seat implemented anything wrongly. go's `path.Match` predates the v1.13
  ruling and was the only reading the text supported; py's interim is the reading being ratified.
- **Not** a claim about `entity-core-rust`'s `name_constraints` site — not read, not measured.
- **Not** a ruling on the other two `<glob>` instances in §2's table; those are `EXTENSION-REVISION`
  sites with their own filings.
