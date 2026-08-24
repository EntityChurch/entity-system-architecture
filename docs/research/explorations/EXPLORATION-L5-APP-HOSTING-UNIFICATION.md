# EXPLORATION — L5 App-Hosting Unification (the "entity app standard" beyond JS iframes)

**Status:** Exploration / cross-team synthesis — **not** a proposal, **not** ratified.
Draws together three already-existing lines of work into one map so the arch + browser +
workbench teams converge on a single app-hosting model. The ratifiable output of this doc
is a future `proposals/` entry (per `CHARTER.md`); nothing here changes a spec.

**Why now.** Two independent lines of work — the **JS-iframe app platform** and the
**generic compute host** — have been converging without anyone drawing the combined
picture, and a rushed alignment pass left the question *"what runs where, in whose peer?"*
unanswered. The rush **overloaded browser-rust** and likely loaded compute programs
**directly onto the system peer** — the thing to *not* do. The POC got far enough to prove
the mechanism works; this doc is the **step back to get the architecture right** before
building further. It names the two lineages, states where each stands, proposes the
unifying frame, and lists the design decisions (and a design-level sequence). Author:
meta/coordination pass; hand to arch as W1 (`applications/`) input.

**Direction: confirmed by the operator (2026-07-20)** — this is the intended design; the
open questions in §5 (esp. the sub-peer isolation model and the monolithic→chunked state
problem, §5.7) are the substance to pick up with the arch team.

**Sibling track (kept separate on purpose).** This doc is the **top** of the stack (L5 app
hosting). Its counterpart `EXPLORATION-MACHINE-BOUNDARY-AND-FULL-STACK-TRAJECTORY.md` is the
**bottom** (the machine boundary / substrate minimality / the long-horizon "entity as its
own full stack" vision). They meet at exactly one seam — the **generic host** (actuation
over hardware below, app execution above) — and share one principle, the **determinism
boundary** (`compute/apply`); otherwise the decisions, teams, and horizons differ, so the
tracks stay distinct documents.

---

## 1. The two lineages (both are called "app" today)

There are **two distinct things** wearing the word "app," and the operator's goal is to
unify them under one L5 standard with several hosting profiles.

### Lineage A — opaque UI apps (shipped, settled)
- **What:** a self-contained single-file HTML/JS app dropped into an
  `<iframe sandbox="allow-scripts">`. The app is a **dumb client — not an entity peer.**
- **Contract:** the app↔host **postMessage lifecycle/state contract** —
  `ready-for-init` → `init{state}` → `viewport` / `audio` → `ready` → `state{…}` →
  `request-state` → `destroy` → `closed` (+ `error`), 150 ms init timeout. State is an
  **opaque `{__v, data}` envelope**; the host is **dumb key→blob storage**, forbidden to
  inspect or migrate it.
- **Where it lives:**
  - Corpus + contract: `entity-apps-htmljs` — `docs/EMBEDDING.md`, `templates/index.html`
    (reference host), `sdk/entity-app.js` (lifecycle). Host asks consolidated in
    `docs/plans/ASK-HOSTING-TEAM-cgid-10-232.md` — the contract is declared **settled**.
  - Host in the browser: `entity-browser-rust` —
    `docs/architecture/specs/REFERENCE-ENTITY-JS-APPS-PLATFORM.md`; three CBOR entity
    types `app/app-catalog` (entries), `app/app-bundle` (the HTML), `app/app-save`
    (opaque state); runs in `<iframe sandbox="allow-scripts">`, postMessage only.
- **State:** shipped, contract frozen; the app is isolated by the iframe and never touches
  the host's tree except through the opaque save.

### Lineage B — compute programs (proven 3-way, convention still draft)
- **What:** an app is `state₀ + step` — a **program descriptor** (entity type
  `app/program/interface`) + a **content-addressed step IR** + an **initial state**. It is
  **entity-native compute**, evaluated via **EXTENSION-COMPUTE**, and it is
  **transferable + deterministic across runtimes** (fetch the three hashes, mount, get
  byte-identical per-tick state hashes on any conformant host).
- **The generic host:** a descriptor-driven harness that mounts *any* program with **zero
  per-program code** — `seed → bind (role,shape) drivers → clock: eval(step)→put(state),
  refresh output ports`. It has **no authority to shard or rewrite** expressions; sharding
  is a program contract, not a host power.
