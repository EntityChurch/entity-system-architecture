# PROPOSAL — Addressable ticks + a wall-deadline coordinate (the clock gains a scheduler surface, no new mechanism)

**Status:** DRAFT (2026-07-22)
**Target:** `specs/extensions/EXTENSION-CLOCK.md` — amend §2.8 (tick paths) + §3.4 (tick op) with the addressable
tick + wall-deadline coordinate, add a §10 note; rides `EXTENSION-SUBSCRIPTION`'s existing `created`-fires
matching (no new mechanism). No V7/wire renumber.
**Provenance:** R5 of `docs/research/reviews/ARCH-RESPONSE-CONTINUATION-CLOCK-ARC-RULINGS.md`, from core-go's
`clock-has-no-scheduler` spec-issue (`entity-core-go`, read-only). Supplies the punctuality + the round identity
`PROPOSAL-CONTINUATION-STANDING-MODEL` §4/§4.1 build on.
**Scope:** **additive; not before-freeze.** The clock today is a **heartbeat, not a scheduler** — there is no
"wake me at tick N", no one-shot, no per-waiter interval anywhere in `specs/`. This adds the scheduler surface
**without a new primitive**: address the future as a path.

---

## 0. The gap (what's missing, exhaustively)

`EXTENSION-CLOCK` §2.8 writes each tick to a single overwritten path `system/clock/tick/latest`; the `tick`
operation (§3.4) subscribes to it. §1.1 puts "periodic clock events **for timed execution**" in scope — but the
mechanism to *express* a timed wait does not exist (core-go, exhaustive `specs/` search):

- **"Wake me at tick N" is not expressible.** Every watcher of `.../latest` receives **every** tick and
  self-filters — wake 1000 times to act once; with no subscription filters, every waiter pays that fan-out.
- **"Wake me in 500ms" is not expressible at all** when `tick_interval` is peer-global 1000ms — a 50ms realtime
  compute join and a 1s heartbeat cannot coexist on one peer.
- **`sequence` is not time.** `wall ≈ sequence × tick_interval` holds only if the interval never changes and no
  tick is missed — neither is guaranteed. A tick count cannot express a wall deadline, which is what
  `completion_deadline_ms` (STANDING-MODEL §4) and every real timeout need.
- **Missed-tick semantics are undefined** — a waiter on a skipped tick N could hang forever (a silent hang, the
  same class as the join wedge §4 exists to remove).

## 1. The mechanism — address the future as a path (no new primitive)

Subscription matching already runs against the **emitted path** with **`created` as a first-class change type**
(`EXTENSION-SUBSCRIPTION` §4.1): **a subscription on a path that does not yet exist fires when that path is
created.** That is the whole mechanism — a one-shot timer in landed semantics, no filter language, no scheduler
subsystem. The tick only has to be **addressable**:

```
today:      system/clock/tick/latest        ; overwritten; every waiter wakes every tick
add:        system/clock/tick/{sequence}     ; created once; a waiter on {N} wakes exactly once, on create
add:        system/clock/at/{ms}             ; a WALL-DEADLINE coordinate; fires when wall ≥ ms
```

- **`system/clock/tick/{sequence}`** — "wake me at tick N" = a subscription on `system/clock/tick/{N}`. Fires once,
  on create, to exactly the waiters that asked. Best fit for **frame-shaped** work (a realtime compute tick; the
  STANDING-MODEL §4.1 `round_id`). `.../latest` stays (the heartbeat pointer).
- **`system/clock/at/{ms}`** — a **time bucket**: "fires when wall ≥ ms". Best fit for **timeout-shaped** work
  (`completion_deadline_ms`, backoff, cron) — expresses a deadline directly and **does not drift** when
  `tick_interval` is reconfigured. §4's join needs both: the round identity is a frame; the deadline is a timeout.

## 2. The rules that make it carry a deadline

- **Missed-tick = at-or-after (MUST — the load-bearing rule).** A waiter on a `{sequence}` or `{ms}` coordinate
  that the peer skipped (asleep/overloaded past it) **MUST still fire, at or after that coordinate** — never
  *never*. A skipped-coordinate silent hang is the same class as the join wedge §4 removes; without this the
  surface cannot carry a deadline.
