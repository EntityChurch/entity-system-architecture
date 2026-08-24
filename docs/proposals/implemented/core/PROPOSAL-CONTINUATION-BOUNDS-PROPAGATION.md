# PROPOSAL — Cross-peer chain bound: wire `chain_depth`, pin cross-peer TTL, de-confound the two

**Status:** rev. 2026-07-19 · **CORE-PROTOCOL DELTAS FOLDED 2026-07-24** — the `ENTITY-CORE-PROTOCOL` §3.11
(`chain_depth` field + `chain_id` single-segment pin), §5.9 (Ruling 1 one-decrement-per-dispatch TTL + Ruling 2
de-confound to ttl 512 = 8× the chain_depth 64 ceiling), and §4.10(b) (Ruling 3 reason-code disambiguation) are
applied to `entity-core-protocol` as amendment **0.8.1** (commit `1fcda2b`). **CONTINUATION-extension-side
reconciliations FOLDED 2026-07-25** (this repo) — §3.6 step-6 refill→decrement + `chain_depth` inherit, §3.7
resume-roots-depth-0, §3.9 wired framing (causal-vs-standing root + `bounds_exceeded` reason, replacing the stale
"not carried on the wire" note), §6.2 monotonic clause — fixed in-place in
`specs/extensions/EXTENSION-CONTINUATION.md`, no rev bump. **VALIDATED THREE-WAY GREEN 2026-07-27** — go/rust/py
`continuation_bounds` cb1/cb2/cb3 = 3/3 each; `cbx_crosspeer_chain_bounds_globally` (anchor-1) PASS on go↔rust
and go↔python; depth-brake marker `bounds_exceeded`/429 byte-consistent. Rust's WARN (2026-07-25) closed by
`e4da0bc` (reactive-path marker self-bind). Report: `entity-core-go/docs/validation/reports/2026-07-27-
continuation-bounds-three-way-GREEN.md`. **Cleared for the 28-peer CBX ratify** (the formal gate for the whole
0.8.1 bundle). Seed ratio **ratified at 8× (512)** (§4a Ruling 2). **O1** (concrete causal-vs-standing keying
signal) — the depth-brake gate converged the *observable*, not the signal; still an open pin, twinned with the
standing-model O1 below. _(Was: DRAFT 2026-07-17, rev. 2026-07-19 against go `e93d36e` / rust `c07cb6c` / py `8fc36e3`;
the "TTL refills" premise was falsified and corrected.)_
**Target:**
- `specs/extensions/EXTENSION-CONTINUATION.md` §3.6 (step 6 — refill→decrement), §3.7 (resume),
  §3.9 (suspension / chain-depth / reason code), §6.2. Fixed in place — cohort finding against
  **landed** CONTINUATION, no rev bump.
- `ENTITY-CORE-PROTOCOL` (we own it; governing record here, lands there at fold):
  **§3.11** — add `chain_depth` to `system/bounds` (all three impls ship it on the wire; the protocol
  spec doesn't list it) and **pin `chain_id` to a single path segment** (currently only a UUID default,
  no format); set the `chain_depth` default ceiling **distinct from** the `ttl` default (§4a Ruling 2).
  **§5.9** — pin cross-peer TTL: uniform default seed (ratio **8× the `chain_depth` ceiling**) +
  **one decrement per *dispatch* (incl. sub-dispatches), never double-counted** (§4a Ruling 1,
  rev. 2026-07-22 — TTL is the resource backstop, orthogonal to the per-hop `chain_depth`). **§4.10(b)** — note that `chain_depth_exceeded` (400) is the *capability*-chain limit and
  the continuation depth brake uses `bounds_exceeded` (429) (§4a Ruling 3).

**Origin:** entity-core-go's step-6 spec-issue (`086705e`) → three-way build → the 2026-07-19 cycle-close
found the runaway bounded on all three impls but at **divergent hop counts** (Rust ~64, Go/Py ~9) with
the depth brake masked by TTL. This rev resolves that: the design converged; the bound was not
deterministic. Answers ruling **15** of `ROUTING-2026-07-16`.