- **The driver ABI:** `(role, shape)` grounded in I/O lineage, registry-extensible —
  outputs `text` / `framebuffer` / `display-list` / `audio-stream` (+ `raw-state` escape);
  inputs `key-set` / `pointer` / `direction` / `event-stream`. Multiple heterogeneous
  output ports per program (e.g. Asteroids emits both `state` and `display-list`).
- **Where it lives:**
  - Arch: `docs/research/explorations/EXPLORATION-GENERIC-HOST-AND-DESCRIPTOR.md`,
    `EXPLORATION-COMPUTE-PROGRAM-RUNTIME-CONTRACT.md`, and the **draft**
    `docs/proposals/PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM.md`. Design rulings in
    `docs/status/HANDOFF-2026-07-17-generic-host-design-rulings.md`.
  - Go host (reference): `entity-workbench-go` — `workbench/program_descriptor.go`,
    `workbench/program_host.go` (proven to name **no** program symbol), with static-k
    sharding landed **into** the host (`docs/.../COMPUTE-SHARDING-INTO-HOST-2026-07-18.md`).
  - Rust/browser host: `entity-browser-rust` mount machinery already passes the **Go
    oracle hash-identical** on Life / Snake / Asteroids fixtures
    (`docs/architecture/reviews/REVIEW-COMPUTE-GENERIC-HOST-BROWSER-2026-07-19.md`;
    commits `9f87416`, `695583a`, `7d19a42` on `dev`).
- **State:** the generic host + descriptor is **proven three ways** (workbench-go + the
  browser mount + the Go oracle), but the `applications/`-domain **convention is still a
  DRAFT proposal** — not cut to vectors, not ratified.

### The FORMAT layer already anticipates active/WASM apps
`specs/applications/APP-CONVENTION-EMBED.md` (v0.2.3 draft) is deliberately built so a
rich-content node can be **active**: an `Embed` may carry `requires` (capabilities) +
`sandbox-constraint`, and its handler may be **an entity-native compute graph
(`expression_path`, needs EXTENSION-COMPUTE) or a WASM handler** — but **active embeds are
deferred behind gate G1** (§7), and the sandbox declaration is deliberately
**substrate-neutral** (no `iframe`/`wasm` literals). So the convention corpus already has a
reserved seat for "an app is live code in a sandbox"; it is named and gated, not designed.

---

## 2. The unifying frame — three independent axes

The confusion dissolves once you stop treating "app" as one thing. An entity app is a
choice on **three independent axes**:

