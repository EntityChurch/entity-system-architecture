# ANALYSIS — compute construction & execution: DSL readiness, Axis-1 guidance, and the one gap both share

**Status:** Analysis / review — 2026-07-21. **NOT** a proposal. Answers two operator questions with a
cross-repo survey (cited), and finds they converge on a single missing artifact. Author: arch workspace.

**The two questions.** (1) *Is the compute system ready for a **DSL**, or keep the hand-authored builder
structures Go/Rust use to build the compute subgraph?* (2) *Does the **Axis-1** interpreter need spec
guidance, or is it left to implementers?*

**The finding in one line.** Neither a DSL nor an Axis-1 representation-spec is the right next move — both
would be premature — and the two questions **converge on the same gap**: there is **no portable cross-impl
conformance corpus** pinning *same IR → byte-identical materialized boundary* across go/rust/py. That one
artifact is the highest-leverage thing to build, and it is the silent dependency under transferable compute,
alt-engine admission, and the sustained-parallel-load gate.

**Survey provenance (read-only, cited).** `entity-core-{go,rust,py}`, `entity-workbench-go`,
`entity-browser-rust`, and this repo's `guides/` + `specs/`. Every impl claim is pinned to `(symbol,
path:line)`; absences were re-run scoped per-repo (a whole-tree grep silently skips `entity-workbench-go`
on this machine — noted so no false-absence slips in).

---

## Part A — Authoring: not a DSL; the toolkit is already emerging; the divergence risk is sidestepped

### A1 — Current state (what actually exists)

- **One fluent builder, and it is workbench-go-only.** `ap.Compute()` → `ComputeBuilder`
  (`entity-workbench-go/entitysdk/compute_builder.go`, 703 LOC): `Literal`/`LookupTree`/`Arithmetic`/
  `Apply`/`Let`/`Lambda`/`Construct`/… each emitting one IR node, `.Build()` doing a post-order put. It
  **already encodes the hard IR rules**: `Let` rejects a `NumericCast` binding at build time (Rule 11),
  `Apply` enforces the F5 capability ceiling, and hashing is canonical-order-stable (`sortedMapKeys`).
- **The core impls have no fluent builder** — Go/Rust/Py expose raw `ComputeXxxData.ToEntity()` structs
  (`entity-core-go/core/types/compute.go`); **Rust and Py have *no authoring surface at all*** — their
  compute crates *evaluate*, never author (`entity-core-rust/extensions/compute` is the evaluator;
  `entity-core-rust/bindings/sdk/src/compute.rs` `ComputeOps` only `eval`/`install`/`list`).
- **The lowering toolkit already exists — in prototype.** `entity-workbench-go/entitysdk/compute_lower.go`
  (577 LOC): `LowerMatch`, `LowerFold`, `LowerFilter`, `LowerMap`, `LowerRecurse`, `LowerArithmetic`,
  `LowerCompare`. So GUIDE-CORE §8's "layer 2 (the lowering toolkit) is the layer to design next" is
  **partly built**, not merely framed. The guide's own roadmap flags the builder as
  "planned-not-yet-specified" (`GUIDE-COMPUTE-PROGRAMMING §2.4`); workbench-go is its first implementation.
- **No DSL / parser / text-format / s-expression exists anywhere** (per-repo search: only the unrelated
  `query` extension parses). Hand-assembly via the typed builder is the only authoring path; Rust/Py drop to
  raw struct construction.

### A2 — "Hand-authored structures Go *and* Rust use" is a slight misframe: only Go authors

Author-once is **settled architecture**, not an open choice. `entity-workbench-go/workbench/
program_authoring.go` is explicit: *"Author\*() runs ONCE and leaves (step IR + projections + state₀ +
descriptor) in the tree. Mount() reads the descriptor and runs the program … A Rust host fetches (descriptor
+ step IR + initial\_state) BY HASH and cannot run buildAsteroidsStepExpr."* Confirmed from the other side:
`entity-browser-rust` "implements `Mount` only — never `Author`. We do not need to port the 458-line
Asteroids IR builder" (`REVIEW-COMPUTE-GENERIC-HOST-BROWSER-2026-07-19`). Rust's only relationship to the
game programs is **oracle-replay of the Go-authored artifact** (`entity-core-rust/…/eval/tests.rs`:
"Go-oracle replay of workbench's Asteroids"). So: **Go authors; Rust/browser mount and evaluate.** The old
"reconstruct in Go at every boot" anti-pattern (the generic-host §1a concern) is **already retired.**

### A3 — DSL readiness: not yet, for three principled reasons

1. **The toolkit (layer 2) must precede a frontend (layer 3), and it is the layer that encodes the *hard
   rules*.** A DSL is a parser that produces IR; if it bypasses the toolkit it will get Rule 11 (no cast
   through a `let`), the lambda-lowers-to-expression rule, and canonical-order hashing **wrong** — exactly
   the traps the builder already guards (`compute_builder.go`, GUIDE-CORE §9). The value is the reusable
   decomposition layer, not the surface syntax; build/finish that, and any frontend is cheap.
2. **The IR/stdlib is still moving.** `range` and `array-concat` are pending (`PROPOSAL-COMPUTE-COLLECTION-
   PRIMITIVES`), `group_by` is out of scope, `compute/match` is deferred (library `LowerMatch` for now),
   sum-types are library-only (`GUIDE-CORE §11`). A DSL authored now targets a moving target; the toolkit
   absorbs stdlib growth without a syntax rev.
3. **Authoring is single-impl by design, so a DSL is *not* a cross-impl concern** — it is a Go-side frontend
   onto the one builder. The guide already frames it this way: "when [a language] lands, the bottom-up
   [builder] pattern is what it will compile to" (`GUIDE-COMPUTE-PROGRAMMING §2.4`).

> **Recommendation A.** **Do not build a DSL yet.** Keep the typed builder as the authoring floor;
> **formalize and complete the lowering toolkit** (`compute_lower.go` → a specified layer-2 surface with
> worked lowerings + tests, the `GUIDE-CORE §11` open item) — that is where authoring leverage actually
> compounds. A DSL is layer 3, appropriate *after* the toolkit stabilizes and the stdlib (`range`/`concat`/
> `match`) settles. The ergonomic friction is real (Asteroids' step is ~520 LOC; "hand-assembled ~25 LOC,
> should be ~5" — `compute_builder_test.go`), but the builder already captured most of that win; the
> residual is a *toolkit* gap, not a *language* gap.

### A4 — The cross-impl authoring-divergence risk: sidestepped today, forward-looking

The hazard I went in worried about — two impls' builders producing *different* content hashes for "the same"
program — is **currently sidestepped, not solved**: only Go authors, so there is no competing builder.
Within Go, determinism is guaranteed by canonical CBOR + sorted args (`compute_builder.go` pitfall #2; the
Life test asserts builder output is byte-identical to a hand-built entity). *If* a second authoring impl is
ever added, the mitigation is already named — "mirror the S1 reference shape" (`GUIDE-CORE §11`) + rely on
the standard canonical encoder (`GUIDE-CORE §9`: "a toolkit relying on the standard encoder gets [identical
hashes] for free; one hand-rolling encoding must replicate it"). But the deeper issue is not *authoring*
divergence — it is *evaluation* divergence, which is Part C.

---

## Part B — Axis-1: leave the representation free; pin the admission contract

### B1 — What Axis-1 is, and why the representation is correctly unspec'd

Axis-1 (`entity-workbench-go/entitysdk/axis1/`, **Go-only**) is a **decode-once resolved-node walker**: an
`Engine{cache map[hash → node]}` keyed by IR content hash, resolving each expression once into a node graph
with child pointers and pre-parsed literals, then walking it. Its speed comes from **lexical-slot addressing**
— a compile-time `compileLevel.resolve(name)→(depth,index)` (de-Bruijn coordinates) replacing Stage-1's
per-node name-keyed `LoadScope`, so a scope read becomes a pointer-hop + array index. Measured **34–163×/tick**
(the `LoadScope` cost collapses O(N²)→O(N)).

The representation is **deliberately unspec'd**, and that is correct: `GUIDE-CORE §6`'s principle —
*"determinism is pinned at the materialized boundary … the per-node content-addressing the interpreter uses
internally is an interpretation artifact the backend drops"* — is exactly what frees the resolved-node graph,
the slot layout, and the decode cache to be per-language. **The representation stays implementer's choice.**

### B2 — But Axis-1 has stopped being isolated

Three dependents now name it as the assumed base layer — and two of them are **this session's own
proposals**: determinism (the equivalence oracle), preemption (`PROPOSAL-COMPUTE-BUDGET-PREEMPTION` — the
explicit-state host for mid-expression suspension), lowering (`PROPOSAL-COMPUTE-LOWERING-CONTRACT` — "Axis-1
first, then Stage-3"; fusion-region discovery over the resolved-node graph). So the thing that needs
guidance is **not the representation** but the **contract** those dependents rely on.

### B3 — What needs spec guidance: the execution-strategy admission contract

Axis-1 (a faster interpreter) and a Stage-3 compiled handler are **both "alternate engines,"** and they need
the **same** admission rule:

> **An alternate execution strategy is conformant iff it is *materialized-boundary-equivalent* to the
> reference interpreter** — same IR + same inputs ⇒ **byte-identical materialized-boundary entities** (state
> writes, `construct`, `apply` args, output ports) — never intermediate state, never content-store contents
> (Axis-1 legitimately leaves *fewer* entities in the store). Plus the invariants that make that hold across
> engines: **impure-frontier preservation** (no reordering/eliding an effect), **laziness** (only the
> demanded subgraph runs), and **metering at the boundary** (the op-budget is over logical graph steps, not
> engine steps).

This is **the same contract** as the lowering proposal's **L-1**, generalized: L-1 covers a compiled backend;
this covers a fast interpreter; they are one admission rule over "any engine." It already exists as a
first-class Go harness (`TestAxis1Equivalence_Differential` — 300 random graphs, canonical-CBOR boundary
comparison) and is **routed to become an `EXTENSION-COMPUTE` appendix** (the handoff's §12.5/§13.9 admission
contract). That routing is correct; this analysis endorses it and notes it is the **same appendix** the
lowering contract should fold into (one admission contract for all alternate engines, not two).

**Landed as a proposal (2026-07-21):** `PROPOSAL-COMPUTE-ALT-ENGINE-ADMISSION.md` carries the ready-to-fold
appendix text — **AE-1** (this rule, generalized over any engine, with the content-store-inequality carve-out
made explicit) + **AE-2/3/4** (impure-frontier / demand / metering) — absorbing lowering's L-1 and this Part-B
contract. Fold into `EXTENSION-COMPUTE` is gated on the corpus (Part C / Rec C).

### B4 — The lexical-slot edge case is handled — and it teaches the meta-lesson

The determinism trap I expected (lexical slots assume lexical scope) **is handled**: an expression fetched at
eval time via `lookup/tree`/`lookup/hash` has no lexical relation to its enclosing graph, so Axis-1 decodes
it with **no lexical context** (every `lookup/scope`→`rootNode`) and evaluates it against a flattened live
scope (`tailIntoDynamic`/`frame.flatten`). Lexical slots apply **only** to genuinely lexical `let`/`lambda`.
The telling detail: the code comments this as a correctness point that *"no Life/Snake test would notice."*
**That is the CDN-corridor meta-lesson** (`AGENTS.md`): a subtle equivalence bug that toy programs pass while
it hides. So the admission contract MUST be exercised by a **differential corpus of random well-formed
graphs** (as Axis-1's Go harness already is), **not** by the game programs — the games are necessary but
categorically insufficient.

> **Recommendation B.** (1) **Leave the Axis-1 representation impl-local** (resolved-node/slots/decode cache
> — per-language, per `GUIDE-CORE §6`). (2) **Pin the execution-strategy admission contract** as an
> `EXTENSION-COMPUTE` appendix (boundary-equivalence + impure-frontier/laziness/metering), *shared* with the
> lowering contract's L-1 — one rule for all alternate engines. (3) Provide **RECOMMENDED** (not MUST)
> interpreter-architecture guidance: an *explicit-state / resolved-node* evaluator is what makes preemption
> and fusion tractable — named so the cohort's fast interpreters converge on a shape those features can lean
> on, without mandating the representation. (4) **Honestly track** that preemption + lowering currently
> depend on a single-impl (Go), *partially* defunctionalized (tail-position only), experiment-tier Axis-1
> with no cross-impl counterpart — done: both proposals now carry that dependency note.

---

## Part C — The convergence: both questions bottom out in one missing artifact

Part A found *"cross-runtime state-hash agreement remains unproven — the corpus does not exist anywhere"*
(`REVIEW-COMPUTE-GENERIC-HOST-BROWSER-2026-07-19`). Part B found *"the equivalence oracle is Go-internal;
there is no portable cross-impl corpus."* **These are the same gap.**

> **There is no portable, cross-impl compute conformance corpus** — a language-neutral set of
> `(IR, inputs) → materialized-boundary-hash` vectors that **go, rust, and py all run and all pass.** Today
> there is exactly **one** ratified cross-impl fixture (a `compute/scope` canonical-encoding anchor,
> `ecf-sha256:3edc5138…`) plus a Go-internal differential harness. That is the entire cross-impl evidence
> base for compute determinism.

This one artifact is the **silent dependency under three of this session's threads**:

| Thread | Its unstated assumption | What the corpus proves |
|---|---|---|
| **Transferable compute** (bridge host, generic host, W-HOSTING) | "author once in Go, fetch by hash, **Rust eval == Go eval**" | that a mounted program produces identical boundary hashes on every runtime — the entire point of transfer-by-hash |
| **Axis-1 / alt-engine admission** (this doc, Part B) | "a fast interpreter is still conformant" | that an alternate engine is boundary-equivalent — the admission contract's conformance evidence |
| **Sustained-parallel-load gate** (W-LOWERING) | "Axis-1(-or-equivalent) exists on Rust/browser" | that a Rust/browser fast interpreter is conformant *before* we rely on it to sustain load |

All three quietly assume cross-runtime evaluation equivalence that **is not proven**. The bridge/generic-host
docs even flag it ("would rather ship a host that reports a hash mismatch loudly than one that assumes
agreement") — the right instinct, but it means the transferability thesis is **honest-but-unverified**.

> **Recommendation C (the headline).** The **single highest-leverage next build is the portable cross-impl
> compute conformance corpus** — port Axis-1's Go differential generator (random well-formed graphs →
> canonical materialized-boundary hashes, with the anti-vacuity guards: closures present, ≥N distinct error
> codes, no fallback) into a **language-neutral vector file** that go/rust/py each execute in CI. It is worth
> more than a DSL, more than Stage-3 lowering, and more than further authoring ergonomics, because **every
> one of those depends on it** and none of them is trustworthy without it. It is also the concrete
> conformance artifact the Part-B admission-contract appendix needs to have any teeth.

---

## Recommendations & routing (summary)

1. **A — No DSL yet.** Finish/formalize the **lowering toolkit** (layer 2; `compute_lower.go` → specified
   surface + worked lowerings, the `GUIDE-CORE §11` open item). DSL is layer 3, after the stdlib settles.
   *(Owning track: compute-standardization / W-COMPUTE authoring.)*
2. **B — Axis-1: free the representation, pin the contract.** Land the **execution-strategy admission
   contract** as an `EXTENSION-COMPUTE` appendix, *unified* with the lowering contract's L-1 (one rule for
   all alternate engines). Representation stays impl-local; add RECOMMENDED explicit-state guidance.
   *(Owning track: W-LOWERING + the compute-extension; fold L-1 into it.)*
3. **C — Build the portable cross-impl conformance corpus.** The gating artifact for transferable compute,
   alt-engine admission, and the sustained-load gate. **Route to the cohort as a first-class build** — it is
   the prerequisite the bridge/generic-host transferability thesis and both compute proposals silently rest
   on. *(Cross-cutting: gates W-HOSTING, W-LOWERING, W-BUDGET's cross-impl determinism.)*

**Net:** the recurring shape across both questions is **"pin the cross-impl-observable contract; leave the
construction and the representation impl-local."** Authoring construction (the builder/DSL) and interpreter
representation (Axis-1) are both correctly implementer's choice; what is under-pinned is the **contract** —
and the corpus is the evidence that makes the contract real.

*Review authored in the arch workspace at the operator's request; the ratifiable follow-ons are (B) the
execution-strategy admission appendix and (C) the conformance corpus — routed, not folded here.*