---

## §1 Problem — a cross-peer continuation chain has no *deterministic* global bound

The goal: a cross-peer causal continuation chain (peer A advances → dispatches to B → B advances →
dispatches back to A → …) must terminate, **the same way on every impl**. As built and verified across
`entity-core-go@e93d36e`, `entity-core-rust@c07cb6c`, `entity-core-py@8fc36e3`, it does not — for three
reasons, none of them a design disagreement (all three impls converged on the *same* design):

1. **`chain_depth` originally could not cross the wire** — it was a per-peer execution-context counter
   that reset at every boundary (landed CONTINUATION §3.9). **Now fixed, three-way:** `chain_depth` is
   a `system/bounds` wire field (Go `core/types/system.go`, Rust `core_types.rs:656`, Py
   `bounds.py:76`), inherited across the boundary, ceiling **64** on all three. The global causal-depth
   brake exists and works single-peer.

2. **Cross-peer TTL is unspecified, so the impls diverge.** `ENTITY-CORE-PROTOCOL §5.9` says only
   *"TTL decrement at dispatch layer"* (default `ttl = 64`) with **no** cross-peer rule (contrast
   `cascade_depth`, which has an explicit "don't reset across the boundary" clause). The three impls
   fill that vacuum differently: **Rust seeds no default TTL** (`ttl = None` → never exhausts → the
   depth-64 brake is the sole terminator → ~64 hops, but *no resource backstop*); **Python seeds
   `DEFAULT_TTL = 64` and decrements at multiple sites per hop** (ingress + each sub-dispatch → ~9
   hops); **Go decrements ~4/hop from 64** (~9–16). Same design, divergent TTL mechanics → a cross-peer
   chain bounds at a **different length on every pair** (the "9-vs-64" split). This is a conformance
   bug, not a design choice.

3. **The confound: TTL default (64) == `chain_depth` ceiling (64).** Two independent bounds at one
   magnitude. Cross-peer, TTL decrements faster than depth increments, so TTL fires first and **masks**
   the depth brake — which is why the depth brake has never been cleanly demonstrated cross-peer (the
   "anchor-1" open item). At 64 it is impossible to tell which brake stopped the chain.