| Axis | Options | What varies |
|---|---|---|
| **① Payload — what the app *is*** | (a) opaque UI bundle (HTML/JS today) · (b) compute program (descriptor + step IR + state) · (c) full **WASM entity peer** (core + generic host + extensions, compiled to WASM) | the app's runtime substance |
| **② Isolation — *where* it runs** | (i) the **system peer** (reserved — avoid) · (ii) a **user peer** · (iii) a **sandboxed iframe** (browser's preferred pattern) · (iv) a **native sub-peer / process** | the trust + failure boundary |
| **③ Boundary contract — *how* host and app talk** | (α) **postMessage UI/state** (dumb client) · (β) **entity protocol over a transport** (the app is a peer — postMessage-tunnelled in-browser, WS/TCP natively) · (γ) **EMBED active-embed handler** (FORMAT layer, gate G1) | the wire between host and app |

Today's two lineages are just two diagonals through this cube:
- **Lineage A** = ①a payload, ②ii iframe isolation, ③α postMessage. (The app is *not* a peer.)
- **Lineage B** = ①b payload, mounted by a generic host — but ② (which peer?) is exactly the
  **unresolved axis** that caused the "off the rails" feeling.

The operator's target cases map cleanly onto the cube:

| Operator's phrasing | ① payload | ② isolation | ③ contract |
|---|---|---|---|
| "embedded iframe JavaScript apps" (today) | opaque UI | iframe sandbox | entity-app contract |
| "a true WebAssembly entity-core running in an iframe … running the generic system host" | **WASM entity peer** | **iframe sandbox** | **entity-app contract (a peer inside)** |
| "just a pure WebAssembly Rust app" | opaque UI (WASM, no entity-core) | iframe sandbox | entity-app contract |
| "NED native … runs its own peer, runs the generic system host" | WASM/native entity peer | native sub-peer (host's choice) | entity-app contract or entity protocol |

Note the isolation column is mostly **iframe** for browser-rust — the host's uniform
mechanism — precisely *because* of P1 (§3): the host treats them all identically. What
differs is the payload, not the seam.

**The through-line is the generic system host.** Whether native (workbench-go) or
WASM-in-iframe (browser phase 2), the *same* descriptor-driven host mounts the program.
"A true WASM entity core running the generic host in an iframe" is simply **Lineage B's
generic host, deployed into an iframe-sandboxed WASM sub-peer** — instead of Lineage A's
dumb-JS-client model.

---

## 3. The two load-bearing principles (the operator's core calls)

### P1 — the host-agnostic boundary contract (the crown jewel)
> **The host does not know or care what runs inside a hosted app. It handles one boundary
> contract — the app starts up, occasionally emits state, the host manages that state — and
> is otherwise blind to the payload.**

This is the most important idea and the thing that makes the whole family one family. To
`entity-browser-rust`, a hosted app is **an iframe with a contract**: it boots, it emits
state now and then, the host persists that state, done. Inside that iframe could be:
- a **JavaScript app** that knows nothing about the entity system ("all I do is emit state
  and run"), or
- a **plain Rust→WASM app** with no entity-core in it at all, or
- a **full entity-core WASM peer running native compute programs on the generic host.**

The host cannot tell them apart, and *that is the point.* The boundary contract is the
universal seam; the payload is opaque. This is why the same app corpus is portable across
hosts (§ below).

### P2 — apps run isolated; the system peer stays reserved
> **Don't fill the system peer with random apps and compute. Hosted apps run isolated — the
> sandboxed iframe is the preferred pattern; a user peer is acceptable; the *system* peer is
> reserved.**

Correction to the first draft: browser-rust is not one peer — it runs **many** (notably the
**system peer**, the IndexedDB-backed control plane). The rule is not "never any peer but
one"; it is **keep application compute *off the system peer*** (it's reserved), and prefer
the **sandboxed iframe** — a little VM-like sandbox in the browser that keeps transferred,
possibly-untrusted compute from running *at the same level as* the entity system. A **user
peer** is a legitimate alternative when you want it; other hosts (godot, workbench-go) may
choose whether to sub-peer-isolate at all. Isolation is the concern; the iframe is
browser-rust's chosen mechanism, not a universal mandate. EMBED's `sandbox-constraint` +
gate-G1 vocabulary (`deny_external_handlers` default TRUE, cross-peer dispatch
refuse-by-default) is the FORMAT-layer seed for the same instinct at the hosting layer.

### The payoff — portability across hosts
Because the contract is host-agnostic (P1) and the generic host is descriptor-driven, **any
host that implements the generic-host contract can run the same transferable compute-native
programs** — `entity-browser-rust` today, a **godot host**, `entity-workbench-go`, a native
binary. Whether each isolates the app in a sub-peer/iframe is *its* call. You author a
compute-native program once, transfer it by hash, and it runs anywhere the contract is
implemented. This is the most interesting property of the whole design.

## 3b. Two orthogonal constructs — keep them distinct

A recurring source of the confusion: **the entity-app contract and the generic host are two
different things that compose, and neither implies the other.**

- **The entity-app contract** = the **boundary/hosting pattern** (P1): boot → emit state →
  host manages state. It is the pattern for *hosting applications*, and the operator wants
  to **keep it, maintain it, and make it THE pattern.** It says nothing about *what* the app
  is.
- **The generic host** = the **execution engine** for transferable **entity-compute-native
  programs** (descriptor + step IR + state).

They cross freely:

| | uses the generic host | opaque payload |
|---|---|---|
| **is an entity-app** (hosted via the contract) | compute program in a sandboxed WASM peer, emitting state to the host | today's JS/HTML apps |
| **not an entity-app** | a generic-host application whose state is managed *elsewhere* (not the entity-app contract) | (n/a) |

So "a fully generic-host application that is **not** an entity-app, expected to be hosted
and emit state to have its state managed elsewhere" is a real, valid quadrant — the
entity-app contract is *the pattern we arrived at*, not the only way to be hosted.

### The "different startup mode" realization
The embedded **entity-core-peer-running-the-generic-host** is not a separate codebase — it
is **`entity-browser-rust` in a different startup mode.** The outer browser-rust is a peer
that hosts peers and manages a distributed entity-OS; the *embedded* instance boots a
minimal mode inside the iframe: no window-manager UI, no peer roster — just load the
state-emission handler, load the generic-host code, bundle the interfaces + whatever custom
code the specific compute-native app needs to build its interface, spin up the generic host,
kick off the entity compute, and emit state as it runs. Same code, stripped startup. The
outer host still just sees "a sandbox emitting state."

---

## 4. Where we actually are (status snapshot)

| Piece | State | Evidence |
|---|---|---|
| JS-iframe app contract (Lineage A) | **Shipped / settled** | `entity-apps-htmljs` EMBEDDING + ASK-HOSTING-TEAM; browser JS-apps platform spec |
| Generic host + descriptor (Lineage B) | **Proven 3-way; convention DRAFT** | workbench `program_host.go` + sharding-into-host; browser mount passes Go oracle; arch DRAFT proposal |
| EMBED convention (FORMAT layer) | **v0.2.3 draft; spine locked 3-way; passive-only** | `APP-CONVENTION-EMBED.md`; active/WASM deferred behind G1 |
| SITE convention | **v0.5 draft; read-path ships at preview** | `ROADMAP-APPLICATIONS.md` |
| **Unified hosting model (this doc)** | **Undesigned — first sketch here** | — |
| WASM-peer-in-iframe host | **Not built** | dependency: 2 wasm32 stubs upstream (below) |
| "NED-native" hosting | **Not formally named in the corpus** | term maps to native sub-peer; see §5 |
| **Chunked/aggregating state emission** | **Not built — the flagged open problem** | see §5.7 |

**Known upstream dependency (not new work to invent, just to name):** the browser compute
review found **two wasm32 stubs in `entity-core-rust`** that block real in-browser compute
— `compute/apply` handler dispatch returns InvalidExpression on wasm32 (no async blocking
equivalent), and the recursion stack-guard is compiled out on wasm32 while MAX_DEPTH stays
1024. These gate the WASM-peer story regardless of how the hosting model lands. (Route as a
spec/impl item to the rust team per the honesty rule — named here as a dependency, not
tracked here as a checklist.)

**Naming flag — "NED native":** the operator's term **does not appear in the arch corpus.**
It maps to the already-sketched **"standalone native binary that bootstraps its own peer +
generic host"** (native sub-peer). *Recommend pinning canonical terminology* before it
spreads informally — either adopt "NED" with a definition, or use "native sub-peer host."

---

## 5. Open design decisions (for the arch cohort)

These are the calls that turn the frame in §2 into a ratifiable convention. Each is a
genuine decision, not a foregone one:

1. **One convention or two?** Is the unified thing **one `APP-CONVENTION-APP` manifest**
   with profiles (opaque-UI / compute-program / wasm-peer), or **distinct conventions** in
   `applications/` that share a hosting envelope? (Lean: shared *hosting envelope* +
   per-payload manifest — the postMessage contract and the program descriptor are too
   different to force into one schema, but the **isolation + lifecycle** layer is common.)
2. **The sub-peer isolation model.** How does a mounted program/app get its **own peer +
   sub-tree**, and what is the exact boundary to the primary (path prefix it may read/write,
   capability grants, cross-peer dispatch policy)? This is the operator's core concern and
   the biggest gap. EMBED §3/§7 (`sandbox-constraint`, G1 conditions C1–C5) is the seed.
3. **The host↔app-peer transport (③β).** For a WASM peer in an iframe, the app talks the
   **entity protocol tunnelled over postMessage** (vs the dumb ③α postMessage UI contract).
   Define that tunnel — is it EXTENSION-NETWORK framing over a `MessagePort`? A reduced
   local transport? This is where Lineage A's postMessage and Lineage B's peer model meet.
4. **Where the JS lifecycle contract sits in the stack.** Is the
   `init/viewport/state/destroy` lifecycle a *profile* of the unified hosting envelope, or
   a legacy adapter the WASM path doesn't use? (It is genuinely useful for *any* embedded
   surface — viewport/audio/teardown are substrate concerns, not JS concerns.)
5. **EMBED active-embed vs app-peer — same mechanism?** EMBED's deferred "active embed =
   WASM/compute handler in a sandbox" (gate G1) and this doc's "WASM app-peer in an iframe"
   look like the **same capability at two scales** (a widget vs a whole app). Decide whether
   they share one sandbox/capability mechanism or stay distinct. If shared, **G1 is the
   security gate for both** — do it once.
6. **Terminology:** pin "NED native" (§4).
7. **Monolithic → chunked/aggregating state emission (the operator's flagged open problem).**
   The state-emission contract is **monolithic today** — an app emits its *whole* state
   snapshot each time, and the host stores one opaque blob per app (Lineage A's `{__v,data}`;
   Lineage B's `put(state_path)` each tick). No limit has been hit yet, but **some apps'
   state emissions are getting large and constant**, and there is **no ability to emit state
   in chunks that aggregate** (deltas / sub-tree-scoped emissions / append-then-fold) rather
   than re-emitting the monolith. This is likely the highest-value optimization and is
   currently **unanalyzed** — flagged for a dedicated look. It touches both lineages: the
   entity-app `state` message *and* the generic host's per-tick `put`. (Note: the
   entity-native tree is *already* `path→hash` with content-store dedup — a sub-tree-scoped
   or delta emission may fall out naturally for Lineage B, which is a reason to design the
   emission contract with chunking in mind rather than bolt it on later.)

---

## 6. Recommended sequence (design-level — get back on track)

Ordered so each step unblocks the next and nothing waits on a decision that hasn't been
made. This is a **design** sequence (what to decide/spec in what order), not an impl
checklist for the teams.

1. **Land this map** (this doc) + socialize with arch + browser + workbench. Agree the
   three-axis frame and the §3 principle *before* more code moves. ← you are here.
2. **Ratify the compute-program convention.** The generic host + descriptor is proven
   three ways; take `PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM` off DRAFT the same way
   EMBED/SITE are going: **cut joint conformance vectors → ratify.** This locks Lineage B's
   descriptor as an L5 convention and gives the browser a stable target.
3. **Spec the sub-peer isolation model** (decision §5.2) — the missing piece. Where a
   mounted app lives, its sub-tree, its capability boundary to the primary. Reuse EMBED's
   `sandbox-constraint` + G1 vocabulary rather than inventing a second sandbox model.
4. **Confirm + correct the browser mount target.** Verify whether the current browser
   mount machinery runs programs in the **primary** peer or a **sub-peer**; if primary,
   that is the concrete "back on track" fix — move it behind the §3 boundary. *(Open
   question to the browser team, not asserted here.)*
5. **Design the WASM-peer-in-iframe host** (decisions §5.3 + §5.5) — the new capability and
   the convergence point of both lineages. Gated on the sub-peer model (step 3) and the two
   upstream wasm32 stubs (§4).
6. **Formalize native ("NED") hosting** — the same generic host + sub-peer model on a native
   substrate (workbench-go / a Godot host already do a version in-process). Mostly a naming
   + boundary formalization once steps 2–3 land.

---

## 7. Provenance (files read for this synthesis)

- `entity-system-architecture`: `specs/applications/APP-CONVENTION-EMBED.md` (v0.2.3),
  `ROADMAP-APPLICATIONS.md`, `docs/research/explorations/EXPLORATION-GENERIC-HOST-AND-DESCRIPTOR.md`,
  `docs/proposals/PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM.md`, `specs/SYSTEM-ARCHITECTURE.md` §2 (L0–L5).
- `entity-workbench-go`: `workbench/program_descriptor.go`, `workbench/program_host.go`,
  `docs/architecture/reviews/COMPUTE-GENERIC-HOST-PHASE1-RESULT-2026-07-17.md`,
  `COMPUTE-SHARDING-INTO-HOST-2026-07-18.md`.
- `entity-browser-rust`: `docs/architecture/specs/REFERENCE-ENTITY-JS-APPS-PLATFORM.md`,
  `docs/architecture/reviews/REVIEW-COMPUTE-GENERIC-HOST-BROWSER-2026-07-19.md`,
  `docs/architecture/specs/DISCIPLINE-REFRAME-BROWSER-SUBSTRATE.md` (L5=DOM).
- `entity-apps-htmljs`: `docs/EMBEDDING.md`, `docs/plans/ASK-HOSTING-TEAM-cgid-10-232.md`,
  `sdk/entity-app.js`.

*This is a coordination synthesis authored in the arch workspace at the operator's request;
the ratifiable output is a `proposals/` entry produced by the arch cohort via
proposal → ratify → fold, not this file.*
