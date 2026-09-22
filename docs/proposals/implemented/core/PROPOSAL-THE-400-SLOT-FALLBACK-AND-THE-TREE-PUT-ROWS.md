# PROPOSAL — the 400 slot already has what the 500 slot was given, two seats read it as a gap anyway, and `tree:put` needs two rows

**Status:** **FOLDED 2026-09-05** — D1–D4 landed and verified: `ENTITY-CORE-PROTOCOL` **0.8.2.9** (§3.3's fallback sentence) and `EXTENSION-TREE` **v4.4** (three Appendix A rows — `put`/`set` at 400 `invalid_request`, 400 `hash_mismatch`, and the 409 CAS row tabulated beside it). The 2-site convergence at go is routed, not owed by this document.
**Target:** `entity-core-protocol` `specs/ENTITY-CORE-PROTOCOL.md` §3.3 (one clarifying sentence) ·
`specs/extensions/EXTENSION-TREE.md` Appendix A (two rows).
**Filed by:** `entity-core-go` spec-issue `2026-09-04-a` (`4dd47e6`) and `entity-core-py` **SA-PY-40**
(`b2b3100`), independently, on the same surface.
**Read at:** arch `5974743` · core-protocol `7ac75fd` (**0.8.2.8**) · go `4dd47e6` · rust `27a1dc3` ·
py `b2b3100` · keystone `e7f905c`

---

## §1 The ask, and the half of it that is already answered

Both seats ask arch to *"rule the 400 slot the way the 500 slot was ruled — either a §3.3-level
default 400 code that the generic case MUST carry, or a directive that each operation's owning
extension MUST enumerate its 400 set."*

**The first option has existed since before this arc.** §3.3's 400 row:

> Default `code` = **`invalid_request`**; more-specific 400 codes where one applies: `invalid_path`,
> `invalid_params`, `unexpected_params`, `chain_depth_exceeded`, `signature_path_conflict`.
> **`invalid_request` … is the generic 400 code an extension handler uses for a structurally invalid
> request.**

A named default, an enumerated cross-cutting specific set, and an explicit sentence assigning the
default to extension handlers. **The 400 row is the most completely specified row in the table** — it
is the row the 500 row was brought up to match at `0.8.2.8`, not the other way round.

### §1.1 Both seats' causal story is off by one version, and the correction matters

go: *"a table landing elsewhere (`EXTENSION-TYPE` Appendix A v1.3) **retroactively turned**
previously-fine 400 spellings into undefined ones, with no diff to prompt the change."*
py: *"a code that was merely **undeclared** while its extension had no table becomes **undefined
against an existing table** the moment one lands."*

**Neither is what happened.** `0.8.2.7` — a day before Appendix A — says *"an undefined spelling is
non-conformant"* and *"a more-specific code MAY be used only where one is **defined for the operation
in a spec code set**."* `decode_error` was non-conformant the moment that landed. **Appendix A did not
create the obligation; it supplied a target to converge to.** The distinction matters because the
inference drawn from it — *"so the 400 slot needs what the 500 slot got"* — is an ask for something
that is already there, and acting on it would mean adding a second default to a row that has one.

**What was actually missing is nothing normative. It is that neither seat applied the fallback.** An
attempted specific code that no table defines **is not a gap — it falls back to the default**, which
is what a default is for. The answer for `decode_error` was `invalid_request` on 2026-09-03, and go
converged to exactly that a day later by a different route.

## §2 But two independent seats misread the same sentence, and that is a spec defect

**This is the part worth acting on.** go and py are the two seats that have read §3.3 hardest this
week — both filed correct, load-bearing findings against it — and **both concluded the 400 slot had no
governing set.** When two careful readers independently reach the same wrong conclusion from a
sentence that is technically correct, the sentence is the problem, not the readers.

The reason is visible in the text: `0.8.2.7`'s slot paragraph states the **permission** (*a specific
code MAY be used where defined*) and the **prohibition** (*an undefined spelling is non-conformant*),
and never states the **consequence** — *what a peer emits when it wanted a specific code and none is
defined.* A reader with an undefined spelling in hand sees a rule that forbids what they have and does
not say what to do instead, so they read it as an unfilled slot.

**Delta D1** closes it in one sentence at the point of the prohibition.

## §3 The real gap, and go handled it exactly right

`system/tree:put` emits two 400s that no table covers:

| Site (go `4dd47e6`) | Condition |
|---|---|
| `core/tree/handler.go:429` | the submitted entity does not decode |
| `core/tree/handler.go:434` | `entity.Validate()` — the content hash does not match |

**`EXTENSION-TREE` Appendix A has no `put`, `set` or `get` rows at all.** Its operation column is
`snapshot` · `diff` · `merge` · `extract` · `create` · `destroy` · `(any)` — the tree-management
operations. **The two core data operations the whole protocol runs on are absent from the only
error-code table their extension has.**

**go held `invalid_entity` as a named divergence and routed rather than inventing.** That is the
correct move under the divergence rule and it is why this is a two-row fold instead of a three-way
split: *"there is no defined code to converge to, and inventing one is exactly the 'three invented
sites is three homes no operator can target' anti-pattern."* Same position `backend_error` was in last
cycle, and it is blessed the same way — **except that neither seat's proposed code is the answer.**

