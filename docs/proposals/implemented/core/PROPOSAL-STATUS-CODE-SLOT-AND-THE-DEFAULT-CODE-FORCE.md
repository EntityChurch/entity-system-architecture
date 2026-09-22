# PROPOSAL — §3.3's default-code column has no stated force, and the sweep that closed one synonym was scoped by the token instead of the slot

**Status:** **FOLDED 2026-09-03** — D1–D5 landed and verified in `ENTITY-CORE-PROTOCOL` **0.8.2.7**
(§3.3 default-code force + code slot + satisfaction mode; 500 row's build-state parenthetical struck;
404 row names §6.2 and excludes the in-handler case; two §9.1 conformance rows). The cohort slot
sweep is routed, not owed by this document.
**Target:** `entity-core-protocol` `specs/ENTITY-CORE-PROTOCOL.md` §3.3 (status table + a new
normative paragraph) · §9.1 conformance list.
**Filed by:** `entity-core-go`, spec-issue
`docs/validation/spec-issues/2026-09-03-a-section-3.3-default-code-normativity-and-the-500-converged-claim.md`
(`1c7813c`) — three questions raised while landing the `0.8.2.6` 501 sweep, two of which correct
arch.
**Read at:** arch `d377a95` · core-protocol `ca7cbf8` (**0.8.2.6**) · go `1c7813c` · rust `77916ed` ·
py `1f6627f` · keystone `9d53346` · workbench-go `0cadfe0` · browser-rust `d403fae`

---

## §1 The three questions, and the two that land on arch

go landed `0.8.2.6`'s 501 remedy (26 emit sites, `0a6d3c9`), then **read the sibling rows to size the
follow-on** — which is the adjacent-row discipline this corpus keeps rewarding — and filed:

- **Q1** — is *"Default `code` = X"* a **MUST** or advisory? The 501 row carries an explicit
  *"MUST NOT be emitted"*; the 404 and 500 rows carry **no normative verb**. A wire check can only
  gate the mandatory reading, so the force has to be ruled per row or every seat guesses.
- **Q2** — the 500 row's justifying prose says *"three implementations had independently converged on
  this spelling."* **go had not.** They measured their own tree and flagged rust/py as unverified
  rather than asserting a cohort claim.
- **Q3** — go's no-handler dispatch boundary emits `not_found` where §3.3 names `handler_not_found`;
  the entity-level 404s inside a registered handler legitimately stay `not_found`.

**Q2 is arch's error and it is the more instructive one.** Q1 is a genuine gap in the text. Q3 is a
divergence that is larger than go could see from their own tree.

## §2 Q2 — the claim is false about all three seats, and the reason is a measurement error worth naming

The `0.8.2.6` rationale rests on a table in this proposal's predecessor
(`PROPOSAL-STATUS-TABLE-DEFAULT-CODES-AND-THE-501-SYNONYM` §4): *"`internal_error` — go 184 · rust 27
· py 18, off-spec"*, summarised as *"three implementations independently converged on `internal_error`
— 229 sites."*

**Those three numbers are occurrence counts of the token `internal_error`. They are not a
distribution.** Censusing the **slot** — every `code` emitted at status 500 — at the commits above:

| Seat | `internal_error` | bare `internal` | other spellings in the same slot |
|---|---|---|---|
| `entity-core-go` | 183 | **58** | `storage_error` 18 · `encode_failed` 14 · `io_error` 7 · `store_put_failed` 6 · `handler_error` 3 · `encode_error` 3 · `backend_error` 3 · `dispatch_failed` 2 · `cap_issuance_failed` 2 · + 6 singletons |
| `entity-core-rust` | 12 | 3 | `result_encode_failed` 3 · `store_failed` 2 · `storage_error` 2 · + 8 singletons |
| `entity-core-py` | 12 | 1 | `io_error` 7 · `invalid_version` 5 · + ~12 singletons |

**`internal_error` is the plurality at every seat and the convention at none.** go's 16 spellings at
one status is the extreme case, but no tree converged, so the sentence is false about all three — not
only about go, which is the half go could not have measured.

