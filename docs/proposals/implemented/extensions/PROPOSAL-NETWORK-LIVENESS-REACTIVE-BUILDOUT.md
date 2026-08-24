# PROPOSAL — NETWORK reactive lifecycle: the liveness write + continuation build-out (Amendment 12)

**Status:** **IMPLEMENTED — RATIFIED 2026-07-14, BUILT + green three-way, and FOLDED IN FULL 2026-08-15.** All Amendment 12 spec deltas are now in `EXTENSION-NETWORK`. This document is the **design record**, not a to-do. **BUILD LANDED — rungs 1–3 built + validated green three-way 2026-07-15…07-17** (go/rust/py; reports `2026-07-15-a12-liveness-harness`, `2026-07-19-reconnect-disconnect-crossimpl`; re-proven live 2026-07-28).

> **CORRECTION 2026-08-13 — this header claimed "§7.2 remote-peer re-subscribe (unbuilt all three)". That was
> false, and it is the sixth carried-forward absence assertion of this cycle.** §7.2 is built in all three:
> go `ext/network/handler.go:213` (`handleRestoreSubscriptions`, advertised in the operations manifest at
> `:153`, i.e. §A5-compliant) · rust `extensions/network/src/lib.rs:895` · py
> `entity_handlers/network.py:409` + `_install_resubscribe_continuation`. Source-read at go `3e8361f`, rust
> `1152d35`, py `ad0ef98`; confirmed by `entity-core-go` (`ROUTING-2026-08-13-k`). **An absence claim expires
> the moment any of the trees moves, and this one was never re-taken.** Retained rather than deleted, because
> the failure shape is the point.

**Fold state — COMPLETE 2026-08-15.** Amendment 12 landed across four partial folds:

| Delta | Landed in `EXTENSION-NETWORK` | Commit |
|---|---|---|
| §A1 + §5.4 join | §5.4a (new), §5.4 pseudocode corrected, §12.1 bullet, two vectors | `58d7e35` |
| §A6.6 `chain_id` | §2.4 field declaration, §4.1 pseudocode | `a14b1be` |
| §A2 / §A6.2–§A6.5 retry lifecycle | §2.2.1 (new), §2.11 (new), §2.2 bounds — fields upstream at core §3.13 `9829c6f` | `dd170bb` |
| **§A3 · §A4 · §A6.0 · §A6.1** | **§4.1.1 (new), §5.4.1 (new), §12.1.1 (new), §4.1 pseudocode, §6.6 `granted_at`** | **this fold** |

**What the final fold changed, and why it is recorded here rather than in the spec.**

- **§4.1.1 — the backoff continuation is STANDING with an `on_error` re-arm.** The one-shot form was
  unimplementable **by ordering, not timing**: the re-install lands *inside* the dispatch while the
  advance's consume runs *after* it, deleting the path just written. Measured identically through three
  independent observables — Go at an external socket, Rust via §3.10 markers in-process, Python on its own
  wire — all agreeing on **2 attempts**, after which an offline peer is never recovered, with no marker and
  a `200`.
