# EXPLORATION — the compute execution model: the budget is a policy, not a law; and lowering to reduce boundary hashing

**Status:** Exploration / design — 2026-07-21. **NOT** a proposal, **NOT** ratified. Reopens two things the
compute track has been treating as settled: (1) the **100,000-op budget** as if it were a principle rather
than a default, and (2) "sharding is the *only* answer to the budget cliff"
(`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §4`). Author: arch workspace at the operator's request.

**Why now.** Every recent doc — including this session's bridge-host and generic-host work — has cited
"~100k `MaxOps`" as a fixed ceiling and concluded "past it you must shard." That is a **point in the design
space, not the shape of it.** The operator's push: *stop reifying a chosen constant; review the closest
parallel systems for the meaningful architecture; and design how to **lower entity compute into dedicated
handlers to cut the hashing at the boundaries.*** Those are two questions with one root — **what the
compute execution boundary is and what crossing it costs** — so they share a document.

**Two load-bearing facts, verified in-repo up front (they change the framing):**
1. **100k is a *recommended default*, parameterized by capability and bounds — not a spec law.**
   `init_budget` returns `operations = min(request_budget, bounds_budget, compute_ops_limit)`
   (`EXTENSION-COMPUTE §5.2`); each term defaults to `peer_default_max_ops` **only when unspecified**
   (`§5.5`), and `peer_default_max_ops = 100,000` sits in the **§9.3 "Recommended Limits"** table. The
   capability constraint `max_compute_operations` is literally *"Maximum evaluation steps **per
   invocation**"* (`§5.5`). So the budget is already **policy carried on capabilities**, per-invocation, and
   the constant is a fallback.
2. **The "drop the per-node hashing" principle already exists in the guide — unbuilt.** `GUIDE-CORE §6`:
   *"Determinism survives compilation because it is pinned at the **materialized boundary** … the per-node
   content-addressing the interpreter uses internally is an **interpretation artifact the backend drops** —
   the compiled interior computes the same boundary entities in registers, materializing (and hashing) only
   where a value re-crosses into the tree."* The Stage-3 native backend that does this is marked **"Unbuilt."**

---

## §0 TL;DR

- **The budget conflates three different jobs into one number** (the `ttl==chain_depth==64` mistake again):
  **termination** (a deterministic evaluator MUST halt), **fairness** (one computation can't starve a
  peer), and **cost-accounting** (DoS / resource bounding). 100k is one point-solution to all three. They
  have *different* natural magnitudes and *different* right behaviors on breach — de-confound them.
- **The central fork the parallels reveal: hard-fail vs preempt-and-resume.** EVM gas and eBPF hard-fail;
  **BEAM preempts at a reduction budget and reschedules.** Entity today hard-fails (`budget_exhausted`
  error). **But entity already has first-class continuations** (suspend/resume with defined resume-safety,
  `EXTENSION-COMPUTE §3.19c`) — so it is uniquely positioned to adopt the **BEAM model**: budget exhaustion
  becomes a **preemption point that yields a standing continuation**, not a cliff. The op-decrement site
  (`§5.3`/eval loop) *is* the natural preemption boundary, exactly like a reduction boundary.
- **So "sharding is the only budget answer" is false outside the hard-cliff point.** There are **three
  reliefs, and they compose**:
  - **Spatial — sharding** (split a `map` into k independent fresh-budget shards). *Landed.*
  - **Temporal — preemption-continuation** (yield at a reduction budget, resume next scheduling turn; the
    whole computation completes across reschedules). *Machinery exists (continuations); not wired to the budget.*
  - **Vertical — lowering/compilation** (compile a pure interior to a native handler; it costs ~**1 op**
    instead of N interpreter ops, and **materializes/hashes only at the boundary**). *Stated in the guide;
    unbuilt.*
- **Lowering is the answer to the second ask (reduce hashing).** The hot cost is per-node `cbor.Unmarshal`
  + scope loads (~75% of a tick, measured). Lowering a **pure interior between impure boundaries** into a
  dedicated handler drops all intermediate hashing — you hash only what re-crosses into the tree. Because
  the IR is content-addressed, the compiled artifact is **memoizable for free** (identical structure ⇒
  identical hash ⇒ valid reuse), which turns into **red-green early-cutoff across ticks** (an unchanged
  input subtree keeps its hash ⇒ skip the recompute). This is Salsa/Adapton, native to a content-addressed
  IR.