**The mechanism, and this is the transferable part.** *"Three implementations converged on X"* is a
positive-sounding sentence whose truth condition is a **negative**: *nothing else occupies that slot.*
`AGENTS-STANDARD`'s *prove a negative before you claim it* is the rule that should have fired and did
not, because the claim does not read as a negative — **it reads as measured, and it cites a number.**
A `grep -c X` returns X's frequency and says nothing about the population sharing its slot. The
enforcement point is in §6.5 and it is cheap: **group by code at that status; publish the
distribution, not the count.**

**It compounded immediately, in the sweep that closed the first synonym.** `0.8.2.6` retired
`unknown_operation` — and censusing the **501 slot** today, after go's sweep landed:

| Seat | `unsupported_operation` | surviving synonyms in the same slot |
|---|---|---|
| `entity-core-go` | 31 | **`not_implemented` 6** (`execute.go:293,667` · `local.go:454` · `ext/compute/handler.go:137,144` · `ext/registry/peerissued/register.go:391`) |
| `entity-core-rust` | 5 | `not_implemented` 2 · `unsupported_mode` 2 · `not_supported` 1 · `handler_error` 1 · `nope` 1 |
| `entity-core-py` | 11 | `not_implemented` 2 · `not_available` 1 · `domain_control_unsupported` 1 |
| `entity-core-keystone` | 22 | **`not_implemented`** — `asm-x86_64` / `asm-arm64` / `riscv64` `dispatch.s` (`ec_not_impl`) and `pd` `main.pd` |
| `entity-workbench-go` | 0 | `remote_unavailable` 2 · `unsupported_algorithm` 1 · `transports_by_hash_unspecified` 1 · `storage_not_supported` 1 |

**`unknown_operation` was one of five spellings, and the sweep retired one.** `not_implemented` is in
**all three ground-up trees plus four generated peers** and means exactly what the 501 row means. A
sweep scoped by a token leaves the slot divergent and reports done — the same
width-of-the-first-incident error L14 records, one level down from where `0.8.2.6` caught it.

