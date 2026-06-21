# Emit Pipeline Implementation Guide

**Status**: Active
**Audience:** Peer implementers (Go, Rust, Python, Godot) — anyone wiring up Phase 1 and Phase 2 plumbing inside a peer. Assumes you've read V7 §6.10 and SYSTEM-COMPOSITION §1–§3.
**Spec reference:** `ENTITY-CORE-PROTOCOL.md` §6.10, §2870 (binding-level atomicity), §2688 (transient I/O failures), `SYSTEM-COMPOSITION.md` §1.3 (Phase 1 / Phase 2), §2.7 (cascade-halt), §3.2 (cascade-depth thresholds), `EXTENSION-SUBSCRIPTION.md` §5.5–§5.6.
**Architectural reference:** `explorations/EXPLORATION-EMIT-DURABILITY-AND-DELIVERY.md` — the "why" behind these recommendations; cross-references the convergence-class framework, the sub-peer pattern, and the boundaries between mathematical / spec / implementation / tooling / user-pattern.
**Related guides:** GUIDE-OPERATIONAL-STATE.md (operational state in the tree; this guide covers operational state for the emit pipeline specifically).

---

## 1. Terminology — aligning vocabulary across teams

Cross-impl conversations have been confused by the same words meaning different things to spec, implementation, and operations audiences. Use these terms consistently:

| Term | Definition | Where it lives |
|---|---|---|
| **Phase 1 consumer** (spec) | A synchronous hook running inline in the writer's call stack as part of `tree.put`. Halts the cascade on non-200 return. Position 0 (persistence) through position 8 (subscription dispatch). | SYSTEM-COMPOSITION §1.3, §2.2 |
| **Phase 2 consumer** (spec) | An asynchronous observer downstream of the post-commit broadcast. Fires after Phase 1 settles. Log-and-continue semantics; cannot propagate errors to the originating `tree.put` caller. | SYSTEM-COMPOSITION §1.3, §2.5 |
| **Sink** (implementation jargon, Go-specific) | A single Phase 2 consumer's input channel. The Go impl supports N sinks, each with its own buffer and drop counter. In other impls this may be called "subscriber," "listener," "observer," or "callback." | Go impl: `core/peer/fanout.go` |
| **Fan-out** (impl) | The mechanism that delivers a single broadcast event to all registered Phase 2 consumers. | Implementation-defined |
| **Saturation** (impl) | A buffer between phases (or between fan-out and a sink) is full and cannot accept more events. | Implementation-defined |

**Translation rule:** when an impl says "per-sink isolation," what they mean in spec terms is "each Phase 2 consumer has its own bounded buffer; saturation at one consumer doesn't backpressure into Phase 1 or block other Phase 2 consumers." The spec doesn't say "sinks" because the spec doesn't prescribe how Phase 2 multiplexes to multiple consumers — that's an implementation choice. The spec just says Phase 2 fires once per write.

---

## 2. Implementation guidance (non-normative)

The following are SHOULD-level recommendations. Implementations may diverge if their workload or backend warrants. Each recommendation cites a problem it addresses and an architectural reference where applicable.

### 2.1 Phase 1 and Phase 2 must have distinct plumbing

**Recommendation:** Phase 1 consumers MUST run in the writer's call stack (inline, synchronous). Phase 2 consumers MUST be fed from a buffer that the writer does not block on. The two phases SHOULD NOT share a channel or a fan-out reader goroutine.