- **The install path moved to `system/network/peers/{peer}/on-reconnect-backoff`**, adopting what **all
  three implementations independently built** against this spec's own stale pseudocode. Three-way
  convergence against stale text is the strongest signal this method produces; the spec states the rule
  (handler-managed state belongs in the handler's namespace) and this document records that the cohort got
  there first.
- **§5.4.1 — `system/peer/status` is transition-written.** Quantified by the Go rung-2 build: ~2
  non-transition events/min/peer/side to every lifecycle subscriber, and ~2,880 superseded status entities
  /day/peer accreting with nothing to collect them.
- **§12.1.1 — the liveness slice is the conformance floor**, so consumers stop blocking on all of §4.1.
- **The `on_error` guidance correction supersedes the rung-1 handoff and `PROPOSAL-CONTINUATION-LOST-ERROR-MARKER-MUST`
  §5's bullet, which is struck.** The blanket *"never route `on_error` to `system/inbox/*`"* left this delta
  with no legal target — a contradiction between two arch documents that the Go seat caught before anyone
  coded against it.

**Build state at fold, source-read 2026-08-15** (`entity-core-go` `2df96f8` · `entity-core-rust` `462f2c2` ·
`entity-core-py` `f33526f`, all clean): the **STANDING** form and the **managed-namespace path** are built in
all three — rust and py both cite *"arch ruling 1"* at the constant. **`on_error` is NOT built in python**,
which binds the §3.10 lost-error marker instead — precisely the category error §4.1.1 names — so that half is
a live cohort delta, not a formality. *(Dated observation, not a standing fact — D1/D8.)*

**This fold releases `PROPOSAL-CONTINUATION-LOST-ERROR-MARKER-MUST` Delta 1**, which was held on the §A6.1
coupling (§2a).

**Target:** `specs/extensions/EXTENSION-NETWORK.md` v1.4 → **Amendment 12** (spec deltas §A) +
cohort build-out (§C). Normative / protocol-touching → proposal-first per AGENTS-STANDARD; routes
to the implementation cohort (Go / Rust / Py + browser-rust) for a **convergence build**, not to a
pre-written vector suite (§D).
**Origin / corroboration:** the entity-browser-rust connect/peer/file-transfer audit and its
`PROPOSAL-NETWORK-EXTENSION-LIVENESS-BUILDOUT` (routed *to* us) + `AUDIT-SYNTHESIS-2026-07-14`.
Independently verified against the cohort tree (§B).
**Container note:** this proposal is the **home for the whole NETWORK liveness/reactive-lifecycle
cycle**. The spec deltas below are what we know now; more is expected to fall out of the build
(§E open work) — new deltas accrete here and fold under the same amendment before this closes.

---

## §1 The finding (one line)

`EXTENSION-NETWORK` is fully spec'd (v1.4, 11 amendments) but its **reactive half — the liveness
signal and the continuation-driven lifecycle — was never built**, because it was never gated by a
conformance vector; the **passive half** (session entity, transport profiles, held-capability
dispatch selection) *was* gated (Amendment 8) and *is* built. The single missing primitive is the
**write of `system/peer/status/{peer}`** on connection state change — the entity everything else
composes on.

## §2 Why we track this and not just patch it

This is the mistake worth recording as loudly as the fix: **eight amendments of prose review and a
3-way-green conformance run did not produce a working extension**, because the green covered the
data shapes, not the reactive behavior. That is our own CDN-corridor meta-rule turned back on us —
*a normative claim is not validated until a cross-impl conformance test exercises it* — and the
browser team rediscovered it from the consumer side (*"a standard without a gate is a suggestion"*).
NETWORK is the sharpest instance in the ecosystem. The correction is **not** "gate everything up
front" (see §D); it is "the reactive surface ships behind a convergence build with real vectors,
same as the passive surface did." Folding these deltas silently into the spec "as if they were
always there" would erase exactly the lesson. Hence: this proposal.

---

## §A Spec deltas (fold as Amendment 12)

### A1 — Transport-error is a liveness trigger (the missing fourth write-site)

**Problem.** Today `system/peer/status/{peer} = "disconnected"` is written from exactly three sites:
keepalive miss (§5.4), release (§4.2), graceful close (§4.4). None of them fire on the trigger that
actually bites: a **failed dispatch over a dead pooled connection**. §10 step 1 does
`send(connection, execute)` and §8.2 `on_outbound_dispatch` calls `send(connection, execute)` when
`connection.status == "active"` — and when that `send()` returns a transport `Err`, **nothing
updates the tree.** The error leaks to the caller as an opaque handler failure, and the next
dispatch repeats it. (Traced live by the origin audit: a `list` over a dropped pooled WS connection
returns `Err("… closed connection")`, surfaced as an opaque "handler error.")

**Delta.** A transport failure observed on a connection whose `system/connection/{peer}.status ==
"active"` MUST demote peer liveness, firing the same subscription the keepalive path fires:

```
; §10 step 1 (and §8.2 direct-send), on transport Err from an active connection:
send_result = send(connection, execute)
if send_result is transport_error:
  ; The connection we believed active is dead. Demote liveness reactively.
  update_peer_status(peer_id, "suspect", reason: "transport-error", last_error: <coded>)
  mark_connection_closed(peer_id)          ; system/connection/{peer}.status = "closed"
  ; This write fires the disconnect subscription → reconnect continuation (§4.1).
  ; Then re-run dispatch from step 1: the demotion means step 1 no longer matches,
  ; so the send re-resolves via held-cap + profiles (§10 steps 2-3) or queues (§8).
  return redispatch(peer_id, execute)      ; one retry through the reachability ladder
```

- **Seam discipline (normative MUST, mirrors Amendment 11).** The demotion write happens at the
  **dispatch caller that both observes the `send()` Err and holds `peer_id`** — never buried inside
  a transport/connection primitive that holds only a socket handle. (Same insertion-site rule as the
  §10.2 fallback seam.)
- **Idempotency / no-clobber (behavioral, not observable — ruled per Go rung-1 ask C).** The
  demotion MUST NOT clobber a concurrently-established live re-entry. The `Arc::ptr_eq` guard the
  cohort already uses at `remove_inbound` is the precedent: demote only if the failed connection is
  still the **currently-bound** one — *however the implementation tracks binding*. A tree read of
  `system/connection/{peer}` and internal pool pointer-identity are equally conformant; the
  observable contract of this delta is the `system/peer/status` write, not the guard's mechanism.
- **Demotion seam scope (pinned per Go rung-1 ask E — normative).** The demotion seams are the
  **direct-dispatch send sites** (§10 step 1 / §8.2) only. The §10.2 dispatch-fallback path and the
  RELAY terminal-hop forward (`SendRawFrameTo`, RELAY §3.1.1) **MUST NOT** demote peer liveness — a
  store-and-forward target is not "a connection believed active," and Mode-S fallback owns that
  path's failure semantics. (Left unpinned this is a cross-impl-observable divergence: one impl
  demoting on a failed relay-forward and another not yields divergent status for the same event.)
