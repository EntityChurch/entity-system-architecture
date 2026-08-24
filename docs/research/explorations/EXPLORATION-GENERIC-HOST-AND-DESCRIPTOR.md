# EXPLORATION — the generic host & the program descriptor (making compute programs transferable)

**Status:** Design exploration — 2026-07-16. Feeds `PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM.md` (the
descriptor + runtime contract this makes concrete). NOT ratified.
**Motivation:** the Asteroids probe (`ABSORPTION-compute-asteroids-actor-probe.md`) shipped a third real
program and exposed the gap that matters most: *the programs are **in** the tree but not **addressable as**
programs.* Everything Asteroids is — `state`, `input`, `step`, `display` — lives in the tree, but the
expressions are assembled by a Go function at startup and the host hard-codes the clock, so **nobody can
pick Asteroids up and wire it to their own front-end.** This doc designs the thing that closes that gap: a
**generic host** that mounts *any* program from a **descriptor**, with zero per-program code — and, the
operator's target, does so *across runtimes* (a browser/Rust host running the same programs).

---

## §0 TL;DR

- **The generic host is a `mount(descriptor) → running program` loop with no program-specific code.** It
  reads a descriptor, seeds input ports, binds every port to a driver **by `(role, shape)`**, clocks the
  tick, and owns run-state. That is the entire contract.
- **The crux is the `(role, shape)` vocabulary** — the *driver ABI*. `role` says "display"; `shape` says
  *how* ("display-list" vs "framebuffer"), which is what actually selects a driver. Asteroids forced this:
  it has **two output ports of different shapes** (state + display-list), which Life/Snake hid because a
  grid's state *is* its display. The vocabulary is **grounded in the lineage of computer I/O** (text →
  raster → vector → audio; keyboard → pointer → events), *not* in games — which makes **`text` a
  first-class shape** (the oldest interface, and the one the game probes never needed) and makes the set
  **extensible by a rule, not a closed enum** (§4/§6): new interface lineages (touch, VR, …) join the same
  way.
- **Transferability is now two ABIs, one per level.** The **compute IR** is the ABI for *evaluation* (each
  runtime brings its engine — §12/§13); the descriptor's **`(role, shape)` vocabulary** is the ABI for
  *I/O* (each host brings its drivers). A program = `(descriptor + step IR + initial_state)`, all
  content-addressed — transfer those three by hash and any conformant host on any runtime runs it.
- **The falsification is built-in:** three shipped programs (Snake, Life, Asteroids) must mount through
  *one* Go host with no per-program Go; then a **browser/Rust host** pulls the same three by hash and runs
  them identically. If one doesn't fit, the descriptor is missing a field — a useful negative.
