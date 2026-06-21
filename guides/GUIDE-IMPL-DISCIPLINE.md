# Guide: Implementation Discipline

**Status**: Draft

**Source:** the Godot report-absorption proposal (Amendment D). Consolidates implementation-discipline patterns codified across the cross-impl ecosystem.

**Audience:** Any team building an implementation of the entity protocol — SDK, binding consumer, application, or substrate. The disciplines apply across roles.

---

## What This Guide Covers

Implementation disciplines that apply across implementation teams (not just SDK consumers). The protocol specs say what is true; this guide says what to *do* when building against them.

Differs from sibling guides:
- `GUIDE-ENTITY-WORKBENCH-APP.md` — app-archetype-specific (workbench applications).
- `GUIDE-EXTENSION-DEVELOPMENT.md` — extension-authoring-specific.
- `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` — peer-archetype framing.
- `GUIDE-PERSISTENCE.md` — persistence layer specifics.

This guide is general implementation discipline — code patterns, audit conventions, process rules — that apply to every impl team.

---

## 1. Overview — the discipline charter

The entity-protocol ecosystem builds OS-foundation substrate. Production-grade reliability requires more than spec conformance: it requires patterns of construction that prevent foreseeable failures, and patterns of audit that surface unforeseen ones. The disciplines below codify cross-impl-validated patterns from real failure modes that shipped (or nearly shipped) and were caught — turned into rules so the same class doesn't recur.

### 1.1 Discipline numbering and provenance

Disciplines D1–D8 were codified by the Godot frontend team during the substrate-sandwich reframe. D9, D10, D11 come from the foundation audit that surfaced a catastrophic phantom-host accumulation. D12 comes from the γ.4.2 misframing retrospective (the LLM-summary-confidence failure mode).

All twelve have been adopted ecosystem-wide. Workbench-go and egui-rust have mapped them to their own existing practices and cosigned.

### 1.2 Discipline catalog

| # | Name | Where treated | Provenance |
|---|---|---|---|
| D1 | Use the kernel — don't reinvent extensions | SDK-OPERATIONS §2.7.2 (kernel-vs-handler) | Godot |
| D2 | L1 dispatch default; L0 is the carve-out | SDK-OPERATIONS §2.7 (normative pin paragraph) | Godot |
| D3 | Capability-typed dispatch | (Godot-specific impl pattern) | Godot |
| D4 | Bounded interfaces — no global broadcast | (Per-impl UI discipline) | Godot |
| D5 | Declarative composition | (Godot-specific autoload pattern) | Godot |
| D6 | Per-host namespaces | (Godot-specific topology) | Godot |
| **D7** | **Kernel keeps working when applications misbehave** | §2 (stub — multi-session authoring) | Godot |
| **D8** | **Trust the spec; surface drift, don't normalize** | §3 (stub) | Godot |
| **D9** | **Memory Store Accounting (Pixel-Perfect-style)** | **§4 (authored)** | Godot (foundation audit) |
| **D10** | **Real-Session Coverage** | §5 (stub) | Godot |
| **D11** | **Inventory-Boundary Declaration** | §6 (stub) | Godot |
| **D12** | **Read canonical sources before designing against them** | §7 (stub) | cross-impl alignment cycle |

D1, D2, D5, D6 are SDK-OPERATIONS-blessed or impl-specific; they live in their respective documents. The remaining six (D7-D12) are universal cross-impl disciplines authored here.

### 1.3 Architecture-side mirror disciplines (A-D1..A-D6)

Implementation teams aren't the only ones with discipline gaps. The architecture team has a parallel set of disciplines codified to address the "we aligned but never followed up" pattern. These mirror impl-side disciplines on the trigger/notification/follow-up loop:

| # | Name | Mirrors impl-side |
|---|---|---|
| A-D1 | Per-extension absorption-status tracking | D8 (surface drift) — on the SDK absorption side |
| A-D2 | Session-start drift-check | D8 (cadence) — on the architecture-team side |
| A-D3 | Triage-on-extension-landing | D8 (cadence) — on the workstream-handoff side |
| A-D4 | Verify-source-not-synthesis | D12 (canonical sources) — on the review side |
| A-D5 | PARITY-MATRIX as standing artifact | (structural) — addresses the workflow trigger gap |
| A-D6 | Confidence-decay across summaries | D12 (LLM-summary hazard) — on the session-handoff side |

