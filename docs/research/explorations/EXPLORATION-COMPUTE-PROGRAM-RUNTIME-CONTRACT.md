# EXPLORATION — Hostable compute programs & the runtime interface contract

**Phase:** Exploration / analysis (pre-proposal). 2026-07-13. **Not a proposal.**
**Purpose:** work out what it takes to run a *real, interactive program* on compute — not by
running WebAssembly behind the substrate (known; not new), but by expressing the program's
**deterministic core as compute IR** and defining the **standard interface contract** that lets any
host (browser, workbench-go, Godot, …) pick that program up, wire its input/output, and drive it.
The long-term target is not Doom — it is **compiling what we already have into entity-native compute**
and, eventually, **the peer itself as a hostable compute program.** Toys are the stepping stones and
the **cost-model probes**.
**Method:** decompose an interactive program (Doom as the reference probe), survey minimal-VM /
fantasy-console I/O contracts (Varvara, WASM-4) for the interface vocabulary, and sketch Conway's Game
of Life as the first end-to-end. Cited §Sources.
**Related:** `EXPLORATION-RUN-ENVIRONMENTS-AND-WORKLOAD-ACTUATION.md` (the WASM-first actuation layer —
this is its *inspectable-workload* sibling); `GUIDE-CORE-COMPUTATIONAL-ARCHITECTURE.md` (compute = the
content-addressed IR; N/S/T; the gradient backend; the entity-native-peer endgame).

---

## §0 TL;DR

- **Three architectures for "run a real program"; only one advances our understanding.** (A) run
  WebAssembly behind the actuation substrate — *known, practical, not new.* (B) express the program's
  **deterministic simulation as compute IR**, dropping to native only for the effectful hot paths
  (render/audio) — *the interesting one.* (C) build an imperative VM inside compute — *rejected as
  stated, but it names a real need:* B is not hostable without a thin **runtime interface contract**
  around the pure program.
- **The architecture is three layers:** (1) **the program** — a pure, transferable compute subgraph
  that is just `state₀` + `step: (state, input) → state'`, and knows *nothing* about I/O; (2) **the
  interface descriptor** — a small standard manifest declaring the program's **ports** (input paths the
  host writes, output paths the host reads) + the **step/tick contract**; (3) **the runtime** — a
  per-host harness that reads the descriptor, binds ports to real I/O, and clocks the tick. *Not a VM —
  a harness.*
- **Minimal runtimes already prove the interface vocabulary.** Varvara (the device layer over the Uxn
  CPU) and WASM-4 both reduce the *entire* I/O contract to **named I/O slots + a periodic vector/update**.
  Varvara even splits **portable-computation (Uxn) from device-contract (Varvara) from per-host runtime**
  — our exact three layers, already shipping. Their "memory-mapped port" is our "declared tree-path
  port"; their "60 Hz vector" is our "host-clocked tick."
- **A hostable compute program is the second workload kind in the actuation family** — beside the WASM
  component — differing in the one way that matters: the workload is **inspectable, transferable data**
  (Class T/1), not an opaque blob (Class 4). That is the entity-native-peer endgame in miniature.
- **The tick must be host-driven, not self-triggered** (a key finding, §5): the discrete `state → state'`
  loop is clocked by the runtime via explicit eval; *reactivity* is reserved for **projections**
  (`framebuffer = f(state)`), which recompute when state changes. A self-feeding reactive tick is an
  uncontrolled cascade.
- **First probe: Conway's Life** — no input seam yet, exercises **state + step + tick + output**, and
  gives a raw **entity-ops-per-tick throughput number** (the cost-model deliverable). Progression:
  Life → Snake (input) → Tetris (deterministic RNG) → Doom (scale + native render drop-down + audio).
- **Verdict from the Life sketch (§6.1): B needs *zero* compute-extension changes.** The step is
  expressible in existing primitives; the "update" is not a compute feature but a **host-clocked
  `eval`+`tree.put` loop.** The only new spec surface is the **interface descriptor** (an L5 convention)
  and the **per-host runtime** (host code). This is *why* the architecture felt fuzzy — there is almost
  no new machinery: one expression, a 3-line loop, a small manifest. **Three programs (Life/Snake/Tetris,
  §6.1–6.3) confirm the language has the coverage** — even Tetris's line-clear (the textbook dynamic-array
  op) lowers to `filter` + index arithmetic on a fixed board. The only recurring friction is **ergonomic**:
  a carried `[0…N-1]` index array in every sketch → an **indexed `map`/`fold`** (or `range`) is the one
  solid compute-amendment candidate, and it is a convenience, not a capability gap.
- **The frontier holds, and the endgame coheres (§7.1–7.2, §9).** Doom's genuinely-new demands
  (non-local spatial queries, heterogeneous actors) are still **expressible** — blockmap = a fixed grid
  (Life-shaped), BSP = value-recursion over the map modeled as **one navigable content-addressed entity**
  (not hash-linked nodes — keeps the impure surface small). And **"prove-then-compress" grounds in the
  gradient (§6):** content-addressing the *state boundary* is orthogonal to compiling the *transition*, so
  the always-interpreted skeleton (ports + hashed state) preserves replay/reactivity while the pure step
  compiles to native. **Determinism is the single root** of expressibility, replay, *and* compilability —
  the refined end-state (I/O through compute, state machine as a compressed handler) is the entity-native
  peer in miniature.
- **The A-vs-B framing dissolves (§9.1).** Per-intermediate hashing is an **interpretation artifact**, not
  a semantic requirement — determinism lives at the *materialized boundary* (same output entity = passes
  the same conformance vectors), so the interior hashing can be dropped freely. "Compression" is a normal
  lowering pass (compute is already an IR): pure sub-DAGs → native straight-line (no hashing), impure ops →
  boundary host-calls. And **compute → WASM is where B, compiled forward, *becomes* A**: one gradient
  `entity-IR (T/1) → WASM (T/4) → native (N/4)`, A and B its two ends, not rivals. **Prove in B, compress
  toward A/native**; every rung yields the same materialized entities.