### §3.1 `invalid_entity` is in neither corpus; `hash_mismatch` is in both, for exactly this failure

Grepped `specs/` and `guides/` in both repos:

- **`invalid_entity` — 0 occurrences.** It is a go-local spelling, and py disclosed a mirror of the
  same class (`invalid_strategy`). Blessing it would mint a code where the corpus already has one.
- **`hash_mismatch` — 8 occurrences**, and it splits cleanly by failure:

| Status | Failure | Sites |
|---|---|---|
| **400** | **a content hash does not match the entity it addresses** | `EXTENSION-CONTENT` §923 — *"included entity hash does not match key"* · `EXTENSION-COMPUTE` :769 |
| 409 | a CAS `expected_hash` precondition lost a race | `EXTENSION-SUBSCRIPTION` §2.2 · `EXTENSION-CONTINUATION` :1523 · `ENTITY-CORE-PROTOCOL` :1343, :1349 |

**`EXTENSION-CONTENT` §923 is the exact analogue of go's `handler.go:434`** — `content_hash(entity) !=
hash` → `400 hash_mismatch`. Same failure, same status, same reasoning, already normative.

**So the two rows are:**

| Site | Code | Status | Why |
|---|---|---|---|
| entity does not decode | **`invalid_request`** | 400 | The generic structurally-invalid case. §3.3's 400 row assigns it in terms; no new code |
| entity hash does not match | **`hash_mismatch`** | 400 | Corpus-existing for this exact failure (`CONTENT` §923). Not `invalid_entity`, which the corpus does not contain |

**And the two `hash_mismatch` statuses are two failures, not one code used loosely.** 400 is *this
entity is not what it claims to be* — a defect in the submission. 409 is *someone else wrote first* —
a defect in nobody, retryable. Naming them together in the row is the AP-22 discipline pointed the
other way: **the same token at two statuses is fine when the failures differ; what is forbidden is two
tokens for one failure.**

## §4 The 400-slot census py asks for — scoped, not ordered

py: *"no seat has run the 400-slot census."* True, and it should be run — but **not as a sweep to the
default.** The 500 census produced a genuine finding (`io_error` vs `storage_error` are two
conditions) precisely because it asked what each site *meant*. Scoped:

1. Group every 400 emit by code, per seat, keyed on the status **value** (AP-21).
2. For each undefined spelling, ask **does this name a condition a caller would branch on?**
   - **No** → it is the generic case; emit `invalid_request`. No routing.
   - **Yes** → hold it as a named divergence and **route it**, as go did here. Do not invent a site.
3. Arch answers each routed one by **grepping the corpus for the failure first** — `hash_mismatch` was
   already there, and so was `backend_error`'s justification.

**This is the process that worked twice. It is being written down rather than re-derived.**

## §5 `storage_error` — confirmed, no edit

go asks arch to confirm line 836 governs `storage_error`. **Confirmed.** `ENTITY-CORE-PROTOCOL` §3.3's
500 row at `0.8.2.8` enumerates `io_error` (an OS I/O operation failed) and `storage_error` (a
content-store or tree bind/read failed) as the more-specific 500 set. go and py concur and both have
swept. Nothing to fold.

## §6 Delta table

| # | Document | Section | Change |
|---|---|---|---|
| **D1** | `ENTITY-CORE-PROTOCOL.md` | §3.3, slot paragraph | One sentence at the prohibition: an attempted specific code that no spec code set defines **falls back to that status's default**; the absence of a table is not an unfilled slot. |
| **D2** | `EXTENSION-TREE.md` | Appendix A | Two `put` / `set` rows — `invalid_request` (does not decode), `hash_mismatch` (content hash mismatch) — with the 400-vs-409 distinction named. |
| **D3** | `EXTENSION-TREE.md` | line 3 | Version bump. |
| **D4** | `ENTITY-CORE-PROTOCOL.md` | line 3 | Version → **0.8.2.9**. |

## §7 Cohort impact

Emit-side only. **Not a flag day, no divergence unit.**

| Seat | Owed |
|---|---|
| `entity-core-go` | 2 sites: `core/tree/handler.go:429` → `invalid_request`, `:434` → **`hash_mismatch`**. The type-surface convergence they already landed is correct and is not revisited |
| `entity-core-py` | Their disclosed `invalid_strategy` mirror, and any `tree:put` equivalents — run the §4 census; sweep the generic ones, route the rest |
| `entity-core-rust` | Run the §4 census. Nothing known-owed |
| `entity-core-keystone` | Vendor step only — but they are **six bumps behind** and that is now the track's blocker, not this fold |

## §8 Open items

**None from this fold.** The two rulings both seats named as remaining are answered here (§3, §5), and
§4 converts the open-ended *"census the 400 slot"* into a bounded procedure with a routing path. After
this, the impl-side work on this track is **2 sites at go plus a census at each seat**, and the track
closes on a 3-way drive against `0.8.2.9`.
