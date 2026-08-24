# PROPOSAL — Lost-error marker: SHOULD → MUST, under the handler's own authority

**Status:** **IMPLEMENTED — FOLDED 2026-08-15 as `EXTENSION-CONTINUATION` v1.23.** Both deltas are in
the spec. **Delta 2 (handler-own-authority bind) was demonstrated three-way 2026-07-27** — the
`continuation_bounds` cb2 gate green go/rust/py, with rust's `e4da0bc` closing its WARN by self-binding the
`bounds_exceeded` lost marker on the reactive path under the handler's own authority — and had already
landed as §3.10.7. **Delta 1 (SHOULD→MUST) was held on the NETWORK-LIVENESS §A6.1 `on_error` carve-out and
is released by that fold**, which completed the same day.

**What landed, and what was already there.** Delta 2 (§3) and the §3a.1 body schema were **already in the
spec** when this fold ran — §3.10.7's *"chain-error markers MUST bind under component-owned authority"* and
`chain_id`/`step_index` in the reserved body-field table. **Verified before editing rather than assumed**,
which is why this fold is smaller than the proposal. What was owed and landed:

| Delta | Landed as |
|---|---|
| §2 Delta 1 — three cases `SHOULD` → `MUST` | §3.4 A.1 / v1.13 / v1.16 marker cases |
| §2a — the obligation is for *unhandled* failure; a handled loop MUST NOT manufacture markers | new §3.4 preamble |
| §2b — handler MUST set `bounds.chain_id` on a known chain; constant sentinels non-conformant | §3.10.5 |
| §3a — path-safety for **every** interpolated coordinate, sentinel collapse, no hashing | §3.10.5 |
| §5 — collection `MUST`, self-collected, `system/config/chain-errors` → `retention_ms` (24h) | §3.4 + Appendix A + §8.1 |

**The §2a coupling is why this waited, and it is worth keeping.** A universal MUST-bind written in the same
cycle as an infinite retry loop, never cross-checked, would have obliged every implementation to write
~1,440 markers per day per dead peer — *"a record that nothing is required to collect."* The fix was at the
source (give the loop an `on_error`, NETWORK §4.1.1), not in collection, and the spec now says so in both
places.

**Target:** `specs/extensions/EXTENSION-CONTINUATION.md` §3.4 / §3.10 (the lost-error marker
family). Separate from NETWORK Amendment 12 — this is a CONTINUATION-spec change that NETWORK
forces; it routes to the cohort alongside the NETWORK convergence pass.
**Origin:** `ABSORPTION-workbench-continuation-error-model.md` (the elevation ask) →
entity-core-go rung-1 feedback **F2** (the conformance-trap catch) → rung-2 observations §2
(option-(ii) feasibility verified in Go). `ARCH-RESPONSE-NETWORK-A12-RUNG{1,2}-GO` carry the
ruling trail.

---

## §1 Problem

The lost-error marker (`system/runtime/chain-error-lost`, §3.4/§3.10.1) is the substrate's *only*
observability backstop for a fire-and-forget chain whose error path fails — the thing standing
between a developer and "I installed the chain, fed it input, nothing happened, no errors
anywhere." Today it is:

1. **`SHOULD`-bind** — an impl is conformant with every chain failure invisible (core-go's own
   validator can only `WarnCheck` its absence);
2. **divergent** — Rust keyed the marker on `cascade_depth`, Go on `RequestID` (the §3.4 v1.14 pin
   already rules for `RequestID`, but a pin under a SHOULD has no gate);
