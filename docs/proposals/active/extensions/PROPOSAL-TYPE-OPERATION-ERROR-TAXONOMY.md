# PROPOSAL — `EXTENSION-TYPE` declares eight operations and zero error codes, so the one seat that built them minted its own

**Status:** DRAFT
**Target:** `specs/extensions/EXTENSION-TYPE.md` — new §8.5 (operation error codes).
**Filed by:** arch, self-found while sweeping `ENTITY-CORE-PROTOCOL` §4.7's `MUST NOT mint a synonym`
clause by subject (`PROPOSAL-GENERIC-400-SYNONYMS-AND-THE-PRE-ESTABLISHMENT-EXECUTE` §5).
**Read at:** arch `60da9ef` · core-rust `1ded022` · core-go `4936cf2` · core-py `4c7a6bc`

---

## §1 The gap, measured

`EXTENSION-TYPE` declares the request/result types for validation, comparison, compatibility,
reconciliation, convergence and adoption (§7, §8.3–§8.4). Grepped the whole document for
`invalid_request`, `invalid_params`, `bad_request`, `Errors:` and `error code`: **zero hits.** The
document specifies what each operation *returns on success* and says nothing about how any of them
refuses.

**One seat has built them.** `entity-core-rust` emits `bad_request` at **8 sites** —
`extensions/type-system/src/validate.rs` (6) and `constraint.rs` (2) — for "decode params",
"entity.type missing", and "type_a and type_b required". `entity-core-go` and `entity-core-py` have not
built these handlers, searched at the commits above.

**So there is no measured divergence yet, only a latent one** — and `bad_request` is already forbidden
by `ENTITY-CORE-PROTOCOL` §4.7 independent of this proposal. rust's rename is forced today; **this
document is about the eight codes nobody has had to pick yet.**

## §2 The distinction the taxonomy turns on, and it is already in the spec

`system/type/validate-result` carries `valid: bool` + `violations` + `unevaluated_fields`. **A type
validation failure is therefore a `200` with `valid: false`, not an error code.** The same holds for
`compare-result` (`compatibility-report`) and `reconcile-result`: the *analysis outcome* is the payload.

Error codes on these operations are therefore reserved for **structural defects of the request itself** —
the operation could not be performed at all. That is a much smaller set than it first looks, and it is
why every one of rust's 8 sites is a params defect rather than a typing verdict. **This is the load-
bearing sentence of the proposal**: a peer that returns `400` for `valid: false` has made the operation
useless to a caller doing schema exploration, and nothing in the current text stops it.

## §3 Proposed §8.5 — the codes, all from `ENTITY-CORE-PROTOCOL` §3.3's sanctioned set

No new codes are minted. Every row is `invalid_request`, `invalid_params` or `not_found`, which §3.3
already declares.

| Failure | Code | Status |
|---|---|---|
| Params undecodable as the declared request type | `invalid_request` | 400 |
| A required params field absent (`entity`; `type_a`/`type_b` for the §7 pairwise ops) | `invalid_params` | 400 |
| `entity.type` absent **and** `type_path` absent — no type to validate against | `invalid_params` | 400 |
| `type_path` (or `type_a`/`type_b`) names a type not present in the type system | `not_found` | 404 |
| The entity fails validation | **none — `200` with `valid: false`** | 200 |
| Type comparison finds the pair incompatible | **none — `200` with the report** | 200 |

**The 404 row is the one worth arguing about and it is deliberate.** A named type that does not resolve
is a lookup miss, not a malformed request — the caller's params were well-formed and it asked about
something that is not there. Collapsing it into `invalid_params` would make "you spelled the field
wrong" and "that type does not exist" indistinguishable, which is exactly the remedy-selection failure
§4.7 cites as its reason for splitting `connection_sequence_error` from `invalid_request` at 0.8.2.4.

## §4 Why this is DRAFT and not folded

The forced half — **stop emitting `bad_request`** — needs no proposal: §4.7 binds it already, and it is
routed to rust as a correction rather than an assignment.

The taxonomy needs **one build** before it lands. The failure set above is derived from the declared
request types, and a derived failure set is a claim about what these operations *can* fail on — the
kind of claim that is checked in a tree, not in a document (**L8**'s fifteenth form: our own spec text
is not evidence about an implementation). rust has the only implementation and is the seat that will
find a seventh failure mode if there is one. **Routed to rust alone until built or confirmed there**
(**L15**), then folded.

**What is NOT waiting on that build:** the §2 distinction. If the cohort builds `valid: false → 400`
in the meantime it is a cross-peer-observable divergence on the primary success path, so §2 is routed
now as the thing to get right, not later.

## §5 Delta table

| # | Document | Section | Change |
|---|---|---|---|
| **D1** | `specs/extensions/EXTENSION-TYPE.md` | new §8.5 | The §3 table, plus the §2 outcome-is-not-an-error rule as a `[MUST]`. |
| **D2** | `specs/extensions/EXTENSION-TYPE.md` | header | Version bump on fold. |

## §6 Open items

1. **Is the §3 failure set complete?** Owner `entity-core-rust`, resolved by building against it.
2. **Do the §7 analysis ops (`converge`, `adopt`, `reconcile`) refuse on anything the table misses?**
   Same owner, same build. `adopt-request` in particular mutates the type system and may need an
   authorization row, which the table deliberately does not guess at.
