# PROPOSAL — the materialized `compute/error` entity is content-hashed over `code` alone

**GATE RE-DEFINED 2026-08-16 — and NOT met. `[read this before ratifying §2.4]`**

**The gate moved when the ruling moved, and the vector that carries its name now proves the
opposite thing.** The gate was *"a corpus vector that materializes an error into a construct/tree,
run three-way."* Under (B) **an error never materializes into a construct** — so
`worked/error/materialized-into-construct` now reads `error(corpus_materialized_error)` in all three
trees. That is a clean three-way lock **of propagation**, which is v3.23's rule and is already
folded. **It exercises this proposal's rule — the code-only content hash of a *materialized* error —
nowhere at all.**

**§2.4's only surviving surfaces are the §7.2 `result_path` write and SA-9 `store`, and no vector
covers either.** `entity-core-go`'s own report says it: *"the 331-vector corpus exercises the
value-form error at **no** consumer or crossing"* — and their recommendation **asks arch for** the
family that would route one *"through the reactive `result_path` crossing … writes code-only."*
**A seat asking for the vector is the clearest possible evidence the coverage does not exist.**

**The decisive evidence is go's own defect, not an argument.** Probing under v3.23 they found their
**reactive `result_path` crossing wrote the error verbatim, message intact — a live §2.4 violation**,
in the reference implementation, invisible beneath a 331-green board until the ruling made them look.
**That is exactly the surface this proposal governs.** Ratifying §2.4 now would ratify it against a
vector that proves something else, on a surface that was wrong in one of three trees until today.

### The re-defined gate — adopting go's §2131 family and extending it

A **value-form** `compute/error` (a literal, **and** a lookup resolving to a stored error) routed:

1. **through each consumer** — arithmetic, compare, logic, field, index, `if`-condition, construct,
   apply — asserting **error-kind out, code preserved** (v3.23's rule, now with the representation
   that was never tested); **and**
2. **through the §7.2 reactive `result_path` crossing and SA-9 `store`**, asserting the written
   entity's **content hash is over `code` alone** — *this half is §2.4's gate and is the one this
   proposal waits on*; **and**
3. **three-way, with `GIT_COMMIT` stamped**, so the number is citable as `N·0F @ <oracle-commit>`
   per **ADR-0012**. go flagged the stamp themselves and declined to claim pinned commits without it.
   **The stamp is not why the gate is unmet — item 2 is — but a ratification number without it is
   not publishable.**

> **Why the two-representation split is the corpus's problem and not a seat's.** Both go and py had
> ~100% coverage of the *minted* error form and **0% of the value form**, independently, and neither
> could see it. **Two impls, one blind spot, one corpus** — a green board coexisting with a latent
> divergence in two of three trees is a coverage gap, not two local bugs.

**UNBLOCKED 2026-08-16 — `PROPOSAL-COMPUTE-IS-ERROR-PREDICATE` ruled (B) propagate and folded at v3.23.** **This proposal survives and NARROWS:** an error never materializes at a `compute/construct` field, so §2.4's code-only content-hash rule governs exactly the sites where an error is *written* — the §7.2 `result_path` and SA-9 `store`. **The gate vector now expects rust's outcome**; go pre-committed to flipping. Re-run before ratifying §2.4.

**Superseded blocker note (kept for the record):** The gate vector this proposal
waited on was authored and run three-way by `entity-core-go` (`71071c6`) and **does not lock**: the
three trees disagree on whether a `compute/error` reaching a `compute/construct` field is embedded
or propagated. **That is this proposal's own premise** — it asks how a *materialized* error is
hashed, and whether materialization happens at all is now the open question. Under a propagate
ruling this proposal survives but narrows to the `result_path` (§7.2) and `store` (SA-9) sites.
**Do not ratify §2.4 until the predicate lands.**

**Status:** **DRAFT — folded provisionally at `EXTENSION-COMPUTE` §2.4; the cohort gate is SENT 2026-08-15** (`ROUTING-2026-08-15-e` §1). **Stays active until the cohort answers** — this is the one open proposal whose remaining work is not arch's. It was never waiting on evidence; it waited ~4 weeks on someone to ask, which was arch's failure and is recorded as such in the packet.
**Domain:** core / `system/compute` — `EXTENSION-COMPUTE §2.4` (the `compute/error` type) + the two reactive
materialization sites (§7.2 pseudocode).
**Source of the finding:** the compute corpus's first cross-impl run (core-go, 2026-07-22) —
`ABSORPTION-compute-corpus-first-crossimpl-run`, `ARCH-RESPONSE-COMPUTE-CORPUS-FIRST-RUN §Q1`.
**Fold target:** `EXTENSION-COMPUTE §2.4/§9.1-adjacent`, version → **v3.21**.
**Status update (2026-07-23):** applied in **Go + Rust** (materialize errors code-only); **held in Python**. The
corpus LOCK (330/330 three-way) does **NOT** cover Q1: the corpus compares error **code** only and no vector yet
materializes an error *into a construct/tree*, so Q1 state differences are **latent** (real but unexercised).
**Real gate:** Python (+Rust) materialize `compute/error` code-only per this §2.4, and a corpus vector that
materializes an error *into a construct/tree* runs three-way. That is all.

