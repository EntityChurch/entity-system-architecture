# PROPOSAL — §3.3 names a default code for three statuses and leaves three bare, and the cohort minted a synonym in one of the gaps

**Status:** **FOLDED 2026-09-03** — D1–D4 landed and verified: `ENTITY-CORE-PROTOCOL` **0.8.2.6** (§3.3 404/500/501 default codes; §6.2 generalized) and `DOMAIN-LOCAL-FILES` §937. The ~66-site cohort sweep is routed, not owed by this document.

> **CORRECTED SAME DAY by `PROPOSAL-STATUS-CODE-SLOT-AND-THE-DEFAULT-CODE-FORCE` (0.8.2.7).** The
> ruling stands; three claims in the reasoning below do not, and they are left in place as the record
> rather than rewritten. **(1) §4's *"three implementations independently converged on `internal_error`
> — 229 sites"* is false about all three trees.** `184 · 27 · 18` are occurrence counts of the token
> `internal_error`, not a census of the code slot at status 500; censused, go carries 16 spellings
> there (183 `internal_error`, 58 bare `internal`, 60 across 14 others), rust and py likewise. The
> parenthetical this claim justified is struck from the spec at 0.8.2.7. **(2) §2 and §8 charge
> `entity-browser-rust` with 1 `unknown_operation` site.** The token is absent from that tree at
> `d403fae` on every branch searched — the row was an assignment to a seat that owed nothing, and it
> is withdrawn. **(3) The sweep this proposal ordered was scoped by a token and closed one of five
> synonyms in the 501 slot;** `not_implemented` survives it in all three ground-up trees and four
> generated peers. The successor rules the **slot**, not the spelling.
**Target:** `entity-core-protocol` `specs/ENTITY-CORE-PROTOCOL.md` §3.3 + §6.2 ·
`specs/domains/DOMAIN-LOCAL-FILES.md` §937.
**Filed by:** `entity-core-rust` (`ROUTING-2026-09-02-g` §3), who declined to change 27 sites unrouted
and said the subject belonged in the generic-400 sweep rather than in a TYPE-only §8.5. Correct on
both counts — the sweep missed it.
**Read at:** core-protocol `d43d66d` (**0.8.2.5**) · go `78a9f2d` · rust `77916ed` · py `4c7a6bc` ·
keystone `6dcbd22` · browser-rust / workbench-go same day

---

## §1 What rust found, and why my own sweep did not

`PROPOSAL-GENERIC-400-SYNONYMS` swept `ENTITY-CORE-PROTOCOL` §4.7's *"MUST NOT mint a synonym"*
clause **by subject** and found three violations. It swept the subject *"a code minted for the
generic-**400** class"* — and the clause's reasoning (*"both are minted codes in no spec code set"*)
is a property of **any** minted code at **any** status. **The sweep was scoped by the status of the
incident that produced it**, which is the same width-of-the-first-incident error L14 records.

rust filed the miss: `unknown_operation` and `unsupported_operation` are two spellings of one
failure — *a handler is registered at the path and does not implement the named operation.*

## §2 The measurement — and it is not one seat's 27 sites

rust framed this as *"27 sites at this seat riding on the answer."* Searched **every tier**
(**L16**), it is not:

| Seat | `unknown_operation` | `unsupported_operation` |
|---|---|---|
| `entity-core-go` | **28** | 7 |
| `entity-core-rust` | **27** | 6 |
| `entity-workbench-go` | **10** | 0 |
| `entity-browser-rust` | **1** | 0 |
| `entity-core-py` | 6 | **33** |
| **`entity-core-keystone` — 46 generated peers** | **0** | **22** |

**And the status differs, which makes it more than a spelling.** go and rust emit **`400
unknown_operation`**; py and every generated peer emit **`501 unsupported_operation`**. 400 tells a
caller *your request was malformed — fix it and do not retry as-is*; 501 tells it *this peer does not
implement this — degrade or go elsewhere.* **A client keying on `result.data.code` gets a different
remedy depending on which implementation refused**, which is precisely the contract §4.7 says the
code table exists to provide.

**It is already costing something, and the receipt is in go's tree.**
`cmd/internal/validate/reachability.go:189` has to accept all of `501`, `404`, and `400` with
*either* spelling, and its comment names the divergence outright — *"entity-core-py answers this …
entity-core-rust answered this."* **That is a measured cross-impl divergence recorded in a source
comment and never routed** — the exact shape L17's first instance records, where convergence (here,
divergence) lives in the comments and no gate can reach it. The harness is not defective; a
reachability probe asking *"did this peer decline?"* is right to be tolerant. What is defective is
that nothing else asked.

## §3 The spec already answers it — in a section headed for a different reader

**§6.2, line 3102:** *"**`501 unsupported_operation`** — a handler IS registered at the path, but
does not implement the named operation. The caller's authority is irrelevant; the operation does not
exist on this handler."*

The **sentence is general** — it says *a handler*, not *a capability handler*. The **heading is
not**: *"Capability handler operation status codes."* §6.5's line 3651 confirms the general reading
from the other side, discussing `501`'s leak properties as an ordinary dispatch outcome.