- **`suspect` vs `disconnected`.** A single transport error writes **`suspect`** (one failure is not
  proof of a dead peer — it may be this pooled socket only); the keepalive/grace path (§5.4) is what
  escalates `suspect → disconnected`. A consumer subscribed to status sees `suspect` immediately and
  can stop trusting the "Connected" label without yet tearing the session down.

This closes the browser team's #1 user-visible symptom ("browse dies on a dead pool") *in the spec*,
not just in one app.

### A2 — `reason` + `last_error` on the peer-status entity

**Problem.** `system/peer/status` carries a bare lifecycle enum (`connected` / `suspect` /
`disconnected` / `reconnecting`). A consumer — and the reconnect continuation itself — cannot tell
*why* the peer left, so it cannot choose a recovery: a transient drop wants backoff-reconnect; an
auth rejection (403) wants re-handshake (§6.3); a deliberate peer shutdown wants *stop trying*. §10
has this fork for **dispatch** (403 → handshake), but the **liveness** path collapses every cause to
"disconnected." This is precisely the "interfaces to track errors / reset if needed" the consumer
asked for.

**Delta.** Add two OPTIONAL fields to the status entity:

```
system/peer/status (extends the §3.13 operational entity):
  status:      "connected" | "suspect" | "disconnected"
             ; unchanged — the ENTITY-CORE-PROTOCOL §3.13 three-state enum. CORRECTION
             ; (rung-1 ask D): an earlier draft of this delta listed "reconnecting" here;
             ; that is a system/network/peer-summary (§2.8) derived value, NOT a status-
             ; entity state. Arch-authored D-class divergence, caught at rung-1 review.
  reason?:     "transport-error" | "keepalive-miss" | "auth-rejected"
             | "peer-shutdown" | "peer-idle" | "peer-migration" | "local-release"
             ; OPTIONAL kebab enum; why the status last changed. Reader treats an
             ; unrecognized value as generic (MUST-ignore-unknowns) and falls back to backoff.
  last_error?: text                ; OPTIONAL; coded/opaque detail for humans + logs, never parsed
```

**Canonical shape + put-site rule (ruled per Go rung-1 ask D).** The full field set of
`system/peer/status` is declared **once**, at ENTITY-CORE-PROTOCOL §3.13 (`peer_id`, `status`
required; `connected_at`, `last_seen`, `connection` OPTIONAL). NETWORK's put-sites (§4.2, §6.2,
this delta) are **minimal writes, not exhaustive shapes** — a bare `{peer_id, status}` write is
conformant; no impl derives the type shape from an example write. Amendment 12 adds this pointer
line at the NETWORK put-sites. `last_seen` is refreshed at most at keepalive cadence (never
per-message — §6.6); whether keepalive-cadence refresh is noisy for lifecycle subscribers is a
rung-2 observation item, not pre-legislated.

Recovery mapping the reconnect continuation reads off `reason`:

| `reason` | Recovery |
|---|---|
| `transport-error`, `keepalive-miss` | backoff-reconnect (§4.1) |
| `auth-rejected` | re-handshake (§6.3), then reconnect — do NOT reuse the held cap |
| `peer-shutdown`, `local-release` | terminal — stop; session ended deliberately (§6.1) |
| `peer-idle`, `peer-migration` | preserve subscriptions; expect resume (§9.1) |

- Keys `reason` / `last_error` are snake-adjacent single tokens; values are kebab per
  `STYLE-NAMING-CONVENTIONS`.
- **Declaration home — RULED (rung-1 ask A): §3.13 note, upstream.** The fields land in
  ENTITY-CORE-PROTOCOL §3.13 — the type's single canonical declaration home — precisely because of
  what ask D demonstrated: two specs each declaring fields on one entity type is how impls end up
  with two shapes. Additive OPTIONAL (`omitempty`), no wire change, no V7 renumbering. The §3.13
  delta travels to `entity-core-protocol` as a routed spec delta at fold (we don't edit the sibling
  repo from here). NETWORK keeps the *lifecycle semantics* (the enum meanings + recovery mapping,
  this section) and cross-references §3.13. Go's additive `omitempty` landing is conformant as-is.

### A3 — The minimal composable slice, named as the build floor

**Delta (framing, not new mechanism).** §4.1 `maintain-peer` bundles connect + subscription +
reconnect-continuation + keepalive + pending-drain as one operation. That is the *full* build. The
spec should name the **minimal composable slice** that every consumer actually blocks on, so the
cohort can ship it first and independently:

> **The liveness slice.** Write `system/peer/status/{peer}` on connection state change — on establish
> (`connected`, §6.2), on transport error (`suspect`, §A1), on keepalive miss (`suspect →
> disconnected`, §5.4) — and nothing more. These are ordinary tree entities → subscribable via
> `system/subscription` → consumers react without polling. This slice requires **none** of
> `maintain-peer`, the continuation graph, or the outbox; those compose on top of it.