§9 (stub — multi-session authoring) covers these in detail.

### 1.4 Anti-pattern catalog

Twelve disciplines describe what to do. Five anti-patterns describe specific failure modes to recognize:

| # | Name | Class |
|---|---|---|
| 5.7 | Defined-but-uncalled cleanup primitive | D9-runtime |
| 5.8 | Borrowed framing | D12 (LLM-summary hazard) |
| 5.9 | Field-name pattern-match | D12 |
| 5.10 | Conceptual model projection | D12 |
| 5.11 | Bypass the ergonomic layer | D12 |

§10 (stub) covers each with source bugs cited.

---

## 2. D7 — Kernel keeps working when applications misbehave

*(Stub — multi-session authoring. Source: Godot REFRAME-EOS-DISCIPLINE §3 Discipline 7.)*

The cardinal seL4 / Fuchsia / Plan 9 invariant. An application crash, hang, or misbehavior never takes down the kernel, the dispatcher, the modal stack, or other applications' state.

Concrete invariants (Godot's worked enforcement):
- Panel crash → host stays alive.
- Modal that doesn't clean up → pop guaranteed by host teardown.
- Peer failure → workspace tree doesn't corrupt (system/revision rollback when configured).
- Misbehaving action → dispatcher catches, reports, doesn't re-throw.

Per-impl containment audit listing each crash/hang/misbehavior path and naming its protection is the canonical enforcement convention.

---

## 3. D8 — Trust the spec; surface drift, don't normalize it

*(Stub — multi-session authoring. Source: Godot REFRAME-EOS-DISCIPLINE §3 Discipline 8 + ecosystem promotion in the Godot frontend-feedback response.)*

Five-step process (Godot's worked form):
1. Cross-reference both `core-protocol-domain` and `sdk-domain` specs.
2. File in per-impl `QUESTIONS-FOR-ARCHITECTURE-{date}.md` durable tracking.
3. Annotate call site with question reference + repair path.
4. Package arch-team-facing extract under `docs/architecture/reviews/`.
5. Record working position + keep moving (filing ≠ blocking).

Architecture's paired RESPONSE convention: `RESPONSE-{IMPL}-{date}.md` under `entity-system-architecture/docs/architecture/v7.0-core-revision/reviews/`.

Standing convention for impl ↔ architecture interaction across the ecosystem.

---

## 4. D9 — Memory Store Accounting (Pixel-Perfect-style)

**Status: full normative treatment.**

**Source:** the Godot foundation audit (Phase 5 + Phase 7) — surfaced a catastrophic 28-phantom-host accumulation across boots that shipped despite 104/104 tests green.

**User framing (verbatim)**: *"every block of memory, every bit that goes through the system. Just like we do the Pixel Perfect accounting, we got to do the Memory Store accounting. Nothing accumulates, we know it and when it does we know why and it's because we chose it too."*

The principle generalizes across implementations. The discipline has two halves, because two layers of memory are typically the impl team's responsibility to account for.

### 4.1 D9-runtime — runtime allocation accounting

**Every allocation an implementation holds open while the process runs is accounted for at construction; every teardown removes what construction added.**

The specifics depend on the runtime model (Godot Node graph, Rust ownership/Drop, Go GC + explicit Close, browser-side GC, etc.), but the structural commitment is the same: at the point of construction, the teardown path is identified and committed to.

#### 4.1.1 The accounting categories (universal)

For each category, the impl identifies the construction point and the corresponding teardown path:

**1. Nodes / handles / refcounted resources.** Every `Node.add_child`, every `Rc::new`, every `gd::Gd::from_init_fn`, every `Box::leak` (rare), every `runtime.AddListener`. The teardown path is identified at construction (`queue_free`, drop-on-scope-exit, ownership transfer, explicit `Close()`).

**2. Signals / event subscriptions / callback registrations.** Every `signal.connect`, every `subscribe`, every `RegisterHandler`, every `OnX` registration. Either matched by explicit `disconnect`/`unsubscribe`, OR scoped such that the runtime's auto-cleanup applies (shared-tree-exit, Drop chain, etc.). Cross-component / cross-process connections always need explicit care.

**3. Timers / periodic tasks.** Every `Timer.new`, `time.Ticker`, `setInterval`, `tokio::spawn(periodic)`. Lifecycle bound to a documented owner; cancellation/stop path identified at creation.

**4. Buffers and caches in-process.** In-memory state in singletons / autoloads / managers. Each cache declares its bound: fixed-size, lifecycle-coupled to a documented event (e.g., per-window cached state with a `forget_X(id)` called from the X teardown path), or time-bounded with eviction policy.

**5. Subscriptions / futures / channels.** L1 subscription handles held implementation-side, in-flight async tasks, open channels. Drop semantics are runtime-specific (refcount, GC, explicit close). Single-owning-field-per-consumer is the rule; aliasing the handle to a longer-lived owner is the leak.

**6. Goroutines / threads / async tasks (where applicable).** Each spawned task has an exit condition reachable from the parent's teardown. `runtime.NumGoroutine()` monitoring is the canonical Go-side observability hook.

#### 4.1.2 The autoload-state addendum (per the Godot meta-pass)

When a singleton/manager caches per-X state, an explicit `forget_X(id)` is required AND must be called from the X teardown path. **Caching without an eviction hook is a leak by construction.**

This is the worked source of anti-pattern 5.7 (defined-but-uncalled cleanup primitive). The discipline is: not just defining `forget_X` (half-discipline), but wiring it AND verifying production call-sites at PR time.

#### 4.1.3 Anti-patterns (each with a source bug from real audits)

- **Add without paired remove** — every `tree_put`, `signal.connect`, `Node.add_child`, `Timer.new+add_child` lands with the matching teardown identified at the same change. Source: Godot's B7 + B3 bugs.
- **Latent infrastructure rots** — when infrastructure is added to support a use case, the use case must ship a test exercising it in the same commit. Source: Godot's `peer_data_dir` parameter unused for months. The infrastructure looks correct in isolation; nothing catches that it's unused.

### 4.2 D9-persistence — entity-tree write accounting

**Every entity an implementation writes into the entity tree has a documented writer / reader-at-boot / GC story OR an explicit exemption with rationale.**

This half of D9 addresses a real architecture-team gap: the entity tree itself does NOT auto-GC what implementations write. Kernel-side GC is on the production-readiness roadmap but not yet specified. **Until that lands, every impl is responsible for cleanup of its own writes.** If an impl doesn't `tree_remove`, its writes live forever.

#### 4.2.1 Per-store accounting requirements

For each persistent location the impl writes, the discipline requires the following documented:

**1. Writer.** A single owner on the impl side. Multiple writers to the same path is a divergence hazard (race conditions, last-write-wins, divergent state).

**2. Reader at boot.** The code path that reads at startup and what it does with the result (instantiate from? populate cache? validate? migrate?). If nothing reads at boot, the entity is write-only — likely a candidate for either removal or repurposing.

**3. GC / cleanup story.** When does an entry leave? Options:
- **Bounded.** Fixed-size (e.g., LRU cache of N entries); cleanup is structural.
- **Lifecycle-coupled.** Removed when X happens (peer disconnect, window close, session end). The teardown path explicitly removes.
- **Reference-counted.** Only removed when no remaining reference. Requires the refcounting infrastructure.
- **Manual.** Operator action removes. Document this — operators need a procedure.

**4. Explicit exemption.** If an entry intentionally lives forever (e.g., identity bundle, configuration root), document the rationale and the size-bound assumption.

#### 4.2.2 Cross-impl shape (per workbench-go's contribution)

Workbench-go's `DEPLOYMENT-DIRECTION.md` documents writer/reader per entity-tree namespace. The shape is reusable:

| Namespace | Writer | Reader at boot | GC story | Exemption rationale |
|---|---|---|---|---|
| `app/{app-id}/workspace/...` | the app's persistence layer | restore-windows-on-boot | lifecycle-coupled (close window → remove state) | — |
| `system/peer/transport/{peer_id}` | connection establishment | reconnect on boot | manual (operator removes stale entries) | known persistent network roster |
| `system/identity/...` | identity bootstrap | identity restoration | exempt — identity roots persist | structural identity stack |

Each impl publishes its own version of this table. **The table itself is the discipline-enforcement artifact** — gaps in the table are gaps in the accounting.

#### 4.2.3 Anti-pattern 5.7 — Defined-but-uncalled cleanup primitive

**Pattern**: a cleanup primitive (`forget_X`, `remove_handler`, `unsubscribe`, `tree_remove_X`) is defined, with the intent of being called from a teardown path. The teardown path is never wired. The primitive sits dormant; the leak ships.

**Provenance**: appeared three times in Godot's codebase pre-audit (M1, N1, N2). Workbench-go owes a sweep of `Close()`/`Cancel()`/`Unsubscribe()` call-sites per anti-pattern 5.7.

**Detection**: PR-time grep for cleanup primitives that have no caller. Languages with dead-code lint catch the easy case; the hard case is the primitive that's "almost called" (defined in path A, called from path B, but the missing path C is where the leak lives).

**Cross-impl PR-time grep convention** (proposed for adoption):

```
# Pseudocode — adapt per language
grep -r 'fn forget_\|fn remove_\|fn unsubscribe_\|fn close_' src/ | \
  while read fn; do
    name=$(extract function name from fn)
    callers=$(grep -r "$name(" src/ | grep -v "fn $name")
    if [ -z "$callers" ]; then echo "ORPHAN: $fn"; fi
  done
```

Per-impl analog. Godot's form is documented in the Godot foundation audit (Phase 5).

### 4.3 D9 enforcement workflow

The discipline operates at three points:

**1. Construction-time discipline.** At the point of writing the `Add` / `Subscribe` / `tree_put`, the teardown path is identified and committed to in the same change. This is the "every block of memory" half of the user framing.

**2. PR-time audit.** Reviewers look for the construction → teardown pair. A construction without identifiable teardown is a PR blocker, not a follow-on.

**3. Foundation audit cycle.** Periodic (per-release or per-major-version) full audit:
- D9-runtime: walk every singleton/manager; verify each entry has a teardown caller AND an audit test exercising the teardown.
- D9-persistence: refresh the writer/reader/GC table; verify every tree-write has each documented.
- 5.7 sweep: grep for orphan cleanup primitives.

**The foundation audit cycle is the load-bearing one** — it catches what construction-time discipline missed AND surfaces patterns that need new discipline. Source bugs are then added to the anti-pattern catalog.

### 4.4 D9 and the Architecture team

D9-persistence has a load-bearing arch-team dependency: until kernel-side GC is specified and implemented, **every impl is responsible for its own cleanup.** This is acknowledged in the production-readiness roadmap. When the kernel-side GC contract lands, the D9-persistence story shifts from "impl cleanup is the only cleanup" to "impl cleanup is the first-line; kernel GC is the safety net."

Until then, D9-persistence is a hard discipline.

### 4.5 Cross-references

- Godot REFRAME-EOS-DISCIPLINE §3 Discipline 9 — the worked Godot form.
- The Godot foundation audit (Phase 5 + Phase 7) — the surfaced bugs that motivated D9.
- `entity-workbench-go/perfreview/` — `DroppedDeliveries()` counter; subscription tracking; channel lifecycle. Workbench-go's existing form maps to D9-runtime.
- The architecture-side kernel-GC production-readiness roadmap.

---

## 5. D10 — Real-Session Coverage

*(Stub — multi-session authoring. Source: Godot REFRAME-EOS-DISCIPLINE §3 Discipline 10.)*

Cross-boot stability + headed-only paths + real-store inspection. Universal test discipline.

The principle generalizes; the enforcement is per-impl (Godot's `HEADED=1` gates / `UI.boot_main_scene` / `viewport.push_input`; workbench-go's `perfreview/` build-tag probe pattern). Tests must verify what survives a real boot, not just synthetic harness state.

Workbench-go's `perfreview/` is the canonical reference shape (per arch promotion decision; see Amendment D promotion-candidate decisions). Per-impl: workbench-go publishes the build-tag pattern; egui-rust and Godot port to their respective runtime models.

---

## 6. D11 — Inventory-Boundary Declaration (meta)

*(Stub — multi-session authoring. Source: Godot REFRAME-EOS-DISCIPLINE §3 Discipline 11.)*

At audit open: declare what's in scope AND what's not.

Surfaced from a real meta-audit failure where inventory-driven Phase 5 fixes missed in-memory caches outside the entity-store inventory. The discipline is cheap and high-leverage: if the boundary isn't named, it isn't covered.

---

## 7. D12 — Read canonical sources before designing against them

*(Stub — multi-session authoring. Source: cross-impl alignment cycle.)*

The LLM-summary-confidence failure mode is real. Session handoffs preserve confidence but don't preserve verification. Each summary-of-summary erodes the underlying source-grounding. The class of error: pattern-matching from prior mental models (git for revision; CRUD for query; "just dispatch" for new extensions) without consulting the canonical PROPOSAL/EXTENSION text.

Mechanical rule:
- Before filing a binding request, consult the PARITY-MATRIX. If the ergonomic layer exists, request the wrapper. If absent, request the build-out across layers.
- Before designing against an extension whose proposal was recently absorbed, re-read the PROPOSAL-*.md to catch reshapings.
- L5-direct-`execute("system/X", ...)` is an anti-pattern that bypasses the spec-encoded shape carried by the ergonomic layer.

---

## 8. D13 — Namespace-match diagnostic at publish boundary

**Rule (SHOULD, not MUST):** When an SDK's publish operation accepts both a target store and a publisher identity (keypair / peer-id), and the publisher's identity does not match the store's canonical peer-id, the SDK SHOULD surface a clear diagnostic to the caller BEFORE proceeding. Silently using the publisher's identity against a foreign store (or silently using the store's identity ignoring the publisher's) produces wrong-output without error — the worst failure mode.

**Provenance:** workbench-go corridor-close hit this with `entity-publish` invoked against a store written by another peer's id; publish silently returned bootstrap entries instead of intended content. Fixed empirically with explicit `-keypair PATH` flag.

**Discipline applied:**

1. SDK MUST verify the store's canonical peer-id at publish-init.
2. If publisher's identity (from keypair / env / flag) does not match, SDK SHOULD:
   - Print or return a diagnostic naming both peer-ids.
   - Either prompt for confirmation, require an explicit `--force-cross-namespace` flag, or refuse to proceed (impl chooses).
3. Default: refuse-to-proceed for write operations; warn-and-proceed for read operations.

**Substrate position:** NOT a protocol MUST. The protocol's namespace canonicalization (V7 §1.5) is correct; the ergonomic gap is SDK-level. We do not add substrate guard-rails for SDK ergonomic bugs.

**Cross-references:** the workbench-go corridor-close review; Round-6 ratification per the CDN-corridor Round-6 ratifications proposal (item #5).

---

## 9. Cross-impl process

*(Stub — multi-session authoring.)*

D8 intake cadence + `docs/architecture/reviews/` convention + paired RESPONSE / ACK pattern. The standing convention for impl ↔ architecture interaction, formalized at the Godot round-trip-1 sign-off.

---

## 10. Architecture-side mirror disciplines (A-D1..A-D6)

*(Stub — multi-session authoring. Source: cross-impl alignment cycle §3.)*

Six disciplines parallel to impl-side D7-D12, addressing the architecture team's own failure modes (the "we aligned but never followed up" pattern). Workbench-go gave concrete teeth in their feedback memo:
- A-D1: header line on every `EXTENSION-*.md` ("Absorbed in SDK-EXTENSION-OPERATIONS: v__ (date); next absorption owed at v__")
- A-D3: landing artifacts MUST list "Absorption owed in `SDK-EXTENSION-OPERATIONS.md §X`" + "Cross-impl impact" row per impl naming who-owns-what + PARITY-MATRIX row update.

A-D4 (verify-source-not-synthesis) and A-D5 (PARITY-MATRIX as standing artifact) are the highest-leverage architecture-side disciplines.

---

## 11. Anti-pattern catalog

*(Stub — multi-session authoring. Source: cross-impl alignment cycle.)*

Five named anti-patterns: 5.7 (defined-but-uncalled cleanup primitive), 5.8 (borrowed framing), 5.9 (field-name pattern-match), 5.10 (conceptual model projection), 5.11 (bypass the ergonomic layer).

§4.2.3 has the full treatment of 5.7. The other four get treatments in the multi-session expansion.

---

## 12. References

Internal:
- The Godot report-absorption proposal (Amendment D) — the proposal that authored this guide
- The Godot frontend-report response (§5) — the D9/D10/D11 promotion verdicts
- The workbench-go cross-impl-alignment feedback — adoption mappings
- The egui-rust review of the Godot report-absorption proposal
- `CROSS-IMPL-PARITY-MATRIX.md` — the load-bearing structural artifact
- The outer-limits status board — items O-21 / O-22 / O-23 / O-24 / O-25

External:
- The Godot REFRAME-EOS-DISCIPLINE charter — the discipline charter Godot codified
- The Godot foundation audit — the D9 source-bug catalog
- `entity-workbench-go/perfreview/` — D10 reference shape
- The workbench-go cross-impl-alignment feedback (§4) — discipline-mapping table
