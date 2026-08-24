# PROPOSAL — Standing continuations: own authority, own completion policy (the subscription-shaped model)

**Status:** ✅ **IMPLEMENTED — CYCLE CLOSED 2026-07-29** (opened DRAFT 2026-07-18; both facets folded to
`EXTENSION-CONTINUATION` v1.21, converged byte-identical three-way, nothing owed by any impl — core-go
cycle-close packet `ROUTING-2026-07-29-cycle-close-arch-packet.md` acknowledged). **§3 (authority) ✅ IMPLEMENTED — FOLDED 2026-07-28** (AT-1..AT-4 converged
three-way, core-go `b98e08b`; edit applied to EXTENSION-CONTINUATION §3.1/§3.1b/§6.1). **§4 (join completion)
✅ IMPLEMENTED — FOLDED 2026-07-29.** The three residuals are built + green three-way (Go `1ccdd51` reference;
Rust `aec13b1` O5 sweep-all; Python `1590d8a` O4 `slot` + `fire-partial` + O5) — live three-way re-run clean
(continuations 61/61, bounds 3/3, liveness shared skips; in-process rust 72/72, py 2936+8). Reap/`fire-partial`
are **peer-local** (no wire category) — their convergence is source-reconcile + each in-process suite + a
no-wire-regression run; the cross-peer surface (round_id guard, O4 drop-body) is what the wire run exercises.
Folded to **EXTENSION-CONTINUATION §2.3** (join fields) **+ §3.5** (round-identity guard + round_id increment)
**+ new §3.5a** (deadline / abandon / fire-partial / sweep-all / round identity / delivered-error /
marker+drop-body). **O6** (quiescent-peer abandon) stays deferred — no impl does it. **Prior residual pins (now
discharged by the fold):** **O4** drop-response body (Py added `slot`); **O5** reap scope — Go sweeps all tracked
joins on touch, siblings extended touched-only → sweep-all (real Rust+Python build, no timer).
**Target:** `specs/extensions/EXTENSION-CONTINUATION.md` — §3.5/§3.6 (advance authorization), §6.1
(privilege-escalation security), §2.3/§3.5 (join entity + slot advancement), and a new standing-model
subsection. Fixed in place — cohort findings against **landed** CONTINUATION, no rev bump. **No wire
change** (authority is a check-site correction; join adds entity fields + a reaper on the existing
marker-sweep pathway).

