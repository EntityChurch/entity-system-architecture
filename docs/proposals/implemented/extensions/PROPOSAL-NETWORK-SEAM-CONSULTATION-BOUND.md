# PROPOSAL — §10.3 obligation 6: a bound on seam-consultation frequency

**Status:** RATIFIED 2026-08-16 — folded to `EXTENSION-NETWORK` §10.3 obligation 6 (v1.7)
**Target:** `specs/extensions/EXTENSION-NETWORK.md` §10.3 (new obligation 6);
`EXTENSION-SIGNALING.md` §13 item 6 → resolved disposition
**Provenance:** filed by `entity-core-rust`/`entity-browser-rust` as
`PROPOSAL-ESTABLISH-CONSULTATION-BACKOFF` §7 (draft normative text, written to be ruled on rather
than adopted verbatim), with a measured remedy; the cross-impl divergence read from
`entity-core-go` `core/peer/remote.go` and re-verified by arch at `3d62604`.
**Adjudicable:** **yes — N=2, divergent, landed surface.** This is the case the seam ruling exists
for, and it is why this could be ruled while the NAT/WebRTC surface could not.

---

## 1. The gap

**§10.3 bounds third-party load on two axes and leaves the third open.**

| Obligation | Face of the invariant | Bounds |
|---|---|---|
| **4** | the **retry** face | nested retry loops multiplying against the carrier |
| **5** | the **fan-in** face | concurrent triggers to one peer — single-flight |
| **— missing —** | the **sequential** face | the same caller consulting the seam again, and again, forever |

**Each member of the sequential series is individually conformant**, and **§11.5 cannot see it**
because §11.5 bounds deposits *per establishment* — the series is a count *of establishments*.
Measured: **~5 offer deposits/second per open conversation with an unreachable peer, indefinitely**
(196 s window, 974/902 negotiations, ~2000 deposits into one bucket, still climbing at teardown).

**The load lands on shared carrier and reflector infrastructure** — which is the stated rationale
that already makes obligations 4 and 5 MUSTs rather than tuning notes: *invisible to the offending
peer's own tests, visible to every provider.* **Obligation 6 is not a new principle. It is the third
axis of a principle the spec already committed to.**

## 2. The divergence that makes it adjudicable

**Two conformant implementations, three orders of magnitude apart, on one seam.** Verified by arch
in both trees rather than taken from either party's report:

- **`entity-core-go` `3d62604`, `core/peer/remote.go`.** Two arms in one switch. **`establishDispatch`
  calls `tryEstablishLive` unconditionally** — no memo check, no bound. **`establishReconnect`**
  checks `prefersRelay`, makes one attempt, then `markPreferRelay` — *skip the seam for the rest of
  the session.*
- **`entity-core-rust` `55cc507`.** A per-peer consultation bound at the ladder call site
  (`RemoteState::note_establish_attempt`, `ESTABLISH_FREE_CONSULTATIONS`): grace window, then spacing.
  Measured 974/902 → **28/30** negotiations, healthy path **byte-identical**.

**Neither is unconformant today**, and that is the finding. Obligation 4's closing sentence already
blesses the prefer-relay memo as a MAY. So the same seam ranges from *"skip forever after one
failure"* to *"~1000 consultations per conversation"* depending on which arm a caller enters, and
**nothing in the spec or the suite can see the difference.**

## 3. What is ruled

### 3.1 Placement — caller-owned, and the seam does not change

**The stop condition belongs to the caller, at the §10.3 call site.** No new `NoPath` variant, no wire
signal, no seam-signature change.

> **The precondition that was attached to this and is not part of the ruling.** The filing seat
> originally required that, if the caller owns the stop condition, the seam MUST first separate
> `NoPath`-unreachable from `NoPath`-not-yet. **They withdrew it and they were right to**: the
> distinction is required only to **abandon** a peer, and **a backoff never abandons** — it spaces,
> and a counterpart that becomes reachable still connects, at most one cooldown late. The conflation
> costs bounded latency, not a lost peer. **It returns only if a future ruling wants a genuine
> give-up.**

### 3.2 Charge the ask, not the answer `[the part most likely to be got wrong independently]`

**A consultation is charged when it is *started*, not when it returns.** A cancelled consultation
returns nothing to charge while the negotiation it started has **already reached the carrier**.
Measured: charging at the outcome produced **57 negotiations where the schedule permits ~34**;
charging at the start closed it to **28**. **A bound charged at the outcome exempts precisely the
attempts a loaded caller makes most of.**

### 3.3 Rate zero is bounded — obligation 4's memo survives `[arch's correction to the draft]`

**The draft text would have made `entity-core-go`'s prefer-relay memo non-conformant**, because it
requires spacing *"up to a cap"* and *"the cap MUST be small enough that a counterpart which becomes
reachable is noticed within it"* — and a memo never notices. **That contradicts obligation 4, which
blesses the memo as a MAY.** A new obligation MUST NOT silently reverse a landed one.

**Resolved: obligation 6 bounds the rate; it does not impose a floor.** A caller that stops
consulting entirely — by memo, or by falling to a lower rung of `EXTENSION-REGISTRY` §3b.4's ladder
— consults at rate zero and is bounded **trivially**. The "noticed within the cap" requirement binds
only a caller that **keeps** consulting: *if you back off, do not back off past usefulness.*

**So what obligation 6 actually forbids is the unbounded middle** — consulting forever at full rate.
Both landed reconnect behaviours stay conformant; **`entity-core-go`'s `establishDispatch` arm does
not**, and that is the one real non-conformance this ruling creates.

### 3.4 The bound's value is implementation-defined; its existence is not

Consistent with obligation 5 (*"how it coalesces is impl-idiomatic"*). The **observable** requirement
is bounded third-party consultation load per peer. Grace-window size, growth curve, and cap are
idiomatic. **Two peers with different caps still interoperate** — the divergence costs a provider
money, it does not split a pair — but it is a MUST for the same reason 4 and 5 are: the cost is
externalized onto third parties.

## 4. Normative text (folded)

See `EXTENSION-NETWORK` §10.3 obligation 6. Derived from the filing seat's §7 draft with §3.3's
correction applied and the reset condition kept.

## 5. What each seat does

| Seat | Action |
|---|---|
| **`entity-core-go`** | **Bound the `establishDispatch` arm.** The reconnect arm is already conformant (rate zero via the memo). This is the latent unbounded series — it has no 5 Hz caller today, which is why it has not been measured. |
| **`entity-core-rust`** | **Nothing normative.** The landed bound is conformant; confirm it charges at *start*. |
| **`entity-core-py`** | Read your establishment arm against obligation 6. **Not read closely by arch or by the filing seat** — stated rather than assumed. |

## 6. Evidence

**Held:** the measured remedy (974/902 → 28/30, healthy path byte-identical, mutation-checked); the
57→28 charge-point measurement; the two-arm read of go's tree, re-verified by arch.
**Not held, and not needed:** a three-way run. **This ruling is about a property of a caller's own
behaviour toward third-party infrastructure, not a cross-peer wire agreement** — the divergence is
observable by reading two trees, which is how it was found.

## References

- `EXTENSION-NETWORK.md` §10.3 obligations 4, 5 · §10 step 3b
- `EXTENSION-SIGNALING.md` §7.2 / §7.2.1 (exchange budget, `caller_owns_retry`) · §11.5 · §13 item 6
- `EXTENSION-REGISTRY.md` §3b.4 (the prefer-cheap-path ladder — where a memoing caller goes instead)