`unknown_operation` appears in the entire corpus **once** — `DOMAIN-LOCAL-FILES` §937, in a
parenthetical about omitting `watch` from a manifest. That is the same fact §6.2 governs (an
operation the handler does not implement), spelled differently, in a **domain** spec that is
downstream of the core.

## §4 The structural cause, and it is the half worth fixing

**§3.3's status table names a default code for 400, 401 and 403, and leaves 404, 500, 501 bare.**

| Status | §3.3 today | What the corpus/cohort actually uses |
|---|---|---|
| 400 | default `invalid_request` + five specifics | — |
| 401 | codes named | — |
| 403 | default `capability_denied` + specifics | — |
| **404** | *"Not found"* | `handler_not_found` — **5 corpus sites**, §6.5 |
| **500** | *"Internal error"* | `internal_error` — **go 184 · rust 27 · py 18**, off-spec |
| **501** | *"Not supported"* | `unsupported_operation` — §6.2, and 46 generated peers |

**So an implementer at a 501 site has nothing to look up in the table where every other default code
lives.** They reach for §6.2, read a capability-handler heading, conclude it is scoped, and mint. The
synonym is not carelessness — it is the predictable output of a half-populated table. **This is L23's
second shape: the home that states the rule is not the home the reader is in.**

**The 500 row is the same gap with a happier outcome and it proves the mechanism.** Three
implementations independently converged on `internal_error` — 229 sites — with **no spec text
anywhere naming it.** Convergence happening in the code where no gate can reach it, one refactor from
decaying. rust's single `encode_failure` is the first divergence and it is why this row is in scope.

## §5 The ruling

1. **`501 unsupported_operation`** is the code for *handler registered, named operation not
   implemented* — **generally**, on any handler. Derived from §6.2's own sentence and §3.3's 501 row.
2. **`unknown_operation` is a minted synonym and is retired**, on the `incompatible_key_type` /
   `invalid_signature` / `bad_request` precedent.
3. **§3.3's 404, 500 and 501 rows carry their default codes**, so the next implementer looks the
   answer up instead of deriving it.
4. **§6.2's sentence is marked general and names §3.3 as the authority** for the pair, so the copy is
   readable as a copy (**L23** fourth shape).

**Not derived from the cohort (L18).** The derivation is §6.2's text and §3.3's existing 501 row.
That 46 generated peers and py already do it is **corroboration, cited last, and verified in the
tree** — 22 sites read at `6dcbd22`, 0 of `unknown_operation`. If every seat had implemented the
other spelling the argument would not change, because §6.2 would still say what it says.

## §6 Cost, honestly, and the direction of travel

**~66 sites move** (go 28 · rust 27 · workbench-go 10 · browser-rust 1) plus py's 6, plus rust's one
`encode_failure`. That is the largest sweep this corpus has asked for.

**The alternative is worse and is not "do nothing."** It is a permanent two-spelling,
two-status divergence on the code clients branch on, with the conformance anchor's 46 peers on one
side and the two largest ground-up trees on the other — and no gate anywhere, which is why it has
survived this long.

**Not a flag day (L21 third shape).** This changes what a peer **emits**, never what it accepts.
Nothing branches on the code across a connection, and go's own harness already tolerates both
spellings and both statuses during any transition, so seats may land it in any order with no
partition. Stated explicitly so nobody sequences around a risk that is not there.

## §7 Delta table

| # | Document | Section | Change |
|---|---|---|---|
| **D1** | `ENTITY-CORE-PROTOCOL.md` | §3.3 | 404 row → default `handler_not_found`; 500 row → default `internal_error`; 501 row → default `unsupported_operation`. |
| **D2** | `ENTITY-CORE-PROTOCOL.md` | §6.2 | Mark the 501 sentence **general, not capability-handler-scoped**; name §3.3 as the authority; name `unknown_operation` non-conformant. |
| **D3** | `DOMAIN-LOCAL-FILES.md` | §937 | `unknown_operation` → `unsupported_operation`. |
| **D4** | `ENTITY-CORE-PROTOCOL.md` | line 3 | Version → **0.8.2.6** (shared with `PROPOSAL-CONNECT-PREAUTH-SCOPE`, one vendor step). |

## §8 Cohort impact

| Seat | Owed |
|---|---|
| `entity-core-go` | 28 sites `unknown_operation` → `unsupported_operation`, 400 → 501. `core/protocol/connect.go:338` additionally carries the code on an **unreachable** default (`execute.go:90` intercepts first, correctly) — hygiene, not a conformance defect, but it becomes one the day the intercept is refactored. |
| `entity-core-rust` | 27 sites; plus `encode_failure` → `internal_error` (1 site). |
| `entity-workbench-go` | 10 sites. |
| `entity-browser-rust` | 1 site. |
| `entity-core-py` | 6 residual `unknown_operation` sites; already conformant at 33. |
| `entity-core-keystone` | **none — already conformant**, and the vendor step for 0.8.2.6. |

## §9 Open items

**A conformance check.** There is none for this pair at any seat, which is how a 66-site divergence
reached two releases. It is the **L17** missing half on a rule that turns out to be four years'
worth of sites, and it is filed rather than built: the check is *"call a registered handler with an
operation absent from its manifest; assert `501 unsupported_operation`"*, and it belongs in the
`validate-peer` register. Owner: `entity-core-go` per the test-vector convention.