3. **capability-fragile** — the bind rides the advance-caller's context, so when the chain's
   propagated cap doesn't cover `system/runtime/chain-errors/**`, **the marker bind itself 403s**
   (Go's F11 record) — the backstop fails precisely for the most-misconfigured chains, the ones
   that need it most.

NETWORK (Amendment 12) is the forcing case: the first extension shipping continuations as a
load-bearing feature to **all** impls, built below the SDK. If the marker stays a non-uniform
SHOULD, NETWORK reconnect failures are observable in Go and invisible off-Go — "green ≠ working"
one layer down.

## §2 Delta 1 — MUST-bind

§3.4's three marker cases (`on_error_dispatch_failed`; no-`on_error` forward non-2xx; `merge_value_
not_map`) are elevated **SHOULD → MUST**. Everything else about the marker is unchanged: type
pinned `system/runtime/chain-error-lost`, v1.20 per-occurrence path scheme, **MUST NOT** trigger
any reactive behavior, GC-eligible after the retention window.

**Scope: universal**, not "extension-shipped chains only." A scoped MUST is untestable (nothing
distinguishes an extension's chain from an app's on the wire) and the observability floor is not
extension-specific — the forcing case is NETWORK, but the floor is general. *(Explicit decision
point for cohort review; the fallback is scoping to chains installed by system handlers, if any
impl shows universal cost.)*

> **§2a — Scope correction (rung-3, three-way measured): the MUST is for *unhandled* failure, and
> a handled loop must not manufacture it.** The first draft of this elevation was written in the
> same cycle as `PROPOSAL-NETWORK-LIVENESS-REACTIVE-BUILDOUT`, whose central mechanism is an
> **infinite loop of expected failures** — and the two were never cross-checked. NETWORK §4.1's
> `backoff_continuation` carries no `on_error`, so every failed retry is exactly the v1.13 case and
> MUST-binds a marker: ~1,440/day/dead-peer, forever. Go's sentence belongs in this record: *"every
> impl is now required to write a record that nothing is required to collect."*
>
> The principle stands — silent chain failure is unacceptable — but the **v1.13 case is the
> silent-burn backstop**, and a failure that is *broadcast on `system/peer/status`* and *handled by
> a re-arming loop* is neither silent nor unhandled. Applying the marker there adds **zero**
> observability at unbounded cost.
>
> **The fix is at the source, not in GC:** the chain author gives the loop an `on_error` (NETWORK
> §A6.1), the v1.13 case stops firing, and the marker returns to catching the *exceptional* failure
> (A.1). **Corollary — a general authoring caution this elevation now carries:** a chain that
> fails *by design, repeatedly* MUST handle its own failure (`on_error`) rather than rely on the
> substrate to record each occurrence. The marker is not a log.

**Key ratification (F1).** The marker step-key is the **original request ID** — the elevation
gates the existing §3.4 v1.14 pin. ~~This ratifies Go's shipped shape; **Rust migrates** from
`cascade_depth`.~~ **Superseded by rung-3 measurement (2026-07-16)** — the carried framing was
stale. Rust's live `step_index` is the constant `internal` (neither the pin nor `cascade_depth`);
Python's is a continuation path. F1's **destination is unchanged** (original request ID); its
*migration list is both siblings*, and Go is **not** the reference on `chain_id` either (see §2b).
Evidence beats the carried assumption.

### §2b — The coordinate (ruled on three-way evidence, 2026-07-16)

Measured, identical scenario, three impls, three incompatible coordinates: Go
`chain_id = notif-sub-<id>` (fresh per dispatch, 5 nodes in 2 families from one run); Rust
`chain_id = step_index = internal` (constant); Python `chain_id = unknown`,
`step_index = cont-forward-system/inbox/…` (constant + path). `reason` converged
(`connection_failed`). **All three fail the same question:** *which peer relationship is failing,
and for how long?*

- **`chain_id` is declared at ENTITY-CORE-PROTOCOL §3.11 (`system/bounds`)** — not by this
  extension, not by NETWORK. It ships with `parent_chain_id`, and **§3.11 already defines the
  correlation model**: *"Set when a continuation dispatches a new sub-chain… The new chain gets a
  **fresh chain_id** and records its parent here… **Enables process tree reconstruction via field
  queries.**"* Correlation is a **process-tree walk**, not a shared id.

  > **Ruling corrected (2026-07-16).** An earlier draft of this section required all failures of
  > one relationship to land under **one** `chain_id` node and called Go's per-dispatch ids
  > non-conformant. **Both were wrong** — they contradict §3.11's sub-chain model and would have
  > made all three impls newly non-conformant. Go's fresh ids (`network-advance-<id>`,
  > `notif-sub-<subid>-<id>`) are **sub-chains getting fresh ids: exactly §3.11's model.** The
  > error was ruling on `chain_id` semantics from the two extensions that *consume* it without
  > reading the core type that *declares* it — this proposal's own authoring rule (§5) failing on
  > its author.

- **The parent-edge ruling is WITHDRAWN (2026-07-16), and `parent_chain_id` is RESERVED.** An
  earlier draft ruled *"`parent_chain_id` MUST be set when a continuation dispatches a sub-chain."*
  **It has no setter.** Verified at §3.6 step 6: `chain_id: context.chain_id or generate_id()` —
  a fresh chain is born **only when `context.chain_id` is absent** (a root dispatch); everything
  downstream inherits. **No dispatch class in the current model mints a sub-chain**, so there is no
  site that could record a parent. Confirmed dead cohort-wide: 35 sites across three trees (5 Go /
  12 Rust / 18 Python), every one a type definition, wire decode, or read-and-propagate — nobody
  originates it. §3.11's process-tree reconstruction is **unreachable on every seat**, and all
  three implemented step 6 correctly. `parent_chain_id` is **speculative surface: do not implement,
  do not invent a sub-chain trigger.** Revisit if a real fan-out consumer lands (a join
  continuation spawning parallel branches is the plausible first).

- **The correlation fix — the handler supplies the context it already has.** The problem the
  withdrawn ruling addressed is real: a retry dispatch is a *fresh root* by step 6's rule
  (timer-fired, no inbound context) → mints a fresh id → markers scatter, and the lifecycle id
  appears nowhere. The answer needs no new mechanism:

  > **A handler that originates a dispatch belonging to a known chain MUST set `bounds.chain_id`
  > to that chain's id**, rather than dispatching with empty context and letting step 6 mint a
  > fresh one.

  For NETWORK: the backoff / retry / keepalive dispatches carry `network-maintain-{session}` —
  which the handler already holds on the session. Every failure of one relationship then lands
  under **one** node (`lost/network-maintain-{session}/…`) and *"which relationship is failing?"*
  is answered by the field that already works. The earlier ruling was solving the right problem
  with the wrong tool: the chain_id was never supposed to be minted there; the handler simply never
  supplied the context it had.
- **Constant sentinels (`internal`, `unknown`) are NOT conformant.** They collapse every marker
  from every chain and peer into one node — satisfying the letter of MUST-bind while delivering
  none of its purpose. **A MUST whose conformant implementation carries zero information is not a
  MUST.** Pinned so a probe can reject it.

## §3 Delta 2 — the marker binds under the continuation handler's own authority

**The F2 trap:** bare MUST-bind is unsatisfiable — the bind can legitimately 403 under the chain's
propagated cap, so elevating without resolving write authority just moves the silent black box one
level up (a MUST that cannot be met).

**Resolution (option ii, adopted):** `system/runtime/chain-errors/lost/**` is the **continuation
handler's managed namespace**; the marker bind is authorized by **the handler's own grant** — the
EXTENSION-NETWORK §11 pattern ("the handler's own grant authorizes all writes to its managed
namespace"). The principled line:

> **Author surface rides the author's cap; substrate surface rides substrate authority.**
> `on_error` delivery is the *chain author's* compensation action — it rides the chain's
> `dispatch_capability`, unchanged (the workbench sink pattern, including the author-provisioned
> cap for a custom sink, is untouched). The lost marker is the *substrate's* observation of a
> failure — it rides the substrate's own authority, so it cannot be broken by the very
> misconfiguration it exists to observe.

**Invariant (from the Go rung-2 verification, adopted verbatim):** the marker is bound at the
**observing peer's own tree under its own authority — it never rides the chain's propagated cap
and never crosses a peer boundary**. This is what makes MUST-bind unconditionally satisfiable.

**Feasibility, per impl:**

- **Go — verified, no cap-check rework** (rung-2 observations §2): the `selectCapability` fallback
  at the handler write seam already implements caller-cap-else-handler-grant with the W6
  attribution split; the change is the continuation handler's grant declaring the lost-marker
  namespace. Precedent already shipping: the §3.10.7 receiver-side `rejected` marker binds under
  component-owned authority with no HandlerContext cap check.
- **Rust / Python — the open check** this proposal routes to them: confirm the continuation
  handler can bind under its own grant at advance time in their cap-check flow. (If an impl cannot,
  the fallback is option (i) — MUST-bind paired with MUST-provision, the chain carrying a cap
  covering `system/runtime/chain-errors/**` — recorded here as **rejected-unless-forced**: it
  pushes provisioning burden onto every chain author and re-creates F11 for any author who forgets.)

**Security bound — and it is COUPLED to §3a; the two are not safe apart.** Handler-authority
binding is not a general write primitive: the path scheme is **fixed at four segments**
(`.../lost/{chain_id}/{step_index}/{reason}/{marker_hash}`) **per §3a below**, the body is derived
from the observed failure, the marker carries no reactive behavior, and markers are collected per
§5. The attribution split (caller vs handler cap, **W6 — MUST**) is the only record of *who caused*
a substrate-authorized write, and is therefore security-load-bearing, not cosmetic.

> **The coupling, stated because omitting it was a real hole (2026-07-16).** Moving the bind to
> substrate authority removed an *accidental containment*: under the chain's propagated cap, a
> hostile path would have 403'd outside the cap's scope. Substrate authority + attacker-controlled
> path segments = **a write-anywhere primitive with the substrate's own grant behind it**. The
> dispatcher site is the sharpest — the `rejected` marker (§3.10.7) binds precisely *because* the
> sender's cap check failed, so **an unauthorized caller reaches it by construction**. Measured by
> the Go seat: `bounds.chain_id: "../../../../authority/keys"` escaped `system/` entirely, because
> an *interior* `..` is **resolved** by path-cleaning rather than rejected (the leading-`../` form
> that cleaners reject is not the escape). **Wherever a marker binds under substrate authority,
> every interpolated segment MUST be validated path-safe at construction (§3a).** This proposal's
> §3 and §3a ship together or not at all.

### §3a — Coordinate segments MUST be single path segments (correcting "fixed")

**This draft's "the path scheme is fixed" was false as written, and it caused a live bug.**
Nothing constrained `chain_id`/`step_index` to a single segment, and in practice they are not:
NETWORK §4.1's own mandated `chain_id = "network/maintain/" + session_id` is **three segments**;
Python's `step_index` is five, putting its markers at **depth 7**. The cost was concrete: Go's
validator walked with a depth cap written against my "four levels" sentence, truncated one level
**above** Python's conformant markers, and **nearly filed a false "Python is missing the marker"
gap**. A spec sentence asserting "fixed" over a variable shape doesn't just mislead — it
manufactures false cross-impl bug reports, the most expensive failure mode we have.

**Ruling — single-segment**, on an argument stronger than tooling convenience:

> **Variable-depth coordinates make the path unparseable.** Given
> `lost/network/maintain/abc/req-1/connection_failed/<hash>` there is no way to know where
> `chain_id` ends and `step_index` begins. "List all failing chains" is not awkward — it is
> **impossible**. The path is an index; an ambiguous index is not one.

- **Producer side (necessary, NOT sufficient):** `chain_id`, `parent_chain_id`, and `step_index`
  MUST each be a single path segment (MUST NOT contain `/`) — routed to ENTITY-CORE-PROTOCOL §3.11
  where `chain_id` is declared. **NETWORK §4.1's format is corrected** to
  `"network-maintain-" + session_id` (companion delta: `PROPOSAL-NETWORK-LIVENESS-REACTIVE-BUILDOUT`
  §A6.6). Go's dash convention is promoted from unstated accident to the rule.
- **Consumer side (the load-bearing half):** *every* interpolated coordinate segment MUST be
  **validated path-safe at construction**, regardless of source. Impls **MUST NOT** trust
  `bounds.chain_id`, `request_id`, or `result.data.code` as path components — **all three are
  wire-supplied**, and a hostile peer ignores the producer-side constraint by definition. (Today
  §1.4 pins path-safety for `{reason}` and says nothing for `{step_index}` or `{chain_id}` — the
  two that are actually attacker-controlled.)
- **Sanitization behavior — collapse to a fixed sentinel; the body carries the original.**
  A segment that fails validation collapses to a fixed sentinel (`{reason}` → `unspecified_error`
  per the landed §3.10.5; one sentinel each for `{chain_id}` / `{step_index}`). The original value
  is preserved in the marker body (§3a.1). **Safe values pass through byte-identical**, so no
  conformant coordinate changes — which is what makes this landable mid-reconciliation, and why the
  safety half needs no ruling and should land immediately.

  > **AMENDED 2026-07-17 — this bullet previously read "hash unsafe values; do not drop or collapse
  > them… distinct failures MUST keep distinct coordinates; the original survives in the marker
  > body."** Both halves were wrong, and Rust caught it by reading the spec against the code:
  >
  > 1. **The distinctness rationale was already satisfied.** §3.10.1's v1.20 scheme puts each
  >    distinct occurrence at its own **`{marker_hash}` terminal**. Values collapsing to the same
  >    intermediate node still produce distinct terminals (different bodies → different hashes).
  >    **Collapsing loses no occurrence** — the terminal already carried the distinctness I thought
  >    I was protecting.
  > 2. **"The original survives in the body" described a design, not the spec.** No `chain_id` body
  >    field exists. Hashing it is a **one-way loss**: the operator sees an opaque node and cannot
  >    answer *"what did the attacker send?"* — in the one marker whose purpose is observing a
  >    hostile failure.
  > 3. **Hashing re-opened the vector the fix closed:** each distinct hostile value → a distinct
  >    hash → a distinct node, so an attacker mints **unbounded path nodes**. A sentinel bounds it
  >    to one quarantine node.
  >
  > The landed §3.10.5 had the right answer all along (sentinel + raw-in-body); this proposal
  > contradicted it. Rust's reconciliation — *"collapse where the body recovers the original, hash
  > where it doesn't"* — is correct, and the sharper form is **collapse always, and make the body
  > always recover** (§3a.1): "the body doesn't recover it" is a gap to fix, not a reason to hash.
  > Cost acknowledged: this churns a behavior all three impls converged on byte-identically
  > (`invalid-<hash>`, unprompted). A constant is more trivially convergent than a hash
  > construction; they re-converge for free.

#### §3a.1 — The marker body schema (PINNED)

Three seats split three ways (Go `original_request_id`, Python `step_index_original`, Rust nothing)
because the body schema was never pinned. **The body is the record; the path is an index.** Each
field holds **the original value**, matching §3.10.5/§3.10.6's landed pattern (path `{reason}` =
`unspecified_error`, body `code` = the raw code):

| Body field | Holds | Status |
|---|---|---|
| `code` | the original `result.data.code` | exists (§3.10.5) |
| `step_index` | the original request ID | exists in `lost` (§3.4, denormalized); **MUST be added to `rejected`** |
| `chain_id` | the original `bounds.chain_id` | **NEW — the gap** |

Plain names, no `original_`/`_original` prefixes — the body field *is* the original by definition.
This un-regresses observability immediately on seats whose sanitizer now correctly refuses a
multi-segment fallback key: the information is recoverable from the body before the single-segment
spelling (§3a, fallback rule) restores path-level grouping.
- **Sanctioned no-request-id fallback:** a **single-segment synthesized key** with the natural key
  preserved in the body — i.e. the hash rule above, generalized. Python's
  `cont-error-{continuation_path}` (a tree path, sanctioned under F1 *before* single-segment was
  ruled) becomes `cont-error-{hash(continuation_path)}`: same information, addressable, path stays
  parseable.
- "Fixed four levels" then becomes **true**, and marker-reader tooling can rely on it.

## §5 Retention: collection becomes MUST, with a named actor and a named key

**The record, corrected.** It is *not* true that no retention window is defined:
`EXTENSION-CONTINUATION.md` §3.4 states, twice (lines 577, 601), *"Implementations MAY
garbage-collect markers after a configured retention window (suggested default: 24 hours)"* — plus
§3.10 (line 1055) and Appendix A (1244–1246). A suggested default **exists**.

**And it changes nothing**, because the real defect is the modality, not the value:

- collection is **MAY** — no impl is required to collect;
- **no actor is named** — "GC-eligible" names a property with no subject;
- **no config key exists** — "a *configured* retention window" names a knob never defined.

> **A MUST-write paired with a MAY-collect is a leak by construction**, whatever default the MAY
> suggests. This elevation created exactly that pairing.

**Deltas:**

- **Collection becomes MUST for any peer that MUST-binds**, actor = **self-collection**: the peer
  that bound the marker collects it. This falls out of §3's invariant — markers are bound in the
  observing peer's **own tree under its own authority**, so the binder can always collect.
  *The authority answer and the actor answer are the same answer.*
- **Key named:** `system/config/chain-errors` → `retention_ms`, **default 24h** (retaining §3.4's
  suggested value, now with a home), mirroring `EXTENSION-DISCOVERY`'s `candidate_history_retention`
  shape.
- With the §2a scope fix, retention returns to **ordinary hygiene** rather than the last line of
  defense against an unbounded generator.

## §4 Conformance (converged, per the Amendment-12 §D method — not pre-authored)

Expected vector shapes, reconciled at the NETWORK rung-3 convergence pass (the reconnect chain is
the natural test subject):

- **Bind-under-denial (the F2 vector):** install a chain whose `dispatch_capability` does NOT
  cover `system/runtime/chain-errors/**`; force an `on_error` dispatch failure; the marker MUST
  land (handler authority), keyed by the original request ID.
- **No-`on_error` non-2xx:** forward chain, no `on_error`, target returns 403/404/500 → marker at
  `.../{result.data.code}/...` MUST land; `remaining_executions` decrements normally; no retry.
- **Non-reactivity:** marker bind MUST NOT advance, retry, or suspend anything (assert no
  additional dispatches after the marker).
- **Key convergence:** same scenario on Go/Rust/Py → same `(chain_id, step_index, reason)`
  coordinate (content hashes agree per the pinned type).

## §5 Non-goals

- No change to `on_error` semantics (best-effort compensation, author-capped, workbench sink
  pattern intact).

  > **Struck (2026-07-16) — this bullet previously read "…including that `on_error` MUST NOT route
  > to `system/inbox/*`."** That blanket prohibition contradicted the rung-3 ruling in the same
  > document set (which corrected it) and left NETWORK §A6.1's "give the loop an `on_error`" with
  > **no legal target** — caught by the Go seat before anyone coded against it. **Corrected rule:**
  > inbox-routing *advances* the continuation bound at that path — use it when the error is
  > **meant** to drive the next step (a retry; NETWORK §4.1's loop is the legitimate case); use a
  > `system/runtime/chain-errors/*` sink when you want **passive observation**. The trap is
  > *unintended advancement*, not inbox-routing itself.
- Not the continuation observability contract (`ContinuationView` / trace-by-`chain_id`) and not
  the NETWORK completion contract — both tracked in `PROPOSAL-NETWORK-LIVENESS-REACTIVE-BUILDOUT`
  §E as separate deliverables.
- No wire change, no new entity type, no new error code. The marker type, path scheme, and
  non-reactivity are all already spec'd; this proposal changes **bind obligation** and **bind
  authority** only.