This is a §12 conformance note ("A1 liveness slice is independently conformant and is the required
floor; the `maintain-peer` automation of §4.1 SHOULD build on it") — it does not change any
mechanism, it names a shipping boundary.

**Floor clarifications (ruled per Go rung-1 asks B + C):**

- **Keepalive is inside the floor.** The slice's three writes include the keepalive-miss demotion,
  so the floor = the status writes **plus** the §5 keepalive loop (Go's rungs 1+2 together). §12.1
  already makes keepalive MUST; rung 1 alone is a build stage, not a claimable conformance point.
  **Consumer latency contract:** a consumer of any NETWORK-conformant peer MAY rely on an idle-dead
  connection demoting within the keepalive envelope (`interval_ms × max_missed + timeout_ms`;
  defaults ≈ 100 s; values impl-defined per §12.4).
- **`system/connection/{peer}` is NOT part of the floor.** It stays MUST at full NETWORK
  conformance (§12.1), write-on-transition per ENTITY-CORE-PROTOCOL §3.13 ("written … on connection
  establishment; updated on close or failure"). The floor's observable contract is
  `system/peer/status` only; a consumer of a floor-only peer MUST NOT assume the connection entity.
  Division of labor: `status` answers *is the peer here* (subscribe); `connection` answers *how am
  I attached right now* (read-on-demand diagnostics).

### A4 — The status entity is transition-written; cadence freshness is impl-internal (rung-2 ruling)

**Problem (quantified by the Go rung-2 build — the ruling-D observation deliverable).** §5.4's
success path `update_last_seen(peer_id, now())` is a tree write per keepalive tick on
`system/peer/status/{peer}` — the entity this amendment establishes as *the subscribed lifecycle
signal*. Cost, measured: 2 non-transition events/min/peer/side to every lifecycle subscriber (who
must all diff for transitions), and ~2,880 superseded status entities/day/peer accreting in the
CAS (nothing GCs operational state). The spec's own rationale convicts it: §6.6 rejected
`last_active` on the session entity *because* cadence writes mean "write amplification →
subscription/revision/history fan-out" — then §5.4 put the same amplification on the subscribed
entity.

**Delta.**

- `system/peer/status/{peer}` is written **on transitions only** (`status` or `reason` change).
- §5.4 `update_last_seen` becomes impl-internal bookkeeping (it already must exist for §5.4
  adaptive suppression); it is **not** a tree write. Verbatim target: §5.4 keepalive-loop success
  branch.
- §6.6 `granted_at` bullet: "(keepalive-updated, …)" → "(snapshot at transition; per-tick
  freshness is implementation-internal)". `last_seen` stays §3.13-OPTIONAL; its semantics at a
  transition write are "last heard as of this transition" (on a demotion write, the demotion's
  evidence).
- **GC note:** superseded operational-state entities (§3.13 SHOULD-populate self-description,
  explicitly non-durable) are **GC-eligible**; mechanism impl-defined.
- Liveness freshness for consumers is already the ruling-B envelope (`status == connected` +
  demotion within `interval × max_missed + timeout`); a cadence-visible heartbeat surface is
  deferred until a consumer demonstrates the need.

### A5 — Advertise-what-you-dispatch: `ping` in the connect manifest (rung-2 cohort-probe finding)

Rust and Python both *answer* the §5.1 ping conformantly but neither *advertises* `ping` in the
`system/protocol/connect` operations manifest — dispatchable-but-undeclared, because §5.1 shows the
exchange but never pins the manifest entry. **Delta:** every operation a handler dispatches on an
established connection MUST appear in that handler's operations manifest; concretely, `ping` MUST
appear in the connect handler's operations map (established connections only, cap-free like the
handshake ops — matching the Go landing). Second incident of the "observable surface shown by
example, never pinned" authoring class (first: the rung-1 declaration-home coda) — the general
authoring rule is drafted for `SPECIFICATION-FORMAT.md`'s next pass.

### A6 — The retry lifecycle: self-handling loop, retry-forever normative, retry state in the tree (rung-3 rulings)

**Problem (observed three-way at rung-3 green).** The failure path is under-specified in five
compounding ways: the reconnect loop has **no exit edge**, `"exponential"` has **no attempt
counter and no delay mechanism** in the spec's own model, and — the root — §4.1's
`backoff_continuation` is written with **no `on_error`**, so every failed retry is precisely
CONTINUATION §3.4's **v1.13 case** (a no-`on_error` forward continuation receiving a non-2xx;
`maintain-peer` returns `502 connection_failed`) and MUST-binds a lost marker. **The retry loop is
a marker generator by construction**: ~1,440 markers/day/dead-peer, forever. Full rulings +
rationale: `docs/research/reviews/ARCH-RESPONSE-NETWORK-RECONNECT-LIFECYCLE.md`.

**A6.0 — The backoff continuation is STANDING (`remaining_executions: null`) — the cohort blocker,
ruled.** §4.1 installs it `remaining_executions: 1 ; one-shot per retry`, intending
`maintain-peer` to re-arm it. **That is unimplementable by ordering, not by timing:** the re-install
lands *inside* the dispatch, and the advance's consume runs *after* the dispatch returns and deletes
what it just installed.

