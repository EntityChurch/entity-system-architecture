# PROPOSAL — the lowering toolkit as a specified layer-2 surface

**Status:** DRAFT (2026-07-21)
**Domain:** compute authoring / `GUIDE-CORE-COMPUTATIONAL-ARCHITECTURE §8–§9` (the layer above the IR-assembly SDK)
**Scope:** promote the built-in-Go **lowering toolkit** (`entity-workbench-go/entitysdk/compute_lower.go`, Phase H.2) from a single-impl artifact to a **specified cross-impl layer-2 contract** — the reusable functional decompositions a language frontend compiles down to.
**Depends on:** the v3.19 compute value model (settled, three-way ratified — `GUIDE-CORE §11`); the S1 builder reference design (built, F1–F8 closed).
**Folds to:** `GUIDE-CORE-COMPUTATIONAL-ARCHITECTURE §8` (the authoring stack), `§9` (lowering patterns — its worked content), and `§11` (moves the toolkit line from *open* to *specified, Go-built, cross-impl-unmirrored*).
**Tracked as:** W-COMPUTE (authoring track). **Precedes:** any DSL/frontend (`ANALYSIS-COMPUTE-CONSTRUCTION-AND-EXECUTION-READINESS` Rec A — "finish the toolkit before the language").

---

## §1 Problem — the toolkit exists, but only as Go, and the guide still calls it unbuilt

`GUIDE-CORE §8` names the high-leverage product precisely: **not a language — the lowering toolkit**, the reusable decompositions ("what a language would need, factored out of any particular language"). `§11` lists it under *Open — the next substantive build*.

That framing is now **stale on the build axis and unmet on the spec axis**:

- **Built + tested — in one impl.** `compute_lower.go` (Phase H.2) ships nine decompositions with a 29 KB test file (`compute_lower_test.go`); F9 (closure-over-scope) and F11 (builtins arg-name friction) are closed inside it. So the toolkit is not "unbuilt" — it is **built and tested in Go, single-impl.**
- **Unspecified as a cross-impl surface.** There is no arch document that pins *which* decompositions are the surface, *what IR each must produce*, and *which IR evaluation rules each must encode* — independently of the Go builder's ergonomics. Rust/Py have no toolkit and nothing to mirror against.

The correct next step is therefore not "build the toolkit" (done in Go) and not "write a DSL" (premature) — it is **specify the toolkit's contract** so a second impl can reproduce it and so its outputs become conformance vectors. This is the same discipline the whole compute arc runs on: **pin the contract, free the construction.**

## §2 The contract split — what is pinned vs. what is impl-local

A lowering is a function `(source construct) → compute IR graph`. Two halves, pinned differently:

- **PINNED (cross-impl-observable): the decomposition → IR contract.** For a given source construct and inputs, every conformant toolkit MUST produce a graph whose **materialized-boundary entities are byte-identical** — the same rule that governs alternate engines (`PROPOSAL-COMPUTE-LOWERING-CONTRACT` L-1) and preemption (`…-BUDGET-PREEMPTION` BP-1). Because compute IR is content-addressed and `compute/apply` args are canonically sorted (length-then-lex, §4), two toolkits that make the same decomposition choices produce **hash-identical IR** — the contract can be pinned at the *graph hash*, not merely at eval output.
- **IMPL-LOCAL (free): the builder surface.** Free functions vs. methods, `*ComputeBuilder` vs. some Rust equivalent, error ergonomics, naming — none of it crosses a peer boundary. `compute_lower.go`'s "free functions layered above S1, not methods" choice (its own header) is a Go style call, **not** part of the contract.

Consequence: the spec is a **table of decompositions + the invariants each encodes**, plus a **worked-lowering vector set**. It is emphatically *not* a Go API port.

## §3 The specified surface — the nine decompositions

Grounded in the H.2 code; the **contract** column is the cross-impl-observable obligation, the **encodes** column is the IR evaluation rule the decomposition MUST get right (the §4 catalog).

