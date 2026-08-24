# EXPLORATION — The Machine Boundary & the Full-Stack Trajectory

**Status:** Exploration / cross-track synthesis — **not** a proposal, **not** ratified.
Pulls the Keystone cohort's *substrate / machine-boundary* research into the arch workspace
so the architecture team can pick up the longer-horizon "entity system as its own full
stack" question. The detailed, ratifiable spec asks already live as routed hand-offs in
`entity-core-keystone/research/stewardship/` (see §9) — this doc is the **vision + the
three-track convergence + the honest proven-vs-open split**, not a re-listing of those asks.

**Sibling track.** This is the **bottom** of the stack; its sibling
`EXPLORATION-L5-APP-HOSTING-UNIFICATION.md` is the **top** (L5 app hosting). They are kept
**separate on purpose** (different altitudes, different teams) but they meet — see §8. The
app-hosting doc asks *"how do apps run on top of the entity system?"*; this doc asks *"how
little does the entity system need underneath it, and could it eventually become the
bottom?"*

**Provenance.** Synthesizes `entity-core-keystone/research/`:
`SUBSTRATE-MINIMALITY-AND-THE-MACHINE-BOUNDARY-2026-07-20.md`,
`SUBSTRATE-THEORY-ALIGNMENT-2026-07-19.md`, `SUBSTRATE-TAKEAWAYS.md`,
`CONVERGENCE-MAP-2026-07-19.md`, `COMPUTE-PARADIGM-AND-META-REVIEW-2026-07-19.md`, and the
two `stewardship/HANDOFF-TO-ARCH-2026-07-{18,19}-*.md`. It also leans on **Paper 04 «The
Entity Machine Boundary»** as referenced by those docs (not re-verified here). Section marks
below cite `(SUBSTRATE-MINIMALITY §N)` etc. against those files.

---

## 1. The machine boundary — and the incident that prompted the question

**The machine boundary** is the interface between **entity computation** (deterministic,
content-addressed, capability-dispatched) and the **physical / host substrate** (an OS, a
WASM runtime, an ISA, silicon). The question it forces: *how much of the host does the
entity system actually depend on?* (SUBSTRATE-MINIMALITY §0, §1, §3c)

The investigation was prompted by a **host failure**, not theory. A commodity Linux host
under heavy concurrent container load can hard-lock: a rootless container teardown wedges a
task in expedited-RCU (`synchronize_rcu_expedited`); the stalled grace period holds a cgroup
lock every process spawn/teardown needs; the whole process lifecycle halts — new shells
never reach a prompt, `init` itself blocks, the stuck task takes no signal (not even
`SIGKILL`), and only a reboot recovers. This is a **known class of monolithic-kernel
failure**, a clean specimen of *shared mutable state under aliasing, defended by lock
hierarchies* (SUBSTRATE-MINIMALITY §1). *(Operationally relevant: this is the same failure
class behind the load-related lockups we're currently seeing on the other box — the incident
is not hypothetical.)*

---

## 2. The diagnosis and the entity wager

The claim (a provocation, held honestly as such): the **mutable memory address** is the
foundational commitment of mainstream computing, and much of the stack — cache-coherency
(MESI/MOESI), lock hierarchies and inversions, virtual-memory complexity, the very RCU
grace-period machinery that caused §1 — is downstream of *managing shared mutable state
under aliasing* (SUBSTRATE-MINIMALITY §2).

Entity-core makes the **opposite bet on all three axes that produced the failure:**

| The failure's ingredient | The entity commitment |
|---|---|
| shared **mutable** state under aliasing | **immutable, content-addressed** content — no aliasing, no coherency traffic |
| a **lock hierarchy** that inverts | **single-writer ownership / message-passing** — no lock to invert |
| **weak fault isolation** (one task freezes all) | **capability-scoped authority** — blast radius = the granted scope |

