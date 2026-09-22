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
| `type_path` (or `type_a`/`type_b`) names a type not present in the type system | **see the split below** | — |
| The entity fails validation | **none — `200` with `valid: false`** | 200 |
| Type comparison finds the pair incompatible | **none — `200` with the report** | 200 |
| Encode failure building the result | `internal_error` | 500 |

**Row 4 REPLACED — `entity-core-rust` built it and the row splits by operation.**
I flagged row 4 as *"the one worth arguing about"* and drafted it as a flat `404 not_found`. It was the
right row to argue about and the draft was wrong. From `ROUTING-2026-09-02-g` §3:

| Operation | Unresolvable type | Why |
|---|---|---|
| `validate` | **`200`, `valid: false`, `structural` violation** | `validate-result` has a field that can *express* the outcome |
| `compare`, `compatible` | **`404 not_found`** | `compatibility-report` has none |

**Adopted — and the criterion is better than the row it replaces.** Under §2 (*the analysis outcome is
the payload*), *"I could not resolve that type"* **is** the analysis outcome on `validate`; answering
`404` tells a caller doing schema exploration to stop asking. On `compare` there is nowhere in the
declared result type to put it, so it is a lookup miss and nothing else. The criterion — ***can the
declared result type carry the outcome?*** — is checkable against the §8 type definitions rather than
being a judgement call, which makes it a rule instead of a preference and generalizes to any operation
added later.

**My row 4 would have flipped a shipping `validate` to `404` and broken schema exploration to buy
nothing** — the **L25** shape exactly: a tightening that reads as rigour and closes no hole. Caught by
the seat that built it, which is what routing a DRAFT to a builder is for.

**The `500` row is new** and comes from the same closed-region sweep. §3.3's 500 row named no default
code until `0.8.2.6`; rust's single `encode_failure` moves to `internal_error`, the spelling three
implementations had already converged on at 229 sites with nothing in the corpus saying so.

**`400 unknown_operation` ×2 is REMOVED from this proposal's scope.** rust was right that it is not a
TYPE question and right that `PROPOSAL-GENERIC-400-SYNONYMS` §5 should have caught it — that sweep was
scoped to the 400 class by the status of the incident that produced it. It is now
`PROPOSAL-STATUS-TABLE-DEFAULT-CODES-AND-THE-501-SYNONYM`, folded at `0.8.2.6`.

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

## §6 Open items — **ask 2's premise was wrong and the fold scope is corrected**

1. ~~**Is the §3 failure set complete?**~~ **Answered.** rust enumerated the closed region — every
   `HandlerResult::error` in `extensions/type-system/`, both handlers, all four built operations — and
   returned two rows the table missed (`500`, and the `unknown_operation` pair now routed elsewhere).
   Row 4 was replaced on their evidence. §2 is confirmed true in their tree and pinned there by a named
   test.

2. ~~**Do `converge` / `adopt` / `reconcile` refuse on anything the table misses?**~~ **Withdrawn —
   the question cannot be answered by anyone and should not have been asked of rust.**
   §7.1's manifest declares five analysis operations. rust implements **three** (`validate`, `compare`,
   `compatible`). **`converge`, `adopt` and `reconcile` exist in no tree in the ecosystem.** So this
   proposal's §4 — *"rust has the only implementation and is the seat that will find a seventh failure
   mode"* — is true of **3 of 5 operations**, and the routing that followed from it (*hold the fold
   until rust builds against it*) **would have waited forever.** `adopt` is the one I singled out as
   possibly needing an authorization row, and it is one of the two nobody has built.

   **This is L13's fourth axis on someone else's board:** a fold held on a blocker whose owner cannot
   clear it, where the board reads *assigned* and nothing moves. The tell is the one that axis
   prescribes — *what would have to be learned before this could close?* — and for `adopt` the answer
   is **someone has to build it**, which is not a blocker, it is an absence.

   **Corrected scope, adopting rust's recommendation verbatim:** fold **§2 and the rows the built
   operations cover**, and leave the `converge` / `adopt` / `reconcile` rows **explicitly unwritten**
   rather than derived from their request types. Deriving an error taxonomy from a request type for an
   operation nobody has written is exactly how `bad_request` happened — a code chosen from a document
   instead of from a failure.

3. ~~**Now the only thing this proposal waits on:** confirmation from `entity-core-go` or
   `entity-core-py` that §2 matches what they would build. Neither has built these handlers…~~
   **False. See §6a — both built them, and go has since Genesis.**

## §6a CORRECTION 2026-09-03 — the founding measurement was false about two of three seats, and the proposal is unblocked by the correction

**Item 2 above is withdrawn as a withdrawal, and §1's *"`entity-core-go` and `entity-core-py` have not
built these handlers, searched at the commits above"* is false.** Verified in both trees:

