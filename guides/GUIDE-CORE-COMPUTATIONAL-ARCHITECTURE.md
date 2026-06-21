# Core Computational Architecture

**Tier:** Foundation. The "why and how" of computation in the entity system — the umbrella over `GUIDE-COMPUTE.md` (the expression-language reference) and `GUIDE-COMPUTE-PROGRAMMING.md` (the SDK authoring surface). Read this first for the model; read those two for the language and the API.

**Status**: Active

**Spec reference:** `EXTENSION-COMPUTE.md` v3.19c (the value model: kind-tagged scope bindings, navigate-by-kind, construct-materializes-bare, error-as-value at 200); `ENTITY-CORE-PROTOCOL.md` §6.6 v7.49 (dispatch uniformity), §3.7 (entity-native handlers), §5.8 v7.50 (cross-peer closure transfer, registered).

---

## 1. The one frame

> **The entity system is a distributed compiler. Compute is its portable, content-addressed intermediate representation (IR).**
>
> **Frontends** (host-language builders, pipeline/query DSLs, eventually source languages) → **compute IR** → **the gradient backend** (interpreted now; native per local machine later).

Everything in the compute arc is a facet of this single sentence. The rest of this guide unfolds it: what makes compute an IR (§3), how work is placed against it (§4), how the positions compose (§5), how the backend compiles it (§6), what the spec must pin for it to travel (§7), how you author it — from hand-assembly up to languages (§8–§9), and how to choose (§10).

The framing is not an analogy borrowed from LLVM. It is the operational mechanism: a transferable description (the IR, living as content-addressed entities in the tree) is locally expressed as activity (evaluation, eventually native), and the description persists for re-expression elsewhere. Frontend / IR / backend = the transcription / translation / expression of a genome.

---

## 2. Why an IR at all

A handler is work. The system has three ways to place work:

- write it in the host language (a compiled or SDK-registered native handler),
- agree on it across implementations (a named, spec-contracted operation every peer provides identically),
- or express it *as data that any compute-capable peer can run* (a compute expression).

The third is the interesting one, because it is the only one that lets computation **move between peers and run identically on arrival**. That capability is what an IR buys you, and it is the reason compute exists as a layer rather than as "a little expression evaluator." Once computation is a transferable artifact, the system stops being "peers that talk" and becomes "peers that can send each other programs" — a distributed compiler whose object code is the tree.

---

## 3. Compute *is* the IR

A compute graph is an intermediate representation — but unlike LLVM IR or WASM bytecode (opaque, in-memory or binary blobs), this IR is **content-addressed entities in the tree**. That one difference *is* the system's character. The IR is:

- **Transferable** — an expression is an entity; send it, it runs elsewhere (this is the genome).
- **Memoizable** — content-addressed, so compile-time collapse and result caching are free and structural: identical structure ⇒ identical hash ⇒ valid reuse, with no separate proof layer.
- **Inspectable** — you can read it, render it, audit it, diff it, like any other entity.
- **Reactive** — its impure edges (`lookup/tree`) *are* dependency edges; the same graph that computes also says what to recompute.

**It is typed-functional, not "simplified LLVM."** Compute belongs to the GHC-Core / CPS family, not the SSA-basic-block family (the IR-landscape cross-check confirmed this against LLVM/WASM/MLIR/JVM/Cranelift/GHC-Core). Content-addressing makes SSA's versioning unnecessary — the hash *is* the version. What looks "missing" relative to an imperative IR is the family's shape, not a hole:

| You might expect | Compute's form |
|---|---|
| loops | recursion + tail-call optimization (TCO) |
| early exit / exceptions | error-as-value (errors propagate like NaN; §7.2 of the spec) |
| mutable variables | `let` bindings + scope |
| arbitrary CFG | nested `if` / `apply` (CPS-transformable) |
| basic-block versioning (SSA) | content hash |

Understanding this family shape is the difference between fighting the IR and lowering to it cleanly (§9).

---

## 4. N / S / T — the language's own structure (and the two axes)

The three ways to place work (§2) are the language's own internal structure — IR / standard-library / FFI:

- **T — compute** is the **IR**: portable, content-addressed, runs on any COMPUTE-bearing peer.
- **S — a named cross-impl operation** is the **standard library**: a spec-contracted function every implementation provides identically (`revision:pull`; the collection builtins once pinned). libc-for-compute.
- **N — a native handler** is the **escape into the host language**: I/O, host libraries, SIMD, hot paths — reached via the `compute/apply` drop-down (§5). Loosely "FFI" by analogy, but *not* a cross-runtime ABI crossing: a native handler is a same-process host function called over a Go/Rust/Python interface; the boundary is entity↔host-type conversion, so the drop-down is cheap and frequent, not the costly crossing "FFI" connotes.

### The two axes (terminology, pinned)

Two orthogonal coordinates describe any piece of work. They have been used loosely in the exploration record; this guide is the place they are stated cleanly. **No new names — just the relationship made precise.**

- **Transferability — how the work moves: `N` / `S` / `T`.** Where the work lives and how it reaches another peer. `N` doesn't move (host-local). `S` moves *behaviorally* (every peer implements the same contract). `T` moves *as data* (the IR travels and runs). **`B` is not a fourth class** — it is the **bridge**: the COMPUTE extension itself, the peer capability that makes `T` possible. "Class B" in older notes means "this peer carries the compute bridge," a property of the peer, not a placement of the work.
- **Convergence / visibility — how inspectable the work is: `1` … `4`.** Class 1 = fully convergent and inspectable (pure compute, every step visible); Class 4 = opaque (a native body you cannot see into). This is the **visibility↔speed dial**, and the gradient (§6) is that dial as a process.

A native handler is **Class N** on the transferability axis **and Class 4** on the convergence axis — two coordinates, not synonyms. The earlier "Class 4 ≈ native" shorthand conflated them; read "Class 4" as *opacity* and "Class N" as *placement*. Pure interpreted compute is Class T / Class 1. A compute handler compiled to native (§6 Stage 3) stays Class T (the source IR still travels) while sliding toward Class 4 on visibility — which is exactly what the dial means.

---

## 5. The drop-down idiom — first-class, not an escape hatch

The three positions are rarely used in isolation. The **dominant practical pattern is their composition**, and it is first-class:

> **The drop-down: a compute (IR) expression uses `compute/apply` to dispatch to a native handler for what is effectful, unbounded, or performance-critical, and threads the result back through the IR. Compute owns the plumbing; native owns the work; `compute/apply` is the seam.**

Experiment C1 measured it: ~3 IR entities of glue around a Go function versus ~25–30 for the same operation in pure compute via cons-cells. This is *where the bulk of real handler work lands*, and it stays the idiom for effects / FFI / hot paths even now that collection navigation has landed (which closed the *pure list-transform* cases, not the effectful ones). It is also how Class S handlers are built internally — `revision:pull` is a spec-contracted op whose *implementation* drops to native for the network/trie work.

No new primitive is needed for this: `compute/apply` over a registered handler (or a `system/compute/builtins/*` path) already is the seam. An SDK authoring surface must make this seam frictionless on day one (§8). One wrinkle the SDK must absorb: a handler-mode `compute/apply` returns its result wrapped in `compute/result {value, expression}` (the SA-4 contract), so consuming the inner value in further compute means a `field("value", …)` (or an unwrap helper) — see GUIDE-COMPUTE-PROGRAMMING §2/§7.

---

## 6. The gradient is the backend

How fast compute runs is a **backend** question, and the backend is a gradient — the same IR, compiled progressively further down per local machine:

| Stage | What it is | Status |
|---|---|---|
| **1** | Interpreted entity-graph walk. Always works; baseline correctness. | **Today** (`entity-core-go/ext/compute`). |
| **2** | Content-addressed memoization at the key/lookup layer — "free and structural." Pure sub-expressions collapse once across all dispatches; reactive recomputation skips unchanged subtrees. | Unbuilt (core-go track). |
| **3** | Native compilation per architecture — compile a handler's IR once at register time, run native per dispatch (the chain-step payoff, where Stage 2's top frame misses by design). | Unbuilt. |
| **4** | Machine code. | Future. |

Three load-bearing properties hold across the gradient:

- **Compilation is optional.** Interpreted Stage-1 compute always works; nothing forces anyone down the gradient. "Transferable IR that always runs as-is" is the floor; "compiles to fast native locally" is the upside. The build track must never make the upside a precondition of the floor.
- **The four-way coincidence at the impure boundary.** Four concerns — purity (*what can collapse?*), compilation (*what stays runtime?*), reactivity (*what edges re-trigger?*), visibility (*what's inspectable?*) — all anchor on the *same* impure operations (`tree.get`/`tree.put`/`dispatch`/`check_permission`). That co-location of *position* is real and load-bearing; it is **not** an identity of *meaning*. The gradient can decouple them: a compiled interior is opaque (visibility lost) yet still deterministic (purity preserved). The consequence is clean — pure interiors collapse to native, the impure skeleton stays, and **reactivity is invariant under compilation** because the reactive edges *are* the preserved boundary. The partition compute already makes by lookup type (`scope` pure / `tree` impure-reactive / `hash` pure-fixed) *is* the visibility frontier.
- **Preserve the IR in the tree (load-bearing invariant).** A compiled handler is *in addition to*, never *instead of*, its source compute entities. The endgame — the system expressed in entity-native compute over a minimal native bootstrap evaluator (the genome) — requires the source IR to survive in tree form so a self-hosting compiler can read it. A Stage-3 surface that emits "compiled-only" handlers would be a regression for self-description: such a handler can't travel as `T` and can't be recompiled or inspected. **Any authoring surface that produces a handler MUST produce (or keep) the source compute entities; "compiled" is a cache beside the source, not a replacement.** (Pin this in the spec's registration section when the Stage-3 surface is actually designed — there is nothing to constrain until then, but the invariant is stated here so it is not lost.)

---

## 7. Transferability ⇒ MUST (the standard-IR floor)

The IR's whole value is that an artifact authored on one peer evaluates **identically** on any COMPUTE-bearing peer. That carries a principle the spec now enforces:

> **Every standard-library capability the transferable IR depends on must be (a) MUST-given-COMPUTE — present on every peer that implements COMPUTE — and (b) cross-impl-deterministic — spec-pinned semantics, not implementation-owned.** MAY-level or impl-owned-semantics is *incompatible* with transferability: it means "the same expression may behave differently, or not run, on another peer," which is the one thing the IR exists to prevent.

The sharp test for what must be pinned (the stopping condition): **does the spec fix the capability's *semantic* behavior — output value, error type, side-effect shape — observable from a transferable artifact? → MUST-given-COMPUTE + pinned. Is it a *resource bound* — depth, budget, timeout — or opt-in per program? → stays flexible.** Semantic outputs are non-negotiable; resource bounds are universally negotiable.

The v3.14–v3.18 floor applied this: `compute/index`/`compute/length` + the `map`/`filter`/`fold` stdlib + entity-native handler evaluation + `store` all converged to MUST-given-COMPUTE; the integer model (WASM/LLVM/JVM: sign-agnostic `add`/`sub`/`mul`, signed-default `div`/`mod`/`compare`, signed-canonical encoding, point-of-use `numeric-cast`) pins integer arithmetic; cascade-depth (a resource bound) stayed flexible. The discipline generalizes into a recurring **transferability-MAY audit** across the other extensions (CONTINUATION / SUBSCRIPTION / REVISION / …) — compute went first.

---

## 8. The authoring stack: from IR assembly to languages

Here is the part the synthesis called "framed, not defined," now framed concretely — and where the genuine design opportunity is.

**Today we author the IR by hand.** The S1 builder (`ap.Compute()` in workbench-go; GUIDE-COMPUTE-PROGRAMMING §2) is typed *IR assembly*: `Literal`, `LookupTree`, `Arithmetic`, `Apply`, `Let`, `Lambda`, … each constructor emits one IR node, you compose them, `.Build()` puts the graph bottom-up. That is the right floor — it closes the hand-assembly friction (F1–F8) and gives every higher layer something to target. But it is assembly. **You would not, in the end, write programs this way — a compiler would emit this.**

**The opportunity is not to "build a language." It is to build the functional decomposition a language frontend needs.** A conventional frontend is lexer → parser → AST → semantic analysis → *lowering to IR*. The first stages are language-specific and well-understood; the last stage — turning parsed structure into correct compute IR — is the part that is specific to *this* IR and is reusable across every frontend. So the high-leverage product is the **lowering toolkit**: the reusable, composable decompositions that translate language constructs into compute graphs. Build that, and:

- a parser for any surface syntax (a small shell DSL, an S-expression form, a host-language closure, eventually a real language) targets the toolkit instead of re-deriving IR generation;
- you can support *many* frontends cheaply, because they share the lowering machinery;
- and you can always drop to the low-level builder directly when you want raw control.

The three authoring layers, then:

1. **IR assembly (the builder).** Typed constructors, one per IR node. The floor. Built and tested (S1).
2. **The lowering toolkit (the functional decompositions).** The reusable translations: how to lower a conditional, an iteration, a pattern-match, a data-structure access, a binding, a function call, signed/unsigned integer semantics. This is the layer to design next — it is "what a language would need," factored out of any particular language. §9 is its first content.
3. **Frontends.** Parsers/surfaces that produce IR via the toolkit: the shell as the smallest meaningful DSL (`compute build <expr>`); a type-system hook that lifts typed SDK functions into compute (depends on 4-B, §11); panels that render and author; multi-language frontends (Rust/Python authoring SDKs are separate, unscheduled tracks). The validate-driver's typed helpers are the proto-frontend.

The invariant from §6 applies at every layer: each layer ultimately emits *source compute entities in the tree*; none emits compiled-only output.

---

## 9. Lowering patterns

Lowering is where frontends meet the IR's family shape (§3). The recurring decompositions — the seed of the §8 layer-2 toolkit:

- **Loops → recursion + TCO.** A counting or accumulating loop becomes a tail-recursive lambda stored at a tree path and self-referenced via `lookup/tree` (the spreadsheet semantic). The evaluator guarantees tail-position calls iterate without stack growth.
- **Collection iteration → the stdlib.** `map`/`filter`/`fold` over a CBOR array, built on `compute/index`/`compute/length`. Plain list transforms live in pure compute now (§7); per-element *effectful* work drops down (§5).
- **Pattern match → `if`/`eq` chains** on a `.data` discriminant tag (`LowerMatch`); a variant lowers to an entity carrying that tag (the entity-native answer to 4-B — there is no type-system `union_of` to discriminate over, and `field` reads only `.data` not the entity `type`). A first-class `compute/match` primitive is a deferred v3.20 candidate (§11). See `PROPOSAL-COMPUTE-RECURSION-AND-SUM-TYPES.md`.
- **Data-structure access → `field` / `index` / `length` / `construct`.** Records by field name, arrays by integer index, length checks, structured results by construction. **(v3.19 value model — three rules a lowering author must encode.)** Navigation **composes** (`field`/`index`/`length` to arbitrary depth, `field` over an entity *or* a record), and disambiguation is by the value's **`kind`, never by sniffing keys** (N3) — so the toolkit threads kind on in-flight values rather than shape-detecting. A **`construct`** result, once it leaves compute, **materializes to a plain bare entity** (entity fields → bare `system/hash` refs, no kind-tags — `EXTENSION-COMPUTE` §2.3, v3.19c α): a lowered `construct` produces exactly what a hand-built entity would, which is what keeps it interoperable and dedup-correct. And **navigating back into a materialized entity returns a `system/hash` field as the hash** — the toolkit must emit an explicit `lookup/hash` to follow it, never auto-resolve, and never size a hash by a fixed byte length (it is variable-length, V7 §1.2).
- **Signed/unsigned integer semantics → point-of-use casts. (The Rule 11 worked example.)** Integers are 64-bit two's-complement; `add`/`sub`/`mul` are sign-agnostic, `div`/`mod`/`compare` are signed by default, and **unsigned `div`/`mod`/`compare` is reached by a `numeric-cast → primitive/uint` that is the *direct operand* of the operation.** The cast is **eager and consumed at the operation; it does not flow through a `let` binding or an `if` branch.** So:

  ```
  div(numeric-cast(y, uint), 2)              // unsigned — cast is the direct operand
  let u = numeric-cast(y, uint) in div(u, 2) // SIGNED-default — the binding drops the intent
  ```

  A frontend lowering unsigned division **must inline the cast at the use site**, never bind it. This is surprising enough that the S1 builder enforces it: `Let` rejects a `NumericCast` binding at build time, citing Rule 11. Any frontend that lowers integer semantics will hit this; it is the canonical example of "the lowering toolkit must understand the IR's evaluation rules, not just its node types."

Two more pitfalls a frontend/toolkit must encode (from the S1 reference design, grounded in the evaluator):
- **Lambdas lower to `compute/lambda` *expressions*, never to pre-evaluated `compute/closure` values.** A closure is a runtime artifact; storing one as a builtin argument is non-portable. Pass the lambda expression's hash to `map`/`filter`/`fold`.
- **`compute/apply` args are canonically sorted (length-then-lex) by the CBOR encoder.** Identical logical expressions built in different argument-insertion orders hash identically — which is what makes Stage 2 memoization correct. A toolkit relying on the standard encoder gets this for free; one hand-rolling encoding must replicate it.

---

## 10. The decision guide — "I want to do X"

Start from what the work *is*; choose the position; weigh the trade-off (including what has an SDK surface today).

| If the work is… | Use | Why / trade-off |
|---|---|---|
| a single-value reshape between two dispatches | **transform_ops** in the continuation step | trivial, total, declarative. No compute needed. |
| a conditional, scalar, or struct transform | **compute** (dispatched from a chain, or installed reactive) | visible, bounded, transferable (Class T/1). Authored via the S1 builder. |
| **iteration over a collection / array** | **pure compute** — `index`/`length` + the MUST `map`/`filter`/`fold`; **drop down** to native via `compute/apply` for *effectful / perf-critical* per-element work | pure-compute iteration is expressible today; the drop-down (~3 entities of glue) is the idiom for effects/FFI/hot-paths, not a workaround. |
| a **canonical, federation-wide recipe** over standard primitives | **named cross-impl op** (Class S) | portable-by-spec, native speed, clean contract — but every conformant peer must ship it, and it is harder to evolve than a local handler. |
| real I/O, FFI, genuinely unbounded work, or a hot path | **native handler** (Class N / Class 4) | the genuine escape; opaque. You can't force these into compute — they are the cliffs. |

Then orient on the axes (§4): must it be **transferable** — *behaviorally* (→ S) or *as data to any compute peer* (→ T, the only option that sidesteps op-presence) — or is it inherently local (→ N)? Must it be **visible/convergent** (→ Class 1–2) or is opacity an acceptable trade for isolation/speed (→ Class 4)? And — the pragmatic gate — does the position you want have an SDK surface yet?

**The composability contract:** positions compose through `compute/apply` and through continuation steps; a compute handler is a verb-op (a typed, dispatchable function) indistinguishable at the dispatch boundary from a native one (V7 §6.6 dispatch uniformity). That uniformity is what lets a chain step, a panel, or another handler call any position without knowing which it is.

---

## 11. Settled vs open

**Settled:**
- The model and the IR frame (compute = the content-addressed IR; N/S/T = IR/stdlib/FFI; the system = a distributed compiler).
- The gradient as the backend, with reactivity/visibility/transferability behavior pinned across compilation (§6); compilation optional.
- Transferability ⇒ MUST and the v3.14–v3.18 floor; the integer model; dispatch uniformity (V7 §6.6 v7.49).
- **The compute value model (v3.19, three-way ratified across Go/Rust/Python):** navigation composes and disambiguates by `kind` not shape (N.5/N3); scope bindings are kind-tagged and inherit the closure's authorization (v3.19b); `compute/construct` materializes to a plain bare entity (== hand-built) with the value model staying compute-internal (v3.19c α); an evaluated `compute/error` is a value returned at status 200 (F10); cross-peer closure transfer is semantics-pinned / wire-deferred (N7, V7 §5.8). The model is settled; what is *not* yet settled is the authoring/test sweep over it (below).
- The drop-down idiom as the first-class composition pattern.
- The authoring floor: the S1 builder reference design (built, tested, F1–F8 closed), which becomes the cross-impl reference (Rust/Python have no builder yet).

**Open — and explicitly wanting more test cases / confirmation before we call it closed:**
- **The authoring/test sweep over the value model.** The *semantics* are now pinned and three-way ratified (the v3.19 value model, above): `index`/`length`/`map`/`filter`/`fold`, navigation composition, kind-tagged scope, construct-materializes-bare. What still wants a dedicated sweep is the *authoring* story over them — how you build, translate, and interact with structured data through the builder; the edge cases (nested arrays, mixed records, empty/heterogeneous collections, deep `field`/`index` chains, read-back of a materialized `system/hash` field); what the translated IR actually looks like. This is the most-requested confirmation area, and it is exactly what workbench-go's battle-test (Scenario D + the aggregate verb, in progress) feeds back into.
- **The lowering toolkit (§8 layer 2).** Designing the reusable functional decompositions — and proving them with worked lowerings + tests — is the next substantive build. The innovation is the toolkit, not a language.
- **Sum-type / `compute/match` (4-B) — reframed, partially resolved (`PROPOSAL-COMPUTE-RECURSION-AND-SUM-TYPES §3`).** Two grounding corrections: there is **no `union_of`** in the type system, and `compute/field` reads only `.data` (not the entity `type`). The entity-native answer ships as a library decomposition — variants are entities with a `.data` tag, `match` lowers to an `if`/`eq` chain (`LowerMatch`). **Validated (workbench-go):** `LowerMatch` lowers Option / Result / a 3-variant enum / a default-arm cleanly, and a first-class **`compute/match` primitive is *not* justified** — tag-in-data is sufficient; exhaustiveness is a frontend concern, not a runtime-IR one (confirming this guide's frontend/IR/backend split). **Confirmed deferred (tracked-not-built):** the `compute/match` primitive (+ `compute/type-of`) and a type-system `union_of`, each with named re-open triggers (entity-`type` discrimination; IR-level exhaustiveness for an optimizer; measured `if`/`eq` perf at many variants).
- **Presence-soundness** (profiles / discovery). The IR floor partially closes the **T** axis — a COMPUTE-bearing peer is *guaranteed* to run a transferred compute handler (N.3) — but only *given* COMPUTE is present; the COMPUTE-presence question, and the **S**-axis op-presence question, remain the deferred profiles/discovery design work.
- **The backend stages** (Stage 2 memoization, Stage 3 native) — unbuilt; core-go track.
- **Multi-language frontends** — Rust/Python authoring SDKs, separate unscheduled tracks; they will mirror the S1 reference shape.

---

## 12. References

- **Spine:** `SYNTHESIS-COMPUTATIONAL-ARCHITECTURE.md` (the cohered framework + corpus index); `STATUS-COMPUTE-STANDARDIZATION.md` (operational tracker).
- **Explorations:** `EXPLORATION-CORE-COMPUTATIONAL-ARCHITECTURE.md` (the map, decision guide, axes), `EXPLORATION-COMPILATION-GRADIENT-AND-SYMBIOSIS.md` (the backend), `EXPLORATION-COMPUTE-AS-PORTABLE-IR.md`, `EXPLORATION-COMPUTE-IR-LANDSCAPE.md` (the IR cross-check), `EXPLORATION-COMPUTING-LANDSCAPE-GAP-HUNT.md` (the wide gap-hunt), `EXPLORATION-CANONICAL-PLACEMENT.md` (the positive placement — what it canonically is + where it sits in the foundations), `EXPLORATION-NATIVE-COMPUTE-PEER.md` (the frontier), `EXPLORATION-COMPUTE-AND-HANDLER-SDK-SURFACE.md`, `EXPLORATION-COMPUTE-BUILD-TRACK.md`.
- **Spec:** `EXTENSION-COMPUTE.md` v3.18; `ENTITY-CORE-PROTOCOL.md` §6.6 (dispatch uniformity), §3.7.
- **The layer below:** `GUIDE-COMPUTE.md` (language reference), `GUIDE-COMPUTE-PROGRAMMING.md` (SDK authoring).
- **Empirical / impl:** workbench-go Experiments A/B/C + the S1 reference design (`entity-workbench-go/entitysdk/compute_builder.go`; the COMPUTE v3.18 validation + SDK-reference feedback; `EXPLORATION-COMPUTE-FRONTEND-WORKBENCH-GO.md`); `entity-core-go/ext/compute` (the Stage-1 evaluator).

---

*Foundation-tier guide. The entity system is a distributed compiler; compute is its portable, content-addressed IR. Frontends lower to it, the gradient compiles it, the tree holds it. We assemble the IR by hand today; the work ahead is the lowering toolkit that lets languages target it — and the test cases that confirm the data-structure and translation story end to end.*