> **§3 status update (2026-07-27) — the authority split is now built three-way; this corrects the
> 2026-07-25 arch note that read Go as over-restricting.** All three impls now key the advance-time
> authority decision on a deliverer-declared **`reactive_trigger`** signal: Go
> `ext/continuation/advance.go` (`if !hctx.ReactiveTrigger { CheckPathCapability(...) }`, commented
> *"split by trigger kind per §3 (Q2 ruling, MUST)"*), Rust `extensions/continuation/src/lib.rs`
> (`ctx.reactive_trigger`), Python `continuation.py` (`if not ctx.reactive_trigger and
> ctx.caller_capability`). The over-restriction is gone. **NOT yet ratify-ready:** (a) no dedicated
> cross-peer *authority* conformance test exists — the three-way GREEN `continuation_bounds` gate
> exercises the depth brake, **not** the authority split; per the CDN-corridor meta-rule an
> authorization claim is not validated until a cross-impl test exercises it; (b) **O1 residual** — Go
> keys on `reactive_trigger` alone; Python adds a `caller_capability`-presence secondary condition. They
> likely converge behaviorally (an absent caller cap makes Go's `CheckPathCapability` a no-op) but this
> is exactly the seam O1 exists to pin. **Path to fold:** pin O1 (§7) → cohort builds the authority test
> (reactive-cross-peer-no-cap advances; administrative-invoke 403s) → run three-way → fold §3 into
> `specs/extensions/`. This is now the top next-fold candidate.

**Two lineages, one primitive.** This proposal exists because two *independent* tracks, touching
CONTINUATION from different concerns, converged on the same missing capability — a **standing
continuation that outlives its trigger and must act as a first-class entity, not an extension of
whatever poked it**:

- **Network lineage** — NETWORK Amendment 12's reconnect/`maintain-peer` graph + the browser-defer
  goal. Surfaced the **authority** facet (entity-core-go Q2, spec-issue at core-go `2302c09`, pinned
  to `CheckPathCapability`/`CallerCapability` in the advance path).
- **Compute lineage** — the compute-program continuation-managed sharding substrate
  (`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM` §4a). Surfaced the **completion/lifecycle** facet
  (workbench-go Q3, pinned as a running wedge test `TestContFork_FailureAsymmetry` @ workbench
  `compute-program-poc 4902236` against core-go `a0d9ea6`).

They are two facets of one entity. This proposal states the shared model **once** and derives both —
rather than shipping two proposals that would specify the same primitive twice and risk diverging at
the exact cross-peer seam this cycle exists to close. (Supersedes the standalone
`PROPOSAL-CONTINUATION-JOIN-COMPLETION-POLICY` draft, absorbed here as §4.)

---

## §1 Problem — the spec's continuation model is a *forward chain*, not a *standing entity*

`EXTENSION-CONTINUATION` was built around the fire-and-forget **forward chain**: a caller installs a
continuation, it advances once (or a fixed number of times), it is consumed. Everything the spec pins
about authority and lifecycle assumes that shape. Two consumers now install **standing** continuations
(`remaining_executions: null`, §6.3) that persist and re-fire on external events — and both hit a wall
the forward-chain model never specified:

1. **Authority (network).** A standing continuation triggered **cross-peer** self-authorizes its
   advance against **the trigger's** capability. So a remote peer must hold *advance rights on the
   browser's own continuation* to poke it — the browser cannot install a reactive continuation that an
   arbitrary reconnecting peer triggers without handing that peer authority over its internals. This
   **blocks browser-defer** directly.
2. **Completion (compute).** A standing join barrier that misses one slot **wedges forever** — it
   never fires and, being standing, never resets, for that round *and every subsequent round*. There
   is no timeout, no partial path, no abandonment (verified: the only time-sweep in the extension is
   `CollectExpiredMarkers`, which reaps error markers, not joins).

Both are the same defect in different clothes: **a standing continuation's authority and its lifecycle
were left coupled to the trigger, and a standing continuation by definition outlives its trigger.**

## §2 The model — a standing continuation is subscription-shaped

The system already has a first-class standing, event-triggered, owner-authorized entity: the
**subscription**. A subscriber installs a subscription under its *own* authority; an emitter *triggers*
the notification, which fires under the **subscriber's** authority; the emitter never holds the
subscriber's grant, and the gate on the emitter is **reachability** (can its write touch a watched
path), not possession of the subscriber's rights. A subscription also owns its liveness independent of
any one emitter.

**A standing continuation MUST behave the same way.** State it once, generally, so every future
standing-continuation consumer inherits it:

> **A standing continuation is an owner-authorized, delivery-triggered entity. It advances under its
> *own* stored authority; triggering it is gated by *delivery-reachability* to its trigger channel,
> not by the trigger holding advance rights over it; and it owns a completion policy independent of any
> single trigger.**

§3 is the authority half of that sentence; §4 is the lifecycle half.

## §3 Facet A — Authority: advance under own authority, trigger by reach (Q2)