- **Budget and lowering co-evolve.** Lowering collapses op-count (a native handler = 1 interpreter op), so
  the op-ceiling no longer bounds the work *inside* the handler — that is exactly what
  `PROPOSAL-COMPUTE-APPLY-RESOURCE-CEILING` (landed F1–F5) is for. The mature model is **two bounds**: an
  reduction budget bounding the interpreter (preemptive), and an apply-resource-ceiling bounding native handlers
  (per-invocation). One number stops doing both jobs.

---

## §1 The budget is a policy, not a law (the reframe, with the spec)

We keep writing "~100k" as though it were a physical constant. The spec says otherwise:

```
init_budget(params, capability, bounds):                 ; EXTENSION-COMPUTE §5.2
  request_budget = params.budget or infinity
  bounds_budget  = bounds.budget or peer_default_max_ops
  compute_ops_limit = capability.constraints["system/compute"].max_compute_operations
                        or peer_default_max_ops
  return { operations: min(request_budget, bounds_budget, compute_ops_limit),
           depth: compute_depth_limit }
```

- The effective budget is a **`min` of three policy inputs** — the caller's request, the wire bounds, and
  the **capability constraint**. 100k appears **only** as `peer_default_max_ops`, the fallback when a term
  is absent (`§5.5` per-key fallback), and it lives in a table headed **"Recommended Limits"** (`§9.3`).
- `max_compute_operations` is *"Maximum evaluation steps **per invocation**"* — so the budget is already
  **per-invocation and capability-scoped.** A grant can hand a large budget to a trusted heavy computation
  and a tiny one to untrusted transferred code. That is the architecture; the constant is a default nobody
  is obliged to keep.

**The real mistake is conflation, and we have made it before.** One number is silently doing **three
different jobs**:

| Job | What it really guards | Natural magnitude | Right behavior on breach |
|---|---|---|---|
| **Termination** | a deterministic evaluator MUST halt (no infinite loop wedges the peer) | must be *finite*, else unbounded | **hard stop** (there is no valid continuation of a true non-terminator) |
| **Fairness** | one computation can't monopolize the scheduler / starve reactive+emission work | *small* (a reduction budget — a slice of steps) | **preempt & reschedule** (the work is legitimate, just yield) |
| **Cost-accounting** | DoS / metered resource / untrusted-code bound | *policy-sized* (per capability, per trust) | **hard-fail** (this is a genuine resource-limit refusal) |

100k tries to be all three at once, so it is simultaneously *too big* to be a fairness reduction budget (a 100k-op
tick blocks everything else for its duration) and *too coarse* to be a real cost policy (it's a peer-wide
default, not a per-grant price). This is the **exact shape** of the `ttl` default (64) colliding with the
`chain_depth` ceiling (64) that masked the depth brake in W-CONTINUATION — *two magnitudes wearing one
number.* De-confound them (§4/§5).

---

## §2 What the budget must actually guarantee (the invariants under the number)

Strip the constant and three requirements remain — these are load-bearing; the number is not:

- **I-T (termination).** Evaluation of any expression MUST terminate in bounded steps on every conformant
  peer. This is *why* a bound exists at all; without it a peer can be wedged by a non-terminator, and worse,
  two peers could disagree on whether something halts (a determinism fork). This one is non-negotiable and
  is the only one that *must* be a hard stop.
- **I-F (fairness / liveness).** A single computation MUST NOT starve a peer's other work (reactive
  re-eval, inbox drain, cross-peer service). This wants a *small reduction budget*, not a large cap.
- **I-A (accounting).** A peer MUST be able to *price and refuse* work per capability/trust (untrusted
  transferred code, metered service). This wants a *policy value on the grant*, and hard refusal on breach.

The claim of this doc: **entity currently serves all three with one hard-cliff op-count, and that is why the
compute track keeps hitting "you must shard."** Separate them and the design opens up.

---

## §3 How the closest parallel systems bound execution (comparative, verified)

*(Figures citation-pinned via a source-verification pass; primary sources named inline.)*