**Two things `0.8.2.6` also got wrong on the same axis, both corrected here.** Its §2 table charged
`entity-browser-rust` with **1** `unknown_operation` site. At `d403fae` the token appears **nowhere**
in that tree — not in `*.rs`, not in `*.md`, not on any branch searched — and browser-rust emits
`handler_not_found` at 7 sites, so it is conformant on the 404 row as well. **That row was an
assignment to a seat that owed nothing.** And go's own close-out reported the sweep
*"`unknown_operation` grep-clean"*; it is **emit-clean**, with two correct harness references
remaining (`reachability.go:193`'s deliberate tolerance and a comment at `typeext.go:451`) — a
precise statement rather than a defect.

## §3 Q1 — the force is not being invented; it is already stated one paragraph below, for two rows

**§3.3's own *Authorization-path code discipline* (normative, v7.71) is exactly the regime Q1 asks
for**, landed, for 401/403:

> A request-time authorization `DENY` … MUST be surfaced with … `result.data.code` set to the
> authorization domain's **defined** code — `capability_denied` **by default**, or a more-specific
> **defined** authorization code where one applies. Implementations **MUST NOT** surface a generic
> catch-all default (e.g. `verification_failed`) on an authorization path; the catch-all is a sign
> that an authorization failure escaped its defined code.

Three clauses: **a default for the generic case**, **a more-specific code only if defined**, and **no
minting**. §3.3 line 851 supplies the vocabulary the second clause needs — *"connection errors (§4.7),
tree errors (`EXTENSION-TREE` Appendix A), and domain handler errors each define their own `code`
values"* — and `EXTENSION-TREE` Appendix A is the worked model: a per-operation table of
`(operation, code, status, description)`.

**So the answer to Q1 is derivable from landed text and is uniform: the default is MANDATORY for the
generic case at every row that names one.** The column is not documentation; it is the same contract
v7.71 already binds on the authorization path, stated at the table where the reader is. This is the
identical move `0.8.2.6` made for §6.2's 501 sentence — the rule was general, the heading was not.

**Not derived from the cohort (L18).** Nothing above rests on what any seat emits. If all six seats
emitted a different spelling the derivation would not move, because v7.71's paragraph would still say
what it says. The census in §2 is the **measurement of the gap**, never the argument for closing it.

### §3a What the mandatory reading does NOT license, and this is the L17 half

The undeclared more-specific codes — go's ~60 sites at 500, rust's ~13, py's ~16, the app tier's — are
**not swept by this proposal, and asking for that would be unlandable.** The escape clause requires
the code be *defined in a spec code set*, and **most extensions declare none**: `EXTENSION-TREE` has
Appendix A, and the rest of the family has no error-code table at all. **A MUST whose escape hatch has
no declared site is L17's exact violation** — *a rule two conformant peers cannot both implement,
because there is nowhere for the agreement to live.*

So the ruling lands on the **generic case only**, which is what the column names and all it can
support today. The per-extension error-code table is a real gap, it is arch's, and it is §9's open
item — not a sweep routed to five seats against a vocabulary that does not exist.

### §3b Satisfaction mode per row, because one of these rows cannot be gated

`GUIDE-CONFORMANCE` §5.2b.1 requires the satisfaction mode stated at the point of the MUST, and
**L19** requires the class named from the section that owns the surface. Per row:

| Row | Mode | Driver |
|---|---|---|
| **501** | wire-drivable | dispatch an operation absent from a registered handler's manifest → assert `501 unsupported_operation`. `validate-peer` register; the check `0.8.2.6` filed |
| **404** | wire-drivable | dispatch to a path with no handler registered, **targeting the local peer** → assert `404 handler_not_found`. Distinct from the §5.2a foreign-namespace row, which is `400 invalid_request` |
| **500** | **NOT oracle-drivable** | a conformant peer cannot be made to fail internally on demand over the wire. Satisfaction is a **source audit** of the emit sites at a named commit |
| 400 / 403 | as today | enumerated specific sets + the v7.71 paragraph |

**The 500 row's mode is stated so the check is not written and then found unbuildable.** A
`validate-peer` check for the generic 500 would need fault injection the protocol does not expose;
recording that now is cheaper than discovering it in a harness.

## §4 Q3 — `handler_not_found` was already normative in five homes, and two trees do not emit it

`0.8.2.6` did not create this obligation; it **tabulated** one that four other sites already stated
(**L23** — a rule has every home it is stated in, and the row was the tabulation):

| Home | What it says |
|---|---|
| §6.2 line 3128 | *"**`404 handler_not_found`** — no handler is registered at the dispatch path, **on a path that targets the local peer**"* — the definitional home |
| §6.2 line 3138 | *"The cross-peer behavior of an unbacked advertisement **is** `404 handler_not_found` at the dispatch boundary"* |
| §5.2a line 2433 | the foreign-namespace refusal *"MUST NOT be reported as `404 handler_not_found` (§6.2)"* — a MUST NOT that presupposes the code |
| §6.7 line 3678 | RT-8 masking **MAY** return *"`404 handler_not_found`"* |
| §9.1 line 4232 | the dispatch-routing conformance row, same MUST NOT |

**Censused at the dispatch boundary:**

| Seat | no-handler 404 code | Evidence |
|---|---|---|
| `entity-core-rust` | **`handler_not_found`** | `core/peer/src/connection.rs:1937,3030,3046,3670`; **pinned by tests** at `core/peer/src/lib.rs:4425` and `bindings/sdk/src/sdk.rs:6557` (*"R2 must surface handler_not_found code"*) |
| `entity-core-keystone` — generated peers | **`handler_not_found`** | 146 references across the generated tier |
| `entity-browser-rust` | **`handler_not_found`** | 7 sites |
| `entity-core-go` | `not_found` | `core/protocol/execute.go:284,625` — **2 sites** |
| `entity-core-py` | `not_found` | `peer/peer.py:4063` via the shared `ExecuteResponse.not_found` factory (`protocol/messages.py:354`) — **1 site**; `handler_not_found` occurs **0 times** in py source |

**go's Q3 framing understates it in one direction and arch's row understated it in the other.** go
asked *"divergence if mandatory"*; it is a divergence against **rust, the 46 generated peers, and
browser-rust**, with rust holding two tests that pin the string. And the fix is smaller than a factory
split: py's boundary is **one call site**, not the four that share the factory.

**go is right that the entity-level 404s are out of scope.** `core/tree/handler.go:182,194,415` —
handler registered, entity absent — are a different fact and stay `not_found`. Any 404 alignment is
**2 sites at go, 1 at py**, and MUST NOT touch the entity-level emits.

**And py's own module docstring asserts the code py does not emit.**
`capability/advertisement.py:5` reads *"produces `404 handler_not_found` at the dispatch boundary."*
That is **L20's corollary** — *a comment asserting a behaviour is a claim to test, and it is the
highest-value line in a file to write a test against, because it is where the author has told you what
they believe.* py read §6.2, wrote its rule down, and emits the other string.

## §5 Where the divergence went to die, both times

Two harness sites in `entity-core-go` absorbed exactly these two divergences:

- `cmd/internal/validate/reachability.go:193` — accepts `400` with *either* `unknown_operation` or
  `unsupported_operation`, with a comment naming which seat does which. Routed at `0.8.2.6`.
- `cmd/internal/validate/registry_issuer.go:1506` — `status == 404 || code == "handler_not_found" ||
  code == "not_found" || code == "unknown_handler"`. **Never routed**, and it names a fourth spelling
  (`unknown_handler`, which no tree emits — go holds the only occurrence, in this line).

**Neither harness is defective.** A probe asking *"did this peer decline?"* is right to be tolerant,
and the `status == 404 ||` short-circuit makes the code terms decoration rather than a contract. **What
is defective is that a tolerance is where a measured cross-impl divergence is recorded and then
forgotten** — the same fact as `0.8.2.6`'s finding, and the second instance of it in two days.

**A claim arch nearly published and withdrew before routing, recorded because the withdrawal is the
point.** A grep for `code == "X" || code == "Y"` across go's harness returned **seven further sites in
`authz.go`** tolerating `403 verification_failed` — the exact code v7.71 outlaws and
`AUTHZ-NO-CATCHALL-1` exists to fail. **Opened, all seven are `FailCheck` branches** with
spec-cited diagnostics (*"v7.71 §3.3 prohibits catch-all defaults on the authz path"*). They are the
opposite of a tolerance: a precise negative-diagnostic ladder, and go's authz harness is exemplary
there. **The grep was the artifact and it was one step from being read as a conclusion** — L8, caught
by opening the file, which is the only thing that ever catches it.

## §6 The ruling

1. **The §3.3 default-code column is MANDATORY for the generic case at every row that names a
   default.** Derived from §3.3's own v7.71 authorization-path paragraph, generalized to the table it
   sits under. Advisory is refuted: it makes `result.data.code` unbranchable for the one case that has
   no other information in it.
2. **A more-specific code is permitted only where one is DEFINED for the operation in a spec code
   set** (§4.7 · `EXTENSION-TREE` Appendix A · a domain handler's own table). An **undefined spelling
   is non-conformant** — the v7.71 no-catch-all clause, generalized.
3. **The unit of conformance and of any sweep is the CODE SLOT** — every code emitted at that status
   — **never a single spelling.** Stated normatively, because `0.8.2.6` retired one of five 501
   synonyms and reported the class closed.
4. **The 500 row's build-state parenthetical is struck.** It is false about all three ground-up trees,
   and a build-state claim inside normative spec text is the failure this repo's *spec text is not our
   log* rule exists to prevent — a stale claim about a peer, re-read as a rule, outliving its truth.
   The rationale lives here.
5. **`handler_not_found` is the code for the local no-handler dispatch boundary**, restated at §3.3
   with §6.2 named as the defining home (**L23** fourth shape — a restatement names its authority).
   Entity-level 404s inside a registered handler are a different fact and are untouched.
6. **Satisfaction mode is stated per row** (§3b), including that the 500 row is **not oracle-drivable**
   and is satisfied by source audit.
7. **Not a flag day.** Every item changes what a peer **emits**, never what it accepts. Nothing
   branches on these codes across a connection, and go's two tolerances absorb both transitions, so
   seats may land in any order with no partition. **Divergence unit: none.**

## §7 Delta table

| # | Document | Section | Change |
|---|---|---|---|
| **D1** | `ENTITY-CORE-PROTOCOL.md` | §3.3 | New normative paragraph after the status table: default-code force (generic case, MUST) · the defined-code escape · **the code slot** as the unit · satisfaction mode per row, with 500 marked not oracle-drivable. |
| **D2** | `ENTITY-CORE-PROTOCOL.md` | §3.3, 500 row | Strike *"(0.8.2.6 — the row named no code, and three implementations had independently converged on this spelling with nothing in the corpus saying it)"*. Default code stands. |
| **D3** | `ENTITY-CORE-PROTOCOL.md` | §3.3, 404 row | Name §6.2 as the defining home; state that entity-level 404s inside a registered handler are not this case. |
| **D4** | `ENTITY-CORE-PROTOCOL.md` | §9.1 | Two conformance rows: the 501 slot check and the 404 local-no-handler check, each naming the code and distinguishing the adjacent case. |
| **D5** | `ENTITY-CORE-PROTOCOL.md` | line 3 | Version → **0.8.2.7**. |

## §8 Cohort impact

Emit-side only; land in any order.

| Seat | Owed |
|---|---|
| `entity-core-go` | **501 slot:** 6 `not_implemented` → `unsupported_operation`. **404:** 2 dispatch-boundary sites `not_found` → `handler_not_found` (`execute.go:284,625`); entity-level `not_found` untouched. **500:** 58 bare `internal` → `internal_error` (the generic sites go itself identified). The ~60 undeclared specific spellings at 500 are **held** — see §9. |
| `entity-core-rust` | **501 slot:** `not_implemented` 2 · `unsupported_mode` 2 · `not_supported` 1 · `handler_error` 1 (+ `nope`, believed a fixture — confirm). **500:** 3 bare `internal` → `internal_error`. **404: already conformant, with tests** — the reference for this row. |
| `entity-core-py` | **501 slot:** 6 residual `unknown_operation` + `not_implemented` 2 · `not_available` 1 · `domain_control_unsupported` 1. **404:** 1 site (`peer.py:4063`) — do **not** change the shared factory; the other three call sites are entity-level. **500:** 1 bare `internal`. **And correct `advertisement.py`'s docstring or the code it describes** — they disagree today. |
| `entity-core-keystone` | **Not already conformant on the slot** — correcting arch's `ROUTING-2026-09-03-b`. `not_implemented` in `asm-x86_64` / `asm-arm64` / `riscv64` `dispatch.s` and `pd`. Their own `asm-x86_64` `SPEC-AMBIGUITY-LOG` **A-ASM-007** records adopting it as a hang fix and marks it non-blocking — filed in their log, never routed. `pd/src/main.pd` additionally draws a distinction the corpus does not make (*"unauthored pattern → 501 `not_implemented`"* vs *"op outside the ladder → 501 `unsupported_operation`"*); a runtime-registered pattern with no body is a registered handler that does not implement the op, so it is `unsupported_operation`. **404: already conformant** (146 refs). Vendored snapshot is **v0.8.2.3** — three bumps behind; 0.8.2.7 is the next vendor step. |
| `entity-workbench-go` | 10 `unknown_operation` (OP-1, stands) **plus 4 undeclared 501 spellings**. |
| `entity-browser-rust` | **Nothing. OP-2 is withdrawn** — the token is absent from the tree, and they emit `handler_not_found` at 7 sites. |

## §9 Open items

1. **Per-extension error-code tables — arch's, and it is the L17 site this ruling's escape clause
   needs.** `EXTENSION-TREE` Appendix A is the only worked one in the family. Until an extension
   declares its codes, its handlers' more-specific spellings are undeclared by construction, so
   ~90 emit sites across three trees sit in a state this ruling can name but not resolve.
   **Sized:** one appendix per extension that emits domain codes, seeded from the census in §2. This
   is the arch-owned follow-on and it is not a cohort sweep.
2. **The two conformance checks (D4) are filed, not built** — `validate-peer` register, authoring
   owner `entity-core-go` per the test-vector convention, and go asked for exactly this: one §3.3 pass
   asserting the **whole default-code table** rather than a 501-only vector rebuilt later. That
   sequencing is granted.
3. **A gate for the slot rule.** Ruling 3 is mechanically checkable in an implementation tree —
   *group emit sites by status, assert one code per generic slot* — and it is the check that would have
   caught `not_implemented` surviving `0.8.2.6`. Filed for the harness tier, not built here.