The bet is **coherent and testable, not rhetorical**: peers that get store-safety *by
construction* (actor-isolation, dataflow variables, STM, Unison's single-`MVar`) are immune
to this failure class because they refuse shared mutable state (SUBSTRATE-MINIMALITY §2;
SUBSTRATE-TAKEAWAYS §4).

---

## 3. Three tracks, one object

The same question — *what must the substrate actually provide?* — was answered independently
at three altitudes. **They do not conflict; they stack** (SUBSTRATE-MINIMALITY §3).

- **3a — Top-down (Paper 04, analytic floor).** A bootstrap evaluator (**~400–500 LOC of C,
  a design estimate**) boots the whole system from a conforming tree. It rests on **seven
  irreducible machine primitives** (byte ops, SHA-256, CBOR, string compare, integer
  arithmetic, allocation, I/O), **four fixed points** that survive every compilation stage
  because they touch reality (tree writes, cross-peer exchange, capability checks, handler
  transitions), and **five profiles** tracing how much of the machine gets absorbed:
  1 compute-only · 2 storage · **3 OS-hosted (today)** · 4 hybrid-kernel · **5 bare-metal**.
  (SUBSTRATE-MINIMALITY §3a)
- **3b — Bottom-up (the 43-peer cohort, empirical).** 43 peers across ISAs, runtimes, and
  paradigms pass the **same core gate** — concrete proof the floor is genuinely thin. The
  §7b **concurrency taxonomy** bounds *how thin the concurrency guarantee can be*: **a
  single-threaded event loop suffices** (Pd, Io, TurboWarp); threads/preemption/shared-memory
  are **not in the floor**. The **Unison peer** sharpened the crypto axis — a managed runtime
  with **no C-FFI** still hosts a peer (hand-rolled Ed25519 in pure Unison), so "can call C"
  is not in the floor either. (SUBSTRATE-MINIMALITY §3b; SUBSTRATE-TAKEAWAYS §4)
- **3c — Sideways (device / host / scheduler, from the runtime side).** Independently
  reconstructed from the compute-program problem as **sense → schedule → actuate**:
  `system/device` = layer-1 sensing (read-only host-fact advertisement); the **generic host**
  = layer-3 actuation (`mount(descriptor) → running program`, zero per-program code); the
  scheduler = layer-2 (future). Two tracks converged on one primitive —
  **`system/device/host/offered`**, the *hostability contract* (the peer's dispatchable host
  capabilities, written by the sensor, read by every actuator's admission). The
  **determinism boundary is load-bearing**: the CPU/GPU line *is* the `compute/apply` seam —
  entity-compute stays a deterministic CPU tree-walk; GPU/DSP work lives on the native side
  as a driver, never as entity-compute (= Paper 04's Category-C native-handler boundary,
  reached from the runtime side). (SUBSTRATE-MINIMALITY §3c)

**Why this matters:** three methods (analytic, empirical, runtime) converge on one object
that behaves like a computer architecture, a runtime, and a distributed OS at once — reached
**from the protocol down, not the hardware up.** The convergence *falls out of a shared
minimal core*, which makes it more trustworthy than any single track's claim
(SUBSTRATE-MINIMALITY §9).

---

## 4. The substrate-minimality thesis — where it actually lands

**How much substrate does the entity system need? — very little, and it is
substrate-independent.** Entity-core is **not an OS** in the traditional sense (no scheduler,
memory manager, or drivers); it is a **protocol** whose core floats on an extraordinarily
thin substrate, proven independent by the 43-peer cohort. At Profile 3 it is a **guest** on
Linux/Windows/macOS (SUBSTRATE-MINIMALITY §3, §5).

The thesis lands as a **trajectory, not a resting place**, and the honest split is the whole
point:

- **PROVEN — substrate *independence*.** Nothing in the core depends on any particular host,
  ISA, or runtime. The method was inverted vs every OS effort: author the high-level protocol
  first, treat host/ISA/runtime as **free variables** — and the 43-peer cohort showed the
  inversion held. (SUBSTRATE-MINIMALITY §3b, §9)
- **OPEN — substrate *replacement*.** Whether the entity role can be filled **all the way to
  the metal** — with acceptable performance and without re-growing the complexity it set out
  to avoid — is **unproven and unbuilt**. The cohort proves independence; it does **not**
  prove replacement. *Different claims; only the first has evidence.*
  (SUBSTRATE-MINIMALITY §6, §8)

Keep that line crisp in anything downstream: **independence = fact; replacement = open
question.**

---

## 5. The full-stack trajectory (Profile 3 → 4 → 5)

The longer-horizon vision the tracks jointly sketch — **the guest→host inversion, applied at
the OS level.** Today entity is a guest borrowing scheduling/memory/process-lifecycle and
inheriting the host's failure modes (the §1 lockup). The trajectory flips the relation
(SUBSTRATE-MINIMALITY §5, §5a–b):

- Entity becomes the **host**; Linux becomes a **driver-abstraction guest**.
- The **OS proper is a pristine peer** — no privileged access, just another handler.
- Hardware/architecture support grows through the **computational genome** — the compilation
  gradient + Paper 04's *"machine-architecture-as-entity-domain,"* where instructions,
  registers, and ABIs are themselves **typed entities**.
- Entity-native code runs **pure** — no context-switch tax, no inherited-lock complexity;
  conventional OSes and VMs are hosted *above* the entity layer (drivers for legacy
  compatibility), not beneath it.
- The **Entity ABI** is the concrete redefinition: `syscall→EXECUTE`, `file→tree path`,
  `open→get`, `write→emit`, `process→peer`, `driver→handler` (SUBSTRATE-MINIMALITY §6).
- **"Peer zero" / reverse-`init`.** Paper 04's Entity-ABI table has no row for `init`/PID 1.
  The proposal: peer-zero ≈ **bootstrap evaluator + seed-policy root + persistent identity** —
  and all three already exist, prototyped at Profile 3 by the cohort's L1 (the Unison peer's
  `--debug-open-grants` is literally the degenerate root seed `default→*`, the ur-capability an
  attenuated tree grows from). The missing ABI row is `init/PID 1 → seed peer`.
  (SUBSTRATE-MINIMALITY §5b)

> **Honest status (carry this verbatim in spirit):** the reversal is **entirely unbuilt, a
> long horizon, and not required for near-term value** — but it is **coherent**, and it is
> the *same shape* as the guest→host inversion the cohort already demonstrated at the
> protocol layer (SUBSTRATE-MINIMALITY §5, §9).

**Reframing "what is an operating system."** OS-ness is a **role** — mediate between programs
and a machine, arbitrate resources, provide identity and isolation — not the fixed Unix-ish
artifact (scheduler+drivers+processes) we inherited and then mistook for the whole. An ESP32
running firmware, a unikernel, the BEAM are all "an OS" by role. Entity re-targets the role
at **entity compute** instead of Linux (SUBSTRATE-MINIMALITY §6).

---

## 6. The physics bridge — and the near-term posture that follows

The vision has a hard floor, stated honestly: **informational closure is not physical
closure.** The tree can describe its own evaluator, but a description does not execute
itself — some physical process must run first. Entity-native hardware does not escape this;
it *moves* the bridge from "compile the first evaluator with an external compiler" to
"fabricate the first entity processor." The dependency on the physical substrate is
**irreducible** (SUBSTRATE-MINIMALITY §7).

Two consequences that are **actionable now**, not long-horizon:

1. **The hardware floor sits beneath the whole edifice.** A flaky DIMM, a marginal power
   rail, a silicon glitch live *below* the bootstrap bridge; no protocol or capability model
   reaches them. So the correct posture at Profile 3 is **survivability, not prevention** —
   entity-core cannot *prevent* a host/hardware failure, but it can make one **cheap and
   recoverable**: content-addressed durable state loses no committed work; cross-peer
   continuation lets another peer resume; deliver-or-signal makes the failure **observable
   rather than a silent freeze** (SUBSTRATE-MINIMALITY §7).
2. **The system's honesty is that it *names* the bridge.** JVM/WASM/BEAM hide the machine
   boundary behind an opaque runtime; Paper 04's method makes it explicit — which is what
   lets the system reason about its own physical realization and say precisely where the
   irreducible dependency sits (SUBSTRATE-MINIMALITY §7).

This posture is the **bridge between the two tracks**: survivability is a Profile-3, buildable
concern that the app-hosting layer (sibling doc) directly benefits from.

---

## 7. Substrate theory — the independent second derivation

A separate (academic-framed) analysis ran a 12-step primitive extraction and arrived at
**six primitives — {E, I, T, M, X, P}** — the same structure as the keystone convergence:
**E** entity, **I** identity/canonical-hash, **T** tree (primordial: information, no
time/agency); **M** emit, **X** execution (temporal: time+agency); **P** peer (spatial). The
central structural property: **X is a Kd4 deterministic evaluator** — the property that makes
entity-core a *hard* substrate (SUBSTRATE-THEORY-ALIGNMENT §1–§2).

Keystone's independently-derived kernel and these primitives **converge**, item by item:
content-addressing ↔ E+I, single-writer ownership ↔ P/TP, **determinism-as-master-property ↔
X-at-Kd4** (the strongest single convergence), self-certification ↔ E+I+signatures. Two
independent internal derivations, one structure — **cross-corroboration, not proof, and both
pathways are internal/unreleased** (keep the independence framing; don't dissolve it)
(SUBSTRATE-THEORY-ALIGNMENT §2, §8).

**The one resolved open question — async vs sync.** Request/response is **X in its native
shape** (a deterministic function: input→result); async-mailbox is **X decoupled from its
result through M** (X∘M = CONTINUATION+INBOX). So **sync is more primitive** (it is X alone),
which is exactly why entity kept sync in core (X) and put async above (X+M). Entity ships
three async shapes — sync request/response, async fire-and-forget (`deliver_to` + 202-ack +
inbox), reactive/deferred (standing continuations + subscription) — and deliberately does
**not** make delivery *guarantees* primitives (they live above core by minimality +
agnosticism + policy). (SUBSTRATE-THEORY-ALIGNMENT §4, §9)

**The hosting test.** Seven external traditions — Unison, Croquet/TeaTime, Adapton, blockchain,
CRDTs, ocap/CapTP, the actor model — **all pass**: entity hosts and subsumes each as an
extension/composition, and **zero new primitives survive minimality**. (CONVERGENCE-MAP §1–3)

---

## 8. Where this track meets the app-hosting track

The two explorations are distinct but touch at exactly one seam — **the generic host** — and
share one principle — **the determinism boundary**:

| | This doc (machine boundary) | Sibling (L5 app hosting) |
|---|---|---|
| Altitude | L_native / the substrate below the protocol | L5 / applications above the protocol |
| Core question | how little substrate is needed; can entity become the host? | how do apps run on top; in whose peer? |
| The **generic host** | layer-3 **actuation** driver over `system/device` sensing (§3c) | the engine that runs transferable compute-native app payloads |
| **Determinism boundary** | the CPU/GPU line *is* `compute/apply`; native work is a driver | entity-compute apps stay deterministic; native effects are host drivers |
| `system/device/host/offered` | the hostability contract a scheduler will match against | the host capability set an app-peer's admission checks |

So the generic host is the **shared spine of the whole stack**: the same descriptor-driven
`mount → run` loop is *actuation over hardware* at the bottom and *app execution* at the top.
That is why the full stack is one story told from two ends — but the tracks stay separate
documents because the **decisions, teams, and horizons differ** (near-term app hosting vs
long-horizon substrate replacement).

---

## 9. What's routed to arch (pointers, not a re-list)

The concrete, ratifiable asks already exist as **routed hand-offs in the keystone repo** —
their canonical home; this doc does not duplicate them. Arch should absorb them (candidate
home: `docs/research/reviews/`). Summary of categories:

- **`HANDOFF-TO-ARCH-2026-07-19` (convergence & substrate review):** documentation/framing
  items (state consistency as **single-writer-ownership** with the quadrilemma as frame;
  revocation as an **eventual OR-Set**, transitive revocation as structural invariant);
  documented conscious trades (deliver-or-signal vs at-most-once; `hash(pubkey)` identity +
  no core key-rotation → F-PQ above core; **language-agnosticism as a first-class principle**);
  a rich set of **extension-design inputs** (trust-sized ordering seam; CRDT-typed paths;
  Matrix-style state-res; actor model as entities; ActivityPub/Nostr/ATProto vocabularies;
  compute borrowables — Salsa red-green, cyclic-definition hashing, height-ordered
  stabilization, promise-pipelining-as-continuation); and the honest **open-but-tracked**
  residue (durable at-least-once = deployment convention; cross-peer continuation resumption;
  crash-mid-flight collector; consensus/Raft named-and-deferred).
- **`HANDOFF-TO-ARCH-2026-07-18` (critical-review outputs):** three **core-touching MUST
  one-liners** worth landing **before core freezes** — RT-13a atomic/lock-guarded refcounts
  (§4.8), RT-13b frame-write atomicity (§1.6), RT-14 lowercase-hex in tree paths — each
  surfaced by a specific peer (C, Io, Common Lisp) and **absent from F1–F46**; a design
  proposal, **W6 mint-time resource absolutization** (make the §PR-8 granter-frame bug class
  structurally impossible); and **F-PQ** cross-algorithm identity migration (routes to
  identity/quorum; `PROPOSAL-MULTIKEY-MULTIHASH-ALIGNMENT` has no file in arch yet).

**Near-term experiments that turn theory into evidence** (SUBSTRATE-MINIMALITY §8) — ordered
by leverage:
1. **Measure the bootstrap evaluator** — does ~400–500 LOC hold? The single most load-bearing
   unmeasured number in the whole picture.
2. **A survivability-first Profile-3 deployment** — durable store + cross-peer continuation +
   a host watchdog that panics-and-recovers on a `D`-state hang; turns the §1 freeze into a
   bounded, signalled, auto-recovered handoff. **Buildable now** — the cheapest test of the
   survivability thesis, and the most directly useful given current host trouble.
3. **The browser/Rust cross-runtime generic-host demo** — same program, authored once,
   fetched by hash, run in Go and in a browser with no shared driver code, boundary-hash
   identical. (This is *also* the sibling track's falsification — the seam where the two docs
   share a test.)
4. **An FPGA prototype of the six-stage pipeline** — far-horizon test of whether the von
   Neumann overhead is a translation artifact (Paper 04 §Feasibility Path).

---

## 10. Bottom line

Three independent tracks converge on one substrate-minimal object, reached protocol-down. The
provocation — the mutable memory address is the choice we've been patching, and a
content-addressed, capability-dispatched foundation is the opposite bet — is **coherent and
testable**. What is **proven** is substrate-*independence* (43 peers, thin floor). What is
**open** is substrate-*replacement* (Profile 4→5, entirely unbuilt, long horizon). The
irreducible physics bridge means the near-term goal is not to abolish the physical substrate
but to **meet it honestly and survive its failures** — a Profile-3, buildable posture that
also serves the app-hosting layer above.

*Coordination synthesis authored in the arch workspace at the operator's request; kept as a
separate track from the L5 app-hosting exploration per that call. The ratifiable outputs are
the keystone hand-offs (§9) absorbed via arch's proposal → ratify → fold, not this file.*