> **Status (2026-07-28): ✅ IMPLEMENTED — FOLDED.** AT-1..AT-4 converged three-way (core-go report `b98e08b`:
> rust surface = `ctx.reactive_trigger` on HandlerContext, non-inherited, matches Go; python proved AT-4
> fail-closed live over the wire — a zero-cap external caller denied at the dispatcher, independently reproducing
> Go's finding; O1 holds three-way). The staged **EXTENSION-CONTINUATION §3.1/§3.1b/§6.1** edit is applied
> (`VECTOR-SPEC-2026-07-28-standing-model-authority.md` Appendix). The bounds/CBX gate does **not** cover
> authority — this test does. Published oracle-pinned per [ADR-0012].

**The spec already prescribes this; an impl over-restricted it.** Verified in landed CONTINUATION:

- **§3.5 step 5 (line 751):** `execute.capability = continuation.data.dispatch_capability` — the
  advance dispatches under the **continuation's own** stored capability, validated in-chain at install
  (§3.1a / §3.2 step 4).
- **§3.6 (line 346):** *"Any handler or extension can trigger advancement by dispatching to
  `advance`."*

entity-core-go's advance path additionally runs `CheckPathCapability("advance", path)` against the
**caller's** capability — a gate the spec does **not** define. Locally harmless (the installer holds
its own path). Cross-peer it is the bug: it forces a remote trigger to hold advance-cap on the target
continuation, which is exactly what browser-defer must avoid.

**Ruling (MUST — cross-peer-observable):** distinguish the two ways advancement is reached, mirroring
the subscription model:

- **Reactive trigger (the standing/cross-peer path).** A continuation advanced by delivery of an
  event — a `deliver_to` arrival, an inbox route, a subscription-driven poke — is gated by
  **delivery-reachability to the continuation's trigger channel** (authority the *owner* configured
  when it set the channel up). It then advances under the continuation's **own**
  `dispatch_capability`. The trigger MUST NOT be required to hold `advance` capability on the
  continuation path. This is line 346 made precise for the cross-peer case.