- **Two dependencies surface, both already known:** cross-runtime *evaluation* trust rides on the
  **compute corpus** (§13.9 — a Rust engine must be boundary-equivalent to Go's), and the **shape
  vocabulary** becomes a small new normative surface (which shapes exist, what each carries).

## §1 The gap, precisely

A compute program today is transferable *data* but not a transferable *program*. The distinction:

| has | Asteroids today |
|---|---|
| step expression in the tree (content-addressed IR) | ⚠️ **reconstructed at boot**, not a durable artifact (see §1a) |
| state + input + output entities in the tree | ✅ |
| **a manifest that says which paths are ports, of what shape, and how to clock it** | ❌ — a Go function knows all this; the tree doesn't |

So the program is *in* the tree without being *addressable as* a program. The descriptor is the manifest
that closes it, and the generic host is the proof it's sufficient.

### §1a Authoring is not mounting (the phase-1 deliverable the handoff under-named) — workbench-go §3

The descriptor assumes `step` is *already a hash in the tree*. It isn't, durably: `buildAsteroidsStepExpr`
is ~458 lines of Go that assemble the IR **at boot** (`workbench/program_asteroids.go:607-1064`), and
`NewAsteroidsGameModel` **fuses authoring with running.** So the real phase-1 task is not "write three
descriptor files" — it is the **split**: the step IR must become a **durable content-addressed artifact,
authored once**, rather than reconstructed every boot. **Phase 2 depends on this completely** — a Rust host
fetching `(descriptor + step IR + initial_state)` by hash cannot run `buildAsteroidsStepExpr`; if the
builders stay boot code, "fetch by hash" has nothing to fetch and transferable compute stays an assertion.
This is the load-bearing deliverable; a phase-1 report that shipped descriptors while leaving the builders
at boot would look green and prove nothing. (Accepted as a named phase-1 deliverable, §7.)

## §2 The mount contract — what a generic host does (and does not) do

```
mount(descriptor_path):
  d ← read descriptor entity at descriptor_path
  # 1. seed — F-E1: an unseeded input port read is a compute/error
  for p in d.input_ports:  write p.initial (or the type-default for p.type_ref) at p.path
  # 2. bind — the ONLY place drivers enter; bind by (role, shape), never by program identity
  for p in d.output_ports: bind_output_driver(p.role, p.shape, p.scene) ← reads p.path
  for p in d.input_ports:  bind_input_driver(p.role, p.shape)           → writes p.path
  # 3. clock — host owns time (§5 of the exploration)
  loop at d.tick.rate_hint:
     if d.tick.mode == clock-driven:
        eval(d.step) → put d.state_path                          # plain eval — NO sharding here (see below)
        for p in d.output_ports: materialize/refresh p            # projections
  # run-state (start/stop/restart) is the host's surface, outside the program
```

**What the host must NOT need:** any knowledge of what an asteroid is, what the step computes, or how many
ports a specific program has. It loops over the descriptor's declarations. That "no per-program code" is the
falsifiable claim (§7): the same binary mounts Snake, Life, and Asteroids.

**CORRECTION (2026-07-17 — the base contract does NOT shard).** An earlier draft had this loop *shard by
op_cost* when `N·op_cost > max_ops`. **A generic host cannot do that** (workbench-go, verified against
source): a shard is a **distinct baked expression** (the index set is a `c.Literal(...)` array, the size is
baked as `c.Literal(W/H)` throughout), so `N` is an **authoring-time constant, not a runtime quantity**, and
a harness has no `N` to derive `k` from and no licence to rewrite a `map` inside an opaque IR (that is a
compiler pass — this is a harness, §9). So:
- **The base mount contract is plain `eval → put`.** No sharding. The three product programs (Snake 8×8,
  Life 16×16, Asteroids ~24 slots) are all under budget, so none needs it — sharding is **off the phase-1
  critical path**.
- **Sharding is an opt-in *program contract*, not a host power** — a shardable step declares a **shard-range
  input port** and maps over `compute/range(i1−i0)`; the host, knowing `N` (state array length, or a
  descriptor-named field), writes k ranges and drives k evals + stitch. One step hash, no IR rewriting,
  rides the existing port mechanism. **It requires `compute/range`**, now reclassified **load-bearing**
  (`PROPOSAL-COMPUTE-COLLECTION-PRIMITIVES` §2). It lands as an extension when `range` does — see the
  app-convention proposal §4.

## §3 The descriptor, evolved (what Asteroids forced in)

The `PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM` descriptor already has `state_path`/`initial_state`/`step`/
`input_ports`/`output_ports`/`tick`/`imports`. Asteroids forces four concretions:

```
app/program/interface := {
  state_path, initial_state, step,                 ; the program core (unchanged)
  input_ports:  [app/program/port],
  output_ports: [app/program/port],                ; MULTIPLE, of DIFFERENT shapes  ← Asteroids
  tick: { mode, rate_hint, op_cost },              ; rate_hint is IN the descriptor  ← was host-hard-coded
  imports: [app/program/import]                    ; capability handlers (unchanged)
}

app/program/port := {
  name, path, type_ref, kind,                      ; kind = snapshot | stream (unchanged)
  role:  "display" | "input" | "state" | "audio",  ; the coarse binding hint
  shape: <driver-binding discriminant>,            ; NEW — the actual driver selector (§4)
  scene: { … },                                    ; NEW — per-shape metadata the driver can't infer (e.g. wrap:true)
  initial: <hash>                                  ; F-E1 seed (input ports)
}
```

1. **`output_ports` is genuinely plural and heterogeneous.** Asteroids emits `state` (a snapshot the host
   can persist/replay) **and** a `display-list` (what the renderer draws). Life/Snake made "one output" look
   settled only because for a grid the state *is* the display.
2. **`shape` is the new load-bearing field** (§4) — `role:"display"` is not enough to pick a driver.
3. **`scene`** — a display-list carries properties vertices can't express: Asteroids needs `wrap:true` so the
   renderer tiles an actor's outline at the world seam even though its centre wrapped.
4. **`rate_hint` moves into the descriptor** — the host currently hard-codes 6/8/… ticks/s; a transferable
   program must carry its own clock rate.

## §4 The `(role, shape)` vocabulary — the driver ABI (the crux of portability)

`role` is the coarse hint; **`shape` is the contract a driver implements.** A host binds a port by looking
up its `shape` in a **standardized vocabulary** and invoking the matching driver.

**Ground the vocabulary in the history of computer I/O, not in games** — this is the operator's steer, and
it matters: a game-derived list ("display-list, key-set, …") looks ad-hoc and risks missing the fundamental
interfaces. The lineage of human↔computer I/O is short and well-worn, and it *is* the shape vocabulary:

| lineage (output) | the classic device | our `shape` | note |
|---|---|---|---|
| **text / character** | teletype → glass TTY → VT100 | **`text`** | a character stream *or* a character grid — the **most fundamental** interface; a shell, a REPL, a log, a text UI. **This was the gap.** |
| **raster / bitmap** | framebuffer display | `framebuffer` | pixels; blit; O(pixels) |
| **vector / structured** | vector display, sprite/scene engines | `display-list` | line/poly drawables + kind tags; O(actors), resolution-independent. *(Asteroids ran on a literal **vector display** — the historical fit is exact.)* |
| **audio** | DAC / sound card | `audio-stream` | PCM chunk per tick (stream) |
| **(program-native)** | — | `raw-state` | the state entity itself; renderer knows the type. **Declared escape hatch — the falsification does NOT run through it** (§4a) |

**§4a — `raw-state` is an escape hatch, and the falsification must not use it (2026-07-17, workbench-go).**
A `raw-state` driver is "program-aware," i.e. **it *is* per-program code** — it just moved out of the model
and into the driver. If Life and Snake bind `raw-state`, the phase-1 criterion "zero per-program *Go*"
passes *trivially* while every per-program fact sits in the C# panel, and the three-program falsification
collapses to a one-program (Asteroids) test. So: **`raw-state` stays in the vocabulary as a declared escape
hatch, but the three probe programs must bind *blind, generic* shapes.** Concretely: **Life and Snake bind
`text`** — a Life grid *is* a character grid (alive=`#`, dead=` `), Snake likewise. This is exactly the
I/O-lineage argument applied consistently (Life was a pencil grid; Snake ran on terminals — `raw-state` was
the game-shaped answer), and it *fixes a second hole for free*: it gives the `text` driver two real programs,
so the "text driver works" criterion becomes checkable instead of compile-only.

| lineage (input) | the classic device | our `shape` | note |
|---|---|---|---|
| **character / key** | keyboard | **`text-in`** (stream) / **`key-set`** (held snapshot) | typed characters are a stream; *held* keys are a snapshot bitmask — two shapes, one device |
| **pointing** | mouse / touch / stylus | `pointer` | a coordinate (+ button) snapshot |
| **selection / discrete** | button, gamepad, menu pick | `event-stream` (discrete) / `direction` (enum snapshot) | an ordered event vs. a latched choice |

**Why this framing is better than the game list:** it says *what class of interface each shape is*, so the
vocabulary is visibly **the standard I/O surfaces of a computer**, not a pile of game hooks — and it makes
the gaps obvious (we had graphics and had no **text**, the oldest interface of all). It also keeps the door
open honestly: new interface *lineages* (touch gestures, VR/spatial, haptics, camera-in) become new shapes
by the same rule — *name the device class, pin what the port carries, define what a driver does* (§6).

**Per-shape `scene` / metadata (the driver-can't-infer fields), pinned for the first set:**

| shape | port carries (`type_ref`) | `scene` fields (metadata a driver needs but can't derive) |
|---|---|---|
| `text` | char grid `{cols, rows, cells}` or a char stream | `mode: grid\|stream`, `palette?` (attr colors), `cursor?` |
| `framebuffer` | `{w, h, format, pixels}` | `format` (pinned pixel-format set) |
| `display-list` | `[{verts, kind}]` | `wrap: bool` (seam-tile), `bounds` (world extent → viewport), `z_order?` |
| `audio-stream` | PCM chunk `{sample_rate, samples}` | `channels`, `sample_rate` (must match the mixer clock) |
| `key-set` | bitmask snapshot | `keymap` (bit → semantic action) |
| `pointer` | `{x, y, buttons}` | `space: screen\|world` |

**Why this is the portability crux.** The vocabulary is **shared**: every host — Go, Rust, browser —
implements the *same* shape drivers (a `text` terminal, a `display-list` renderer, a `key-set` source, …).
A program declares only *which shapes it uses*; it never ships a driver. So the same program mounts anywhere
the shapes it needs are implemented. **This is the driver-level analog of "the IR is the ABI"** (§12.1):
there, each runtime brings its own compute engine for one shared IR; here, each host brings its own drivers
for one shared shape vocabulary.

## §5 Cross-runtime transfer — the browser/Rust target, concretely

The operator's target: a **browser/Rust host** builds its own generic host, pulls a program in, understands
the contract, and runs it. Here is exactly what that is, and why it works:

1. **A program is three content-addressed entities:** `descriptor`, `step` (the IR), `initial_state`.
   Transfer = the receiving peer fetches them **by hash** into its own tree (the existing content-addressed
   transfer mechanism — no new wire).
2. **The Rust host reads the same descriptor** and binds the declared shapes to *its* drivers — a canvas
   `display-list` renderer, a DOM-keyboard `key-set` source. It never sees Go code.
3. **The Rust host evals `step` on its own compute engine** (Stage-1, or its own Axis-1) and gets
   **boundary-hash-identical** state, tick by tick — *provided* its engine agrees with Go's, which is
   exactly the §13.9 execution-strategy admission contract.
4. **It runs.** Same descriptor, same IR, same shape vocabulary → same program, different runtime, different
   language, different platform. That is transferable compute stopping being an assertion.

**The two ABIs make this airtight — and expose one real dependency:**

```
program transferability = compute-IR ABI (evaluation)  +  (role, shape) ABI (I/O)
                          ↑ gated on the compute corpus     ↑ gated on the shape vocabulary being standardized
                            (§13.9 — Rust eval == Go eval)     (a small new normative surface, §6/§8)
```

So the browser/Rust demo is *also* the forcing function for the **compute corpus** (§13.9): without it, "the
Rust engine agrees with Go" is hope, not proof. The corpus we flagged as "we'll get there" is precisely what
makes the cross-runtime run *trustworthy* rather than merely *apparently working* — worth stating so the two
threads are sequenced, not surprised.

## §6 Extensibility & admission — the vocabulary is a framework, not a fixed list

The operator's constraint: *keep it open to additional interfaces.* The design principle that delivers
that:

> **The shape vocabulary is a *registry* of standard I/O surfaces, extended by a rule, not a closed enum.**
> Adding a shape means: (1) name the **interface lineage / device class** it belongs to (§4); (2) pin what
> the **port carries** (`type_ref`) and its **`scene`** fields; (3) define **what a driver does** with it.
> Nothing else in the model changes — the mount loop (§2) is shape-agnostic; it dispatches on the field.

Admission is what makes the open vocabulary *safe for transfer*: a **host advertises its supported shape
set** (its driver capabilities), and mounting checks the descriptor's declared shapes against it — the same
"offered capabilities vs. required imports" check the device/actuation work uses, now over shapes. So a new
shape is **additive**: hosts that implement it gain those programs; hosts that don't **cleanly refuse to
mount and say why** (never a half-render). A headless service host supports *no* console shapes and mounts
only capability-port programs (the "console tier is an optional profile," proposal §8); a rich host adds
`text`, `display-list`, `audio-stream`, …; a future VR host adds a spatial shape — all without touching the
descriptor format or any existing program. **Extensible-by-construction, transfer-safe-by-admission.**

Two guards keep the openness from becoming fragmentation: the vocabulary must stay **finite and shared at
any given version** (an ABI, not a free-for-all — a shape means the same thing on every host), and new
shapes are **standardized before they're relied on for transfer** (the normative-home question, §8) — the
same add-as-needed discipline as the compute primitives, applied to interfaces.

## §7 The falsification plan (this is why it's the right next rung)

Three shipped programs make the descriptor **falsifiable**, which no single program could:

1. **One Go host, three programs, zero per-program Go.** Write the generic host; mount Snake, Life,
   Asteroids from their descriptors. **If the descriptor is right, all three run.** If exactly one doesn't
   fit, *that* is the finding — a missing field or shape — and it is far more informative than another
   green game. Asteroids is the stress case (two output shapes, a `key-set` input, a `scene.wrap`, a rate
   the host used to own).
2. **Then the browser/Rust host** (the operator's target). Rust builds its own generic host, fetches the
   three programs by hash, and runs them. Cross-check **state hashes against Go's** per tick — which
   doubles as the first real exercise of the §13.9 cross-runtime evaluation contract. Success here is the
   headline: *the same program, authored once, running in Go and in a browser with no shared driver code.*

## §8 Open questions (honest)

- **Descriptor discovery.** How does a host *find* the descriptor — a well-known tree path convention, or a
  registry entry? (proposal §9). The generic host needs one pinned to mount by path.
- **The shape vocabulary's normative home.** `(role, shape)` becomes a small standardized surface — where
  does it live, and how is it versioned/extended? Likely a table in the app-convention proposal (the L5
  convention), with `scene` fields pinned per shape. It must be *finite and shared* to be an ABI, yet
  *additive-extensible* (§6) to grow.
- **Cross-runtime evaluation trust = the compute corpus (§13.9).** The browser/Rust run is only *proof* if
  the Rust engine is corpus-verified against the reference. Sequence the corpus ahead of quoting the demo
  as validation (though the demo can be *built* first and gated later — the handoff can say so).
- **Scene-property completeness.** `wrap` was the first; a real renderer set (camera, z-order, palette?)
  will surface more. Grow the `scene` vocabulary per shape as programs need it — the same add-as-needed
  policy as the compute primitives.
- **Run-state & lifecycle over transfer.** Who owns start/stop/reseed when a program is mounted on a
  *remote* peer's host? Run-state is the local host's (proposal §5); a transferred program's clock is the
  receiving host's — worth stating so remote-mount doesn't imply remote-control.
- **FUTURE THREAD (parked) — how *applications* (not just game-sims) emit data.** These three probes are
  interactive tick-loops; the output shapes above are I/O *display* surfaces. A broader class — an app that
  emits *data* (a query result, a document, a feed, a form submission) rather than a frame — may want output
  *shapes* of a different family (structured records, streams to another program's input port). The
  `raw-state` shape + program-composition (wire one program's output port to another's input port, §10.3 of
  the runtime-contract exploration) already cover the mechanism; whether "application data emission" needs
  its own shape lineage beside the display lineages is a **separate design conversation** the operator
  flagged and we are **deliberately not opening now** — noted so the vocabulary's §6 extension rule is
  understood to reach it when we do.

*(Capacity note, for the record: the operator's read is that the two scaling strategies — **parallel
sharding** (proven, §2 of the Asteroids probe) and **compression into handlers** (the compile ladder, §12
of the runtime-contract exploration) — together give enough compute at the boundaries for these richer
applications, riding also on general hardware advancement. This design assumes that capacity story; it does
not re-derive it.)*

## §9 What this is / isn't

- **Is:** the manifest + host contract that makes a compute program *addressable and mountable anywhere* —
  and the `(role, shape)` ABI that makes "anywhere" include other runtimes. It is the consolidation the POC
  → Axis-1 → Asteroids arc was building toward.
- **Isn't:** a new compute capability (zero `EXTENSION-COMPUTE` change), a VM (the host is a harness, not an
  interpreter of anything but the descriptor), or a rendering standard (drivers are per-host; only the
  *shape contract* is shared). The console-tier shapes remain an **optional profile** — a headless service
  program uses none of them.