**Why:** Tangling Phase 1 and Phase 2 plumbing is what produced the four-phase incident. The Go impl had a single `events` channel and a single fanout reader; a slow Phase 2 consumer backpressured into the Phase 1 path. Errors-at-saturation-boundaries (the workbench team's correction) restores the structural separation.

**Architecture reference:** `explorations/EXPLORATION-EMIT-DURABILITY-AND-DELIVERY.md` Option A.

### 2.2 Reads must not block on writes

**Recommendation:** Storage backends that have a single-writer constraint (SQLite, bbolt) SHOULD split read and write connection pools. Reads SHOULD use a separate connection pool sized for parallelism; writes SHOULD use a single-connection writer.

**Why:** V7 §2870 contracts per-binding atomicity. Content store entries are immutable. At the protocol level there is no reason a reader from any path should wait for a writer to a different path. The Go impl's initial `MaxOpenConns=1` (chosen to dodge SQLITE_BUSY) serialized reads behind writes — a violation of the protocol's "reads are parallel-safe" property. The fix is a separate read pool in `_mode=ro`.

**SQLite-specific recipe:**
- Writer: one `*sql.DB` (or equivalent) with `MaxOpenConns=1`, `busy_timeout=5000ms`, `journal_mode=WAL`, `synchronous=NORMAL`.
- Reader: one `*sql.DB` with `MaxOpenConns=N` (typically 4–16) and `_mode=ro`.
- Route `Get`/`Has`/`List`/`Len` to reader; `Put`/`Set`/`Remove`/`CompareAndSwap` to writer.

**Other backends:**
- bbolt: has the same single-writer constraint; apply the same split.
- Badger / Pebble / RocksDB: LSM-based with internal pipeline; per-key concurrency available without pool splitting, but the operational property to preserve is the same.
- Memory: no constraint; reads and writes are naturally concurrent.

**Architecture reference:** Exploration §7 (Contention is implementation choice), Option F.

### 2.3 Phase 2 saturation MAY signal to the caller, not just drop silently

**Recommendation:** When the Phase 2 broadcast buffer is full and the implementation must choose between dropping the event and blocking the writer, neither is acceptable as the only option. Implementations SHOULD expose saturation as either:
- A non-fatal error returned from `tree.put` (signaling the binding committed but the event was not delivered), or
- An observable counter / metric that operators can monitor.

**Why:** Silent drop violates Phase 1's convergence promise (consumers expect to see every post-commit transition). Indefinite blocking lets a single slow consumer halt the entire peer.

**Spec status:** the spec does not currently have language for this signaling. The Go impl invented `ErrEventBufferFull` and propagates it through `Set`'s return. This is **spec-permitted, not spec-required** — other impls may handle saturation differently (Rust's `tokio::mpsc::try_send` returns error naturally; Python `asyncio.Queue.put_nowait` raises). Cross-impl alignment on whether to standardize the signal is open; see §5 below.

**Architecture reference:** Exploration §19 (Phase 1 is the contention; Phase 2 just goes), §10 Option A.

### 2.4 Multiple Phase 2 consumers SHOULD be isolated from each other

**Recommendation:** When the implementation supports multiple Phase 2 consumers (e.g., the subscription engine, a reverse-write daemon, an externally-exposed channel for application observers), each consumer SHOULD have its own bounded buffer. A slow or un-drained consumer SHOULD fill its own buffer and drop independently rather than blocking other consumers.

**Why:** Without per-consumer isolation, the slowest consumer becomes the throughput ceiling for all consumers. In the workbench scenario this manifested as a UI consumer (tview update queue) slowing every upstream write.

**Architecture reference:** Exploration §22 (Sub-peer pattern asymmetric coupling) — generalizes the principle to "no single component can stall the system."

### 2.5 Phase 1 with no consumers MUST be cheap

**Recommendation:** When a peer has no Phase 1 consumers registered (or no consumers for a particular event type), `tree.put` SHOULD have effectively zero overhead from the emit pathway — no channel allocations, no goroutine spawns, no per-call dispatch lookup that runs whether or not consumers exist.

**Why:** SYSTEM-COMPOSITION §2.4 already says "a peer implementing only the core protocol has zero synchronous consumers." Implementations should reflect this — costs that aren't tied to actual consumer work should be paid for by registration, not by every write. The workbench's high-write workloads were largely against peers with sparse Phase 1 wiring; the per-write overhead should have been near-zero but was not.

**Implementation pattern:** check `len(consumers) == 0` at the top of the emit dispatcher; skip straight to return.

**Architecture reference:** Exploration §15.2.

### 2.6 Bootstrap-phase emit suppression

**Recommendation:** Implementations that bind system entities (types, handlers, grants, identity) during peer construction SHOULD suppress emit during the bootstrap window. Construction-time bindings are not application events and should not flow through the consumer pipeline.

**Why:** The Phase 2 buffer (or any wired consumer) is generally not ready during construction. Construction-time emits either fill the buffer with non-event data (wasting capacity) or fire saturation signals before the peer has even started.