- **Demand-driven materialization (SHOULD).** The clock writes `tick/{N}` / `at/{ms}` **only if something is
  subscribed** to it (the subscription index answers this directly) — a **timer wheel**: cost ∝ *scheduled work*,
  not elapsed time, and **zero on an idle peer** (vs. 86,400 nodes/day at the 1s default if every tick got a
  path). This also keeps the periodic goroutine armed only while work is scheduled — addressing the "first
  lifecycle in `ext/`" posture concern (the marker sweep deliberately avoided a lifecycle). Unconditional writes
  + a retention sweep (the `CollectExpired*` precedent) are the conformant fallback.
- **Per-waiter intervals: NO — peer-global `tick_interval` stays.** Sub-interval timing is expressed by
  wall/sequence *addressing* (a 50ms compute join and a 1s heartbeat coexist via distinct `at/{ms}` waiters), not
  by a per-waiter `tick_interval`. This makes the §1.1 "sub-interval timing is the application's concern" stance
  **explicit**, which today it only implies.

## 3. `MAY`-level, conditional dependency (no blanket §10.3 elevation)

The `tick` operation **stays MAY** (§10.3): a peer with no scheduled waiters needs no ticks. But a consumer that
**sources a MUST-level behavior from a tick** makes the tick *conditionally* required — stated for the one live
consumer: **a tick-driven standing join (STANDING-MODEL §4.1) requires CLOCK tick emission; a self-round-id join
does not.** The addressable surface here is what lets that dependency stay conditional (a per-request join uses
`at/{ms}` for its deadline and its own round id, touching no tick sequence).

## 4. Cross-impl surface to pin

- **MUST:** the `system/clock/tick/{sequence}` and `system/clock/at/{ms}` path grammars; **missed-tick =
  at-or-after** firing (§2) — a divergence here is a silent cross-impl hang.
- **MUST:** a `{sequence}` path is created **exactly once** and is **immutable** (monotonic, §2.8) — a subscriber
  that fired on `{N}` never re-fires; correlates with the STANDING-MODEL §4.1 `round_id` guard.
- **Impl-defined (MAY diverge):** demand-driven vs. unconditional-plus-retention materialization; timer-wheel
  internals; jitter tolerance (already §10.4 implementation-defined).

## 5. Security

Tick subscriptions already respect the subscription capability model (`EXTENSION-CLOCK` §7.4 → `EXTENSION-SUBSCRIPTION`
§9): a waiter needs capability for `system/clock` `tick` + a valid deliver token. The addressable paths inherit
this unchanged — a `{sequence}`/`{ms}` subscription is an ordinary scoped subscription. **Demand-driven
materialization MUST NOT let an unauthorized subscriber force a write** (the clock materializes a coordinate for a
subscription only after the subscription's own capability check passes).

## 6. Relationship to the rest of the arc

- **STANDING-MODEL §4/§4.1** — this makes the join **punctual** (fires at the deadline via `at/{ms}`, not at the
  next touch) and supplies the **round identity** (`tick/{sequence}`) the cross-round-bleed guard consumes. That
  is the correct dependency direction: **ticks improve §4; they are not load-bearing for it** (§4's deadline is
  evaluated lazily today and degrades gracefully without ticks).
- **EXTENSION-CLOCK R7/R8** (same arc) — the tick paths are excluded from advancement (§4.3) and history-recording
  (§9); the addressable `tick/{sequence}` paths inherit both exclusions (they are engine output).
- **Not before-freeze.** The heartbeat (`.../latest`) + R7/R8 ship now with core-go's tick emission; this
  addressable surface is the sequenced upgrade.

## 7. Conformance & posture

- **No new mechanism, no wire renumber.** New path grammars + one firing rule (at-or-after) + the MAY/conditional
  note. Rides subscription's `created`-fires matching.
- **The CDN-corridor meta-rule:** the at-or-after firing + one-shot-on-create semantics are not validated until a
  cross-impl run exercises a waiter on a skipped coordinate and a waiter on a future coordinate across two
  conformant peers — prose review does not catch a missed-tick silent hang.

## 8. References
- `ARCH-RESPONSE-CONTINUATION-CLOCK-ARC-RULINGS.md` R5 (this proposal's ruling); core-go
  `docs/validation/spec-issues/2026-07-22-clock-has-no-scheduler.md` (read-only, the finding).
- `EXTENSION-CLOCK.md` §1.1/§2.5/§2.8/§3.4/§4.3/§7.4/§9/§10.3; `EXTENSION-SUBSCRIPTION.md` §4.1 (`created` matching)
  /§9 (cap model).
- `PROPOSAL-CONTINUATION-STANDING-MODEL.md` §4/§4.1 (the consumer — punctuality + round identity).