| Decomposition | Source construct → | IR it MUST produce (contract) | Evaluation rule it encodes |
|---|---|---|---|
| **arithmetic** | a host `+`/`−`/`×`/`÷`/`mod` over signed/unsigned operands | `compute/arithmetic`; for unsigned `div`/`mod` a `numeric-cast → primitive/uint` **inlined at the operand site** | Rule 11 — cast is the direct operand, never a `let` binding (§4a) |
| **compare** | a host ordered comparison | `compute/compare`; unsigned wraps each operand in the operand-site cast | Rule 11 (same as arithmetic) |
| **fold** | accumulate over a collection | `BuiltinsCall("fold", {collection, initial, fn:Lambda})` | lambda-as-expression, not pre-evaluated closure (§4b) |
| **filter** | select from a collection | `BuiltinsCall("filter", …)` with the uniform collection/lambda shape | lambda-as-expression; builtins arg-name normalization (F11) is impl-local |
| **map** | transform each element | `BuiltinsCall("map", …)` | lambda-as-expression |
| **recurse** | a self-referential loop / recursion | a `compute/lambda` at a tree path + an `ApplyClosure` invoking it; resolves the fixpoint-bootstrap knot | tail-position calls iterate w/o stack growth (TCO); the lambda is stored, self-referenced via `lookup/tree` (§4c) |
| **match** | a sum-type / variant discrimination | an `if`/`eq` chain over a `.data` tag field read via `field(value, tagField)` | tag-in-`.data` (there is no `union_of`; `field` reads only `.data`, not entity `type`) (§4d) |
| **record** | build a structured record | a `compute/construct` — IR identical to the equivalent hand-written builder call | `construct` materializes to a **bare entity** on leaving compute; field-key canonicalization on CBOR encode (§4e) |
| **numeric-intent** | thread a source type's signedness | (not a graph — the parameter that selects signed vs. unsigned in the four ops above) | the signed-default op set (`div`/`mod`/`compare`); sign-agnostic ops (`add`/`sub`/`mul`) emit bare IR |

The set is **closed for v1** at these nine — they cover conditional, iteration (map/filter/fold), recursion, pattern-match, data access, construction, and integer semantics: the `§8`/`§9` enumeration of "what a language needs." A tenth (data-navigation chains `field`/`index`/`length`) is a **composition of primitives, not a new decomposition** — the toolkit threads it through the above; see §4f.

## §4 The invariants every decomposition MUST encode

The load-bearing content — "the lowering toolkit must understand the IR's **evaluation rules**, not just its node types" (`§9`). A second impl that gets the node types right and these wrong produces boundary-divergent IR that prose review will not catch (the CDN-corridor meta-rule). Each is a MUST on any conformant toolkit:

- **(a) Rule 11 — cast at the use site.** Unsigned `div`/`mod`/`compare` reach unsigned semantics only when `numeric-cast → primitive/uint` is the **direct operand**. A cast bound in a `let` or crossing an `if` branch **silently reverts to signed-default**. The S1 builder enforces this by rejecting a `NumericCast` binding at build time; a conformant toolkit MUST inline, never bind.
- **(b) Lambdas lower to `compute/lambda` *expressions*, never to `compute/closure` values.** A closure is a runtime artifact; storing one as a builtin argument is non-portable. Pass the lambda expression's hash to `map`/`filter`/`fold`.
- **(c) Recursion via stored lambda + `lookup/tree`, tail-position for iteration.** The fixpoint bootstrap (needing the hash to build the body that references the hash) is the toolkit's job to resolve; tail calls MUST iterate without stack growth.
- **(d) Discriminate by `.data` tag; there is no `union_of`.** `match` reads `field(value, tagField)` (`.data` only) and branches by `if`/`eq`; a variant is an entity carrying that tag. No entity-`type` discrimination, no runtime `compute/match` primitive (deferred — `PROPOSAL-COMPUTE-RECURSION-AND-SUM-TYPES §3`; exhaustiveness is a frontend concern).
- **(e) `construct` materializes to a bare entity; keys canonicalize on encode.** A lowered `construct`, once it leaves compute, is byte-identical to a hand-built bare entity (fields → bare `system/hash` refs, no kind-tags — `EXTENSION-COMPUTE §2.3`, v3.19c α). This is what keeps lowered output dedup-correct and interoperable.
- **(f) Navigate by `kind`, never by sniffing keys (N3); follow a materialized hash explicitly.** Data-access composes (`field`/`index`/`length` to arbitrary depth); disambiguation is by the value's `kind`, threaded on in-flight values. Reading a `system/hash` field back out of a materialized entity returns **the hash** — the toolkit MUST emit an explicit `lookup/hash`, never auto-resolve, and never size a hash by fixed byte length (variable-length, V7 §1.2).
- **(g) Canonical `compute/apply` arg sort (length-then-lex).** Identical logical expressions built in any argument-insertion order MUST hash identically — this is what makes Stage-2 memoization correct. A toolkit on the standard CBOR encoder gets it free; a hand-rolled encoder MUST replicate it.