> **An operation cannot re-arm its own trigger through a continuation that consumes itself around
> the dispatch.** The one-shot always loses the race.

Measured three-way through three independent observables — Go (dials at an external socket), Rust
(§3.10 markers in-process), Python (dials on its own wire) — all agreeing on **2 attempts**. Cost of
the defect: **an offline peer is never recovered.** Independent of §A6.1a: `remaining_executions`
decrements on every *completed* dispatch, success or not, so returning 200-on-armed does not save
the one-shot; both deltas are required. **General hazard for the programming guide:** any *re-arm
by re-dispatch* pattern MUST use a standing continuation.

**A6.1 — The loop handles its own failure (the root delta).** §4.1's `backoff_continuation` MUST
carry an `on_error` routing to the retry re-arm. A reconnect failure is neither *silent*
(`system/peer/status` broadcasts it — this amendment's whole purpose) nor *unhandled* (the loop
re-arms; that **is** the handler), so binding a silent-burn marker for it is a category error. With
the `on_error` present, the v1.13 case stops firing for expected retry failures and the marker is
left doing its job — catching the **exceptional** failure (§3.4 A.1: the `on_error` dispatch itself
failing). A dead peer's marker tree goes from ~1,440 nodes/day to **empty**.

> **Guidance correction (supersedes the rung-1 handoff AND the marker proposal §5 bullet, which is
> struck).** The handoff said *route `on_error` to a chain-errors sink, **never** to
> `system/inbox/*`*. Too broad — and left this very delta with no legal target, a contradiction
> between two arch documents that the Go seat caught before anyone coded against it. Correct rule:
> **inbox-routing advances the continuation bound there — use it when the error is *meant* to drive
> the next step (a retry); use a `chain-errors` sink for *passive observation*.** The trap is
> unintended advancement, not inbox-routing itself.

> **Backoff install path — adopt the cohort's converged shape.** §4.1's pseudocode binds the
> backoff at `system/inbox/network/{peer}/on-reconnect-backoff`; **all three impls instead ship it
> in the managed namespace.** That is not drift — it is three-way convergence, the strongest signal
> this method produces, against stale pseudocode. §4.1 adopts the managed-namespace shape the
> cohort built. (Left unfixed, a stale pseudocode block re-arms the contradiction on every new
> reader.)

**A6.1a — `maintain-peer` returns 200 on armed re-entry (the second marker family, killed at its
source).** §4.1 returns `error(502, "connection_failed")` when connect fails — so the backoff
continuation's own re-EXECUTE receives a non-2xx and binds a second marker family
(`network-advance-*`). **The 502 is a misreport:** `maintain-peer`'s contract is *"maintain a
relationship,"* not *"connect right now."* With `reconnect: true`, a failed initial connect means
the operation **succeeded** — the maintained relationship exists and is in retry.

- **200** when the relationship is armed (`reconnect` enabled), regardless of the initial connect
  outcome. **502** only when `reconnect: false` — then a failed connect really is the operation
  failing, with no loop to handle it.
- **No `status` field on `maintain-result`.** `system/peer/status/{peer}` is the canonical liveness
  home; duplicating it into the result would reintroduce the mirror this amendment exists to
  delete. §4.1 states plainly that **200 means "established and armed," not "connected"** — the
  caller reads the status entity for liveness.
- Both marker families thus die from **status honesty**, not bolted-on error plumbing: A6.1 kills
  `notif-sub-*`, A6.1a kills `network-advance-*`. A second `on_error` would have treated the
  symptom.

**A6.2 — Retry-forever is intended; say it normatively.** §4.1 gains the statement: a maintained
peer relationship retries **indefinitely** by default; `release-peer` (§4.2) is the exit. This is
the substrate's semantic, not an oversight — a peer offline for a week and returning is the P2P
norm, and the "give up after N" instinct imports a client-server assumption that does not hold.
Stated normatively so no impl invents its own cap and diverges silently.