**Implementation patterns:**
- Toggle an `emit_suppressed` flag during construction, clear it before the peer accepts external requests (Go's `SetEmitSuppressed`).
- Defer consumer wiring until after construction; emit fires into a no-op until the first consumer registers.
- Replay-on-attach: buffer construction events and replay them to the first registered consumer.

Implementations may choose; none is normative. The Go impl chose the first (flag-based suppression) after an earlier seed-drainer-goroutine approach proved race-prone.

### 2.7 Operational state for the emit pipeline lives outside the cascade

**Recommendation:** Operational state about the emit pipeline itself — drop counts, queue depths, consumer saturation, cascade-halt history — SHOULD NOT be written through the same emit pipeline. Recording "we dropped event X" via `tree.put` re-enters the pipeline that dropped X.

**Implementation patterns:**
- **Operational sub-peer (recommended):** pair the feature-rich peer with a core-protocol-only peer that holds operational records. The sub-peer has no extensions, no cascade, no cycles. See exploration §14 / §22 for the full pattern.
- **Sibling table:** record operational state in a separate storage table the emit pathway doesn't touch.
- **In-memory only:** acceptable for development; lost on restart.

**Architecture reference:** Exploration §14, §22.

### 2.8 UI consumers and in-process observers MUST subscribe by prefix, not consume raw L0 events

**Recommendation:** Any consumer with a stable interest in a subtree (UI panel, application-level observer, in-process bridge) MUST subscribe via pattern-scoped subscription (`OnPrefixChange` or equivalent). The raw L0 tree-event stream is reserved for fan-out infrastructure inside the SDK itself (the watch hub plumbing, the fanout dispatcher) — NOT for application code.

**Why:** This is the textbook case for SYSTEM-COMPOSITION §1.3's Phase 2 consumer model. Each panel is a Phase 2 consumer; the protocol provides pattern-scoped delivery so each consumer sees only its relevant events. Treating the raw L0 stream as the application entry point inverts this — consumers observe every event then filter in-process, paying O(N) work per event for events they don't care about.

The Go workbench accreted this anti-pattern over six months. With ~14K paths in a production-shape store, every UI refresh cost ~22ms of main-thread work because every panel scanned the whole tree and filtered for its own prefix. The fix took half a day once diagnosed. The diagnosis took weeks because the symptom looked like storage performance.

**The anti-pattern, recognizing it:**
- A shared "current peer state" cache that holds a snapshot of all entity paths
- Panels that read the shared cache + filter for their own prefix
- A single refresh tick that fires for every tree event and iterates all visible panels
- Status displays / aggregates that count or list all entities on every refresh
- Inspector-style panels re-fetching the currently-selected entity on every tick rather than on path-or-content change
- Tree-browser panels rebuilding from a full enumeration on every refresh

If panels never appear in the subscription engine's pattern registry, that's the smoking gun.

**The correct shape:** each panel owns a prefix subscription, maintains its own local view-state, updates incrementally via event handlers, renders from local state. No shared cache, no full-tree scans, no per-event work for events the panel doesn't care about.

**Recipe (any language):**

```
Panel construction:
  1. Set local view-state to empty.
  2. Subscribe to prefix via OnPrefixChange(prefix, onEvent).
     SDK seeds with synthetic ChangePut for existing paths,
     then drains live events to onEvent.
  3. Cache the cancel function for teardown.

onEvent(ev):
  - ChangePut(path, hash): fetch entity, decode, set
    localState[path] = decoded. Schedule redraw.
  - ChangeRemove(path): drop localState[path]. Schedule redraw.
  Both cases: IDEMPOTENT. Same event twice = same result.

Panel render():
  - Read localState (mutex/lock-protected).
  - Build UI from local state. NO store reads here.

Panel close():
  - Call the cached cancel function.
```

**For panels displaying a single selected entity** (inspector, markdown-view): TWO subscriptions are needed — one on the selection slot (so the panel knows when the user picks a different path) and one on the currently-selected content path (so it re-renders when the entity changes). The content watch is re-bound on every selection change.

**Architecture reference:** the workbench-go cross-impl UI-patterns feedback (workbench origin memo, with concrete numbers). Exploration §17 (composition introduces this regime; the system's value depends on getting it right).

### 2.9 `LenPrefix(prefix) → int` is a required interface method

**Recommendation:** The location-index interface MUST expose a prefix-scoped count operation that does not materialize entries. Status displays, count-only consumers, and aggregate UI elements call this on every refresh tick; implementing it as `len(List(prefix))` makes those displays an O(N) blocker on every render.

**Contract:**
- MUST NOT materialize entries (no `len(List(...))` stub).
- SQL backends: indexed `COUNT(*)` over the path range. O(log N + matches).
- Memory backends: map walk is acceptable (N is bounded for in-memory use); MAY use a maintained counter for the empty-prefix hot case.
- Empty prefix counts everything in the index.
- Conformance tests cover: all-prefix, scoped-prefix, empty-store, after-remove. Both backends must report identical counts for identical inputs.

**Why:** UI status displays (entity count in status line, panel headers, etc.) call this on every heartbeat. With `len(List(""))` on a 14K-path store, status-line refresh was 16ms. With `COUNT(*)` it's 2.9ms. Without this primitive, every implementation will discover the issue independently and re-implement it; standardizing the name + contract lets the convention be portable.

**Go reference:** `entity-core-go/core/store/store.go` (interface), `memory.go` and `sqlite.go` (implementations), `conformance_test.go` (4 conformance cases verify behavioral parity between backends).

### 2.10 `OnPrefixChange(prefix, handler) → cancel` as the canonical SDK helper

**Recommendation:** SDKs SHOULD expose `OnPrefixChange(prefix, handler) → cancel` as the canonical helper for "subscribe to all changes under a prefix." The shape:

- Wraps the equivalent of `Store.Watch(prefix + "*")` with seed enumeration in an SDK-owned goroutine.
- Handler invoked per matching event with `(path, hash)` or the decoded entity, per language ergonomics.
- Handler MUST be idempotent — "set my view of path P to state at hash H," not "process delta." The watch hub may deliver duplicates after seed (events buffered before subscribe-time get replayed).
- Cancel is idempotent and waits for in-flight delivery to complete.

**Why:** This is the primitive UI panels (§2.8) consume. Naming it consistently across SDKs makes panel code portable in shape, even if implementation languages differ.

**Architecture reference:** workbench-go has the prototype in `workspace_state.go`; generalizing it to the SDK is a small wrapper around the existing watch hub.

### 2.11 Observability surface (minimum suggested)

**Recommendation:** Implementations SHOULD expose, via implementation-defined channels:
- Per-Phase-2-consumer drop counts (with reason labels if available).
- Per-Phase-2-consumer queue depth (gauge) and capacity (constant).
- Per-`tree.put` latency histogram (median + p99 sufficient).
- Cascade-depth distribution (histogram) — leading indicator of approaching depth thresholds.
- Saturation count for the Phase 2 broadcast buffer.

Format is implementation-defined. The Go impl exposes `Peer.Stats()` as a Go-API surface; others may prefer entity-as-telemetry (recorded in an operational sub-peer per §2.7) or Prometheus / OpenTelemetry metrics.

**Why:** The four-phase incident was difficult to diagnose precisely because the implementation had a drop counter that was gated on debug-log being enabled. Without operational visibility, every drop-shaped bug is invisible.

**Architecture reference:** Exploration §25 (the missing layer — analyzer / observability surface), §10 Option E.

---

## 3. Common anti-patterns to avoid

Pulled from the four-phase incident. These are bugs in any reasonable spec interpretation:

| Anti-pattern | Why it's wrong | Correct shape |
|---|---|---|
| `select { case ch <- evt: default: drop }` without a counter | Silent loss; violates Phase 1's convergence promise | Drop with a counter; or surface saturation as error to caller |
| `Set` returns `void` | V7 §2688 normatively requires error propagation | Return error |
| Watcher / background goroutine `continue`-ing past `Put` error | Discards an error the caller is expected to handle | Log + abort, or retry with backoff, or surface to operator |
| Single events channel feeding all Phase 2 consumers via a single fan-out reader | One slow consumer stalls all others | Per-Phase-2-consumer buffer + isolation |
| Phase 2 buffer sized for steady state, not bursts | Saturates on bootstrap, ingest, recovery | Size for expected burst + observability + shed policy |
| Subscribing to `**` (everything) for application data | Captures `system/history/*`, `system/clock/*`, `system/inbox/*` operational writes; amplifies own cascade | Narrow subscription scope to application paths only |
| Writing operational state (drops, failures) through the same `tree.put` that produced the drop | Recursive failure | Use operational sub-peer or sibling storage |
| UI panel reads "all entities of this peer" + filters at the panel | O(N) work per event for events the panel doesn't care about; main-thread blocker at scale | Panel subscribes to its own prefix via `OnPrefixChange`; maintains local view-state |
| Status display uses `len(List(""))` to count entities | O(N) materialization on every refresh tick | Use `LenPrefix("")` (indexed COUNT) — §2.9 |
| Inspector / detail panel re-fetches selected entity on every refresh tick | Get+decode per tick regardless of whether content changed | Watch the currently-selected path; re-render on event only |
| Type-asserting the concrete storage backend (`if sqlite, ok := li.(*SqliteLocationIndex); ok`) | Backend leakage; defeats the abstraction; blocks alternative backends | If a feature needs backend support, add it to the interface + conformance suite, don't reach through |

---

## 4. Cross-implementation status

Tracking what each implementation has landed and where decisions diverge. Keep this updated as cross-impl work progresses.

### 4.1 entity-core-go (reference implementation)

| Area | Status | Notes |
|---|---|---|
| `Set` returns `error` | ✅ Landed (four-phase incident, Phase 2) | Conforms to V7 §2688 |
| Distinct Phase 1 / Phase 2 plumbing | ✅ Landed (followup) | `runCascade` vs `emit + fanOut` |
| Per-Phase-2-consumer isolation | ✅ Landed | `core/peer/fanout.go::fanOut` |
| `ErrEventBufferFull` propagation | ✅ Landed | Sentinel error from `Set` |
| Bootstrap-phase suppression | ✅ Landed | `SetEmitSuppressed` flag |
| `Peer.Stats()` API | ✅ Landed | `EventBufferDrops` + `FanOutSinks` per-consumer |
| SQLite read/write pool split | ✅ Landed | `core/store/sqlite.go` commit 9af9bea; reader p99 = 130µs under cross-peer write pressure |
| `LocationIndex.LenPrefix(prefix) → int` interface method | ✅ Landed | Memory + SQLite + wrappers; 4 conformance cases; status line refresh 16ms → 2.9ms |
| Cross-peer deadlock integration test | ⏳ Recommended, not yet landed | "Currently passes by luck" — Step 2 of CONCURRENCY-CONSOLIDATION-PLAN |
| `OnPrefixChange` SDK helper | ⏳ Recommended (medium priority) | Workbench prototype exists; canonical name to land in entitysdk next round |
| Bounded inbox/deliver-to goroutine pool | ⏳ Latent; unbounded today | 1-day estimate |
| Watcher pending-map cap | ⏳ Latent; unbounded today | 1-hour fix |
| Operational sub-peer pattern | 🔍 Architectural option, not implemented | See exploration §14 |

### 4.2 entity-core-rust

| Area | Status | Notes |
|---|---|---|
| Phase 1 / Phase 2 separation | 🔍 Cross-impl review pending | Needs audit |
| Saturation handling | 🔍 Likely uses `tokio::mpsc::try_send` | Convergent with Go's `ErrEventBufferFull` shape |
| Storage concurrency | 🔍 Cross-impl review pending | If using SQLite, same pool-split recommendation applies |
| Operational observability | 🔍 Cross-impl review pending | |

### 4.3 egui-entity-core-rust (workbench-side Rust)

| Area | Status | Notes |
|---|---|---|
| Phase 1 / Phase 2 separation | 🔍 Not yet at this load; review pending | |
| Saturation handling | 🔍 Pending | |

### 4.4 godot-entity-core-rust

| Area | Status | Notes |
|---|---|---|
| All areas | 🔍 Cross-impl review pending | Godot's signal system has its own delivery model; needs translation |

### 4.5 entity-core-py

| Area | Status | Notes |
|---|---|---|
| All areas | 🔍 Cross-impl review pending | `asyncio.Queue.put_nowait` raises on full; aligns with errors-at-boundaries shape |

---

## 5. Open cross-impl coordination questions

For discussion when cross-impl review resumes. The exploration doc raises these architecturally; this guide tracks them as implementation decisions.

### 5.1 Standardize a saturation signal across impls?

The Go impl returns `ErrEventBufferFull` from `Set`. Rust's `try_send` returns `TrySendError::Full`. Python's `put_nowait` raises `asyncio.QueueFull`. Godot's signal system has different semantics entirely.

**Question:** do we want cross-impl alignment on the *shape* of the saturation signal (e.g., a documented sentinel error type that maps to each language's idiom)? Or is "each impl signals saturation in its native style" sufficient?

**Tentative answer:** native style is fine for now; the *behavior* should converge (binding commits, signal surfaces, caller decides) even if the type name differs. Revisit if cross-impl SDK consumers complain.

### 5.2 Standardize operational sub-peer pattern?

The exploration §14 proposes that feature-rich peers pair with an operational sub-peer. Implementation is straightforward; the pattern is portable.

**Question:** should the cross-impl SDK guide recommend this as the standard pattern for production deployments? Or leave it as "Go-impl experiment" until proven at scale?

**Tentative answer:** recommend it once one impl has it in production. Hold off on cross-impl prescription until then.

### 5.3 Should Phase 2 enqueue saturation be normatively defined?

Currently silent in spec. The exploration §11 proposes amendment text. The minimal coherent answer (§10) doesn't adopt it.

**Question:** do we wait until cross-impl divergence forces convergence, or proactively standardize?

**Tentative answer:** wait. Each impl handles saturation in its native style; we standardize when it becomes a portability problem.

### 5.4 Storage concurrency posture

Single-file SQLite is the Go-impl default. Rust and Python may pick differently (LMDB, Badger, sled, postgres). The implications for read/write isolation differ per backend.

**Question:** does the implementation guide need per-backend recipes, or is the general principle ("reads must not block writes") sufficient?

**Tentative answer:** general principle for the guide; per-backend recipes if and when implementers ask.

---

## 6. Status of the four-phase incident response

| Phase | Status | Where the fix lives | Notes |
|---|---|---|---|
| Phase 1: SQLite production setup | ✅ Landed | core-go `core/store/sqlite.go` | DSN pragmas, MaxOpenConns=1, options struct |
| Phase 2: `Set` returns error | ✅ Landed | core-go many call sites | Watcher / compute / history-recorder etc. all propagate |
| Phase 3 (incorrect): blocking emit | ⚠️ Superseded | — | Replaced by Phase 3-corrected |
| Phase 3-corrected: `ErrEventBufferFull` + per-consumer isolation | ✅ Landed | core-go `core/store/notifying.go`, `core/peer/fanout.go` | |
| Phase 4 (workbench-side): drainers + coalescing | ✅ Landed | workbench `shell/app.go`, `console/main.go` | Belt-and-suspenders; per-consumer isolation makes them redundant for correctness but they're cheap |
| Phase 5 (proposed): read/write pool split | ⏳ Recommended | core-go `core/store/sqlite.go` | Half-day; addresses actual contention |
| Phase 6 (proposed): cross-peer deadlock integration test | ⏳ Recommended | core-go test suite | Validates §22 cycle topology under load |

---

## 7. Performance baseline (Go reference impl, post-incident)

Established by the Go reference implementation after all four-phase-incident changes landed. These are the numbers cross-impl teams should aim to match or beat. All numbers from **production-mode binaries** (no race detector — see §8).

### 7.1 Per-operation cost

| Metric | Value | Context |
|---|---|---|
| Per-Put SDK overhead, sqlite_file backend | **166µs** | Full extension stack wired; storage Set/Put ~40µs of that; WAL fsync ~10µs of that |
| Cross-peer revision sync, 1000 entities | **522ms** (≈520µs per binding end-to-end) | Includes serialization, signature verification, network, ingest cascade |
| Reader p99 latency, under continuous cross-peer write pressure | **130µs** | Requires read/write SQLite pool split (§2.2); without it, p99 is dominated by write latency |
| `SqliteConcurrentReadDuringWrite` test (4 writers × 200 writes each) | **avg 22.7µs / max 810µs** read latency | Verifies the pool split |

### 7.2 UI panel costs (workbench reference, 14,225-path production snapshot)

| Metric | Before refactor | After refactor | Speedup |
|---|---|---|---|
| Per-refresh full-tree scan (`store.List("")`) | 18-40ms | gone | — |
| Per-refresh panel filter pass (6 panels) | ~3ms | gone | — |
| Tree-browser refresh, no event | ~10ms | **406ns** | ~25,000× |
| `PathCount()` (status line, per heartbeat) | 16ms | **2.9ms** | 6× |
| Inspector + markdown-view per refresh | ~50µs Get+decode | 0 (event-driven) | — |
| Refresh tick wall-time, populated store | ~22ms | sub-ms | >50× |

The "after refactor" numbers are achievable only with §2.8 (panel-prefix subscription) and §2.9 (`LenPrefix`). Without those patterns, the refresh-tick cost grows linearly with corpus size and the workbench becomes unusable past a few thousand paths.

### 7.3 Implications for cross-impl

- If an implementation's per-Put overhead is much greater than ~200µs with full extensions wired, audit the extension stack and Phase 1 plumbing.
- If an implementation's reader p99 under write pressure exceeds ~1ms, the read/write pool split (§2.2) is probably missing or misconfigured.
- If an implementation's status-display refresh exceeds ~5ms on a 14K-path corpus, `LenPrefix` (§2.9) is probably implemented as `len(List(""))`.
- If an implementation's UI refresh-tick cost grows visibly with corpus size, panels are probably reading from a shared cache rather than subscribing per-prefix (§2.8).

---

## 8. Benchmarking caveat — race detector × pure-Go SQLite (Go-specific)

The `modernc.org/sqlite` driver is a pure-Go translated SQLite VM. Every load/store gets instrumented heavily by the Go race detector. Measured slowdown:

| Bench | With `-race` | Without `-race` | Ratio |
|---|---|---|---|
| SDK Put (sqlite_file) | 2.92ms | 166µs | 17.6× |
| Cross-peer sync 1000 entities | 9.07s | 522ms | 17.4× |
| Bulk corpus mount 1385 files | 35.7s | 6.9s | 5.2× |

`go test` defaults to no-race; `make test` in entity-core-go forces `-race`. The review chain spent several hours chasing "fsync per autocommit" and "write amplification" hypotheses before noticing.

**Recommendation:** for any cross-impl performance comparison, use no-race builds. The production binary uses plain `go build`. Anyone reading test-suite numbers for perf intuition is reading 17×-inflated numbers.

This trap is Go-specific. Rust (`cargo test`) and Python (`pytest`) do not have an equivalent slowdown. Flagged here because cross-impl perf comparisons should be apples-to-apples.

---

## 9. Storage abstraction discipline — the convention that made fast turnaround possible

The review chain (silent data loss → design correction → SQLite pool split → `LenPrefix` interface addition → UI refactor) was completable in days, not weeks, because the storage abstraction in the Go impl held throughout. Calling this out as a positive cross-impl convention:

### 9.1 The shape

- `ContentStore` and `LocationIndex` are **interfaces**. Two leaf impls (Memory, SQLite) plus composable wrappers (`NotifyingLocationIndex` for events + sync hooks, `NamespacedIndex` for peer-id qualification). Any combination is valid.
- A **conformance test suite** runs the same behavioral assertions against every backend. Adding `LenPrefix` added four conformance cases; both Memory and SQLite passed identical assertions before merging.
- Backend-specific configuration (`SqliteOptions.{BusyTimeout, JournalMode, MaxOpenConns, ReadPoolSize}`) is contained inside the construction call. After construction, callers hold interface types only.
- **No type-assertion-to-concrete-backend leakage in production code.** The escape hatch (`SqliteStore.DB()`) is documented as test-only and is the sole exception.
- The SDK's storage switch (one `case` per backend at construction time) is the only place in workbench code that knows which backend it constructed.

### 9.2 Convention for adding a new backend

For cross-impl teams adding Postgres, BoltDB, libSQL, Badger, etc.:

1. Implement the `ContentStore` + `LocationIndex` (or language-equivalent) interfaces.
2. Pass the conformance suite. **Non-negotiable** — divergence in behavior is a bug, not a feature.
3. Add one construction site (analogous to `entitysdk/app.go::storage switch`).
4. Backend-specific options stay inside the constructor.

Nothing else should need to change.

### 9.3 The anti-pattern

A consumer type-asserting the concrete backend to use backend-specific features:

```go
// WRONG — backend leakage
if sqlite, ok := li.(*store.SqliteLocationIndex); ok {
    // ... use SQLite-specific feature
}
```

If a feature needs backend support, **add it to the interface and the conformance suite**, don't reach through the abstraction. `LenPrefix` is the canonical example: rather than type-asserting to SQLite and calling SQL directly, the interface gained a new method and every backend implements it.

### 9.4 In-memory vs persistent

Both backends are first-class. Behavior MUST be identical (enforced by conformance); performance MAY differ:

| Operation | Memory | SQLite | Notes |
|---|---|---|---|
| `LenPrefix` | O(N) map walk | O(log N + matches) index range | Memory acceptable because N bounded |
| `List` | O(N + sort) | O(log N + matches) index seek | Same shape |
| Write throughput | O(1) hash table | ~340µs (WAL fsync) per commit | Cross-peer ingest tests use memory because they don't need durability |
| Read/write pool | N/A | Distinct pools (§2.2) | Memory has no lock contention |
| Crash recovery | N/A | WAL replay | Test fixtures vs production peers |

**Cross-impl recommendation:** each implementation MUST run a conformance suite asserting behavioral parity across all its supported backends. If `LenPrefix` is added to one backend without the other, discrepancies surface as bugs downstream (status displays showing different counts in tests vs production).

---

## 10. Status of the four-phase incident response (post-close)

Updated with the cross-impl consolidated landing.

| Phase | Status | Where the fix lives |
|---|---|---|
| Phase 1: SQLite production setup | ✅ Landed | core-go `core/store/sqlite.go` |
| Phase 2: `Set` returns error | ✅ Landed | core-go many call sites |
| Phase 3-corrected: `ErrEventBufferFull` + per-consumer isolation | ✅ Landed | core-go `core/store/notifying.go`, `core/peer/fanout.go` |
| Phase 4 (workbench-side): drainers + coalescing | ✅ Landed | workbench `shell/app.go`, `console/main.go` |
| Phase 5: read/write SQLite pool split | ✅ Landed | core-go `core/store/sqlite.go` |
| Phase 6: cross-peer deadlock integration test | ⏳ Recommended (still open) | core-go test suite — "passes by luck" until written |
| Phase 7 (UI refactor): panel-prefix subscriptions | ✅ Landed (workbench-side) | workbench panel implementations |
| Phase 8 (`LenPrefix` interface) | ✅ Landed | core-go `core/store/store.go` + impls + conformance |
| Phase 9 (`OnPrefixChange` SDK helper) | ⏳ Medium priority | entitysdk, next round |
| Phase 10 (cross-impl rollout) | 🔍 Pending | Rust, Python, Godot to adopt §2.2 / §2.8 / §2.9 / §2.10 in their stores and panels |

Active stabilization is complete on the Go side. Remaining items are cleanups (one-line `len(List)` → `LenPrefix` migration in `ext/revision/resolve.go:60`; prefix-successor function consolidation) and forward-pointing (cross-impl rollout, integration testing, `Batch`/`Transaction` primitive design).

---

## 11. References

- `explorations/EXPLORATION-EMIT-DURABILITY-AND-DELIVERY.md` — architectural reasoning behind these recommendations
- The Go-impl concurrency/backpressure review — original Go-impl inventory
- The Go-impl concurrency followup — design correction landing the principle
- The workbench team's event-delivery / backpressure feedback — the workbench team's framing
- The workbench SQLite-busy bulk-ingest report — the four-phase incident report
- The workbench UI-refresh full-scan feedback — the UI anti-pattern incident + 5-step migration
- The workbench post-UI-refactor system review — post-refactor sweep
- The workbench cross-impl UI-patterns feedback — workbench's outbound to cross-impl teams; primary source for §2.8 / §2.9 / §2.10
- The core-go cross-impl consolidated feedback — core-go's response side; source for §7 perf baseline + §9 storage discipline + §10 status
- V7 spec: `ENTITY-CORE-PROTOCOL.md` §6.10, §2870, §2688
- `SYSTEM-COMPOSITION.md` §1.3, §2.7, §3.2
- `EXTENSION-SUBSCRIPTION.md` §5.5–§5.6
- Prior cross-cutting analysis: `entity-core-papers/papers/shared/notes/core/concurrency-and-isolation-update.md`, `.../entity-computation-analysis.md`