## §5 Conformance — the toolkit's output *is* corpus material

A worked lowering is exactly a `(source construct, inputs) → materialized-boundary-hash` triple — i.e. a **row of the portable cross-impl compute conformance corpus** (`ANALYSIS-…-READINESS` Rec C; `WORKSTREAMS.md` cross-cutting). So this proposal and the corpus are the same deliverable seen from two ends:

- Each of the nine decompositions contributes a **worked-lowering vector** (source + expected boundary hash) that go/rust/py toolkits must reproduce. The nine × the §4 edge cases (nested arrays, ragged collections, deep `field`/`index` chains, empty/heterogeneous inputs, unsigned-at-use-site, materialized-hash read-back) is the **authoring sweep** `§11` calls the still-open half.
- Because the contract is pinned at the **graph hash** (§2), a toolkit conformance test is stronger than eval-equivalence: it asserts two toolkits *build the same IR*, not merely that the IR evaluates the same — catching a divergent decomposition before it ever runs.

This is the concrete first content of the corpus: the toolkit vectors are portable by construction (IR + inputs + hash, no Go).

## §6 Honest ledger

- **Built + tested — Go single-impl (workbench-go H.2).** Nine decompositions, 29 KB tests, F9/F11 closed. This is **not** cross-impl evidence — one author's toolkit passing its own tests is cohort-consistent-at-best, and here it is a cohort of one.
- **Unbuilt:** any Rust/Py toolkit; the graph-hash conformance harness. Nothing here is validated across impls yet.

  > **CORRECTED 2026-08-13 — this bullet listed "the portable vector set (the sweep)" as unbuilt. It is
  > built.** `entity-core-go` `cmd/internal/compute-corpus/worked.go` (source-read at `7c3871f`) is the §5
  > worked-vector tranche, and it names this proposal: *"start with the nine lowering-toolkit worked lowerings
  > (`PROPOSAL-COMPUTE-LOWERING-TOOLKIT` §5), then the random sweep"*, with `builder.go` citing §2's
  > pinned-contract / free-builder split. Every vector there names the §4 invariant it pins, *"because a vector
  > that does not pin an invariant is a vector nobody can act on when it goes red."* **So §5/§7's "the sweep
  > completes it" is no longer the open half of this proposal — it is the landed half, in one impl.** Reported
  > by core-go (`ROUTING-2026-08-13-l`), which also corrects §3's framing: `compute/{arithmetic,closure,compare,
  > lambda,match}` **are** landed; only the **builder/toolkit surface** is owed.
  >
  > This is an honest-ledger section that went stale, which is the failure mode the section exists to prevent.
  > A build-state bullet with no `(repo, commit, date)` cannot expire, so it never did.
- **What ports is the contract (§3/§4), not the code.** The Go free-function surface is an ergonomics choice; a second impl satisfies this spec by producing hash-identical IR, however it is structured.

## §7 Non-goals & open

- **Not a DSL / frontend.** This is the layer *below* any surface syntax (`§8` item 2, not item 3). A parser targets this toolkit; designing one is out of scope and premature until the sweep lands (Rec A).
- **`compute/match` primitive + `compute/type-of` + `union_of`** stay deferred (`…-RECURSION-AND-SUM-TYPES §3`, named re-open triggers); v1 `match` is the `if`/`eq` decomposition.
- **The authoring sweep** (`§11` open) is the test-obligation half of this proposal — the edge-case vectors over the value model. Ratify the surface (§3/§4) now; the sweep completes it and populates the corpus.
- **Open-Q1:** does `numeric-intent` belong as a toolkit parameter (as in Go) or as distinct signed/unsigned entry points? Contract-invisible (both produce the same IR); leave impl-local, note it.

---

*Authored in the arch workspace. Ratifiable via proposal → ratify → fold into `GUIDE-CORE §8/§9/§11`. The specified surface (§3) + invariants (§4) are the ratifiable unit; the sweep (§5/§7) completes it and is the first tranche of the portable compute conformance corpus. Recurring lesson: pin the decomposition→IR contract, free the builder.*