| System | Unit of accounting | Breach behavior | The design lesson |
|---|---|---|---|
| **Erlang/BEAM** | **reductions** ≈ function applications; per-process **reduction budget `CONTEXT_REDS = 4000`** (was 2000 pre-OTP-20; `INPUT_REDUCTIONS = 8000`) — `erl_vm.h` | **preempt → reschedule**; process resumes exactly where it stopped, no error. Long C BIFs/NIFs trap-and-reschedule or go to **dirty schedulers** | *Fairness is a small reduction budget you **yield** at, not a cap you **die** at.* The scheduler owns time; the runaway is time-sliced, not killed. **The model entity is built for.** |
| **Wasmtime — fuel** | per-instruction (most = 1; `nop`/`drop`/`block`/`loop` = 0); precise running count, **deterministic** | trap **by default**, or **async-yield** every N units (`fuel_async_yield_interval`) | Deterministic *work* metering; **trap-vs-yield is an independent policy knob on the same meter.** |
| **Wasmtime — epoch** | global counter bumped by a timer thread, checked at calls/loop back-edges; **non-deterministic** | trap or `epoch_deadline_async_yield_and_update` | ~10% overhead, **2–3× cheaper than fuel** — but wall-clock ⇒ **the anti-pattern for a deterministic evaluator.** Meter by op-count (fuel), *never* wall-clock (epoch), or peers fork. |
| **EVM gas** | gas per opcode (transfer floor 21,000) | **out-of-gas → burn-all + revert** (hard-fail); `REVERT`/EIP-140 rolls back *and* refunds. **63/64 rule** (EIP-150) caps call depth ~300–340 | *Accounting done right = per-op **pricing** + hard refusal.* Hard-fail **because** adversarial code under **replicated consensus cannot preempt-and-resume.** This is I-A, not I-F. |
| **eBPF** | static verifier: bounded loops + **`BPF_COMPLEXITY_LIMIT_INSNS = 1,000,000`** explored (raised from 131,072); `BPF_MAXINSNS = 4096` unpriv | **rejected at load**; zero runtime metering | Prove termination **before admission** when the runtime context (kernel) can't tolerate a meter. Entity can't fully (richer IR), but *some* shapes are statically bounded. |
| **Lua** | instruction-count hook (`sethook`/`LUA_MASKCOUNT`, author-chosen N) | **hard-fail** (`luaL_error` unwinds; script terminated). *LuaJIT caveat: count hooks may miss hot compiled loops* | Cheapest retrofit of a bound onto a tree-walker — but it **kills** rather than time-slices. |
| **V8 isolates** | **none** for CPU (heap bytes only) | external watchdog → `TerminateExecution()` (**uncatchable**, non-resumable); Cloudflare adds a 30 s wall-clock cap | With no in-engine meter you can only bound by an **external wall-clock hard-kill** — a fuse, not a scheduler. |
| **Unison** | **abilities** (typed effect row `I ->{A} O`); a handler intercepts each request and holds its **delimited continuation `k`** | handler may **drop `k` (terminate), resume once, or resume many** — **no built-in meter; abilities are the mechanism to *build* one** | Route every effect through a handler-intercepted request — **that handler *is* the metering/preemption boundary.** = entity's `compute/apply` impure boundary. |