- **Administrative invoke (the local/operator path).** A direct `advance` dispatch that is *not* a
  configured delivery — an operator or handler managing continuations — remains capability-gated on
  the continuation path (Go's existing check, correctly scoped to this case).

**Why this is safe (the reviewer's question, answered):** a reactive trigger causes the continuation
to do only what its **owner pre-authorized** at install (the scoped `dispatch_capability`); the trigger
gains nothing it could not already do, exactly as an emitter triggering a subscription gains none of
the subscriber's rights. The residual is **timing/frequency** (a reachable peer could over-trigger) —
a DoS surface, not privilege escalation — bounded by the continuation's own `bounds` (`chain_depth`/TTL
per `PROPOSAL-CONTINUATION-BOUNDS-PROPAGATION`) and by idempotency at the trigger channel. §6.1 gains a
note that the escalation mitigation is the **install-time** in-chain check on `dispatch_capability`,
*not* an advance-time caller check — the latter breaks reactive standing continuations without adding
containment.

**This unblocks browser-defer:** the browser installs a reconnect continuation under its own authority;
a reconnecting peer triggers it by reaching the delivery channel; it fires under the browser's
authority. Rust does not hand remote peers advance rights over browser internals.

## §4 Facet B — Completion: the standing continuation owns a completion policy (Q3, absorbed)

> **Status (2026-07-29): ✅ IMPLEMENTED — FOLDED.** Built + green three-way (Go `1ccdd51` reference; Rust
> `aec13b1`; Python `1590d8a`); live re-run clean (continuations 61/61, bounds 3/3). Folded to
> **EXTENSION-CONTINUATION §2.3 + §3.5 + new §3.5a**. The `round_id` guard, deadline/`abandon`/`fire-partial`,
> and the **sweep-all-on-touch** reap (no timer) are now normative. Reap/`fire-partial` are peer-local (no wire
> category); the cross-peer surface (round_id guard, O4 drop-body) is wire-exercised. O6 deferred.
>
> **Fold audit (source-verified at fold, all three impls at the pinned commits — read, not assumed):**
> (1) the durable marker vocabulary is **three** `lost`-sink reasons — **`join_incomplete`** (deadline, slots
> never arrived; also fire-partial), **`join_late`** (stale-round drop), **`join_error_slot`** (delivered-error
> slot) — at `system/runtime/chain-errors/lost/{chain_id}/{step_index}/{reason}/{marker_hash}` (type
> `system/runtime/chain-error-lost`). **`stale_round` is NOT a marker reason** — it is only the value of the
> `dropped` key in the wire drop-body; the first fold draft conflated the two and was corrected against source.
> (2) The drop-body is the five-key `{advanced:false, dropped:"stale_round", slot, targeted_round, current_round}`,
> status `200`, converged three-way — this discharges the drop-response-shape pin core-go routed to arch
> (`advance.go` "status/code contract here is a pin candidate routed to arch"). (3) **`round_id` = clock
> `tick.sequence`** (§4.1's tick-driven source) is **UNBUILT in all three** — every impl self-increments a private
> counter from 0. Only the **self-supplied monotonic** round id is built/folded as the floor; the tick-source is
> marked **design-forward** in §3.5a and **deferred to the tick-driven W-COMPUTE realtime join consumer**, to be
> validated cross-impl when that consumer lands (CDN-corridor). This is the one genuinely-open §4 sub-item.

*(Full content of the former `PROPOSAL-CONTINUATION-JOIN-COMPLETION-POLICY`, preserved. Origin:
workbench-go `COMPUTE-PARALLELIZATION-TWO-MODELS-2026-07-17` §3.2/§8; seed vector
`TestContFork_FailureAsymmetry`.)*

The join barrier (`system/continuation/join`) fires only on `allSlotsReceived` (§3.5) and a **standing**
join resets `received` only after firing. §2.3/§3.5 define accumulate-under-CAS and fire-on-complete
but **never define what happens when a slot never arrives**. The failure splits into two mechanisms
that MUST NOT be conflated (a build refinement — `processAsyncDelivery` delivers regardless of status):

**Mechanism 2 — undelivered (the barrier wedges).** A dropped trigger, a pool refusal (429), or an
error returning before delivery leaves the slot unfilled; a standing join wedges for that round and
every subsequent round. Add to the join entity (§2.3):

```
system/continuation/join := {
  ...existing (expected, received, target, operation, params, remaining_executions)...
  completion_deadline_ms: {type_ref: "primitive/uint", optional: true}   ; per-round wall budget; absent = wait forever (today's behavior)
  on_incomplete:          {type_ref: "primitive/string", optional: true} ; "abandon" | "fire-partial" ; default "abandon"
}
```

- On install with `completion_deadline_ms`, arm a per-round timer (reset with `received` each round),
  reaped by the **existing `CollectExpired*` sweep extended to joins** — not a new subsystem.
- On deadline with `received ⊊ expected`:
  - **`abandon`** (default) — fail the round: emit a **`lost` marker** naming the missing slots, reset
    `received`, ready for the next round. A standing per-tick join thus **self-heals** rather than
    wedging — the load-bearing change for realtime reuse, and the direct analogue of §3's "owns its
    liveness independent of any trigger."
  - **`fire-partial`** — fire the target with partial `received` plus an explicit `incomplete` marker
    listing missing slots; never the default (a stitch assuming k fragments must opt in). **A
    determinism-critical stitch MUST NOT select `fire-partial`** (R4, 2026-07-22): its output is a
    boundary hash consumed for equivalence (Axis-1 §1.4/§4.3), and a partial fire is the same
    seam-collapse as the straggler bleed (§4.1), opted into. The compute-program descriptor MUST declare
    `fire-partial` **opt-in explicitly** (`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM` §4); Go passing an
    `incomplete` marker makes an error slot *decidable* but nothing forces the target to look, so the
    prohibition is spec-level, not left to the stitch.
- **Absent `completion_deadline_ms` = today's wait-forever.** No silent change; opt-in per join set by
  the substrate that installs it. NETWORK Amendment 12's reconnect graph is a join-free forward chain
  and is unaffected regardless.

### §4.1 Round identity — the `abandon` straggler guard (MUST) [added 2026-07-22, core-go cross-round-bleed finding]

`abandon` resets `received` and readies the next round — but a slot advance carries **no round identity**,
so a *straggler* from the abandoned round (a merely-slow slot, not a lost one — the mainline pool-refusal /
429 case §4 names) is byte-identical at the join to a next-round slot and **lands in it**. The result is a
round stitched from **two generations** → a boundary hash that is **wrong, deterministic-looking, and
reproducible** — the silent seam-collapse the whole compute POC exists to prevent, and one the existing §6
anchor *passes straight through* (a suite gap as much as a spec gap). Before §4 the join wedged loudly;
`abandon` trades the loud failure for a **silent** one, the wrong direction for the determinism bar.

The fix reuses a mechanism that **already exists** (read the source, don't invent — the same pattern
`PROPOSAL-CONTINUATION-BOUNDS-PROPAGATION` §2 applied to `cascade_depth`): the join carries a monotonic
`round_id`, incremented on every reset/fire; every slot advance is tagged with the round it targets.

```
system/continuation/join := { ...; round_id: {type_ref: "primitive/uint"} }   ; current round generation
; a slot advance carries the round_id it targets
```

- **Two sources, one field (MUST).** `round_id` = the clock **`tick.sequence`** (`EXTENSION-CLOCK` §2.8)
  when the fan-out is **tick-driven** — for a realtime per-tick compute join the tick *is* the round
  boundary, so the sequence is not an approximation of the round id, it **is** the round id; the
  **fan-out's own** monotonic round id otherwise (a per-request join supplies its own).
  **[FOLD AUDIT 2026-07-29: the tick-source is UNBUILT in all three impls — every impl self-increments a
  private counter from 0. Only the self-supplied monotonic branch is built/folded (§3.5a floor). The
  `tick.sequence` binding is design-forward, deferred to the tick-driven W-COMPUTE realtime join consumer,
  validated cross-impl when it lands.]**
- **Drop stale, loudly (MUST).** A slot whose `round_id` ≠ the join's current round **MUST NOT be admitted**
  and **MUST be dropped loudly** — a `lost`/`late` marker naming the stale slot (a slot "consistently one
  tick behind" is a useful realtime signal). **A straggler from an abandoned round MUST NOT appear in a
  later round.** Lateness must be *observable*, not merely survived — a silently mixed generation is worse
  than a dropped round.
- **Round-opening collapses into this (R2).** The empty-round gap (a tick where every slot is pool-refused
  never opens a round, so nothing is reported) dissolves: the round is opened by its boundary — the tick, or
  the fan-out's round-open — whether or not any slot arrives. No separate round-opening signal.
- **Conditional clock dependency.** A *tick-driven* join requires `EXTENSION-CLOCK` tick emission
  (`EXTENSION-CLOCK` §10.3 — a MAY the join makes *conditionally* required); a self-round-id join does not.
  The `tick` operation stays MAY in general (no blanket §10.3 elevation).

**Mechanism 1 — delivered-error (the target must reject).** A delivered non-2xx *fills* its slot; the
join fires normally and the failure surfaces at the stitch/target, which has no contract for "one of my
slots is an error." The join's `received` map MUST **preserve each slot's status** (pass a non-2xx /
`compute/error` slot payload through as-is, not coerce it into boundary bytes). A target requiring
all-good slots (the determinism-critical stitch) **MUST reject** on any error slot — surfacing a `lost`
marker for the round rather than emitting a boundary entity computed from an error payload. This keeps
the boundary-equivalence licence honest: an error slot MUST NOT silently produce a boundary hash that
diverges across peers (the cross-peer seam-collapse class the methodology guards).

**Determinism preserved:** both mechanisms are failure-path only; the success path is untouched
(`received` keyed by slot name, read in `expected` order, byte-identical to serial). Failure produces a
`lost` marker, never a partial boundary entity — a failed round is *observably* failed, not silently
divergent.

### §4.2 Finite-continuation exhaustion — delete, not retain (MUST) [close-out 2026-07-29]

**Was this a spec issue? Yes — finalized.** The landed spec said, at both §3.4 (forward) and §3.5 (join),
*"At 0: entity records exhausted state … SHOULD clean up"* — a `SHOULD` that **describes both behaviors and
mandates neither**. Its two conformant readings **diverge across a peer boundary**: a consumed finite
continuation (`remaining_executions` → 0) resolves to *absent* on one peer and to a `remaining_executions: 0`
husk on another — a `tree = path → hash` difference at a path, exactly the cross-peer-observable class the
methodology says to **pin, not leave `SHOULD`**. So it is a spec issue (an underspecified `SHOULD`), and the
back-and-forth was only about *which way* to pin.

**Source-read, all three (not assumed):** **Go deletes** at exhaustion on every fire path; **Rust deletes**
(`lib.rs:1705` "decrement … or delete if last"); **Python is split** — deletes on its `fire-partial` path but
*retains* on its ordinary completion path (an internal Python inconsistency of its own). Two impls delete, the
spec already leans delete (`abandon` deletes a suspended continuation, §3.8 "killing a stopped process"), and
delete-is the lower-churn convergence.

**Ruling (MUST):** at exhaustion the entity **MUST be deleted**, not retained at `remaining_executions: 0` —
one rule for **both** forward and join (stated once at §3.4; §3.5 references it). Delete-if-last is one CAS with
the final decrement, so 0 never persists; a post-exhaustion advance resolves to `not_found`. **Folded to
EXTENSION-CONTINUATION §3.4 (note + pseudocode) + §3.5 (pseudocode).** **✅ BUILT + CLOSED 2026-07-29:** Python's
ordinary completion path now routes through its shared `_age_join_after_fire` delete-if-last helper (py `a4eb929`,
+2 tests), matching Go/Rust and removing Python's internal fire-partial-vs-ordinary inconsistency. Nothing owed.

## §5 The shared invariant (state once — this is the anti-hole)

Both facets are one rule about the same entity; folded as the opening of the new standing-model
subsection so it is not re-derived per consumer:

> A **standing continuation** (`remaining_executions: null`) is decoupled from any single trigger. Its
> **authority is its own** (advances under its stored `dispatch_capability`; triggered by
> delivery-reach, §3), and its **lifecycle is its own** (owns a completion policy that self-heals a bad
> round rather than wedging, §4). A trigger reaches it; it does not own it. New standing-continuation
> consumers inherit both without re-specification.

## §6 Cross-impl surface + conformance

Convergence method (not vectors-first): build, reconcile cross-impl, the reconciled behavior becomes
the suite. Directional anchors:

1. **Authority — reactive cross-peer trigger needs no advance-cap.** Peer B reaches peer A's standing
   continuation via its delivery channel with **no** advance capability on A's continuation path;
   assert it advances (fires under A's `dispatch_capability`). Assert the **administrative** direct
   invoke without the path cap still 403s (proves the split, §3).
2. **Authority — browser-defer acceptance.** A reconnecting remote peer triggers the browser's
   reconnect continuation without holding advance rights on it (the concrete browser-defer signal).
3. **Completion — standing join self-heals.** A standing join with `completion_deadline_ms`/`abandon`
   missing one slot emits a `lost` marker, resets, and **fires clean on the next round** (proves §4
   mechanism 2; seed = `TestContFork_FailureAsymmetry`).
4. **Completion — error slot rejected, not folded.** A delivered-error slot makes the determinism
   stitch emit a `lost` marker, never a boundary entity (proves §4 mechanism 1).
5. **Completion — the straggler bleed is caught (§4.1, R1).** Slot A arrives round N; deadline passes (B
   never arrives); round N is abandoned; round N+1 opens and A arrives; **then B *of round N* arrives
   late.** Assert B is **dropped with a `lost`/`late` marker** and **MUST NOT** fill round N+1's B slot —
   no mixed-generation stitch. This is the anchor the naive "fires clean on the next round" (anchor 3)
   *passes straight through*; it exists specifically to catch the silent cross-round bleed, and asserts on
   the `round_id` guard, not merely on a clean next-round fire.

**Wire pin (R3, cross-impl-observable — a second impl is what's gated on these):** **(a)** `round_id: uint`
on the join and echoed on each slot advance (§4.1); **(b)** the `join_path` / `join_slots` field spellings
(adopt Go's `fd25fa0` spelling, pinned at first cross-impl — a probe today would freeze an unpinned
spelling as the bar); **(c)** the completion reason codes reuse the continuation-family codes — **do not**
mint strings that collide with the bounds family (the §4a Ruling-3 `bounds_exceeded` /
`chain_depth_exceeded` discipline). Name them explicitly at first cross-impl. **(d)** the stale-round
drop-response body is `{advanced: false, dropped: "stale_round", slot, targeted_round, current_round}` (Go's
oracle shape; Python omitted `slot` — converges by adding it) — see O4. **Converged three-way** on the
reconcile of 2026-07-28 (core-go `b98e08b`) on (a)/(b)/(c); the residual on (d) is Python's `slot` add. The reap
scope (O5) is **NOT** converged — Go sweeps all tracked joins on touch, the siblings reap only the touched join;
ruling is converge-on-sweep-all (real Rust+Python build) — see O5.

## §7 Scope, boundaries, open points

**In scope:** the authority check-site correction (§3) + the join completion policy (§4) + the §5
shared-invariant subsection. **Not in scope / stays separate:** `chain_depth`/bounds cross-peer
termination (`PROPOSAL-CONTINUATION-BOUNDS-PROPAGATION` — a *different* axis: bounding causal chains,
not the standing entity's authority/lifecycle) and the marker family
(`PROPOSAL-CONTINUATION-LOST-ERROR-MARKER-MUST` — observability, single-lineage, already landing). The
join entity gains `completion_deadline_ms`, `on_incomplete` (§4), and `round_id` (§4.1) as **entity
fields**; **no V7 wire-format renumber, no new opcode/cap/error code** (completion reason codes reuse the
continuation family). `parent_chain_id` remains RESERVED.

**Open points for cohort review:**

- **O1 — the reactive-vs-administrative signal (§3). [RESOLVED 2026-07-28 — pinned; authority test authored.]**
  **Pin:** the signal is `reactive_trigger` (converged three-way); the administrative **absent-caller-capability**
  case is pinned **fail-closed** — an administrative advance (`reactive_trigger` = false) whose caller presents no
  capability on the continuation path is **DENIED (`403`)**, forcing Go's no-op-on-absent and Python's
  require-cap-present to agree on the outcome. Exercised by AT-4 in the authority vector spec (not assumed). Detail
  below preserved for rationale.
  The concrete signal an impl keys on to classify an advance as *reactive delivery* (delivery-gated,
  own-authority) vs. *administrative invoke* (path-cap-gated) is the one place a divergent reading springs
  apart at the cross-peer seam. **The three impls have independently converged on a deliverer-declared
  `reactive_trigger` flag** (Go `ReactiveTrigger`/`WithReactiveTrigger`, Rust `ctx.reactive_trigger`,
  Python `ctx.reactive_trigger`) — set when the advance is driven by a delivered event (inbox route,
  subscription poke), false for a bare `advance` EXECUTE. **Pin `reactive_trigger` as the signal.** One
  residual to resolve at the pin (and assert in the authority test): Go gates on `reactive_trigger` alone,
  Python additionally requires `caller_capability` present on the administrative path. Specify the
  administrative-path semantics for the **absent-caller-cap** case so the two cannot spring apart
  cross-peer (Go's `CheckPathCapability` no-ops on a zero caller cap, which is *believed* equivalent to
  Python's explicit condition — the authority conformance test MUST exercise it, not assume it). This is
  the twin of the bounds proposal's causal-vs-standing O1 — same shape, same discipline.
- **O2 — join deadline source** — wall-clock vs. logical-tick budget (realtime wants wall;
  deterministic replay wants logical; likely a `deadline_kind`, start wall-clock `abandon`).
- **O3 — 429-as-immediate-undelivered** — a pool-refused shard is knowably undelivered at once; a
  fast-path failing the round on synchronous pool refusal (vs. waiting out the deadline) is a latency
  win, routed as an optimization, not required for the floor.
- **O4 — stale-round drop-response body shape (§4.1). [✅ DISCHARGED BY FOLD 2026-07-29 — Python added `slot` + the two round fields (`1590d8a`); the five-key body is now normative in EXTENSION-CONTINUATION §3.5a.]**
  A stale slot's drop response is returned to the slot sender, which may be a remote peer → cross-peer-observable.
  Go's oracle body carries `{advanced: false, dropped: "stale_round", slot, targeted_round, current_round}`; Rust
  matched it by reading Go's source; **Python's routeback omits `slot`** (and the two round fields). Left unpinned,
  Rust vs Py diverge at the seam. **Ruling (MUST):** the drop response body is Go's full shape —
  `{advanced: false, dropped, slot, targeted_round, current_round}`; **Python adds the missing fields to converge.**
  Keys are snake-adjacent single tokens; the `dropped` value is a status code in the continuation family (snake,
  parallel to `bounds_exceeded`), so `stale_round` is convention-correct — **do not** re-case to kebab. Also pinned
  as R3 wire-pin (d) below.
- **O5 — reap scope: sweep-all-on-touch, no timer (§4). [✅ DISCHARGED BY FOLD 2026-07-29 — Rust (`aec13b1`) +
  Python (`1590d8a`) extended touched-only → sweep-all; now normative (MUST) in EXTENSION-CONTINUATION §3.5a, no
  timer. RULED 2026-07-28 — source-verified; TWO prior passes retracted.]** History, because it is the D7 lesson
  twice over: pass 1 ("deadline-abandon MUST be observable without
  a follow-up advance") rested on a report *summary* that Go auto-abandons a quiescent join — false. Pass 2
  ("touch-driven, no timer, **all three already match, no sibling build**") corrected the timer error but introduced a
  *second* unsourced claim — that the impls match. **They do not.** Arch read all three at source (`cd35ad3` +
  arch's own read):
  - **Go** — `maybeSweepJoins` (`ext/continuation/join_completion.go:293`) → `sweepOneJoin` in a loop: a throttled
    pass over **every tracked deadline-carrying join**, reaping any expired round, on **any** continuation op. **No
    timer** (line 297: a reaper loop "would introduce a lifecycle" — deliberately declined).
  - **Rust** (`extensions/continuation/src/lib.rs:1209-1211`) + **Python** (`continuation.py:1342-1347`) — reap
    **only the join being touched**; both source comments name the divergence ("unlike Go's `CollectExpired*`… a
    round with no further arrivals is never proactively reaped" / "No background sweep subsystem exists in this peer").
  - **The divergence is real and cross-peer-observable:** a deadline-join that goes silent **while the peer stays
    continuation-active** is reaped by Go (its next op sweeps *all* joins) but **never** by the siblings (their next
    op reaps only *its own* join). Go emits the `join_incomplete`/`lost` marker; the siblings never do — a permanent
    divergence on exactly the failure/liveness signal a consumer watches. That is the §4 determinism motive's own
    seam-collapse class → it MUST be pinned, not left implementation-defined.
  - **Ruling (MUST) — converge on Go's sweep-all:** on any continuation op, an impl reaps **all** tracked
    deadline-expired joins (throttled), not only the touched one. **No background timer** (the truly-quiescent
    peer — zero ops — stays uniform: nothing sweeps; that residual is O6). **This is a real sibling build for BOTH
    Rust and Python** (extend reap-the-touched-join → sweep-all-tracked-joins; Rust notes "no separate sweep subsystem
    to extend," so it is a genuine addition, not a tweak). Reference: Go `maybeSweepJoins`/`sweepOneJoin`.
  - *Alternative considered + rejected:* rule touched-only conformant and sweep-all a permissible superset — rejected
    because the silent-join marker then appears in Go and not the siblings, a cross-peer-observable divergence on a
    liveness signal (the exact thing §4 exists to make deterministic). Ties to O2's `deadline_kind`.
- **O6 — quiescent-peer abandonment (deferred, all-three decision, not a convergence gap).** Whether a peer should
  *eventually* abandon deadline-expired joins even with no continuation activity (e.g. a peer-lifecycle sweep) is a
  **separate feature** — **no impl does it today**, so it is not a divergence to close but a design choice for all
  three at once, if a consumer ever demonstrates the need. Deferred; not gating §4.