**Correction to this proposal's earlier draft.** It argued *"step 6 refills TTL to `peer_default_ttl`,
so TTL cannot bound a cross-peer chain."* That is **false against every impl**: none refills — all
decrement, matching §5.9 (Go's code comment calls decrement "the stricter reading" of step 6). So TTL
*does* bound cross-peer chains; it just does so **non-deterministically**. `chain_depth`'s real
justification is therefore **determinism** — a resource-independent chain-length ceiling identical on
every impl — not "TTL can't do it." CONTINUATION §3.6 step 6's `ttl: peer_default_ttl` language
contradicts §5.9's decrement rule and the shipped behavior; it is corrected in §7.

## §2 The mechanism already exists — `cascade_depth`. Read the source, don't invent.

The protocol already solves this exact problem for the *other* recursion axis. `cascade_depth` is a
`system/bounds` field (**ENTITY-CORE-PROTOCOL §3.11**) that:

- rides in bounds **on the wire** (`SYSTEM-COMPOSITION.md` §3.4, "cascade_depth on bounds");
- is **inherited across the peer boundary** — *"the receiving peer reads `cascade_depth` from the
  incoming EXECUTE's bounds and initializes its local cascade from that value… it works on the first
  cross-peer hop without any prior state"* (§3.4);
- bounds a **causal recursion**, while a *fresh external notification* roots a new cascade (async
  cross-peer delivery escapes the synchronous recursion — `SYSTEM-COMPOSITION.md` §4.2).

`chain_depth` is the same structural object for the continuation-advancement axis and behaves the same
way — a wire-carried, inherited bounds field parallel to `cascade_depth`, **not a new mechanism.** All
three impls built it exactly so (Go `core/types/system.go`, Rust `core_types.rs:656`, Py `bounds.py:76`).
This section is now the *confirming precedent*, not a request; §3.11 must be amended to **list** the
field the impls already ship.

## §3 Delta 1 — bounds propagate on cross-peer dispatch (LANDED three-way)

**Cross-impl-observable → MUST**, and **adopted**: every cross-peer `EXECUTE` originated by a
continuation advancement carries `system/bounds` with `chain_id`, `chain_depth`, `ttl`, `budget` (Go
`local.go:259-275`, Rust `connection.rs:2403-2419`, Py `_wire_bounds_for_dispatch`). This rev does not
re-open it; it pins the *values* that ride (TTL mechanics, §4a) and the §3.11 field list. Every cross-peer
`EXECUTE`
with `chain_id`, `chain_depth`, `ttl`, and `budget` populated. The remote-dispatch path MUST NOT drop
bounds (core-go finding 2: today it does). `chain_id` and `chain_depth` are **immutable-preserved /
inherited** on the wire in the sense §3.11 already gives `chain_id` and `cascade_depth`; `ttl` and
`budget` **decrement** per §5.9 / §4a below (not refilled).

## §4 Delta 2 — `chain_depth` becomes a `system/bounds` wire field (§3.11 + §3.9)

**ENTITY-CORE-PROTOCOL §3.11:** add `chain_depth` to `system/bounds` alongside `cascade_depth` —
non-negative integer counter (unlike opaque `chain_id`), inherited across the wire, initialized on the
receiver from the incoming bounds value.

**CONTINUATION §3.9** — replace the per-peer-context framing with the wired one:

> **`chain_depth` is carried in `system/bounds` (§3.11) and inherited across peer boundaries**, exactly
> as `cascade_depth` is. On a continuation advancement dispatch that is *causally within* a triggering
> advancement (§5), the dispatch layer sets `bounds.chain_depth = triggering.bounds.chain_depth + 1`.
> When `bounds.chain_depth` exceeds the peer's maximum, the dispatch layer calls `suspend()` with
> `reason: "bounds_exceeded"` (§4a Ruling 3 — **not** `chain_depth_exceeded`). The maximum is peer-local
> but MUST be uniform across the cohort (§4a Ruling 2); the *value being tested* is global, because it
> is inherited rather than reset. TTL/budget **decrement** (§5.9) does not touch `chain_depth`.

**§3.6 step 6** — the dispatched bounds carry the inherited-and-incremented depth; TTL/budget
**decrement**, they are not refilled (correcting the landed `peer_default_ttl` language, §7):

```
; Step 6: Dispatch. chain_depth inherited+incremented; ttl/budget decrement per §5.9 (NOT refilled).
dispatch(execute, bounds: {
  ttl:         decrement(context.ttl)             ; resource backstop — §4a; §5.9 rule, deterministic
  budget:      decrement(context.budget)
  chain_id:    context.chain_id or generate_id()
  chain_depth: (context.chain_depth or 0) + 1     ; inherited from the wire, monotonic — §4/§5
})
```

### §4a — Q1: `chain_depth` is the deterministic chain brake; TTL is the resource backstop; de-confound them

The runaway is bounded on all three impls, but at a **different hop count on every pair** (Rust ~64,
Go/Python ~9) because cross-peer TTL mechanics are unspecified and each impl seeds/decrements TTL
differently (§1). Safety converges; the *bound* does not — a cross-peer seam divergence. Two things fix
it, and both are forced by the analysis, not chosen:

> **Ruling 1 (MUST — pin cross-peer TTL, `ENTITY-CORE-PROTOCOL §5.9`). [Revised 2026-07-22 per core-go's
> TTL-decrement-rate finding — the earlier "once per hop, not per sub-dispatch" wording made TTL redundant
> with `chain_depth` and therefore inert; corrected.]** TTL is a **resource backstop**, not a second
> per-hop counter: every impl MUST seed the same default `ttl` and **decrement it once per *dispatch*,
> including internal sub-dispatches**, so TTL is fan-out-sensitive and fires on expensive/fan-out steps
> (its stated role, §6). It is **orthogonal** to `chain_depth` (the per-causal-hop brake, §5): one dispatch
> advances `chain_depth` by at most one causal level but may spend several TTL across its sub-dispatches.
> **The exact rule that ends the 9-vs-64 split: a single dispatch MUST be decremented exactly once — never
> double-counted at ingress *and* again on forward.** Concretely: **Rust MUST seed a default TTL** (today
> `ttl = None` → no resource backstop at all); **Python's defect is the double-count** (a single dispatch
> decremented at ingress *and* on forward), **not** that it counts sub-dispatches — counting sub-dispatches
> is correct.