- **The port surface is bounded (§8): everything is a port; effects are *declared*, not *performed*.**
  The step emits effect *intent* as output-port state; drivers actuate it — so the program stays pure
  (replay/lockstep hold). WASI confirms the split: it standardizes streams/sockets/files but **declines
  graphics/audio** (only closed consoles pin those). → **capability tier** (network/storage/devices/
  assets) **reuses existing extensions**; the one **new, small standard is the console tier — display +
  input + audio.** Assets are free (content-addressed WADs via `lookup/hash`); **audio is the only hard
  new port** (a realtime *stream*, vs display's snapshot). The console tier is an **optional profile** —
  the headless entity-native peer needs only the capability tier.
- **The POC validated all of it empirically (§11; workbench-go absorption).** Life + Snake, built
  descriptor-faithfully, ran on the **unchanged** Stage-1 evaluator and shipped as Avalonia panels with
  **zero game logic in the driver** — verified against source three ways. Boundary-equality, two-lowering
  equivalence, and free replay are now *tested*, not argued. The measured cost profile: **~75% of
  interpreted time is per-node decode + scope-load; the materialized boundary is nearly free** — and a
  hard 100k-op budget cliff caps interpreted Life at 24×24.
- **The compile-to-handler problem is the headline endgame, and the POC number shapes it (§12).** "Runtimes
  that share no code" dissolves the way the cohort already works: **the IR is the transferable ABI, the fast
  engine is peer-local** — each arch builds its own entity-compute interpreter (the operator's framing),
  convergent not by shared code but by the shared **boundary-hash oracle** (conformance is the contract).
  Two compilation axes — a *faster general engine per runtime* (Axis 1) vs. a *faster specific program*
  (Axis 2) — and the measured 75% is **general-engine overhead**, so Axis 1 (one artifact per runtime, no
  per-program trust story) captures most of the win *first*. Compiled artifacts are **content-addressed
  tree entities** `compiled/{target}/{ir_hash}` — which is exactly where **keystone's tree-resident
  assembly translation** slots in (the bottom rung, `target = asm`), and where **WASM** is the one
  transferable-*and*-fast rung. **Doom is a rung-2/3 program:** the ladder to it is drawn, its bottom two
  rungs built and measured, the middle (a compiled general interpreter) designed-not-built — which is the
  recommended next empirical rung.

## §1 The three architectures (and why B)

"Run a real program on compute" splits three ways:

- **A — actuated WASM workload.** Compile the program (Doom is portable C) to WASM, run it in the
  actuation substrate, link its imports to entity handlers exposed as WIT interfaces (the run-environments
  doc). **Known and practical — the fastest path to a *usable* system — but it teaches us nothing new
  about entity computation.** The workload is opaque; compute isn't involved.
- **B — deterministic sim as compute IR.** Express the program's pure, deterministic simulation as a
  compute subgraph; drop to native only for the effectful hot paths. This is the one that measures the
  **real cost of entity computation** and moves toward the entity-native peer. *This doc is about B.*
- **C — an imperative VM inside compute.** Rejected as literally stated (simulating a register/opcode
  machine on a pure functional IR is interpretation-on-interpretation; if you want a bytecode VM, WASM
  *is* one — that's A). **But C names a real gap:** B's pure program is only *hostable* if there is a
  standard way to find where its input enters and its output leaves. That standard is not a VM — it is
  the **runtime interface contract** below.

Doom decomposes to make the split concrete: **simulation** (pure, deterministic — Doom demos replay
because a tic is `(state, ticcmd) → state'` over a fixed RNG table) → compute; **rendering** (BSP +
column raycast → framebuffer; hot path) → native drop-down; **I/O** (input, framebuffer, audio) →
the port seam. B keeps the first, drops the second to native, and needs the third specified.

## §2 The three-layer architecture

1. **The program** — *pure, transferable, Class T/1.* A compute subgraph: `state₀` plus
   `step: (state, input) → state'`. It reads declared input paths and derives declared output paths; it
   knows nothing about devices. "There is a state I care about (input) and a state I emit (output), and
   the tick iterates." This is the portable artifact — the thing you could someday `transfer` to another
   peer and have run identically.

2. **The interface descriptor** — *the shared semantic; the L5-standard question, answered.* A small
   manifest entity declaring the program's **seams** so any host can wire it without bespoke knowledge:

   ```
   ; SKETCH — not a ratified type. snake_case keys, kebab type names (STYLE-NAMING).
   app/program/interface := {
     state_path:    system/tree/path            ; where the current state entity lives
     initial_state: system/hash                 ; state₀
     step:          system/hash                 ; expression/closure: (state, input) → state'
     input_ports:   [{name, path, type_ref, kind: "snapshot" | "stream"}]  ; tree paths the HOST writes
     output_ports:  [{name, path|projection, type_ref, kind}]              ; paths/projections the HOST reads
     tick:          {mode: "clock-driven" | "event-driven", rate_hint?}    ; §6.2
   }
   ```

   (`kind` and the two `tick` modes are refinements the Snake sketch forced — §6.2.)

   *This* is what makes the same program hostable in the browser, in workbench-go, and in Godot: they
   all read the same descriptor and bind the same ports. It standardizes the **port entity**, never the
   I/O — the renderer gets an output-state entity and draws it however it likes; the input loop does a
   tree write at the declared input path. (Doom's framebuffer is exactly this: a **constructed entity
   field you read**, per the operator — `compute/construct` materializes bare, §GUIDE-CORE §9.)

3. **The runtime** — *per-host capability, N-class, not spec-defined in its guts.* Browser / workbench-go
   / Godot each implement the **same contract**: read descriptor → bind input ports to real input
   devices → bind output ports to real sinks (native render drop-down for hot paths) → drive the tick
   clock → host the compute eval. The operator's three external actors — **input driver, clock driver,
   output driver** — all live here, attaching at declared port seams. "A runtime with an interface,"
   not a VM.

## §3 Prior art: minimal runtimes already speak this vocabulary

The operator's instinct — *toy VMs will hint at the interface contract* — holds. Two minimal systems
reduce the **entire** I/O story to the same two primitives, and one of them is literally our three-layer
split:

| Concept | Varvara / Uxn | WASM-4 | Our model |
|---|---|---|---|
| Portable computation | Uxn CPU + ROM (stack ISA) | WASM module | **compute subgraph (the program)** |
| I/O contract | **Varvara devices** — 16 devices × 16-byte ports on a 256-byte device page (DEI/DEO) | **memory-mapped I/O** — fixed addresses (gamepad @0x16, framebuffer @0xa0) | **declared tree-path ports** (input/output) |
| Drive / event | **vectors** — `Screen/vector` @60 Hz; `Console/read` fires on input | **`update()`** called @60 Hz | **host-clocked tick** (§5) |
| Output surface | screen device (2 layers, 4 colors) | framebuffer region (160×160, 2 bpp) | **output-port entity** (constructed field you read) |
| Per-host impl | uxnds, uxn-c+raylib, browser | web/native runtimes | **the runtime layer (per-host)** |

Two load-bearing lessons:

- **The whole I/O contract is "named slots + a periodic vector."** Varvara's device *port* = a named
  slot; a device *vector* = a callback fired on an event. We already have both, better-typed: the slot
  is a **tree path holding a typed entity**, and the vector is a **reactive trigger on a write** to that
  path. Where Varvara addresses I/O by byte offset and WASM-4 by fixed address, we address by
  **structured tree path with a typed entity** — strictly richer, and it means our ports carry
  schema, not raw bytes.
- **Varvara *is* the three-layer split, shipping.** Uxn (portable computation) / Varvara (the device
  contract) / each host's implementation. This is the existence proof that the split is real and that
  the standard belongs at the **device-contract** layer, not the computation and not the host. Our
  version swaps the Uxn stack-machine for the compute IR (inspectable, content-addressed) — which is
  the whole point.

## §4 Placement: the inspectable sibling of WASM actuation

A hostable compute program is **another actuatable workload**, beside the WASM component in the actuation
family (`EXPLORATION-RUN-ENVIRONMENTS §7`), under the same "one contract, per-host runtime provider"
discipline:

| WASM actuation (scoped) | Compute-program actuation (this) |
|---|---|
| Component carries a **WIT world** (imports/exports) | Program carries an **interface descriptor** (input/output ports) |
| Host **links** imports to capability providers | Runtime **binds** ports to I/O drivers |
| **Fuel** metering | **budget** metering (already in compute) |
| capability-gated dispatch | grant-gated (already in compute) |
| Workload is an **opaque blob** (Class 4) | Workload is **inspectable, transferable data** (Class T/1) |

The difference in the last row is the reason B is worth the cost A avoids: the workload is the
**entity-native, self-describing** kind. A hostable compute program is the entity-native-peer endgame
(GUIDE-CORE §6, "the system expressed in entity-native compute over a minimal native bootstrap
evaluator") in miniature — Doom-on-compute is a stepping stone toward *the peer itself* being a
hostable compute program.

## §5 The tick must be host-driven (a real finding)

Pure compute has no time; **time is a host concern.** The discrete simulation loop `state → state'` is a
fixpoint iteration the **runtime clocks**, via explicit `eval` of the `step` each frame, writing the
result as the new state. It **must not** be a self-feeding reactive install: if `step` reads `state` and
also writes `state`, the write re-triggers the dependency → uncontrolled cascade → `cascade_limit`
freeze. The clean split:

- **Tick loop = host-clocked explicit eval.** The runtime owns discrete time and drives each step. (This
  is Varvara's `Screen/vector`-@60 Hz and WASM-4's `update()` — the *host* calls the update; the program
  does not schedule itself.)
- **Projections = reactive.** `framebuffer = f(state)` (and other output ports) *can* be reactive
  installs — they recompute when `state` changes and never feed back into it. This is where the reactive
  engine earns its keep for a program: derived outputs, not the tick.

This answers "where does the clock live": **always the host.** It also keeps determinism clean — the
sim is a pure function of `(state, input_stream, tick_count)`, so save-state / replay / lockstep fall
out of content-addressing (the striking B payoff).

## §6 First sketch — Conway's Game of Life

Chosen first because it needs **no input port yet** and still exercises state + step + tick + output —
and because scaling the grid gives a clean **throughput number**.

- **State.** `app/life/grid := {width, height, cells: [0|1, …]}` (flat row-major array; a
  `compute/construct` entity). Content-addressed, so an unchanged grid dedups to the same hash for free.
- **Step.** `grid → grid'`: for each cell index `i`, count the 8-neighborhood live cells and apply
  B3/S23. In IR this is a `map` producing the new `cells`, with a per-cell closure doing `index` reads
  at the 8 neighbor offsets + arithmetic + `if`.
  - **Finding (IR ergonomics):** `map` iterates **values, not indices**, but neighbor-counting needs the
    *index* (to read neighbors). So the step maps over an **index-range array `[0 … W·H−1]`**, not over
    `cells`. Building that range is a `fold`/helper (or a runtime-provided constant). This is a genuine
    friction point the lowering toolkit (GUIDE-CORE §8–9) will want a pattern for — "iterate with index"
    is a first-class need for any grid/array sim.
  - **Decision to pin per-program:** edge behavior (toroidal wrap vs. dead border) — wrap is cheaper to
    express (modular offsets) and standard for Life.
- **Tick.** Host-clocked (§5): runtime `eval`s `step(state)` each frame, writes `state'` to `state_path`.
- **Output port.** The grid *is* the framebuffer here — `output_ports: [{name: "grid", path: state_path,
  type_ref: "app/life/grid"}]`. A display renders `cells` as pixels; a headless profiler just counts. (A
  richer program would add a `projection` output — `framebuffer = f(state)` — as a reactive install.)
- **Descriptor (Life).**
  ```
  app/program/interface {
    state_path:    "app/life/state",
    initial_state: <hash of a seeded grid — e.g. a glider>,
    step:          <hash of the grid→grid' expression>,
    input_ports:   [],                                  ; none yet
    output_ports:  [{name: "grid", path: "app/life/state", type_ref: "app/life/grid", kind: "snapshot"}],
    tick:          {mode: "clock-driven", rate_hint: 10} ; 10 gen/s is plenty to watch
  }
  ```

What Life lets us **measure** (the cost-model deliverable): entity-ops per generation as a function of
grid size; interpreter throughput (generations/sec at Stage-1); the share of cost that is
content-hashing vs. evaluation; and later, what Stage-2 memoization saves when most of the grid is
static. This is the honest read on "what is the real cost of entity computation" — including the
possible finding that it is *too high for realtime* at Stage-1, which is a result we want, not a failure.

### §6.1 The step, concretely — and the verdict (no compute changes)

Assembled bottom-up (leaves first) with **existing primitives only**; shown nested for readability.
`offsets` and `indices` are `compute/literal` arrays carried as **subgraph data** (built once):

```
offsets = literal([[-1,-1],[-1,0],[-1,1],[0,-1],[0,1],[1,-1],[1,0],[1,1]])
indices = literal([0, 1, 2, …, N-1])          ; N = width*height — carried as data

step =                                          ; grid -> grid'
  let g     = lookup/tree(state_path)           ; the current grid entity
      w     = field(g, "width")
      h     = field(g, "height")
      cells = field(g, "cells")
      cell_rule = lambda [i]:                    ; captures w, h, cells
        let x = mod(i, w)
            y = div(i, w)
            neighbor_add = lambda [acc, off]:    ; captures x, y, w, h, cells
              let dx = index(off, 0)
                  dy = index(off, 1)
                  nx = mod(add(add(x, dx), w), w)   ; toroidal wrap
                  ny = mod(add(add(y, dy), h), h)
                  ni = add(mul(ny, w), nx)
              in  add(acc, index(cells, ni))
            count = fold(offsets, 0, neighbor_add)
            alive = eq(index(cells, i), 1)
        in  if( or(eq(count,3), and(eq(count,2), alive)), 1, 0 )   ; B3/S23
      new = map(indices, cell_rule)
  in  construct("app/life/grid", {width: w, height: h, cells: new})
```

The tick is a **host loop, not compute**:

```
; per-host runtime code (NOT a compute primitive):
loop each tick:
  r     = execute(system/compute, "eval", resource:{targets:[step_path]})
  grid' = decode(r)                 ; construct result is a bare app/life/grid entity
  tree.put(state_path, grid')       ; write it back; next tick reads it
```

**Verdict — no compute-extension changes.** Every node above is an existing compute type
(`lookup/tree`, `field`, `lambda`, `let`, `mod`/`div`/`add`/`mul`, `index`, `fold`, `map`, `compare`,
`logic`, `if`, `construct`); the tick is a host `eval`+`tree.put` loop (both already exist). The only
*new* surface is the §2 **descriptor** (an L5 application convention) and the **per-host runtime** — not
a compute primitive between them. Compute itself is untouched. (This directly answers the "is the update
a compute change or a convention?" question: **convention.**)

**Two frictions the sketch surfaced** (both real, neither blocking):

1. **`map`/`fold` expose no index.** Neighbor-counting needs a cell's *position* (= its index), but the
   collection builtins pass values, not indices — so the step maps over an **index array `[0…N-1]`**
   carried as data. Compute has collection *consumers* (map/filter/fold) but no *producer* —
   no `range`/`iota`/array-builder. Fine for fixed-size sims (the index vector is a constant); a
   **candidate future primitive, not needed for the toy**, and a first-class need for any grid/array sim
   the lowering toolkit (GUIDE-CORE §8–9) should carry a pattern for.
2. **Nested-lambda scope capture carries the load.** `cell_rule` closes over `w,h,cells`;
   `neighbor_add` closes over `x,y,w,h,cells`. This works *only because* the lambdas are created **inside**
   the scope they capture and `let` is sequential (`let*`, COMPUTE §2.1). A frontend that hoisted these
   lambdas would break — a concrete lowering rule.

**Cost signal (structural).** ≈ **100 entity-evaluations per cell per generation** (the 8-neighbor fold
is ~12 ops × 8, plus the rule) → ≈ 10²·N per generation; a 32×32 grid ≈ 100k evaluations/gen, each with
content-hash work. Watchable at small N; **orders of magnitude from 35 Hz realtime for Doom-sized work at
Stage-1.** Stage-2 memoization is especially strong here — most of the grid is static each generation, so
unchanged sub-arrays dedup to the same hash and skip recompute. Actual generations/sec is **TBD by a
workbench-go run**; the op-count is the structural floor and the honest "how far" signal.

### §6.2 Snake — the input seam (and the dynamic-collection gap)

Snake adds the one seam Life lacks — **input** — plus deterministic RNG. State is a **grid of timers**
(not a body array — see finding 4): `app/snake/state := {width, height, cells:[int;W·H] (0=empty,
k>0=segment with k ticks of life, head cell=length), head, dir, food, length, rng, status}`.

The step, in **existing primitives only** (`lookup/tree, field, construct, if, eq/neq/lt/gt/gte, and/or,
add/sub/mul/div/mod, index, length, map, filter, lambda, literal`):

```
step =                                            ; (state, input) -> state'
  let s   = lookup/tree(state_path)
      inp = lookup/tree(input_path)               ; latest direction 0..3 (snapshot input port)
  in if( eq(field(s,"status"), "dead"), s,         ; frozen when dead — no-op tick
     let W=…  H=…  cells=…  head=…  dir=…  food=…  len=…  rng=…    ; fields of s
         ndir = if( eq(inp, mod(add(dir,2),4)), dir, inp )         ; 180° reversal guard
         dx = if(eq(ndir,1),1, if(eq(ndir,3),-1, 0))               ; bool is NOT numeric →
         dy = if(eq(ndir,2),1, if(eq(ndir,0),-1, 0))               ;   branch with if, not arithmetic
         nx = add(mod(head,W), dx)   ny = add(div(head,W), dy)
         hit_wall = or( or(lt(nx,0),gte(nx,W)), or(lt(ny,0),gte(ny,H)) )
         nhead = add(mul(ny,W), nx)
         ate   = eq(nhead, food)
         hit_self = gt( index(cells,nhead), if(ate,0,1) )           ; tail vacates when not eating
     in if( or(hit_wall,hit_self),
            construct("app/snake/state", {…all fields…, status:"dead"}),  ; construct has NO spread
            let nlen  = if(ate, add(len,1), len)
                idxs  = literal([0 … W·H-1])                        ; static, carried as data (Life trick)
                ncells= map(idxs, lambda [i]:
                          if( eq(i,nhead), nlen,
                              if( ate, index(cells,i),
                                  let v=index(cells,i) in if(gt(v,0), sub(v,1), 0) )))
                nrng  = mod( add(mul(rng,1103515245),12345), 2147483648 )   ; LCG — pure, deterministic
                free  = filter(idxs, lambda [i]: and( eq(index(ncells,i),0), neq(i,nhead) ))
                nfood = if( ate, index(free, mod(nrng, length(free))), food )   ; pick a free cell
            in construct("app/snake/state", { …, cells:ncells, head:nhead, dir:ndir,
                                               food:nfood, length:nlen, rng:nrng, status:"playing" }) ))
```

Input seam + tick (host code, not compute): on keypress the runtime writes the latest direction to the
input port (`tree.put("app/snake/input/dir", …)`, last-write-wins); the **clock-driven** tick loop evals
`step` (which reads *two* paths — state and the input port) and writes back. Input is sampled at the tick
boundary — classic game-loop handling.

**Findings:**

1. **Input port = snapshot, sampled at tick — validated.** Last-write-wins tree path; two keys between
   ticks → only the last is seen (correct for Snake). A **stream** input port is needed only when events
   can't be dropped (combos, a text field). → the descriptor's `input_ports` needs a **`kind`** field.
2. **Two tick modes, both expressible today.** Snake is **clock-driven** (moves on its own → host loop
   over `eval`); an editor is **event-driven** (tick *on* input → a reactive install keyed on the input
   port). → the descriptor's `tick.mode` names both; the reactive machinery already covers event-driven.
3. **Deterministic RNG is free** — an LCG seed in state advanced by arithmetic (pure → replay/lockstep
   hold; the Doom-RNG shape at small scale). "Pick a *free* cell" is a **`filter`**, not a retry loop.
4. **The real gap — dynamic collections — dodged by representation.** The natural Snake state is a **body
   array** you prepend a head to and drop the tail from each tick — but compute has **no array producer**
   (`cons`/`append`/`slice`/`range` all absent; map/filter/fold only *consume*). The **grid-of-timers**
   representation sidesteps it (fixed-size grid, Life's regime). Lesson: **IR ergonomics push you toward
   fixed-size representations.** Strong motivation for a future `range`/array-builder — *not forced*, you
   re-model. (Doom's mobj list will hit this harder.)
5. **Two minor lowering rules:** bool is **not** numeric (a `compare` → bool; arithmetic on it →
   `type_mismatch`; branch with `if` to get 0/1), and `construct` has **no spread** (copy-all-change-one
   re-lists every field).

**Verdict: still zero compute changes** (given a fixed-size representation). Descriptor survives with two
refinements now folded into §2 (`kind` on ports; `tick.mode` ∈ {clock-driven, event-driven}).

**Cost calibration (the encouraging bit).** Snake's per-tick work is ~O(N) over the grid but *light*
per cell (~10 ops, no neighbor fold): a 20×20 grid at 8 moves/sec ≈ 400 × 10 × 8 ≈ **~32k evals/sec —
comfortably realtime on the Stage-1 interpreter.** So **simple games run realtime on interpreted compute
today.** The cliff is not the loop shape — it is per-entity complexity × count (large-grid Life, Doom's
thousands of mobjs).

### §6.3 Tetris — stress-testing the dynamic-collection gap (and resolving it)

Tetris was chosen to *break* the fixed-size dodge: **line clearing** ("remove full rows, shift down") is
the textbook dynamic-array operation. It doesn't break. The modeling move: **the board is fixed-size, and
the falling piece is scalars** — `{board:[color;W·H], piece:{type,rot,x,y}, rng, gravity, score, lines,
status}`. Rotation/wall-kick tables are static literal data (rotate = a table `index`, not array surgery).

The line clear — the stress test — in existing primitives only:

```
row_full  = lambda [y]:                                   ; row y has no empty cell?
  fold(literal([0 … W-1]), true,
       lambda [acc,x]: and(acc, neq(index(board, add(mul(y,W),x)), 0)))
kept_rows = filter(literal([0 … H-1]), lambda [y]: not(row_full(y)))   ; bounded, ≤ H
num_full  = sub(H, length(kept_rows))
new_board = map(literal([0 … W·H-1]), lambda [i]:                      ; survivors pile at bottom
  let y=div(i,W)  x=mod(i,W) in
  if( lt(y, num_full), 0,
      index(board, add(mul(index(kept_rows, sub(y,num_full)),W), x))))
```

`filter` produces the surviving rows (a bounded variable-length array); the compaction is pure **index
arithmetic** — no `cons`/`append`/`slice`/`range`. The gap is **not hit.** Every "dynamic" Tetris op
reduces to two existing moves — `filter` (bounded subset) and `map`-over-a-static-index-range (transform,
even functional element-update `map(idxs, λj. if(eq(j,i),v,index(arr,j)))`):

| Tetris op | Reduces to |
|---|---|
| rotate / wall-kick | `index` into static rotation/kick tables |
| lock piece into board | `map` over board + 4-cell membership test |
| **line clear** | **`filter` + index arithmetic** |
| collision / valid-move | `fold` over the piece's 4 cells |
| hard-drop / ghost | `fold` over bounded range (≤ H) |
| 7-bag shuffle (optional) | functional update via `map`-over-indices |
| piece RNG | LCG seed in state |

**Verdict: Tetris needs zero compute changes; the language has coverage for this whole class.** The
missing array-producers (`range`/`cons`/`append`/`slice`) are only needed for **unbounded or from-nothing
generation of computed length** — grid games never hit that, because every output is a *subset or
transform of a bounded structure* (`filter` covers subsets; `map`-over-range covers transforms/updates).

**The one real finding — an ergonomic gap, not a capability one.** All three toys carry a static index
array `[0 … N-1]` as literal data, `map`/`filter`/`fold` over it, then `index` *back* into the real data —
in every neighbor loop, board rebuild, and row scan. That double-indirection is the single recurring
friction. Fix candidates (either would be MUST-given-COMPUTE per the transferability rule, GUIDE-CORE §7):
1. **Indexed `map`/`fold`** — the closure receives the **index** alongside the element. Kills both the
   carried index-array and the index-back step. *(Lean: the most direct fix; the strongest lowering-toolkit
   addition, with all three toys as evidence.)*
2. **`compute/range` / `iota`** — produce `[0…n-1]` from a count; removes the literal boilerplate but keeps
   the index-back indirection.
Neither is *required*; this is the exploration's first solid spec-amendment candidate, motivated by
ergonomics, not capability.

**Cost note.** Because the piece is **scalars, not a grid**, most ticks are ~O(1) (move/rotate = a 4-cell
collision check + scalar updates); the board is rebuilt only on **lock + clear** (infrequent). The render
*projection* (overlay piece on board = `map` over 200 cells) runs at display rate. Generalizable lesson:
**keep the mutable-per-tick state small (scalars); rebuild big grids only on discrete events** — the
compute-native form of delta/dirty-rect optimization, forced by the cost model.

## §7 The progression (scope & scale ladder)

Each rung adds exactly one new demand on the contract:

1. **Life** — state + step + tick + output. *No input.* Throughput probe. **(start here)**
2. **Snake** — adds the **input port** (direction), a small **deterministic RNG** (food), score/state.
   First full four-seam program.
3. **Tetris** — adds **deterministic RNG as a first-class concern** (the piece bag; the Doom-RNG-table
   shape at small scale) and gravity-vs-input timing.
4. **Doom** — adds **scale** (thousands of mobjs/tic → the Stage-1 realtime cliff, the real cost read),
   the **native render drop-down** (raycast → framebuffer constructed-entity), and **audio** as a second
   output port. The stress test that tells us how it all maps at real-world complexity.

### §7.1 What Doom actually introduces (the frontier)

The toys are grid-local and uniform; Doom is neither. Separating what merely *scales up* from what is
*genuinely new*:

**What Doom does NOT introduce (dodges, like the toys):**

- **Dynamic heterogeneous actors → a fixed-size object pool.** Doom's mobj list (monsters, projectiles,
  items) spawns and dies each tic — the dynamic-collection case the grids avoided. Resolution is the same
  bounded trick, and it is *literally how idTech works*: a **fixed-size pool (`MAX_MOBJS` slots) with an
  alive flag**, `map`/`filter` over the pool. So spawn/death stay within existing primitives — no
  array-producer needed. (Confirms §6.3's rule at scale: bounded pool, not unbounded growth.)
- **Static world geometry → content-addressed data.** Walls/sectors are read via `lookup/hash` (pure);
  moving sectors (doors, lifts) are small dynamic state. Assets (the WAD) are content-store entities — free.
- **RNG → a carried table.** Doom's 256-byte RNG table is literal data + an index in state — exactly
  Snake's LCG-seed pattern, and the source of demo determinism.
- **Fixed-point math → integer arithmetic — and it is a *gift*.** Doom uses 16.16 fixed-point (never
  float), so its math maps onto compute's 64-bit int ops directly *and* stays cross-impl deterministic —
  float is the determinism risk (RUN-ENVIRONMENTS §4); Doom's historical choice aligns perfectly with the
  compute determinism discipline. The sim stays replay-exact across implementations.

**What Doom GENUINELY introduces (the new structural challenges):**

- **Non-local interaction / spatial queries — the big new thing.** Life/Snake/Tetris are *local* (fixed
  neighbor set). Doom actors interact *non-locally*: hitscan across the map, monster sight/targeting, sound
  propagation, radius damage. Naive "each actor scans all actors" is **O(N²)**. Real Doom uses the
  **blockmap** (spatial hash) + **BSP** (visibility) — fixed-size auxiliary indices, expressible, but now
  the O(N²)-vs-spatial-index choice is a **cost lever *inside* the sim**, and the clean `map`-over-cells
  locality is gone. This is the frontier the toys never touched.
- **Heterogeneous actors → tag-dispatch.** A uniform grid needs no per-element type switch; a mobj pool
  does (monster vs. projectile vs. item behave differently). This is the `LowerMatch` / `if`-`eq`-chain on
  a `.data` type tag (GUIDE-CORE §9, the sum-type pattern) — expressible, but per-actor dispatch is new
  per-tic work.
- **BSP traversal / recursion at scale.** Sight and collision are BSP walks — recursion, which compute
  supports via TCO + `lookup/tree` self-reference, but heavy per-actor-per-tic.
- **The render + audio native drop-downs become load-bearing.** The toys' "render" was a grid blit. Doom's
  renderer (BSP walk + column raycast + texture + lighting) is a serious **native** workload — the
  canonical §5 drop-down. It reads sim state (view, sector heights, sprites) and writes the **framebuffer
  output port**; the mixer reads sound events and writes the **audio stream port** (the hard one, §8). The
  sim/native boundary — sim in compute (deterministic, portable), render/audio native (fast, opaque) — is
  now genuinely exercised, with the output-port entities as the clean seam.

**A striking confirmation:** Doom **demos are a recorded stream-input-port** replayed against a
deterministic sim — *exactly* what this architecture gives for free (§5's replay payoff). The two things
Doom hand-built — a deterministic sim and input-stream recording — are **native** to the pure-compute
model. Doom validates the architecture rather than straining it *conceptually*; where it strains is **cost**.

**Cost verdict.** Thousands of mobjs × heavy per-mobj AI × BSP/blockmap traversals × ~10²-ish entity-evals
per nontrivial op × 35 Hz → **orders of magnitude beyond Stage-1 interpreted realtime.** But it decomposes
cleanly: the **pure sim is expressible and deterministic *now*** — a correct, content-addressed, replayable
Doom-sim is a real artifact (headless verification, demo/netcode reference) *independent of framerate*;
**realtime is a Stage-3 (native-compiled IR) question**, where — unlike Life — memoization helps less
(state churns hard each tic) and native compilation is the lever (§9). Doom is the case that says: the
architecture is *correct* well before it is *fast*.

### §7.2 Blockmap & BSP — the spatial machinery is expressible (coverage confirmed)

Doom's two spatial structures — the reason §7.1's non-local queries aren't O(N²) — check out against the
primitive set with **no gap**:

- **Blockmap (spatial hash)** is a **fixed-size 2D grid of linedef-lists** — structurally *Life's grid*.
  Query: block = `div(x,128), div(y,128)` (a power-of-two divide — no bit-op needed) → `index` the grid →
  `fold` the cell's bounded linedef list. Precomputed at map load → **static content-addressed data**. The
  O(N²)→O(N) index that makes Doom feasible is itself expressible.
- **BSP (visibility / point-in-subsector / sight)** is a **tree walked by recursion**: side-of-line test
  (cross product = arithmetic + `compare`) then recurse into a child; depth ~log(map), within limits;
  sight adds interval clipping (arithmetic). Expressible via TCO; nothing missing.

**Design finding (pin this):** a big static structure like the map is modeled as **one content-addressed
entity navigated internally with `field`/`index`, not as thousands of hash-linked node entities.**
Per-node `lookup/hash` would explode a reactive subgraph's authorized-hash set *and* add impure edges;
navigating one immutable value with `field`/`index` is **pure** and needs no authorization. The BSP
recursion walks the *value*, not a tree of entities — the "carry static data as one literal" pattern at
map scale, and what keeps the sim's impure surface small.

## §8 The port taxonomy — is display + audio "everything"?

Neither A nor B needs a new *execution engine*. What B forces is the **port surface**, and the sharp
question is how big it has to be.

**The model: everything is a port; effects are *declared*, not *performed*.** The step is
`(state, input_ports) → (state', output_ports)`. Input ports are what drivers write *in* (keys, pointer,
incoming packets, the clock); output ports are what drivers read *out* and actuate (framebuffer, audio,
outgoing packets, storage writes). The program **never performs an effect — it emits effect *intent* as
output-port state, and a driver actuates it.** That is what keeps the step pure/deterministic and
preserves the save-state/replay/lockstep payoff, and it collapses display, audio, *and* network into one
"port" concept. (The impure `compute/apply` drop-down stays available for service work that doesn't care
about replay — §5's reactive/impure escape; a game shouldn't use it.)

**Two surfaces, and WASI confirms the split.** WASI 0.2/0.3 standardizes a generic **stream** primitive
and builds sockets/files/http on it — but pointedly has **no standard graphics or audio** interface.
Display/audio are only standardizable inside a *closed console* (WASM-4, Varvara). So:

- **Capability tier (open) — reuse, don't invent.** network, storage, assets, random, devices → existing
  extensions (NETWORK/RELAY/INBOX, DURABILITY/tree, the content store, the device proposal). A program
  reaches them as ports over **existing entity types**, or via the drop-down. **No new standard.**
- **Console tier (closed) — the one small new standard.** display, input, audio. No external standard to
  borrow (WASI declines to); define a closed console like the fantasy consoles did. **Small, bounded.**

So — **is display + audio everything? For the genuinely-new standard, essentially yes: display + input +
audio** (the console tier). Everything else is ports over entity types we already have.

**Two port *kinds* (the mechanics already exist).** Snapshot port = a **tree path** (last-write-wins;
lossy-OK) → display, pointer. Stream port = an **inbox/subscription** (ordered, lossless, backpressured)
→ input events, audio samples, packets. This is WASI's io-stream; our inbox is the mechanism.

**Classification (the whole surface):**

| Surface | Dir | Kind | New standard? | Home |
|---|---|---|---|---|
| Display / framebuffer | out | snapshot | **yes (small)** — frame entity + format set | new console port |
| Input — keys/buttons | in | stream | **yes (small)** — event entity | new console port |
| Input — pointer | in | snapshot | yes (small) | new console port |
| **Audio** | out | **stream** | **yes — the hard one** (PCM + realtime timing) | new console port |
| Clock / tick | in | driver | contract only | host / EXTENSION-CLOCK |
| Random | in | effect | reuse | impure handler |
| **Assets (Doom WAD)** | in | snapshot | **none — content-addressed** | content store (`lookup/hash`) |
| Persistent storage / save | out | snapshot | none | tree / DURABILITY |
| Network | in+out | stream | none | NETWORK / RELAY / INBOX |
| Devices / sensors | in | snapshot | none | the device proposal |

Two things fall out: **assets are free** — Doom's WAD is content-store entities read by `lookup/hash`
(pure, no filesystem port); and **audio is the only genuinely hard new port** — a realtime *stream* with
ordering/backpressure/timing, unlike display's drop-stale-frames snapshot.

**Read both ways (program ↔ host).** The program's descriptor declares its **ports** (in/out) and its
**imports** (which capability handlers its drop-downs need — the WIT-world analog). The host provides
**console drivers** (bind display/audio/input) plus the **handler set** the imports require. Admission =
**match the program's imports against the peer's offered capabilities** — exactly the `system/device`
*offered-capabilities* field the run-environments loop-back scoped (§5/§8 there). The two explorations
meet here; a real program is the concrete *consumer* that justifies specifying host interfaces (there as
WIT, here as **typed tree-path ports**).

**The peer endgame: the console tier is a profile (optional).** A headless *service* program — the
entity-native peer itself — uses **only** capability ports (network, storage) and needs no display/audio.
Display/input/audio are the *interactive-app* profile (Doom). So building the full entity-native peer does
**not** require the console standard; it rides the capability tier we already have.

**The still-new surface, then, is small:** (1) the **interface descriptor** (§2) — an L5 app-convention
(`app/program/interface`, alongside `APP-CONVENTION-*`); (2) the **console-port entity types** (frame /
input-event / audio-buffer) + the snapshot/stream port kinds; (3) the **host clock/tick** contract
(EXTENSION-CLOCK as candidate driver, §5); (4) the thin **runtime contract** (read descriptor → bind
drivers → clock tick → host eval), fat per-host — the device/actuation "one contract, per-runtime
provider" discipline.

## §9 Prove-then-compress — the compilation endgame (what stays in compute)

The long-term shape (operator's framing, grounded in the compilation gradient, GUIDE-CORE §6): **write the
program in compute to prove it correct/portable/deterministic; then "compress" hot paths to compiled native
handlers.** In the most refined case the compute engine handles only the **I/O + state boundary**, and the
Doom state machine is a **compiled handler**. The load-bearing rule for *where the cut goes*:

**Content-addressing the *state boundary* is orthogonal to compiling the *transition*.** Replay /
save-state / lockstep depend only on **state entities being content-addressed at tick boundaries** — not on
the transition being interpreted. So:

- **Always-interpreted skeleton (cheap, ~O(1)/tick):** read input ports → `compute/apply` the step → write
  content-addressed `state'` → write output ports. This thin shell *is* the entity-system properties —
  hashable/trackable/replayable state, reactive ports, transferable state. It is never the bottleneck.
- **Compiled interior (Stage 3):** the step as a native handler invoked via `compute/apply` — internally
  native/fast, externally a content-addressed entity transform `state(hashed) → state'(hashed)`.

Two rules the cut must respect (from §6's four-way coincidence):

1. **The compiled interior must be pure** — a function of its entity inputs, no hidden tree reads; any
   impure/dependency edge stays in the interpreted skeleton, or reactivity goes blind. **Doom's step *is*
   pure** (`(state, input) → state'`; all reads are passed-in state or the immutable map), so it compiles
   cleanly.
2. **Compiled is a cache *beside* the source IR, never a replacement** (§6's preserve-the-IR invariant) —
   inspectability and self-hosting survive; the interpreted version is always the fallback.

The cut is **adjustable** — compile the whole step, or just the hottest sub-function (BSP sight, mobj AI),
leaving the rest interpreted. "Compression" = choosing where to draw the `compute/apply` line; the
content-addressed entities at that line stay inspectable/hashable.

**The through-line: determinism/purity is the single root of three payoffs.** The property that makes Doom
*expressible* in compute (deterministic pure sim) is the *same* one that makes it *replayable*
(content-addressed state) and *compilable* without losing reactivity (pure interior collapses, impure
skeleton stays). Choose B and all three follow from one root. And the refined end-state — I/O + state
tracking through the compute engine, the state machine as a compressed handler — is **the entity-native
peer in miniature**: a minimal interpreted skeleton over compiled hot handlers, source IR preserved for
self-hosting.

### §9.1 How the reduction works — and is the compiled result transferable?

**The realization that makes compression safe: per-intermediate hashing is an *interpretation artifact*,
not a semantic requirement.** Determinism is pinned at the **materialized boundary** — the canonical CBOR
+ content hash of entities that cross into the entity system (state, a stored `construct`, an `apply`
arg). It is *not* "hash every `add`." The cross-impl gate is "same **materialized** hash, == hand-built"
(the validate-peer check) — silent on intermediate hashes. So hashing `state_n → state_{n+1}` is
load-bearing; the interpreter's *interior* cost is an implementation artifact — precisely: **walking and
CBOR-decoding the expression graph every eval, plus materializing + hashing at interior boundaries**
(each `construct` that gets stored, `apply` arg on a hash-typed field, scope capture). A compiler removes
both: the graph is walked once at compile time, and interior values live in registers, materializing only
at the step's I/O boundary. **Dropping the interior work leaves replay/save-state/lockstep untouched** —
they are a function of the boundary entities only.

**The reduction is a standard lowering pass — and compute is already an IR, so it is IR→lower-IR:**

1. **Partition pure vs. impure** — free: compute marks the frontier by lookup type (`scope`/`hash` pure,
   `tree` impure-reactive, §6).
2. **Collapse pure sub-DAGs to native straight-line code** — `arithmetic`→arith, `if`→branch, `let`→local,
   `lambda`/`apply`→function, `index`/`field`→memory access. Intermediates become **registers/stack — no
   entity, no CBOR, no hash.** *This is the reduction:* delete the interior's entity representation.
3. **Emit impure ops as host calls at the boundary** (`lookup/tree` → a real read; state write → a real
   materialization).
4. **Canonicalize only at the boundary** — emit canonical encoding + hash bit-identically to the
   interpreter where a value crosses back; interior stays unhashed.

"Same semantics" is then concrete: the compiled handler produces **bit-identical boundary entities** and
**passes the same conformance vectors** — equivalence is at the materialized boundary, not the internal
steps.

**Transferable without the hashing? Two forms — the second is the target:**

- **Form A — transfer the IR, compile locally.** The source IR stays Class T (unchanged); each peer
  compiles it **per architecture** → the hashing-free fast version is generated on arrival. The *recipe*
  transfers; native code (arch-specific) does not, and needn't. (GUIDE-CORE §1: "interpreted now; native
  per local machine later.")
- **Form B — compile to WASM.** A genuinely **transferable, hashing-free, same-semantics** handler:
  arch-independent bytecode, no per-op content-addressing, native-ish speed. Trade-offs are exactly the
  expected ones — it goes **opaque** (Class 4) and its **impure edges become explicit WASM imports** (the
  capability model re-surfaced: imports *are* the effect boundary).

**The unification: compute → WASM is where Architecture B, compiled forward, becomes Architecture A.** The
gradient is one line, and A/B are its two ends, not rivals:

```
entity-compute IR              →   WASM                        →   native
transferable, inspectable,         transferable, opaque,           local, opaque,
per-op hashed, slow  (T / 1)       not per-op hashed, fast (T / 4)  fastest      (N / 4)
```

So the opening "A (WASM) vs. B (compute)" question **dissolves**: **prove in B** (correct, portable,
deterministic, inspectable), then **compress toward A/native** for speed — and because determinism lives
at the boundary, every rung yields the same materialized entities. Source IR is preserved beside the
compiled artifact (§6), so the inspectable version and self-hosting survive. What you spend sliding right:
interior inspectability + automatic interior memoization (native recompute beats hash-lookup anyway). What
every rung keeps: boundary determinism, replay, reactivity, and the transferable source recipe.

## §10 What determinism buys: netcode, streaming audio, and the non-determinism boundary

### §10.1 Netcode / lockstep — a *better* substrate than hand-rolled engines

Deterministic lockstep (RTS-style: sync thousands of units over tiny bandwidth) needs a bit-identical
deterministic sim — the thing real engines struggle to guarantee. Entity-compute *enforces* it:

- **Input-only sync.** Peers exchange only the input streams (ticcmds) over the existing cross-peer
  inbox/relay seam — never state. Tiny bandwidth.
- **Substrate-enforced determinism (the headline).** The desync-bug class that plagues hand-rolled lockstep
  (float drift, uninitialized memory, ordering) is *structurally prevented* by compute's determinism
  discipline (canonical encoding, integer/fixed-point, no float ambiguity). Determinism is **guaranteed,
  not hoped-for** — and it reduces to compute's **existing conformance suite** (a Go peer and a Rust peer
  converge on the same state hash by the same vectors), *provided* the sim avoids float (the known cohort
  float16 risk) — which fixed-point sims do.
- **Free desync detection** — content-addressed state ⇒ peers compare `state_n` hashes, no checksum
  machinery.
- **Free rollback / save-states** — content-addressed states *are* the snapshots; GGPO-style rollback
  re-sims from a past state hash. State management is free; only re-sim *speed* is the cost question.

Coordination note (runtime, not a gap): lockstep adds a **tick barrier** — the tick fires only once all
peers' inputs for tick N have arrived (a third consideration beside clock-/event-driven).

### §10.2 Streaming audio — chunks over a stream port; the difficulty is rate decoupling

Audio is a **stream output port**: each tick the program appends a PCM chunk of `sample_rate / tick_rate`
samples (Doom: 44100/35 ≈ 1260/tick). The inbox/stream mechanism carries it. The genuine difficulty is
**not the bytes** but **rate decoupling**: audio's consumer clock (the hardware sample rate) is
**authoritative *and* lossless** — you can't drop a sample without a click, and it pulls relentlessly. So
the producer must **stay ahead** via a small look-ahead buffer (buffer depth = latency); a stalled tick
underruns. Standard audio-engineering tradeoff — a **runtime** concern, no compute gap.

### §10.3 The non-determinism boundary (the clarification that closes the model)

**The determinism boundary wraps the sim *state*, not the audio/video *projection*.** The sim emits
deterministic *events/positions* (integer, synced, in state); native drivers render them — the mixer to
PCM, the renderer to pixels (float OK, local, **per-peer, not synced**). **Audio and video are *local
projections of synced state*, never synced themselves** — each peer renders its own. This separates the
deterministic core (integer, hashed, replayed) from local renders (float, native).

Consequently the output-port model sharpens: **materialize a projection as an entity only when you need it
inspectable / transferable / recordable** (gameplay recording; **remote display = subscribe to the
framebuffer output port** — entity-native spectating / cloud gaming). Pure local play lets the native
renderer draw straight to screen. The port is the *contract*; materialization is optional per use.

**The invariant that closes the gap sweep:**

> **Anything non-deterministic or external — wall-clock, true randomness, user input, network — enters as
> an *input port*, recorded in the input stream; the sim stays a pure function of `(state, input-stream)`.**
> The input stream is the **complete record of all non-determinism** — which is exactly why replay works.

Odds and ends all resolve through it: save/load (content-addressed state), pause (stop the tick), frame
interpolation (a local projection lerping two states), wall-clock/random-as-input, program composition
(wire one program's output port to another's input port). **No remaining capability gaps** beyond the
ergonomic indexed-`map`/`fold` candidate (§6.3).

## §11 Open questions & next

- **Descriptor shape** — is it an L5 app-convention or a `system/*` type? (Lean app-convention; it is a
  usage pattern over compute + tree, not a protocol primitive.) Field set now pressure-tested by three
  programs (Life/Snake/Tetris); ready to write up.
- **Indexed `map`/`fold` (or `compute/range`) — the one solid compute-amendment candidate (§6.3).** All
  three toys carry a static `[0…N-1]` index array and index back into the data; an indexed collection
  builtin kills that recurring double-indirection. **Ergonomic, not a capability gap** (Tetris proved
  `filter` + `map`-over-range cover the whole class). Would be MUST-given-COMPUTE. Route into the
  lowering-toolkit / compute-standardization track, not this doc.
- **Clock ownership** — `EXTENSION-CLOCK` as tick driver vs. pure-runtime concern (§8).
- **Audio — the one hard console port (§8).** A realtime *stream* (PCM samples) with ordering,
  backpressure, and timing against the host clock — unlike display's drop-stale-frames snapshot. Does the
  program produce a sample buffer per tick (simple, but couples audio rate to tick rate), or append to a
  stream/inbox port the audio driver pulls at its own rate (decoupled, the real model)? First hard design
  in the console tier; defer until a program actually needs sound (not Life, not Snake).
- **Snapshot vs. stream port kinds (§8)** — pin the two kinds (tree-path last-write-wins vs.
  inbox ordered-lossless) as the port-mechanism vocabulary in the descriptor.
- **Cost model** — ✅ **done** (workbench-go POC, absorbed in
  `docs/research/reviews/ABSORPTION-compute-program-poc-life-snake.md`): Life across grid sizes on the
  Stage-1 evaluator, with the CPU profile (**~75% per-node decode + scope-load, the materialized boundary
  nearly free**) and the 100k-op budget cliff. The compile-to-handler problem it opens is now **§12**.
- **Next:** two tracks are now ready. (a) **Empirical:** run the Life/Snake/Tetris step graphs on the
  Stage-1 evaluator (workbench-go) for real throughput numbers — the cost-model deliverable. (b) **Spec:**
  the descriptor has survived three programs → write it up as a proposal (the L5 `app/program/interface`
  convention + snapshot/stream port kinds + `tick.mode`). The console-port entity types (frame / input /
  audio) and the runtime contract remain exploration until the descriptor lands.

---

## §12 The compile-to-handler problem — transferable compute, local execution

The POC (§11, absorbed) closed the empirical open question and opened the next one, now the track's
headline. §9/§9.1 argued the *mechanism* (compression is a lowering pass; determinism lives at the
boundary; Form A = transfer-IR-compile-locally, Form B = WASM). This section makes it an **ecosystem
architecture** — because the real question was never "can we compile a step," it is **"how does a
compiled fast path stay transferable across go / rust / py / assembly runtimes that share no code?"** The
POC handed us the number that decides the shape.

### §12.0 The problem, restated with the number in hand

We have entity-compute: a portable, content-addressed, purely-functional IR. We have handlers: fast,
native, operational. The compile question is *what carries you from the first to the second without losing
the first.* The POC profile (§11) sharpens it: **~75% of interpreted cost is per-node CBOR-decode +
scope-load — the general interpreter's overhead — not the program's arithmetic** (that is noise) and
**not the boundary hashing** (also noise). So "make compute fast" is overwhelmingly "**stop re-decoding
the graph every eval**," and the part that buys transferability/replay (materializing at the boundary) is
already nearly free. That reframes everything below.

### §12.1 The move that resolves "share no code": the IR is the ABI, execution is peer-local

The worry in the phrase *"runtimes that share no code"* dissolves the same way it already does for the
cohort. Go, Rust, and Py peers **share no implementation code today** — they share the **spec + the
conformance suite**, and interoperate. Compilation is the same pattern one level down:

> **The transferable artifact is the IR. The fast thing is peer-local. They are never the same object.**
> Each runtime brings its own **entity-compute execution engine** (interpreter, then compiled interpreter,
> then native) the way each CPU brings its own decoder for one shared machine code. You ship the *program*
> (the IR + descriptor); the peer supplies the *reader*. Nobody ships readers.

This is exactly the operator's framing — *every architecture builds its own entity-compute handler
interpreter, running the same semantics inside a handler, hashing only at the boundaries.* The IR is the
lingua franca; the compiled interpreter is each arch's fast dialect-reader. **Convergence is guaranteed
not by shared code but by the shared oracle** (§12.5): a Go compiled interpreter and a Rust one agree
because they produce the same boundary-hash sequence on the same vectors — the §9.1 equivalence criterion,
which the POC already tests in miniature (`TestExpLifeD3_LoweringsConverge`: two lowerings, identical
boundary hashes every generation).

### §12.2 Two axes of compilation — and the number says which comes first

§9.1 spoke of "compiling the step." There are really **two distinct things you can compile**, and
conflating them hid which one to build:

- **Axis 1 — a faster *general* engine (one per runtime).** A compiled/specialized interpreter that runs
  *any* IR but skips the general path: it decodes the expression graph **once** into an in-memory form
  (or native closures) and evaluates that, instead of `cbor.Unmarshal`-ing every node every eval. One
  artifact per runtime; runs every program the peer will ever host.
- **Axis 2 — a faster *specific program* (one per program).** Specialize a single step to straight-line
  native/WASM, intermediates in registers (§9.1's lowering pass). One artifact per program; fastest, but a
  compile step and a distribution/trust story *per program*.

**The POC's number says Axis 1 captures most of the win with the least machinery.** The measured 75% is
*general-engine overhead* — per-node decode + scope reload — which Axis 1 removes for **every** program at
once, with a single per-runtime engine and **no per-program artifact to ship, validate, or trust.** The
cheapest rung of all (the Stage-1.5 decoded-node cache, >50% for almost no work — POC §2) is the toe of
Axis 1. Axis 2 is the *second, smaller* increment — it buys the last slice (branch prediction, register
allocation, killing the interpreter dispatch loop itself) and it is where the hard transfer/trust
questions live, so it is right to do it **after** Axis 1, only for the hot programs that need it. The
operator's instinct — build the fast general handler-interpreter per arch first — is the higher-leverage
order, and the profile is why.

### §12.3 The execution-strategy ladder (one gradient, IR-preserving at every rung)

Both axes are rungs of a single ladder. Every rung consumes the **same IR** and is gated by the **same
boundary-hash oracle**; you slide right for speed and lose only *interior* inspectability (§9.1), never
the boundary or the source recipe:

```
rung  strategy                         removes                         artifact / where           transfer
────  ───────────────────────────────  ──────────────────────────────  ─────────────────────────  ────────────────
 1    Stage-1 tree-walking interp      —  (baseline; the fallback)     none (the engine)          IR only
1.5   decoded-node cache               re-decode within a run          none (engine, per-run)     IR only
 2    compiled general interp  ◄Axis1  per-node decode + scope reload  one engine per runtime     IR only
 3a   per-program → WASM      ◄Axis2   interpreter dispatch entirely   compiled entity, per prog  IR + (opt.) WASM
 3b   per-program → native/asm ◄Axis2  everything but the boundary     compiled entity, per arch  IR + (opt.) native
```

Rungs 1–2 ship **nothing but the IR** — the peer's engine does the rest, so they are transferable by
construction and need no new trust story. Rungs 3a/3b optionally ship a *compiled artifact* as an
acceleration cache, which is where §12.4–§12.5 apply. Key invariant, unchanged from §9: **rung 1 is always
present as the fallback** — a compiled artifact is a cache *beside* the IR, never a replacement, so no rung
is a hard dependency and a peer that can't run the fast form still runs the program.

> **A note on "rung" vs. core-go's "Stage-N" — they are different axes; don't conflate the numbers.**
> Core-go's *Stage-N* names an **evaluator-maturity** gradient (Stage-1 tree-walker → **Stage-2
> memoization** → Stage-3 native). This "rung" ladder is a **transferability × compilation** decomposition,
> and the digits do **not** line up: rung 1 *is* Stage-1 and rung 3 *is* Stage-3 (native), but **rung 2 (a
> compiled *general* interpreter, Axis 1) is not Stage-2 (memoization)** — they are orthogonal wins
> (decode-once vs. result-caching) that happen to share a digit. Where this doc says "rung," it means this
> ladder; Stage-N still means the evaluator gradient. The Axis-1 prototype (§13) is **rung 2**, deliberately
> ahead of Stage-2 memoization because the POC profile says decode-once is the bigger, program-independent
> win.

**MEASURED (Axis-1 absorption, 2026-07-16) — the ladder has a near-term terminus at rung 2.** The Axis-1
prototype landed rung 2 and inverted the case for climbing further: **34–163× on the whole tick**, and a
whole-tick profile showing the residual interpreter dispatch is **~32% of a tick** — so a *perfect* rung-3a
bytecode buys only **~1.5×**, against the 34–163× already banked. **The evaluator has stopped being the
bottleneck.** So rung 3 (per-program native/bytecode) is **evidence-deprioritized**: don't open it until a
program's *non-eval* remainder (dispatch, put, decode — now the larger share, and not a compute-extension
concern) becomes the target. The transferability spine (§12.1) is unchanged and *confirmed* — Axis-1 is
exactly the per-language engine the IR-as-ABI model predicted, and the D3 cross-engine oracle held on all
four lowering×engine pairings. Two sharper facts the rung landed: the win is **asymptotic** (F-D2's
per-element `LoadScope` is O(N²) → live frames make it O(N), so speedup grows with collection size and has
no single number — "remove 75% ⇒ 4×" was the wrong frame), and the **budget cliff is orthogonal to the
engine** (op count measures the IR, so *no rung moves it* — §12.7; scale past it is sharding, not
compiling).

### §12.4 The compiled artifact is itself an entity — and that is where keystone/assembly plugs in

A rung-3 compiled artifact is not loose native code — it is a **content-addressed entity in the tree**,
keyed by its inputs:

```
compiled/{target}/{ir_hash}   →   { target, ir_hash, artifact_bytes, oracle_vectors_hash }
```

`target` is the arch/backend (`wasm`, `x86-64`, `aarch64`, `rv64`, a keystone assembly profile);
`ir_hash` is the content hash of the source step. This makes the compilation a **pure function
materialized as an entity**, with three payoffs that fall straight out of the substrate:

- **Caching & distribution are free** — a compiled artifact is fetched, deduped, and pinned like any
  entity; a peer that already holds `compiled/wasm/{h}` reuses it. The content store *is* the artifact
  cache.
- **Provenance is intrinsic** — the artifact's key *claims* "I am the compilation of `ir_hash` for
  `target`." That is verifiable identity (does the key's `ir_hash` match the IR you hold?), separate from
  correctness (§12.5).
- **This is the keystone assembly path, viewed from compute.** The operator's *assembly-language
  translator in the tree* is exactly a rung-3b compiler whose `target` is an assembly profile: it reads an
  IR entity and materializes a `compiled/{asm-target}/{ir_hash}` entity. Compute→native and
  keystone's tree-resident assembly translation are **the same operation at two `target`s** — which means
  the keystone assembly work is not a parallel effort but *the bottom rung of this ladder*, and a native
  compiled *interpreter* (Axis 1 at an assembly target) is the fastest general reader a peer can hold.

### §12.5 Trust & validation — the oracle is the contract, not the artifact

Findings §5 asked: *what does a peer install, and what does it trust?* The ecosystem already has the
answer in its bones — **conformance is the contract, not the version number** (ADR-0012) — and it applies
verbatim: **a peer trusts a compiled artifact exactly as far as it passes the oracle, and no farther.**
Four trust postures, strongest first:

1. **Re-derive locally** — compile the IR with your own compiler; trust only your own toolchain. Always
   available (you hold the IR). The default for rungs 1–2 (nothing is shipped).
2. **Accept + verify against the oracle** — take a shipped `compiled/*` artifact, run the conformance
   vectors through it, check the boundary-hash sequence equals the Stage-1 interpreter's. This is the
   §9.1 equivalence criterion and the POC's D3 test, promoted to an **admission gate**: a compiled handler
   is admissible iff it is boundary-equivalent to the reference interpreter on the vectors. Trust is in the
   *oracle run*, not the shipper.
3. **Accept on provenance** — verify the artifact's `ir_hash` key matches your IR (content-addressing
   gives this for free), and trust a signature for correctness. Weaker; provenance ≠ correctness.
4. **Accept on reputation** — trust the shipper outright. Weakest; for closed deployments only.

The load-bearing design point: **the oracle is per-IR and already exists** — the same conformance vectors
that gate the interpreter gate every compiled rung, so "is this fast handler correct?" is never a new kind
of question, only the existing one run against a new engine. And because rung 1 is always the fallback, a
peer that *distrusts* every artifact simply interprets the IR — degraded speed, identical results. This is
what keeps the system transferable under adversarial or air-gapped conditions: **worst case, you always
have the IR and your own interpreter.**

### §12.6 WASM: substrate vs. target (where the "peers in WebAssembly" feedback lands)

The substrate work — *building the peers themselves in WebAssembly* — is **mostly orthogonal** to this
ladder, and it is worth stating why so the two are not conflated:

- **WASM-as-substrate** (the peer runtime *is* a WASM module) is about **where the engine runs** — it
  hosts rung 1–2 like any other host. It does not change what the IR is or how compilation is validated.
  A peer-in-WASM still ships/receives IR and still gates compiled artifacts on the oracle.
- **WASM-as-target** (rung 3a: compile a *program* — or the *interpreter itself* — **to** WASM) is a rung
  on this ladder, and a **special** one: it is the only rung that is **transferable *and* fast at once**
  (§9.1 Form B — arch-independent bytecode, opaque, near-native). That makes it the natural artifact to
  **ship to a peer that lacks a local compiler**, and the bridge between Axis-1-local-compile and
  Form-B-ship-bytecode.

Where the substrate feedback *does* become load-bearing is the operator's own caveat: **once we compile
down to new architectures from the tree-resident assembly translator** (rung 3b), a peer-in-WASM is one
more `target` in the `compiled/{target}/{ir_hash}` space, and the same peer might hold *both* a WASM
artifact (portable, to hand onward) and a native one (fast, for itself). So the substrate work is not a
fork in the model — it slots in as (a) another host for the engine and (b) another `target` value, the
transferable one. Nothing here fundamentally alters; it fills two cells of a table the ladder already
drew.

### §12.7 Open sub-problems (honest — what §12 does *not* settle)

- **✅ RESOLVED — the compiled-interpreter form.** "Decode once into an in-memory form" was the open
  design; §13 pinned it (resolved-node graph + lexical addressing + live frames) and the Axis-1 prototype
  **built and measured it** (34–163×/tick; §13 header, Axis-1 absorption). A bytecode/threaded-code form was
  the candidate next increment — and the whole-tick profile says **don't build it yet** (residual dispatch
  is ~32% of a tick, a perfect bytecode buys ~1.5×). Settled for now.
- **Cross-runtime artifact portability at rung 3.** A `compiled/wasm/*` entity is portable; a
  `compiled/x86-64/*` is not. The descriptor/admission story for "I have the IR and a WASM artifact but
  not a native one for your arch" needs pinning (fall back to WASM? to interpret? negotiate?). *(Lower
  priority now that rung 3 is evidence-deprioritized — §12.3.)*
- **Budget under compilation — with a new data point.** The 100k-op cliff is orthogonal to the engine (op
  count measures the IR; no rung moves it — §12.3), so "compile past the cliff" is closed and **sharding**
  is the answer (proposal §4). The residual open question is *metering cost*: Axis-1 showed self-metering is
  **cheap at Stage-1 and progressively expensive up the ladder** (a decrement+branch is noise against a
  `cbor.Unmarshal`, but a real tax against a ~ns node visit — the same shape as §9.1's interior-hashing
  argument). So "self-meter vs. trust-to-terminate" **diverges by rung**, not settled once. (GUIDE-CORE §7
  negotiable bounds.)
- **Budget escape & fairness (operator question, Axis-1 §4c).** A `compute/apply` sub-dispatch gets a
  **fresh** budget (the boundary is the handler invocation, not the expression tree — verified), so
  in-compute fan-out is possible and **terminating** (TTL fuses recursion depth; budget bounds each step;
  an anti-runaway guard prevents self-reference). What is *not* guaranteed is **fairness**: the dispatch is
  synchronous, so a large-but-bounded fan-out could occupy the serve loop and starve concurrent reactive/
  emission work. That is a runtime scheduling property to name in the tick contract, and it is the gating
  concern for an in-compute (self-dispatching) sharded step — the array-concat proposal must carry it.
- **Impure-edge capabilities at rung 3a.** §9.1 noted WASM's impure edges become explicit imports — the
  capability model resurfaced. The `imports` admission check (proposal §5) is the entity-native form; the
  mapping to WASM imports for a compiled program is a design point.
- **Who compiles.** Is compilation a peer-local step (each peer compiles what it hosts), a service (a
  compile-farm peer materializes `compiled/*` entities others fetch), or both? Content-addressing makes
  either work; the ecosystem posture is a decision, not a mechanism.

### §12.8 How close are we to Doom?

Synthesizing the POC numbers, the budget cliff, and the ladder — an honest reading:

**Not on rung 1, and not by a little.** The interpreter's hard 100k-op budget caps Life at 24×24 (POC
§2). Doom is a 320×200 framebuffer (64k cells) plus BSP traversal, mobj AI, and collision at 35 Hz — the
sim step is *orders* beyond an interpreted eval, and the cliff is a wall, not a slowdown. Rung 1 runs
Life and Snake comfortably (§11) and that is its ceiling.

**The ladder is exactly shaped for it, though, and the pieces are individually in view:**
- **Budget** — sharding (proposal §4) already splits a too-big step into per-strip evals today; the
  compiled rungs drop per-op budgeting altogether.
- **The 75% is the target** — Axis-1 compiled interpretation removes the dominant cost for *every* Doom
  sub-step at once, no per-function work. That alone may not reach 35 Hz at Doom scale, but it is the
  first and cheapest multiple.
- **The cut is adjustable** (§9) — Doom does not need to be *one* compiled handler. The hot interior (BSP
  sight, mobj think) compiles (rung 3) while the rest stays interpreted, and the framebuffer materializes
  as a `display` snapshot port (§8, proposal §7) — the boundary that is already nearly free.
- **The correctness gate scales unchanged** — every rung is the same boundary-hash oracle, so a compiled
  Doom step is *validated the same way* Life's two lowerings already are.

**UPDATE (Axis-1 landed) — the throughput frontier is gone; the frontier is now heterogeneity.** Rung 2 is
**built and measured**, not designed. With Axis-1 + sharding, Life reaches **64×64 at ~658 gens/s** (and no
ceiling yet in sight) — so the grid sizes this section treated as the Doom-scale frontier are **no longer a
constraint worth designing around.** That retires the part of the Doom question that was about *raw uniform
throughput*. What it does **not** retire is what actually makes Doom Doom, and this is the honest reframing:

- **Doom is not a uniform `map`.** Life's win is asymptotic *because* Life is one big map with a capturing
  closure — the exact shape F-D2 punished and live frames fixed. Doom's cost is **heterogeneous**: BSP
  traversal (value-recursion, §7.2), mobj `think` over a *variable* actor set (not a fixed grid), collision
  against the blockmap. Those are not one big map, so they inherit Axis-1's constant-factor win (~20–50×,
  the floor of the range) but **not** automatically its asymptotic one. Whether they *shard* like Life does
  (independent, read-only, stitchable) is the open empirical question — some will (blockmap queries), some
  are inherently sequential (a mobj that reads another mobj it just moved).
- **The cut stays adjustable** (§9): the framebuffer is a near-free `display` snapshot port; the hot
  interior can compile *if* profiling a real Doom step ever says the evaluator is the bottleneck again —
  but Axis-1's lesson is to **measure the whole tick before assuming it is.**

**So: Doom is a rung-2 + sharding program for its regular parts, and its frontier is now the irregular
parts — heterogeneous actors and spatial recursion — not throughput.**

**UPDATE (Asteroids probe, 2026-07-16 — the heterogeneity questions came back answered).** The
heterogeneous-actor probe ran, and the variable/heterogeneous/interacting shape held up:
- **It shards** — boundary-hash-identical at k=2,5,8 in parallel, with cross-actor reads and a
  length-changing set. The heterogeneous shape does *not* forfeit the sharding scaling path.
- **No new primitive for correctness** — `if` on a `kind` field covers heterogeneous think; a **fixed-cap
  actor array + a free-slot tag** (Doom's own mobj model) carries spawn/split/death with no array-concat.
- **The Doom *renderer* reframed — and per-pixel-compute rendering is ruled out, not Doom.** The probe
  priced the two derived rendering ports and found the decisive shape: a **display list** is **O(actors),
  resolution-independent** (~13k ops for Doom's ~50 mobjs — fits trivially), while a framebuffer *computed
  in compute* is **O(pixels × actors)** because a pure `(state) → frame` cannot scatter and must gather at
  every pixel (~92M ops/frame at Doom scale). So the answer is the exploration's own §8/§9 split: **compute
  emits a display list; a native renderer rasterizes.** "Doom is out of reach" is only true of the
  formulation nobody should use (per-pixel-compute rendering). The real open Doom question is the
  **display-list *vocabulary*** for a BSP/span renderer (spans? walls? sectors?) — a design conversation,
  not a budget wall. *(And the 100k budget is a per-eval default, negotiable per spec and splittable across
  parallel evals — never treat it as a hard ceiling; reformulate the graph and shard.)*

**The sharpest invariant the probe found — cost and parallelism are the same property.** A gather is
O(product) *because* every output cell is independent — which is exactly *why* it shards (the whole scaling
result). A scatter (via a `fold`, or a hypothetical `assoc` indexed update) is cheaper-looking but
**sequential by construction** (step i+1 eats step i's accumulator). So **you cannot have cheap scatter and
free sharding from one structure** — the expensive-but-parallel gather and the cheap-but-sequential scatter
are two faces of one coin. This reframes the collection-primitive question (`PROPOSAL-COMPUTE-COLLECTION-
PRIMITIVES`): `group_by`/`range` are pure wins that keep the parallel structure, while an indexed update
(`assoc`) is a real capability that **moves a program off the shard path** — added as a deliberate author
choice, never an implicit lowering.

The **next rung is now consolidation, not another probe**: three shipped programs (Snake/Life/Asteroids)
make the **descriptor + a generic host** *falsifiable* — if the descriptor is right, all three mount through
one host with no per-program code, and the same descriptor + IR transfer to a *different-runtime* host
(browser/Rust) that supplies its own drivers. That is where "transferable compute" stops being an assertion
(see the generic-host design, `EXPLORATION-GENERIC-HOST-AND-DESCRIPTOR.md`). Doom stays the north star — it
forces the actor-set and spatial-query shapes the uniform-grid probes structurally can't — but the path to
it now runs *through* a landed descriptor, not another game.

## §13 The compiled-interpreter form — a concrete Axis-1 design (so nobody guesses)

§12 said *build Axis 1 first*; this section says **what it is, concretely enough to implement without
guessing**, and — the operator's requirement — **how the one design works across languages and
architectures.** It is the design the Axis-1 prototype handoff carries. Grounded in the Stage-1 reference
(`entity-core-go/ext/compute`, read at the current release): the node tags are entity types
(`arithmetic`, `compare`, `logic`, `construct`, `field`, `lookup/*`, `let`, `lambda`, `apply`, `if`,
`index`, `literal`, `builtins/map|filter|fold`); scope is a `map[string]interface{}` captured to a
content-addressed `compute/scope` entity (`CaptureScope`) and reloaded by hash (`LoadScope`).

### §13.1 One-line contract (and the anti-scope)

> **Same IR in; bit-identical materialized boundary entities out; the interior representation is free.**

- **It IS:** decode the expression graph **once** into an in-memory evaluable form, then evaluate that
  form every tick — deleting the two measured costs (§12.0): per-node CBOR decode and per-element scope
  reload. A *faster general engine*, not a program-specific compiler.
- **It is NOT:** a JIT or native codegen (rung 3b), a *portable* bytecode format (a rung-3a fork,
  deliberately deferred — §13.6), a new IR, or any wire/`EXTENSION-COMPUTE` change. The source IR is
  untouched and remains the transferable artifact; this is a peer-local execution strategy.

### §13.2 The two costs, and the one technique each (the whole design in two lines)

| Measured cost (POC §2) | Stage-1 cause (reference) | Axis-1 technique |
|---|---|---|
| ~54% per-node `cbor.Unmarshal` | every node re-decoded from CBOR each eval | **decode-once** → resolved in-memory node graph, evaluated many times |
| LoadScope (67% in the F-D2 case) | closure env captured to a `compute/scope` entity, re-fetched + re-decoded **per element** | **live frames** → the interior environment is an in-memory slot array, never round-tripped through `CaptureScope`/`LoadScope` inside a pure loop |

Everything else in §13 is making these two precise and pinning what must *not* change.

### §13.3 The decoded form (recommended representation)

Recommend a **resolved node graph** (decode-once AST), not yet a linear bytecode:

- Decode each expression entity once into a typed in-memory node (`Add{lhs,rhs}`, `If{cond,then,else}`,
  `Construct{type, fields}`, `Index{coll,i}`, `Lit{value}`, `Lambda{params, body}`, `Map{coll, fn}`, …).
  Child references are **resolved pointers**, not hashes to re-fetch. Literals are pre-parsed to native
  values once.
- Evaluate by walking the resolved graph over a **frame stack of value slots** (§13.4). No CBOR touched on
  the hot path; boundary encode/decode only where a value crosses into/out of the entity system (§13.5).
- **Why not bytecode first:** a resolved-node walker removes the dominant ~54% + the LoadScope cost with
  far less surface than a stack VM, and keeps a 1:1 correspondence to the IR that makes the equivalence
  oracle (§13.8) trivial to reason about. A linear bytecode/threaded-code form is a *later* increment on
  the same rung (kills the remaining dispatch overhead) — note it, don't build it now.

### §13.4 Lexical addressing — the LoadScope kill (the load-bearing part)

This is the piece most likely to be got wrong if left to guess, so it is pinned:

- **Resolve variable references at decode time.** Walk the graph tracking lexical scope; rewrite each
  `lookup/scope("x")` to a **slot coordinate** `(depth, index)` into the frame stack — an array access, not
  a string-keyed map lookup and *never* an entity reload. `let` bindings and `lambda`/closure params define
  slots; `map`/`filter`/`fold` element params are slots in the loop frame.
- **The environment is a live frame, not a content-addressed entity, for all *interior* evaluation.** The
  Stage-1 round-trip (`CaptureScope` → hash → `LoadScope`) is exactly the §9.1 "interior hashing is an
  interpretation artifact" thesis applied to *scope*: a captured `compute/scope` entity is only
  semantically required when a scope/closure value **crosses the boundary** (is stored or returned as a
  materialized entity). For a pure `map`/`fold` over interior data it never does — so the compiled form
  invokes the **once-decoded** closure body per element against a **fresh element slot in a shared frame**,
  with the captured environment sitting in parent slots. That is the direct, general fix for F-D2 (the
  O(N²) per-element reload); the point-fix routed to core-go is its toe, this is the whole foot.

### §13.5 The boundary contract — what MUST stay bit-identical (the gate)

The interior is free; the boundary is not. A conforming Axis-1 engine MUST preserve, unchanged from the
Stage-1 reference:

- **Materialized boundary entities** — every entity that crosses into the entity system (a stored
  `construct`, an `apply` arg on a hash-typed field, the written `state'`, a scope/closure that is actually
  materialized) is **canonical-CBOR bit-identical and same-content-hash** as Stage-1. This is the *only*
  equivalence that is checked (§9.1), and it is the whole conformance surface.
- **The impure frontier** — `lookup/tree` stays an impure boundary host-call; `lookup/scope`/`lookup/hash`
  are pure. Reactivity depends on this frontier being exactly where Stage-1 draws it (§6).
- **Evaluation semantics** — error-as-value propagation (`compute/error` poisons its consumers), **lazy
  `if`** vs **eager `let`**, canonical map-key ordering, integer/fixed-point determinism (**no float
  introduced** — the cohort float16 hazard), and the S1 builder's `let`-name-sort visibility (F-E-series).
  The resolved form inherits these; it must not "optimize" a lazy branch into eager evaluation or reorder
  effectful boundary calls.
- **Budget** — the op-cost accounting (`EXTENSION-COMPUTE §9.3`, `DefaultMaxOps`) must count equivalently
  *or* the engine must be explicitly declared to change the metering contract (an open point, §12.7). For
  the prototype: keep counting the same logical ops so the budget cliff moves for a *known* reason
  (fewer artifacts), not silently.

### §13.6 Cross-language & cross-architecture — how *one* design works everywhere

This is the operator's explicit requirement, and the resolution is the §12.1 move made operational:

- **The form is peer-local and per-language — and that is correct, not a compromise.** Go builds Go
  structs, Rust an `enum` tree, Python objects, a native target its own layout. **The decoded form is never
  transferred**; only the IR is. So "works across N languages" does **not** mean "one shared data
  structure" — it means **each language builds its own resolved form and all are validated against one
  oracle** (§13.8). The cross-language contract is **semantic + boundary-equivalence, not a byte layout** —
  the same way go/rust/py peers interoperate today through the spec + conformance suite, not shared code.
- **Architecture is abstracted by the host language at this rung.** A resolved-node walker is ordinary
  code; endianness/word-size never surface because all determinism-bearing values are canonical-CBOR at the
  boundary and integer/fixed-point in the interior. Architecture only becomes first-class at **rung 3b**
  (native/assembly codegen — the keystone `target`), which is *out of scope here* and handled by the
  `compiled/{target}/{ir_hash}` model (§12.4).
- **The one fork to name, not decide:** a *portable* decoded form (a canonical entity-compute **bytecode**
  materialized as `compiled/entity-compute-bc/{ir_hash}`) would make the fast form itself transferable
  (rung 3a). It is attractive but it is a **separate, larger commitment** (a second normative artifact with
  its own conformance surface). The Axis-1 prototype stays **peer-local per-language**; whether a portable
  bytecode is worth standardizing is a follow-on the prototype's numbers should inform. **UPDATE (Axis-1):**
  the *speed* motivation for any bytecode form is now closed — the evaluator isn't the bottleneck, so a
  bytecode buys ~1.5×/tick (§12.7). A *portable* bytecode remains a live but separate question decided on
  **transferability** grounds (ship one fast artifact to a peer that can't compile locally), not speed — so
  it rides on the rung-3a demand, which is itself deprioritized (§12.3). Not now.

### §13.7 Caching & lifecycle

- **Key the decoded form by the IR content hash** (`step` is a fixed entity across ticks): decode once per
  unique step hash, reuse across every tick and every program instance that shares the hash. The content
  store already dedups the IR; the decoded-form cache is its in-memory shadow.
- Invalidation is trivial: a different step is a different hash is a different cache entry. No mutation, no
  staleness.

### §13.8 The validation harness (the admission gate, concretely)

This is §12.5 posture 2 turned into runnable tests — and the prototype's *primary deliverable* beside the
engine, because it is what lets every future rung be trusted:

1. **Interpreter-equivalence sweep.** Run vectors plus the Life/Snake probe steps through *both* the
   Stage-1 interpreter and the Axis-1 engine; assert **identical materialized-boundary-hash sequences**.
   Any divergence is an Axis-1 bug by construction (Stage-1 is the reference). This is the reusable
   definition of "a compiled engine is conformant." **CORRECTION (Axis-1 §1 / gaps doc):** the handoff said
   "the *existing* compute conformance vectors" — **there are none.** No portable compute corpus exists in
   the cohort (each impl has private in-package tests; zero shared bytes). Workbench substituted a
   **differential/property sweep** — 300 generated well-typed graphs through both engines — which is
   *stronger* than a fixed corpus at catching reimplementation drift and cannot go vacuous, but
   structurally **cannot pin cross-impl (go-vs-rust) agreement**. That gap is the corpus dependency (§13.9).
2. **Reuse D3 as a cross-check.** `TestExpLifeD3_LoweringsConverge` (two lowerings, identical boundary
   hashes) run under the Axis-1 engine must still hold — and cross-engine (Stage-1 arith vs. Axis-1 table,
   all four pairings) must agree. D3 is the track's compiler oracle (§12.5); this widens it to engines.
3. **Re-run the cost sweep** (POC §2) on the Axis-1 engine: Life throughput + CPU profile + budget cliff.
   The deliverable number is **how much of the ~75% one rung removes**, and whether the budget cliff moves
   (fewer materialized artifacts per eval).

### §13.9 Where it might live in the spec (placement — flagged, not decided)

The operator's open question ("extension? amendment? appendix?") resolves cleanly along the repo's
pin-the-observable-surface doctrine, and it splits in two:

- **The decoded form is an internal → it needs NO spec.** Per-language, peer-local, "leave internals to
  converge." Standardizing a Go struct layout would be exactly the over-pinning the doctrine warns against.
- **The equivalence/admission contract is cross-impl-observable → it is the candidate spec surface.** "An
  alternate execution engine is conformant iff it is materialized-boundary-equivalent to the reference on
  the conformance vectors" is a real normative statement other impls must agree on. Its natural home is an
  **appendix or amendment to `EXTENSION-COMPUTE`** defining *execution-strategy conformance* (the engine is
  swappable; the boundary oracle is the contract) — **not** a new extension, because it adds no new
  operation or wire surface. Decide the exact form at fold, once the prototype confirms the oracle is
  sufficient. *(A future portable bytecode — §13.6 — would be the one thing that DID need its own normative
  artifact; that is the reason to keep it a separate decision.)*
- **⚠ GATING DEPENDENCY (Axis-1 §1 / gaps doc §1) — the contract quantifies over a corpus that does not
  exist.** "…materialized-boundary-equivalent on the conformance vectors" is unwritable until compute *has*
  vectors as a shared artifact. Compute **has** been validated (the go validate-peer runs compute, including
  transferable compute); what is missing is a **standardized, portable, cross-impl corpus** go/rust/py can
  each load — so nothing today would catch go and rust *disagreeing* about compute evaluation. Authoring it
  is arch's (the ECF pipeline exists and works — arch authors canonical bytes, byte-locked 3-way), and its
  vendoring home is the **extensions-generation repo** (a keystone-analog for entity-systems extensions),
  **not keystone** (which is core protocol). Per the operator's steer this is "we'll get there as we go,"
  but the admission contract cannot be *ratified* ahead of it — record the dependency, don't block on it.

### §13.10 Prototype scope boundary (in / out — the fence for the handoff)

- **In:** the resolved-node decode-once engine; lexical addressing + live frames (§13.4); the §13.8
  equivalence harness; the Life budget/throughput re-run. Experiment-tier, beside Exp-D/E, reusing their
  oracle.
- **Out:** JIT / native / assembly codegen (rung 3b); a portable bytecode (§13.6 fork); per-program
  compilation (Axis 2); any `EXTENSION-COMPUTE`/wire/IR change; any spec edit (the §13.9 placement is
  arch's to fold *after* the numbers land).

---

## Sources

- Varvara / Uxn devices: [XXIIVV — Varvara](https://wiki.xxiivv.com/site/varvara.html),
  [uxn-impl-guide — devices](https://github.com/DeltaF1/uxn-impl-guide/blob/main/devices.md),
  [Fantasy Console Wiki — Varvara](https://fantasyconsoles.org/wiki/Varvara)
- WASM-4: [Memory Layout](https://wasm4.org/docs/reference/memory/),
  [WASM-4 intro](https://wasm4.org/docs/)
- WASI streams / the capability-tier split: [wasi-io streams.wit](https://github.com/WebAssembly/wasi-io/blob/main/wit/streams.wit),
  [Thinking about streams in WASI (sunfishcode)](https://blog.sunfishcode.online/preview3-streams/),
  [WASI 0.3 launched](https://bytecodealliance.org/articles/WASI-0.3) — note: WASI standardizes
  streams/sockets/files/http but **not** graphics or audio (the console-tier boundary).