**Not a gate — a phantom I withdraw.** An earlier revision routed a "sync `ENTITY-CORE-MACHINE-SPEC.md`'s
`compute/error` block (declares `message` required)" as the blocker, because Python deferred to it. That doc is a
**derived condensed summary** (its own header: `Source: … EXTENSION-*.md`, "no rationale, no history"), it is
already generally version-skewed, and the protocol repo has an open **keep-or-retire** decision on it. §2.4 is
the canonical home for `compute/error`; the machine-spec is downstream. If it is stale it should be **regenerated
from §2.4 or retired** — the protocol maintainer's hygiene, **not** a Q1 gate. Python deferring to a stale
derivative was backwards. Until the real gate above is met the §2.4 edit stays **v3.21-provisional** (stamped in
the spec).

**Fold gate (CDN-corridor meta-rule):** this is a "what dedups" change — **not validated until a materialized-error
vector runs three-way** against the amended rule. The in-place §2.4 edit is landed **provisionally**.

## The defect

`§2.4` defines `compute/error := {code, message, at?, expression?}`, and all four fields enter the entity's
content hash when it materializes. The reactive pseudocode materializes error entities directly to the tree
(`§7.2`, e.g. `{code: "cascade_limit", message: "…", at: entry.expression_uri}` → `entity_tree.put(result_path, …)`):

- `code` — deterministic (§9.1 enumerated). ✓
- `message` — "human-readable description"; nothing pins the string. ✗
- `at` — impl-dependent attribution point (`entry.expression_uri`). ✗
- `expression` — hash of *whichever* sub-expression an impl blames. ✗

So two conformant peers writing the **same** semantic error (same `code`) to the **same** `result_path` produce
entities with **different content hashes**. This breaks:

- **AE-1 materialized-boundary equivalence** for the ~40% of the input space that errors (the surface where an
  alternate engine is *most* likely to diverge — error paths are re-derived, not shared);
- **V7 content-dedup** (identical errors don't dedup);
- **cross-peer sync** (same error, different hash);
- any **reactive consumer** keyed on the result hash.

Empirically the first run confirms the surface: **135 of 328 vectors agree on `code` and differ on `message`.**
A Rule-4 / cross-peer-seam determinism defect that passed prose review.

## The rule (normative)

A **materialized** `compute/error` — written to a `result_path`, placed in a `compute/construct` field, or sent
on the wire as part of a materialized subtree (the compute→non-compute crossings, §2.3 N1) — is content-hashed
over **`code` only**. `message`, `at`, and `expression` are **diagnostic-only**: they MAY appear in the
*in-flight / dispatch-boundary* representation of an error (the §3.7 status-200 error-as-value return; a
debugging view) and MUST **NOT** be part of the bytes V7 content-addresses.

**Any materialized error field beyond `code` MUST first be proven a pure function of the IR + inputs** (not of
impl attribution).

## Why code-only is sufficient (not a loss)

No value semantics read `message`/`at`/`expression` for control flow: the §2049 short-circuit propagates by
*kind* (`is_error`); reactive handling (§2040) re-errors downstream regardless of location; frozen-subgraph
recovery (§872) clears-and-re-evals without consulting them. Two "divide-by-zero" errors **are** the same
materialized entity — correct content-addressing. Diagnostics (where/why) are served by the in-flight
representation, out of band from the deterministic tree. This is the error-side of the existing "only the
materialized form is normative; the in-flight representation is implementation-private" rule (§2.3 N1).

## Fold delta (landed in place, provisional)

1. `§2.4` — type comment marks `message`/`at`/`expression` `optional`, diagnostic, not materialized; new
   normative paragraph (the rule above). ✅ landed (`fold` commit).
2. `§7.2` reactive pseudocode (the two `error_entity` sites) — materialize `{code}` only. ✅ landed.
3. `GUIDE-CONFORMANCE §7c.3` — the error boundary is `code`-only; the gate's `code`-comparison *is* the
   materialized-error-hash comparison. ✅ landed.

## Route

- **Cohort:** re-materialize `compute/error` as code-only; re-run the corpus's error vectors three-way. This
  gives **AE-1** a well-defined error boundary (its fold was gated on the corpus, `PROPOSAL-COMPUTE-ALT-ENGINE-ADMISSION`).
- **On three-way green:** flip `EXTENSION-COMPUTE` header to v3.21 (Source += this proposal, implemented).