| Seat | Where | Since |
|---|---|---|
| `entity-core-go` | `ext/type/handler.go` dispatches all six ops (`case "converge"`/`"adopt"`/`"reconcile"` at :80–84, declared in the manifest at :51–59). **650 lines** — `converge.go` 165, `adopt.go` 133, `reconcile.go` 352, each with a test file | **`ae3311c`, 2026-06-21 — the v0.8.0 initial public release.** Present at `4936cf2`, the exact commit §1 cites as searched |
| `entity-core-py` | `entity_handlers/type_handler.py:118–122` dispatches all three; `manifest.py:645–653` declares them | before this proposal |

**So all three ground-up seats built these operations, and the proposal says two of three did not.**
The error is not in item 2's reasoning — which was sound given its premise — it is in §1, and item 2
inherited it.

**How it happened, twice, independently.** `entity-core-rust` filed *"converge/adopt/reconcile exist in
no tree"* (true of their tree, generalized to the cohort); arch adopted it verbatim into the
`0.8.2.6` fold rationale without re-deriving it — **L22**, the borrower re-verifies, and arch was the
borrower. **But arch had already made the same wrong absence at §1 two weeks earlier**, before rust
said anything. Both searches looked for the **token** — the code spellings rust uses (`bad_request`,
`invalid_request`) and rust's crate layout (`extensions/type-system/`) — inside a tree that spells the
same thing `decode_error` in `ext/type/`. **An absence found by searching for the vocabulary of the
seat you already read is not an absence.** That is **AP-21's absence form**, third instance in this
arc, and it is the rule rust's own ratchet states from the other side: *prove an absence by
construction binds hardest outside your tree, and a wrong absence handed upstream gets ratified into a
routing and returns as everyone's premise.*

### §6a.1 The correction unblocks the proposal, and there is a real measured divergence

§1 said *"there is no measured divergence yet, only a latent one."* There is, and it has existed the
whole time. Censused at go `b98e636` · rust `f0a399b` · py `72ffa54`:

| Status | `entity-core-go` | `entity-core-rust` | `entity-core-py` |
|---|---|---|---|
| **400** | `decode_error` ×6 · `invalid_request` ×3 · `invalid_strategy` · `invalid_entity` | `invalid_request` ×8 | `invalid_request` ×5 |
| **404** | `type_not_found` ×6 | — | `not_found` ×1 |
| **500** | `encode_error` ×3 | — | — |

**Three seats, three answers on 404, and two on 400.** This is the strongest input a taxonomy can
have — derived from three failure sets rather than from a request type — and it is exactly what item 2
said the proposal could not get.

### §6a.2 The three rows, derived by rust's own criterion

Row 4's criterion — ***can the declared result type carry the outcome?*** — is checkable against
§7.1's manifest, and it answers all three without a judgement call:

| Operation | `output_type` (§7.1) | Can it express *"a named type did not resolve"*? | Unresolvable type → |
|---|---|---|---|
| `validate` | `system/type/validate-result` | **yes** — `valid` + `violations` | `200`, `valid: false` (unchanged) |
| `compare` / `compatible` | `system/type/compatibility-report` | no | `404` (unchanged) |
| **`converge`** | **`system/type`** — a bare merged type entity | **no.** There is no failure field at all | **`404`** |
| **`adopt`** | **`system/type`** | **no.** Same | **`404`** |
| **`reconcile`** | `system/type/reconcile-result` | **no.** `incompatibilities` describes *fields that could not be reconciled across sources that resolved*; nothing carries *a source path that did not resolve* | **`404`** |

**All three go to `404`, and the criterion did the work rather than a preference.** Note this is the
opposite of what a "be permissive" instinct would produce for `converge`/`adopt`: those two have the
*least* room to carry a failure, because their result type is a bare `system/type`.

### §6a.3 The 404 code — and this is OP-3's first instance, not a §3.3 row

The 404 here is **not** §3.3's `handler_not_found`: a handler **is** registered, and the resolved
*type* is what is missing. Under `0.8.2.7` a more-specific code is permitted only where **defined for
the operation in a spec code set**, and `EXTENSION-TYPE` has none — which is precisely the gap filed as
`COHORT-OPEN-ITEMS` **OP-3**. **So this proposal becomes OP-3's first worked instance:** §8.5 is the
error-code table, and it is seeded by three built implementations rather than authored from nothing.

**`type_not_found` is the row**, adopting go's spelling: it is the only one of the three that says
which lookup missed, and bare `not_found` is ambiguous with the entity-level 404 that `0.8.2.7`'s 404
row explicitly carves out. rust's row-4 `404 not_found` narrows to `type_not_found`; py's single
`not_found` moves; go is already there.

### §6a.4 The two undeclared spellings resolve to the defaults

`decode_error` (go ×6) and `encode_error` (go ×3) are undeclared at their statuses and there is nothing
to declare them **as** — they are the generic case. Params that will not decode is §3's row 1
(`400 invalid_request`); an encode failure building the result is §3's last row
(`500 internal_error`). **go aligns; nothing new is minted.** `invalid_strategy` and `invalid_entity`
are candidates for §8.5 rows rather than sweeps — go should say whether they name distinct failures a
caller would branch on.