> **Ruling 2 (MUST — de-confound magnitudes).** The `chain_depth` ceiling and the TTL hop-budget MUST be
> **different** magnitudes, with `chain_depth` ceiling ≤ TTL hop-budget, so `chain_depth` is the
> **deterministic primary chain-length brake** and TTL is a pure resource backstop that only fires on
> expensive/fan-out steps. Today both default to **64**, so TTL masks the depth brake and it is never
> cleanly demonstrable cross-peer. Pin the two defaults apart (§7 / §3.11 defaults). This makes anchor 1
> demonstrable **without** a probe-TTL hack — the depth brake becomes the thing that actually fires.
> **[2026-07-22] Pin the *ratio*, not merely "different":** the TTL seed MUST be **`8 × chain_depth`
> ceiling** (Go ships 512/64 = 8 — the number of dispatch operations one causal level may spend before the
> resource backstop displaces the depth brake). Under Ruling 1's resource reading the **ratio** is the
> conformance-relevant quantity, not the bare seed; three impls agreeing on `512` while decrementing at
> different rates would still bound at different hop counts. **Recommended default ratio = 8× (seed 512),
> measured on the 2026-07-27 three-way run** — `cbx_crosspeer_chain_bounds_globally` PASS on go↔rust and
> go↔python; anchor 1's "same hop count on every pair" is demonstrated. **The ratio is NOT a conformance
> constant** (keystone 2026-07-27 §6b — that framing over-reached against the §4.10 doctrine): the
> conformance requirement is the *property* — the deterministic depth brake, not TTL, terminates a runaway
> chain — and a deployment MAY retune the ratio to suit its sub-dispatch fan-out. Distinctness + ordering
> (ceiling ≤ seed) are the structural MUSTs; 8× is the recommended default that satisfies them.

> **Ruling 3 (MUST — reason code, resolve the collision).** The continuation depth brake MUST report
> **`bounds_exceeded` (429)** — **NOT** `chain_depth_exceeded`, which is the **capability**-chain depth
> limit (`400`, `ENTITY-CORE-PROTOCOL §4.10(b)`). Go already uses `bounds_exceeded`; **Rust and Python
> currently emit the colliding `chain_depth_exceeded` and MUST migrate.** Two mechanisms MUST NOT share
> a reason string.

This supersedes the earlier "TTL refills, so it can't bound" framing (§1 correction): TTL *does* bound,
but only `chain_depth` bounds *deterministically* and *resource-independently*.

## §5 Delta 3 — causal depth vs. a standing continuation re-firing (the load-bearing distinction; NETWORK compat)

**This is the delta that keeps the fix from breaking retry-forever, and it is a checkpoint for cohort
review.** NETWORK's backoff / retry / keepalive dispatches all carry one stable `chain_id`
(`network-maintain-{session}`, proposal §A6.6) and are **intended to retry forever** (Amendment-12
ruling 6). If `chain_depth` incremented on every retry, a correctly-retrying peer would hit the
maximum and suspend — a regression.