**A6.3 — §2.2 gains OPTIONAL bounds.** `max_attempts` and `max_elapsed_ms`, **default unset =
retry forever** (additive; today's behavior preserved). For deployments that do want give-up
(mobile, constrained devices, short-lived agents).

**A6.4 — The terminal is a `reason`, not a new state.** Exhausting a configured bound writes
`status: disconnected` + **`reason: retry-exhausted`** (new value in the §A2 vocabulary; recovery
mapping: **terminal — stop; cleared only by an explicit re-`maintain-peer`**). No fourth enum value:
rung 1 ruled `system/peer/status` a 3-state enum (§3.13), and `reason` is exactly the field that
carries *why, and what to do about it* while `status` carries *where the peer is*.

**A6.5 — Retry state lives in the tree, as one transition-written field.**

- **`failing_since`** (epoch ms, OPTIONAL) — set on the transition into `suspect`/`disconnected`,
  cleared on the transition back to `connected`. A **transition write** → free under §A4, no
  per-attempt fan-out, no CAS accretion. Joins `reason`/`last_error` in the routed §3.13 delta
  (one batch, one trip).
- **`attempt` and `next_attempt_at` are DERIVED, never stored** — computable by any reader from
  `(failing_since, now, §2.2 config)`. Storing them would mean a tree write per attempt: exactly
  the amplification §6.6 rejected and §A4 removed.
- **The inversion is PINNED normatively** (per the Go rung-3 ask — an unpinned derivation just
  relocates the divergence: three impls would invert the series three ways):

```
; Retry schedule — a PURE FUNCTION of (failing_since, backoff-config §2.2).

delay(k, cfg)  =                              ; delay BEFORE the k-th retry; k is 1-indexed
    exponential : min(min_ms * 2^(k-1), max_ms)     ; §2.2 default
    linear      : min(min_ms * k,       max_ms)
    constant    : min_ms

elapsed_to(n, cfg) = Σ(k=1..n) delay(k, cfg)  ; elapsed_to(0) = 0
                                              ; ms after failing_since at which retry n fires

attempt(now)    = max { n ≥ 0 : elapsed_to(n, cfg) ≤ (now − failing_since) }
next_attempt_at = failing_since + elapsed_to(attempt(now) + 1, cfg)

retry_exhausted ⟺ (max_attempts   set ∧ attempt(now) ≥ max_attempts)      ; §A6.3
                ∨ (max_elapsed_ms set ∧ (now − failing_since) ≥ max_elapsed_ms)
```

  Pinned explicitly at the three divergence points: **`k` is 1-indexed** (first retry is `k=1`);
  **`attempt` counts retries *fired*** (0 immediately after `failing_since` — the drop itself is
  not a retry); **`elapsed_to(0) = 0`**. Worked example (defaults, exponential): delays
  `1,2,4,8,16,32,60,60…s` → `elapsed_to = 1,3,7,15,31,63…s`; at `now − failing_since = 10s`,
  `attempt = 3` and `next_attempt_at = failing_since + 15s`. **No jitter in v1** (deterministic;
  cross-peer decorrelation comes free from the natural spread in `failing_since`) — deferred until
  a deployment demonstrates thundering-herd.
- **Payoff:** because the schedule is a pure function, its conformance vector is a **pure-function
  table** — `(failing_since, cfg, now) → (attempt, next_attempt_at)`. No peers, no sockets, no
  timing flake. The most divergence-prone surface in the retry design becomes the most cheaply
  converged one.
- This closes `"exponential is unimplementable"` (the counter has a defined home; it simply isn't a
  stored field) and fixes **restart-hammering** for free: with `failing_since` in the tree, a
  restarting peer resumes at the **correct escalation** instead of dropping back to `min_ms`.
- **The delay mechanism:** the **timer is host-provided; the schedule is spec-defined.** §5.4's
  pseudocode already assumes a host `sleep(interval_ms)` — this is an existing blessed pattern, not
  a new concession. §4.1 states it so Rust/Py don't each invent a different shape and call it
  backoff.

**A6.6 — §4.1's `chain_id` is slash-free; correlation is the `parent_chain_id` edge.** Format
changes to `"network-maintain-" + session_id` (was `"network/maintain/" + session_id` — three path
segments, which made the marker path unparseable; see the marker proposal §3a). §4.1 is the one
known violator of a constraint that belongs at **ENTITY-CORE-PROTOCOL §3.11**, where `chain_id` is
actually declared: *`chain_id`/`parent_chain_id` MUST each be a single path segment* — routed with
the §3.13 delta as one core-protocol trip. Not a compatibility event: no installed base
(AGENTS-STANDARD), and `chain_id` is opaque (§3.11 gives it no format; nothing parses it), so the
correction is three impls emitting a different generated string plus a probe update.

**Correlation — the handler supplies the chain_id it already has (2026-07-16; supersedes the
parent-edge framing).** §3.6 step 6 mints a fresh chain only when `context.chain_id` is **absent**
(a root dispatch); everything downstream inherits. A retry dispatch is a *fresh root* by that rule
(timer-fired, no inbound context), so it mints a fresh id and the lifecycle id appears nowhere.

> **A handler that originates a dispatch belonging to a known chain MUST set `bounds.chain_id` to
> that chain's id.** For NETWORK: the backoff / retry / keepalive dispatches carry
> `network-maintain-{session}` — which the handler already holds on the session.

Every failure of one relationship then lands under **one** node
(`lost/network-maintain-{session}/…`). **`parent_chain_id` is RESERVED — do not implement**; it has
no setter in the current model (no dispatch mints a sub-chain), and an earlier arch ruling requiring
the parent edge is **withdrawn**. Full corrected ruling: marker proposal §2b.

**Depth rides the same signal (cross-ref).** The very "timer-fired, no inbound context" property that
makes a retry a *fresh root* for `chain_id` also roots `chain_depth` at 0 — so the backoff loop
retries forever (ruling 6) without ever tripping the `chain_depth` bound, even though its stable
`chain_id` correlates every retry. Identity and depth are separate axes; the wire-carried `chain_depth`
bound that makes NETWORK's *cross-peer* advancement chains terminate is
`PROPOSAL-CONTINUATION-BOUNDS-PROPAGATION` §5 (the causal-vs-standing distinction is built on exactly
this NETWORK case).

---

## §B Verification (proven negative, per AGENTS-STANDARD)

> **⚠ POINT-IN-TIME SNAPSHOT — HELD 2026-07-14, NOW SUPERSEDED. DO NOT RE-CITE AS CURRENT.** The negative
> below was true on 2026-07-14 and was **superseded within ~24 hours**: the §C rungs 1–3 landed 2026-07-15…07-17
> and were **validated green three-way** (reports `2026-07-15-a12-liveness-harness` — Go/Rust 4/4, Py 3+1 WARN;
> `2026-07-19-reconnect-disconnect-crossimpl` — Go/Rust/Py 5/5), re-proven live on the wire 2026-07-28. A
> proven-negative is a timestamped claim; re-asserting it later requires a **fresh** wire run, never a re-read
> (this fossil was mis-cited on 07-28 to flip the tracker to "unbuilt" — see `docs/DOCTRINE-COHORT-STATE-TRACKING.md`).

Checked against the cohort tree, not memory — **as of 2026-07-14:**

- `maintain-peer` / `"system/network"` handler — **zero hits** in `entity-core-rust`; no
  `maintain-peer` in `entity-core-go`. The NETWORK handler does not exist.
- No `system/peer/status` write of `connected`/`disconnected`/`suspect` anywhere in
  `entity-core-rust/core/peer/`. The liveness write does not exist.
- `remove_inbound` (`entity-core-rust/core/peer/src/remote.rs:510`) — the disconnect seam — mutates
  the in-memory pool only; no tree write.
- **Contrast (the passive half that IS built + green):** `session_entity.rs`, `transport_profile.rs`,
  `held_capability`, reachability-class selection all present in `entity-core-rust/core/peer/` —
  Amendment 8 landed 3-way. Its own doc-comment (`session_entity.rs:15`) *names*
  `system/peer/status/{peer}` as the liveness home — and nothing writes it. That comment is the
  fossil of the gap.

The passive/reactive split in §1 is this evidence, not an assertion.

---

## §C What the cohort builds (and in what order)

1. **A1 liveness slice** — the three status writes + `reason`/`last_error`. First deliverable;
   unblocks every consumer (the browser team deletes its `connection_health` mirror against it).
2. **Keepalive loop** (§5) — the `suspect → disconnected` escalation the slice's grace path needs.
3. **`maintain-peer` + the continuation graph** (§4.1) — reconnect-on-disconnect, subscription
   restoration (§7), the backoff continuation, `on_error` routing.
4. **Pending delivery** (§8) — outbox queue + drain on resume. (Note: Rust + Python ship no §8
   outbox today per Amendment 11; their step-4 terminal is a bare error. §8 is the OPTIONAL rung.)

Each rung is independently observable → each converges its own vectors (§D).

---

## §D Validation: convergence build, NOT vectors-first

**Rejected approach (recorded so the reasoning is on disk):** an earlier suggestion in this cycle was
to *write the conformance vectors first, then have the cohort build against them.* **Rejected.** Our
method is the opposite and for good reason: **the peers implement, and cross-impl convergence
produces the vectors.** Vectors authored ahead of a real implementation encode one author's guessed
shape and mislead the build (a vector can assert the wrong thing — ADR-0012); vectors distilled from
three impls reconciling their behavior encode what actually had to agree. Amendment 8 landed exactly
this way (R6 validated 7/7, §10 priority-selection 3-way green — *after* the impls built it). The
gate that was missing for the reactive half was never "vectors up front"; it was "run the reactive
surface through the same convergence the passive surface got."

**So the process is:** fold Amendment 12 (§A) → cohort builds §C rung by rung → each rung's
directional behavior (status flips on connect / transport-error / keepalive-miss; reconnect
continuation fires; pending drains on resume) is reconciled cross-impl → the reconciled behavior
**becomes** the vector suite → that suite is the standing gate that keeps the reactive half from
drifting back to unbuilt. Convergence first, vectors as its residue, gate thereafter.

---

## §E The real weight of this: NETWORK is our first continuation-shipping extension

Continuations have been a metasystem primitive we exercised in practice; **NETWORK is the first
extension that ships them as a load-bearing product feature** — the reconnect/backoff/restore
lifecycle *is* a continuation graph in the tree (§4.1). That is new operational surface, and it is
where most of the unknowns live. Two of these now have concrete direction absorbed from
workbench-go, the ecosystem's most mature continuation consumer — see
`docs/research/reviews/ABSORPTION-workbench-continuation-error-model.md`. Open work this proposal
opens (deltas expected to land back here):

- **Observability.** How does an operator *see* a live continuation graph — the pending reconnect,
  the backoff timer, the chain that is mid-flight? `maintain-result.chain_id` names it. **Direction
  (absorbed):** workbench's `ContinuationView` + `TraceChain(chain_id)` are the worked shape, but
  *continuations have no spec-level observability* — it is all SDK convention, which is why
  browser-rust has none. **Spec ask:** lift a minimal `ContinuationView` + trace-by-`chain_id` into a
  cross-impl observability contract, and have **NETWORK declare its continuation completion contract**
  (what "reconnect chain completed" looks like in the tree) so trace can tell success from never-ran
  (DIAGNOSTIC-DIRECTION §2.1 #8). (Ties to the browser team's 7-window sprawl → one derived view.)
- **Error comprehension.** There are **two** error surfaces (CONTINUATION §3.4), not one: `on_error`
  (author-set compensation delivery, best-effort, *silently lost by design*) and the **lost-error
  marker** (`system/runtime/chain-error-lost`, the substrate's `SHOULD`-bind observability backstop).
  **Two findings:** (1) `on_error` MUST route to a `system/runtime/chain-errors/*` sink, **never** to
  `system/inbox/*` — the inbox handler drives advancement, so an error would spuriously fire another
  step (workbench's documented trap; corrects the earlier §4.1 "route to inbox" instinct). (2) The
  lost-error marker is a **non-uniform `SHOULD`** (impls diverged: Rust `cascade_depth` vs Go
  `RequestID`) — so a reconnect chain's failures are observable in Go and invisible off-Go. **Spec
  ask:** elevate the lost-error marker to **`MUST` for extension-shipped continuations** (NETWORK is
  the forcing case). The `reason` field (§A2) is the per-status brick; this is the chain-level brick.
  **Blocking rider (Go rung-1 ask F2 — the catch of the cycle):** bare MUST-bind is a conformance
  trap — the bind itself can 403 when the chain's propagated cap doesn't cover
  `system/runtime/chain-errors/**` (Go's F11 record), re-creating the black box for exactly the
  most-broken chains. The marker proposal MUST resolve write authority before shipping the
  elevation. Two candidate resolutions (see `ARCH-RESPONSE-NETWORK-A12-RUNG1-GO.md` §F2): (i) pair
  MUST-bind with MUST-provision (chain carries the marker-path cap — Go's framing); (ii) **arch's
  lean** — re-home the marker write under the **continuation handler's own grant** (the NETWORK-§11
  managed-namespace pattern): the marker is the *substrate's* observation, not the chain's action,
  so author surface (`on_error` delivery) rides the author's cap while the substrate surface (the
  lost marker) rides substrate authority — MUST-bind becomes unconditionally satisfiable, and the
  backstop can't be broken by the very misconfiguration it exists to observe. **RESOLVED (rung-2):**
  Go verified (ii) feasible with no cap-check rework (`selectCapability` handler-grant fallback +
  the §3.10.7 component-authority precedent); the elevation is now drafted as
  `PROPOSAL-CONTINUATION-LOST-ERROR-MARKER-MUST.md`, routing to Rust/Py for their feasibility
  check.
- **Reset / re-establish semantics.** "Reset if needed" — is that `release-peer` + `maintain-peer`,
  or do we need an explicit reset that tears the continuation graph without discarding the session?
  §4.2 release + §6.2 resume cover parts; the clean "reset this relationship" verb is unspecified.
- **Testing a reactive graph.** How do we drive + assert a continuation lifecycle in a test — inject
  a disconnect, assert the graph advanced, assert idempotency under concurrent re-entry? This is the
  thing the browser team flagged (the shipped Direct arm was e2e-blind). The convergence build (§D)
  is where this test methodology gets forged; it is a deliverable, not a given.

These are named, not solved. They are why this proposal is a **container** (header note), not a
one-shot patch.

---

## §F Non-goals

- Not building discovery / relay / NAT / sync-under-partition (DISCOVERY, RELAY, REVISION, and the
  §10.2 fallback seam already own those).
- Not the REGISTRY petname/authorized half of "remembered peer" (that is EXTENSION-REGISTRY local-name
  + capability, per the pull-in map — do not fold it into NETWORK).
- Not the poll-fallback for non-listening targets (R4 / `delivery_mode: poll`, Phase-2).
- Not changing the wire core, adding an opcode, or adding a capability/error code. §A1 reuses the
  existing status entity + subscription mechanism; §A2 is an additive optional field.

## §G Conformance (converged, per §D — not pre-authored)

Emerges from the cohort build; expected shape:

- **A1 (MUST).** A transport error on an active connection writes `system/peer/status = suspect` +
  fires the disconnect subscription; the write happens at the dispatch caller (seam rule); it is
  idempotent under concurrent re-entry. Directional vector: two-peer WS session, drop one side,
  assert the *other* side's status flips without a poll.
- **A2 (SHOULD).** `reason` is set on every status transition; a reader treats an unknown `reason` as
  generic-backoff. **Ownership CLOSED (arch, 2026-07-28):** fields declared upstream at core §3.13
  (see §A2 declaration-home ruling); NETWORK owns the enum semantics. No longer blocked — rides §A1.
- **A3 (MUST).** The liveness slice is independently conformant with no `maintain-peer` / outbox
  present.
- **Full lifecycle (SHOULD, per rung §C).** reconnect continuation fires on the status write;
  subscriptions restore (§7); pending drains in sequence on resume (§8.3).