**The fork, verified and sharpened:** every system is **hard-fail** (EVM, eBPF, Lua, V8) or
**preempt-and-resume** (BEAM; Wasmtime *optionally* on either meter; Unison via the continuation handler),
and **the choice is forced by trust + determinism context, not taste.** EVM/eBPF *must* hard-fail —
adversarial code under replicated consensus (EVM) or a zero-overhead kernel (eBPF) **cannot** resume. Entity
is *neither* a public consensus VM nor a kernel, so it is **not** forced into their corner. Two systems it is
genuinely closest to — **BEAM** (deterministic reduction budget, scheduler-owned preemption) and **Unison**
(effects-as-handler-boundary over delimited continuations) — both **resume**, and Unison shows entity
**already owns the machinery** (its continuations are Unison's `k`). And steal **Wasmtime's two-axis
insight**: *trap-vs-yield is an independent policy knob layered on one meter* — so entity can offer a
hard-ceiling (untrusted/non-resumable frames, EVM/Lua-style) **and** yield-resume (cooperative default,
BEAM-style) over the *same* deterministic op-meter. That is exactly the two-behaviors framing of §4/§5.

---

## §4 The insight entity is built for: budget exhaustion → yield a continuation (BEAM, natively)

BEAM's move — *hit the reduction budget, suspend the process, reschedule it* — requires a runtime that can
**suspend and resume a computation.** Most systems bolt this on with heroics. **Entity already has it as a
first-class, conformance-defined primitive:**

- `compute/closure`, suspended continuations, and pending reactive evaluations are **retainable suspended
  state** with a defined **resume-safety** contract (`EXTENSION-COMPUTE §3.19c`: a resumed computation whose
  scope binding is missing yields a clean `scope_unreachable` error value, *"never a silent substitution … a
  peer MUST NOT resume a closure with a missing binding as though it were present"*). The GC invariant that
  keeps a suspended computation's captured scope reachable **across a peer restart** is already pinned.
- The **op-decrement site is already the preemption point.** In the eval loop, `budget.operations -= 1; if
  budget.operations <= 0: return error("budget_exhausted")` (`§4.1`/`§5.3`). That is *exactly* a BEAM
  reduction boundary. Today it returns an error; the BEAM variant **captures the current evaluation
  continuation and yields it to the scheduler** for resumption with a fresh reduction budget.
- **Reactive re-eval already treats exhaustion as non-terminal.** `§7.4`: budget exhaustion during reactive
  re-evaluation *"writes a `compute/error` … but does **NOT** freeze the subgraph — budget exhaustion may be
  transient."* The system already half-believes exhaustion is recoverable; preemption-continuation makes
  that belief the general model instead of a reactive special case.
- **Unison names the exact shape.** In Unison an effect handler intercepts a request and **holds its
  delimited continuation `k`**, and may **drop `k` (terminate), resume once, or resume many** — and Unison
  ships **no built-in meter** precisely because *"abilities are the mechanism to build one."* Entity's
  continuations **are** that `k`; so budget-metering is not a trap bolted into the evaluator, it is a
  **handler decision at the effect boundary** — the same `compute/apply` impure boundary where entity
  already materializes and hashes (§6). This is the deep unification (§8): one boundary, three roles.

> **Design proposal (D1) — a fairness *reduction budget* that yields, distinct from a cost *ceiling* that fails.**
> Introduce a small **reduction budget** (BEAM-reduction-sized: enough work to be efficient, small
> enough to keep the peer responsive). When a computation consumes `q` ops without finishing, the evaluator
> **suspends into a standing continuation** (the machinery of `§3.19c`) and reschedules it — it does **not**
> error. The large **cost ceiling** (the capability's `max_compute_operations`, I-A) remains a **hard
> `budget_exhausted`** — because *that* breach is a real resource refusal, not a fairness pause. Two limits,
> two behaviors: **`q` preempts; the ceiling fails.**

**The honest hard part (do not gloss it).** BEAM preempts at *function-call* boundaries; suspending a
*tree-walk mid-expression* means capturing the evaluator's own stack as a resumable continuation. Entity's
trampoline already reuses the depth frame for tail calls (`§4.1` TCO), which is a step toward an explicit,
resumable evaluation state — but full mid-expression suspension needs the evaluator in **defunctionalized /
CPS / explicit-stack form** (the **Axis-1 decode-once resolved-node walker** is the natural host for this;
it already keeps an explicit resolved-node state). So D1's realistic first form is **preemption at natural
boundaries** — reduction budget checkpoints at dispatch points, `map`/`fold` element boundaries, and trampoline
iterations — *not* arbitrary mid-node suspension. That is still BEAM-grade (BEAM also preempts at
boundaries, not mid-instruction). Pin the boundary set; don't promise mid-node.

**What D1 buys:** a heavy-but-legitimate tick (a big `map`, a deep recursion under the *ceiling* but over
the *reduction budget*) **completes across scheduling turns instead of failing** — no author-side sharding required
for the *fairness* problem. Sharding remains the answer for *parallelism* (using k cores), but it stops
being forced on you merely because one tick is big. And it unifies W-COMPUTE with W-CONTINUATION at the
substrate, not just at the join (`WORKSTREAMS.md`): the standing continuation is the shared primitive.

---

## §5 Three reliefs, composed — reopening "sharding is the only answer"

`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §4` states *"sharding is not the preferred budget answer — it is
the only one,"* justified by "op count measures the IR, not the engine, so no faster engine moves the
cliff." That reasoning is correct **for the hard-cliff, pure-interpreter point** — and only there. Widen the
frame and there are **three orthogonal reliefs**:

| Relief | Axis | Mechanism | Bounds what | Status |
|---|---|---|---|---|
| **Sharding** | **spatial** | split a `map` into k independent shards, each a fresh top-level eval (fresh budget) | uses k cores; each shard under the ceiling | **landed** (static-k floor; §4/§4a of the proposal) |
| **Preemption-continuation (D1)** | **temporal** | yield at the reduction budget, resume next turn — the whole computation runs across reschedules | the *fairness* breach, not the ceiling | **machinery exists** (`§3.19c`), unwired to the budget |
| **Lowering / compilation** | **vertical** | compile a pure interior to a native handler: **~1 op** instead of N, hash only at the boundary | collapses op-count; shifts the bound to the apply-resource-ceiling | **stated** (`GUIDE-CORE §6`), **unbuilt** |

They **compose**: a program can shard a `map` (spatial), have each shard's heavy interior lowered to native
(vertical), and still preempt (temporal) if a shard exceeds the reduction budget. The proposal's claim should be
amended to: *"sharding is the answer to the **parallelism/throughput** ceiling; preemption is the answer to
the **fairness** breach; lowering is the answer to the **per-op cost**. Do not force one to do all three."*
(A ruling reopen, routed as an amendment to that proposal — not a silent contradiction.)

**Why vertical relief is real and not double-counting.** Op-count measures *interpreter node-visits*
(`GUIDE-CORE`: "op count measures the IR"). A native handler reached via `compute/apply` is **one node-visit
to the interpreter** — the interpreter decrements one op for the `apply`, then the handler runs opaquely. So
lowering a 10k-op pure interior to a native handler makes it cost **1 op**, and the 100k budget suddenly
goes ~10,000× further *for that interior*. This is not gaming the bound — it is that the op-count was only
ever bounding the *interpreter*, and the work moved off the interpreter. The moved work now needs its **own**
bound: `PROPOSAL-COMPUTE-APPLY-RESOURCE-CEILING` (landed, F1–F5) is exactly that — a resource ceiling on the
`apply`/handler boundary. So the mature budget model is **two bounds by construction**: reduction budget (bounds
the interpreter, preemptive) + apply-resource-ceiling (bounds native handlers, per-invocation). The single
100k was standing in for both.

---

## §6 Lowering to reduce boundary hashing (the second ask, and it is already the stated architecture)

The hot cost is **not arithmetic** — it is **decode + hash of intermediate nodes**. Measured (Life 16×16
arith, `ABSORPTION-compute-program-poc-life-snake`): ~87% in `compute.Evaluate`, of which **~54% per-node
`cbor.Unmarshal` + scope-load ≈ ~75% of everything**; arithmetic and boundary hashing are noise. The
operator's ask — *"lower entity compute into dedicated handlers … to reduce hashing where possible"* — is
the direct attack on that 75%, and `GUIDE-CORE §6` **already specifies the principle:**

> Determinism is **pinned at the materialized boundary** — the canonical encoding + content hash of entities
> that cross into the tree (state writes, a stored `construct`, an `apply` arg). The per-node
> content-addressing the interpreter uses internally is an **interpretation artifact the backend drops** —
> a compiled interior computes the same boundary entities in registers, hashing **only where a value
> re-crosses into the tree.** (`GUIDE-CORE §6`, line 103; the Stage-3 native backend that does this: *Unbuilt*.)

So the design is not to invent a mechanism but to **build the one the guide already pins**, and to say
*where the boundary is*:

**The lowering unit is a *pure interior between impure boundaries*.** `GUIDE-CORE §6`'s "four-way
coincidence": purity, compilation, reactivity, and visibility **all anchor on the same impure ops**
(`tree.get` / `tree.put` / `dispatch` / `check_permission`). The partition compute already makes by lookup
type — `scope` pure / `tree` impure-reactive / `hash` pure-fixed — **is the visibility frontier.** So:

- **Cut at the impure ops.** Everything between two impure operations is a **pure subgraph**; that is the
  unit you fuse into a dedicated handler. The impure skeleton (the `tree.get`/`put`/`dispatch` calls, and
  the reactive edges, which *are* the preserved boundary) stays as-is.
- **Inside the fused interior: no entities, no hashing.** The compiled handler computes intermediate values
  in registers/native structures. Nothing is CBOR-encoded, nothing is hashed, no scope entity is loaded per
  node — the entire ~75% evaporates *for the fused region*.
- **At the boundary: materialize and hash canonically, exactly as before.** The values that re-cross into
  the tree (the state write, an `apply` arg, a `construct` that leaves compute) are canonically encoded and
  hashed — and **the conformance gate checks precisely these**, so equivalence is *"same materialized
  boundary entities ⇒ same vectors,"* never "same internal steps." **This is why it is safe to compile.**
- **Keep the source IR (load-bearing, `GUIDE-CORE §6` line 104).** The compiled handler is a **cache beside**
  the source compute entities, never instead of them — the self-hosting genome needs the IR in tree form.

### §6a The content-address makes it memoizable — and that is red-green early-cutoff for free

The IR is content-addressed, so (`GUIDE-CORE §3`) *"compile-time collapse and result caching are free and
structural: identical structure ⇒ identical hash ⇒ valid reuse, with no separate proof layer."* Two
compounding wins beyond the per-tick fusion:

- **Compile-once, keyed by the subgraph's content hash.** The compiled artifact for a pure interior is
  cached under that interior's hash. Every program with the *same* interior (every peer running the same
  transferable step) reuses the same compilation — the Truffle/JAX "trace-compile-**cache**" pattern,
  except the cache key is *already* the content hash, not a fingerprint we have to compute.
- **Red-green early-cutoff across ticks (Salsa/Adapton, native).** If a pure interior's **input** subtree
  hash is unchanged tick-to-tick, its **output** hash is unchanged — so **skip the recompute and reuse the
  cached output hash.** For a sparse update (Doom actors: O(actors) change, the rest of the world is
  identical — `EXPLORATION-COMPUTE-WHOLE-STATE-TICK-COST`), only the changed subtrees recompute; the
  unchanged majority costs a **hash comparison**, not an eval. This is the same mechanism as the
  content-store dedup invariant (#2), lifted from storage to computation. *(It also quietly addresses the
  monolithic→chunked state-emission problem, `L5 §5.7` — an unchanged subtree keeps its hash, so it is
  neither re-hashed nor re-emitted — though that optimization is deprioritized and only noted here.)*

**This is the deforestation angle too.** Haskell stream fusion eliminates *intermediate data structures*; a
fused entity interior eliminates *intermediate hashed entities*. Same idea, and here the intermediates are
the expensive thing (each is a CBOR encode + SHA-256), so the payoff is larger than in a normal language.

---

## §7 Lowering — how the closest parallels do it (comparative, verified)

| System | Mechanism (verified) | The design lesson for us |
|---|---|---|
| **GraalVM / Truffle** | **self-specializing AST** (nodes rewrite on observed types, insert **guards**) + **partial evaluation**: once hot, PE holds the AST nodes **constant** and inlines the `execute` methods so **dispatch is constant-folded away** → native code for that one program. Measured: naive fib(20) ~6,028 µs/op → ~100 µs/op (**~60×**, ~2–3× native). Invalidated guard → `transferToInterpreterAndInvalidate()` **deopts to the interpreter** | *You do not write a compiler; you partially-evaluate `eval` against a fixed program.* Our "fixed program" is the **content-addressed step IR** — an ideal PE target (immutable, hashed). **Take the whole discipline, including deopt (§6b).** |
| **Futamura projections** | (1) specialize interp w.r.t. program → a compiled program; (2) w.r.t. interp → a compiler; (3) w.r.t. itself → a cogen. Truffle = the **first projection** in production | **Stage-3 lowering *is* the first Futamura projection over the compute IR.** Names what `GUIDE-CORE §6` gestures at. |
| **JAX / XLA, `torch.compile`** | **trace** the pure fn on shape/dtype tracers → compile → **cache keyed by the abstract shape/dtype signature** (Dynamo: an FX graph + **guards**; recompile on structural change, bounded then eager fallback) | *Trace-compile-cache-**by-structure*** — and our cache key is the **content hash**, computed for us. The guard-set = the boundary hash check. Pure-fn-only = the pure-interior restriction (§6). |
| **XLA fusion** | fused intermediates have **"no intermediate storage materialized in HBM… passed through registers or shared memory"** — no addressable existence | The exact "don't hash intermediates": once a subgraph is fused, its intermediate entities **don't exist to hash** — only inputs + final output cross the seam. |
| **Salsa / rust-analyzer (red-green)** | revision counter + **backdating early-cutoff**: a re-executed query whose output **equals** the prior value **stops propagation** — dependents don't re-run | §6a directly: unchanged output hash ⇒ halt the invalidation cascade. Native to a content-addressed IR (equality is a hash compare). |
| **Adapton** | demanded-computation DAG of memoized nodes; **Eq-gated** change propagation | The general theory behind §6a; the DAG is our expression graph. |
| **Nix CA / Bazel** | address outputs/actions by **content hash of inputs**; a **bit-identical re-derivation skips the whole downstream** | Macro-scale proof that content-hashing = free memoization + early cutoff. We get it at eval granularity. |
| **Haskell stream/deforestation** | Wadler deforestation + Coutts stream fusion: **no intermediate structure allocated** in a producer→consumer composition | Fuse away intermediate **hashed entities** — the largest per-item cost here (each is a CBOR encode + SHA-256), so the highest-value fusion. |

**Synthesis — the JAX/torch.compile loop already fuses all three families, and it is our architecture:**
*trace the pure interior once → partially-evaluate + fuse into one native handler (Truffle/XLA) → cache it
keyed by the interior's **content hash** (Nix-CA/Salsa/JAX-signature) → at the boundary, check the hash and
run the handler; re-specialize only on a structural (hash) change or a failed guard.* Truffle supplies the
lowering rigor **and the deopt safety-net**; XLA/deforestation supply "intermediates never materialize";
Nix-CA/Salsa supply hash-keyed early-cutoff. For **the budget**: **BEAM** (preempt at the reduction budget,
reschedule) + **Wasmtime fuel** (deterministic op-meter with trap-or-yield as a policy knob) — *never* epoch
(wall-clock ⇒ non-deterministic ⇒ forks).

### §6b Lowering needs a deopt safety-net (Truffle's load-bearing lesson) — the correctness pin

Truffle's speculation is only safe because **every specialization is guarded and an invalidated guard
`transferToInterpreterAndInvalidate()` falls back to the interpreter** — the deopt makes a wrong speculation
a *correctness-preserving fallback*, never a miscompile. Entity's Stage-3 lowering MUST carry the same net,
and it maps cleanly onto what already exists: **the source compute IR is preserved in the tree**
(`GUIDE-CORE §6` line 104 — "a compiled handler is a cache *beside* the source, never instead of it"), so
**the interpreter over the source IR *is* the deopt fallback.** Concretely: a lowered handler's output MUST
be validatable against the interpreted result at the **materialized boundary** (identical boundary entity
hash), and any mismatch — or any input outside the specialization's assumptions — **falls back to
interpreting the source IR**, which is authoritative. This is why keeping the IR in the tree is not just a
self-hosting nicety (the genome) but the **safety net that makes compilation legal at all.**

---

## §8 The synthesis (one execution model)

**The deep result — the two problems share one seam.** The impure/effect boundary (`tree.get`/`put`/
`dispatch`/`check_permission` — `GUIDE-CORE §6`'s four-way coincidence) turns out to be the **same** boundary
in three roles at once, and this is why the budget question and the hashing question are one design:

1. **It is where you preempt** (Problem 1): the effect boundary is where a handler holds the delimited
   continuation and may yield-and-resume (BEAM/Unison). Between two impure ops, a pure interior runs
   uninterrupted to a boundary — the natural preemption point (§4).
2. **It is where you materialize and hash** (the determinism pin): only values that *cross* the boundary
   into the tree are canonically encoded and hashed; the conformance gate checks *exactly* these (§6).
3. **It is where caching/fusion validity ends** (Problem 2): only the **pure fragment** between boundaries
   may be fused (XLA/deforestation) or memoized by content hash (Salsa/Nix-CA); the impure boundary is
   precisely what fences off the non-deterministic parts that must **not** be cached or fused.

So "meter it," "hash it," and "may I reuse/compile it" are **three questions answered by the same
partition** — purity at the impure boundary. Get that boundary right once and all three fall out; that is
the single most load-bearing idea in this doc. Concretely, the model stops being "a 100k wall you shard
around":

- **Two bounds, not one.** A small **reduction budget** `q` that **preempts into a continuation** (I-F, BEAM/fuel)
  + a capability-scoped **cost ceiling** that **hard-fails** (I-A, EVM-style pricing on the grant). Plus the
  **apply-resource-ceiling** (landed) bounding native handlers. Termination (I-T) is guaranteed by the
  ceiling being finite.
- **Three composable reliefs** for a heavy computation — **shard** (spatial/throughput), **preempt**
  (temporal/fairness), **lower** (vertical/per-op-cost) — picked by *which* pressure you're under, not
  "always shard."
- **Lowering doubles as the hashing fix:** fuse pure interiors between impure boundaries into dedicated
  handlers (hash only at the materialized boundary), compile-once keyed by content hash, and reuse across
  ticks via red-green early-cutoff. The measured ~75% decode/hash cost is the target; the boundary is the
  impure ops; the safety is the materialized-boundary conformance gate.

None of this is a wire or core-protocol change to the *bound's existence* — it is: (a) recognizing the
budget is already capability-policy; (b) adding the reduction-budget/preemption behavior (a continuation wiring, not
a new primitive); (c) building the Stage-3 lowering the guide already specifies.

---

## §9 Open questions & what to route

1. **The reduction budget (D1).** What size, and is it a **new** field or a *second interpretation* of the
   existing budget (e.g. `bounds.budget` = ceiling; a peer-local reduction budget = fairness slice)? Lean:
   the reduction budget is a **peer-local scheduler knob**, not on the wire (fairness is a local concern; the
   *ceiling* is what travels). Confirm against `SYSTEM-COMPOSITION` cascade limits (a sibling local safety bound).
2. **Preemption boundary set.** Pin the exact points where the evaluator may suspend (dispatch points,
   `map`/`fold` element boundaries, trampoline iterations) — *not* arbitrary mid-node — and prove
   resume-safety (`§3.19c`) holds at each. Needs the **Axis-1 explicit-state evaluator** as the host; route
   with the Axis-1 build.
3. **Determinism of a preempted computation.** Two peers must agree on the *result*, not on *where* each
   preempted (preemption points are local scheduling, invisible to the boundary hash — the same
   "equivalence at the materialized boundary" argument as compilation). State it as a MUST and add a vector:
   same step, different `q`, identical per-tick state hash.
4. **Stage-3 lowering scope.** First target = the pure interior of the three probe programs' steps; measure
   the decode/hash cost eliminated vs the Axis-1 interpreter's ~20–33× (is lowering worth it *after*
   Axis-1, or does Axis-1 already move the bottleneck enough?). This is the honest gating question —
   `GUIDE-CORE §6` says the gradient is optional; measure before building Stage-3.
5. **Red-green cache invalidation & memory.** The output-hash cache is unbounded without eviction; ties to
   the **deferred GC collector** (`§3.19c`, keystone crash-mid-flight collector). Route together.
6. **Amend the "sharding is the only answer" ruling** (`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §4`) to the
   three-reliefs framing (§5) — a proposal revision, not a silent edit.
7. **Where this lands.** The budget-policy reframe + the reduction-budget/preemption is a **core/compute-extension**
   concern (routes to `entity-core-protocol` / `EXTENSION-COMPUTE §5`); the lowering is the
   **compute-standardization / Stage-3 backend** track (impl, per-peer, `GUIDE-CORE §6`/§8). Keep them
   sequenced: the budget reframe is cheap and unblocks honest framing now; Stage-3 is a measured build.

*Authored in the arch workspace at the operator's request; ratifiable outputs are (a) an `EXTENSION-COMPUTE
§5` amendment de-confounding reduction-budget/ceiling + wiring preemption to the continuation substrate, and (b) a
Stage-3 lowering proposal on the compute-standardization track — both via proposal → ratify → fold, not this
file. Comparative figures (§3/§7) verified against primary sources (BEAM `erl_vm.h`, Wasmtime `Config`/`Store`
docs, EVM EIP-150/EIP-140, eBPF kernel docs/commit `c04c0d2b968a`, GraalVM PLDI 2017, JAX/XLA/Salsa/Nix-CA
docs).*