The resolution is the `cascade_depth` model exactly: **`chain_depth` bounds *causal advancement depth
within one triggered flow*; it does not count the lifetime firings of a standing continuation.**

- **Increment** when an advancement is dispatched *causally from within* another advancement's
  execution (A→B→C as one flow) — it inherits and +1's the triggering `bounds.chain_depth`. This is
  the runaway case (§1), and it is now bounded on the wire.
- **Root at 0** when a **standing continuation** (`remaining_executions: null`) fires on a **fresh
  external trigger** — a timer tick, a `system/peer/status` write, a new inbound message — with no
  triggering advancement bounds in context. `chain_id` stays stable (correlation/observability is a
  *separate axis* from depth), but depth roots fresh, exactly as a fresh notification roots a new
  cascade.

**Why NETWORK is safe under this rule:** the backoff re-arm (proposal §A6.1) installs a continuation
that fires on the **next scheduled tick** — the retry schedule's timer break severs the causal chain,
so each retry roots `chain_depth = 1`, never accumulating, regardless of how long the peer stays down.
`chain_id` remains `network-maintain-{session}` throughout for `TraceChain` correlation. Retry-forever
and the depth bound coexist because they live on different axes.

> **Corollary (worth pinning for the programming guide):** a *synchronous, zero-delay* re-dispatch
> loop **would** accumulate `chain_depth` and eventually suspend — which is the **desired** safety
> behavior. "Retry forever" is safe *because* it is paced by a schedule; an unpaced self-redispatch is
> the runaway the brake exists to catch.

## §6 Delta 4 — the division of labor: chain_depth = length, TTL/budget = resource

Stated once so it stops being re-derived. The two bounds measure **different quantities** and must not
be confounded (§4a Ruling 2):

| Bounds field | Bounds what | Cross-peer behavior | Role |
|---|---|---|---|
| `chain_depth` | **causal chain length** (advances) | inherited + `+1` per advance (§4/§5) | **deterministic primary chain brake** |
| `ttl` / `budget` | **resource** (all dispatches, incl. sub-dispatches) | **decrement** once per hop, uniform, no refill (§5.9) | resource backstop; fires only on expensive/fan-out steps |
| `chain_id` | **correlation identity** | stable across the flow | identity, not a bound |

The earlier draft had `ttl` "refilled per advancement" and called it the fresh-resource source — that
was the wrong model (no impl refills; §1 correction). `ttl` decrements; its default and `chain_depth`'s
ceiling MUST differ (§4a Ruling 2) so the two never collide at one magnitude.

## §7 Delta 5 — correct §3.6 step 6, fix the false invariants, reset depth on resume

**§3.6 step 6** — the landed text says *"dispatch with fresh bounds: `ttl: peer_default_ttl`."* That
contradicts `ENTITY-CORE-PROTOCOL §5.9` (TTL decrements at the dispatch layer) and every shipped impl.
Replace the refill with **decrement** (§4a pseudocode). There is no `peer_default_ttl` refill source;
the only seed is the initial default TTL, applied once at chain origin, then decremented.

**§3.9 "Cross-peer limitation" note** — *"Global chain length is bounded by TTL on the wire and by
deliver token expiry"* is **false** (TTL is a resource bound, and non-deterministic across impls) and
MUST be replaced:

> Global chain length is bounded by `chain_depth`, carried in `system/bounds` (§3.11) and inherited
> across peer boundaries (the `cascade_depth` precedent), suspended uniformly at the cohort-wide ceiling
> with `reason: bounds_exceeded`. TTL/budget are a *resource* backstop that decrements per §5.9; they do
> not define chain length. Deliver-token expiry remains a wall-clock backstop, not a hop bound.

**§6.2** — *"Chain depth is a monotonically increasing counter, not reset by fresh bounds"* becomes
**literally load-bearing**: once `chain_depth` rides in bounds, the §5.9 TTL/budget decrement MUST leave
`chain_depth` untouched (it only increments). Add that clause.

**§3.7 resume** — `handle_resume` reconstructs with `bounds: params.bounds or peer_defaults`. Resume is
an **operator-authorized fresh dispatch** after a suspension (often *caused by* `bounds_exceeded` on
depth); it MUST **root `chain_depth` at 0**, else a resumed chain re-suspends immediately. State this in
§3.7 (fresh operator intent = fresh root, same as a fresh external trigger).

## §8 Cross-impl surface + conformance (what the cohort converges)

Per the convergence method (not vectors-first): build the wired counter, reconcile directional
behavior, and the reconciled behavior **becomes** the vector suite. The directional anchors:

1. **Runaway bounded by depth, deterministically, on every pair.** Two peers, a continuation that
   causally advances A→B→A→…; assert it suspends with `reason: bounds_exceeded` **on depth** (not TTL) at
   the cohort-wide ceiling, at the **same hop count on every impl pair** (proves §4a Rulings 1+2 killed
   the 9-vs-64 split), and that the depth at suspension is the *global* count (proves inheritance, §4).
2. **Retry-forever is not bounded.** A NETWORK backoff loop against a down peer runs past the
   `chain_depth` maximum without suspending (proves the causal-vs-standing split, §5) while its
   `chain_id` stays stable (proves the identity axis is independent).
3. **Bounds ride the cross-peer EXECUTE.** Inspect an originated cross-peer advancement; assert
   `bounds.{chain_id, chain_depth, ttl, budget}` are all populated and that `ttl` decremented by exactly
   one hop-unit (proves Delta 1 + §4a Ruling 1).
4. **Resume roots fresh.** Suspend on depth (`bounds_exceeded`), resume, assert the resumed dispatch runs
   (proves §7's depth-0 reset).

## §9 Scope, boundaries, open points

**In scope:** the CONTINUATION deltas (§3.6/§3.7/§3.9/§6.2, in place) + the core-protocol §3.11/§5.9/
§4.10(b) deltas. **Already adopted three-way (not re-litigated here):** the `chain_depth` wire field,
cross-peer bounds propagation, and the causal-vs-standing split — this rev only *pins the parts that
diverged* (TTL mechanics, magnitudes, reason code). **Not in scope:** the marker/injection work; NETWORK
Amendment 12 itself; the standing-model authority/join facets (`PROPOSAL-CONTINUATION-STANDING-MODEL`).
No wire-core renumber — `chain_depth` is additive, MUST-ignore-unknown, beside `cascade_depth`.

**What each impl changes (from the 2026-07-19 survey):**

- **Rust** — MUST seed a default TTL (today `ttl = None` → no resource backstop); migrate the reason
  code `chain_depth_exceeded` → `bounds_exceeded`.
- **Python** — MUST decrement TTL **once per cross-peer hop**, not at ingress *and* every sub-dispatch
  (that's what makes it die at ~9); migrate the reason code → `bounds_exceeded`.
- **Go** — already `bounds_exceeded` and one-decrement-ish; align to the pinned per-hop rule + the new
  distinct ceiling default.

**Open points for cohort review:**

- **O1 — the causal-vs-standing signal (§5).** Pin the *concrete* signal each impl keys on (triggering
  `bounds.chain_depth` present vs. fresh handler entry). The twin of the standing-model's O1.
- **O2 — the two default magnitudes (§4a Ruling 2).** Pick the `chain_depth` ceiling and the `ttl`
  default so ceiling ≤ TTL-hop-budget and they're **distinct** (today both 64). Proposed: keep
  `chain_depth` ceiling 64, raise the continuation TTL seed so depth binds first — or lower the ceiling.
  Cohort converges the exact pair; the MUST is that they differ.
