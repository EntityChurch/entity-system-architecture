# PROPOSAL — the deadline is §6.11(c)'s only expressible input, and the fan-out is not needed because §4.8 forbids collapsing the invocations

**Proposes:** one optional parameter — `deadline_ms` — on `system/validate/dispatch-outbound`
(`GUIDE-CONFORMANCE` §7a.1), with a *no-silent-substitution* rule and the code its expiry surfaces.
**And it withdraws the conditional second ask**, on the reading question that ask was conditioned on.

**Status:** **DRAFT 2026-09-16 · revision 1.**
**Depends:** `GUIDE-CONFORMANCE.md` §7a.1, §7a.1a, §7a.2, §7a.2a · `ENTITY-CORE-PROTOCOL.md` §6.11,
§6.12, §4.8, §9.4
**Kind**: proposal · **Authority**: non-binding until folded · **Governed-by**: `SPECIFICATION-FORMAT.md`
**Audience**: implementers of the scaffold handlers, of the validator side, and of the conformance
driver that exercises both.

**Answers:** `CQ-20`/`X5`, `CQ-21`/`X6` (round 3, `entity-system-conformance`).

---

## §0 Summary

| | |
|---|---|
| **`CQ-21` asked first, and it is conditional** | *"A fan-out form (N originations from one invocation) — **only if** you read N concurrent invocations as NOT satisfying §6.11(a)'s 'a handler originates concurrently'."* |
| ✅ **`CQ-21` — WITHDRAWN. N concurrent invocations satisfy it.** | §6.11(a)'s subject is literally *"multiple outbound EXECUTE calls on the same pooled connection"*, and under §7a.2a every reentry origination goes back **over the caller's own inbound connection** — so N invocations put N outbound EXECUTEs on one pooled connection, which is the clause's own wording |
| ⭐ **And the load-bearing half is a different section: `§4.8`** | The worry a fan-out would answer is *the peer serializes its own inbound dispatch, so the N invocations never overlap.* **§4.8 forbids exactly that** at MUST level — while a handler is processing a frame, the peer MUST be able to read and dispatch further frames on that same connection. **A peer that collapses the N is already non-conformant**, so the driver is sound *because another rule holds the premise up* |
| ⚠ **What a fan-out WOULD buy is diagnosis, not measurement** | N invocations cannot distinguish *the peer serialized OUTBOUND dispatch* (§6.11(a)) from *the peer serialized INBOUND dispatch* (§4.8) — both produce one observable. A fan-out isolates §6.11(a) with §4.8 held constant. **That is a triage refinement and it is the seat's to propose on measured evidence, not arch's to mint in advance** |
| ✅ **`CQ-20` — GRANTED, and it is not a convenience** | **§6.11(c) is a MUST with no expressible input.** A probe can observe a deadline's *consequences* only if it can *set* one, and nothing in any handler contract lets it. **A normative MUST naming a capability has to name a declared site a peer can carry, and this one
never did — for several revisions** |
| **The parameter** | `deadline_ms: uint`, optional. Present → the peer MUST apply it as the per-request deadline on the reentry origination, enforced at the request layer per §6.11(c) |
| ⭐ **The rule that makes the negative control trustworthy** | **No silent substitution.** A peer that cannot honour the requested value MUST refuse with `400 invalid_params` rather than apply a different one — otherwise *"the control arm failed"* cannot distinguish *deadline ignored* from *deadline capped*, and the filing seat's own SKIP-vs-FAIL branch turns on exactly that |
| ⭐ **What we are NOT adding, and why it is better without** | **No elapsed-time reporting.** The suite is the counterparty and holds both endpoints of the interval already. **A self-reported duration is the peer grading its own timing** |
| **Cost** | one optional param · one new §7a.1b · **additive, so `GUIDE-CONFORMANCE` takes a minor bump** under `SPECIFICATION-FORMAT` §9.1 — except that it carries no `**Version**:` header at all, which is named in §5 as owed |

---

## §1 `CQ-21` — the reading question, answered from the text

**The ask:** do N concurrent invocations of `dispatch-outbound` satisfy §6.11(a)'s *"a handler
originates concurrently"*, or is a fan-out form needed?

### 1.1 §6.11(a)'s subject is the connection, not the invocation

> **(a)** An implementation that pools connections per peer-pair MUST NOT hold per-connection
> serialization across the send+recv cycle of an outbound EXECUTE. **Multiple outbound EXECUTE calls
> on the same pooled connection MUST be able to proceed concurrently.**

**The clause counts EXECUTEs on a connection. It does not count handlers, and it does not count
invocations.** §7a.2a pins where the reentry origination goes — *"back to the caller over the same
connection"* — so N concurrent invocations of `dispatch-outbound` place **N outbound EXECUTEs on one
pooled connection**, which is the sentence's own subject with nothing left to interpret.

⇒ **The condition on `X6` is not met, and it withdraws.**

### 1.2 ⭐ The premise that needed checking is in §4.8, not §6.11

**The real objection to N invocations is not about (a)'s wording. It is this:** if the peer processes
inbound EXECUTEs on one connection **one at a time**, the N invocations are serialized *before they
ever reach the handler*, the probe never gets two originations in flight, and it measures nothing —
while the peer looks conformant.

**§4.8 forecloses that at MUST level, in the peer's own direction:**

> **Implementations MUST support inbound frame processing concurrent with outbound dispatch initiated
> from handlers.** … while a handler is processing a frame received on a connection, the
> implementation MUST be able to read and dispatch additional frames received on that same
> connection …

⚠ **And §4.8's permissions do not re-open it.** It allows bounding concurrency *"via worker pools,
semaphores, or back-pressure"* and queuing dispatched work — but it states the reconciling test
itself: *"the requirement is that inbound frame processing not block on outbound dispatch from the
same connection."* **A per-connection concurrency cap of one blocks inbound dispatch on the
completion of a handler that is itself awaiting an outbound response** — inbound processing blocking
on outbound dispatch, transitively, which is the thing forbidden. **Non-conformant**, from the
paragraph's own reconciling sentence rather than from an inference over it.

⇒ **N invocations is a sound driver for §6.11(a), and it is sound *because §4.8 holds the premise
up*.** That is worth saying rather than just answering *yes*: the two rules compose, and a seat
reading §6.11 alone cannot see why the driver works.

### 1.3 ⚠ What a fan-out would actually buy — and why we are not minting it

**A failing N-invocation probe cannot say WHICH rule broke.** *The peer serialized its outbound
dispatch* (§6.11(a)) and *the peer serialized its inbound dispatch* (§4.8) produce the **same
observable**: the N originations arrive one after another. A fan-out form — N originations from
**one** invocation — isolates §6.11(a) by holding §4.8 constant, because one inbound frame cannot be
serialized against itself.

**So `X6` has a real justification, and it is not the one it was filed under, and it is weaker.** It
is a **triage** refinement — spec-gap-vs-impl-bug is Stage 2 work and it is the filing seat's — not a
capability the measurement needs. ⇒ **Not minted here.** If the ambiguity is hit in a real run and
the attribution matters, that is the evidence to propose it on, and the proposal should come from the
seat that hit it.

⚠ **Recorded so the absence does not read as an oversight:** this is a deliberate non-grant of a
conditional ask whose condition failed, with a second ground for it named and declined on its own
terms.

---

## §2 `CQ-20` — a MUST with no expressible input

### 2.1 What is actually missing

> **§6.11(c)** Per-request deadlines MUST be enforced at the request layer, not via connection-wide
> deadline primitives … that would race across concurrent in-flight requests on the same connection.

**The obligation is a behaviour and the text names a mechanism** — §6.11 says so itself
(*"purely behavioral"*, *"impl-private and unobservable"*) — so the requirement is the behaviour the
forbidden mechanism breaks, in **both** directions a connection-wide deadline races: **shortening**
(one request's expiry fails another that had longer) and **extending** (a later deadline overwrites
an earlier one, so the first does not expire when it should).

**One probe exposes both: stagger two deadlines and answer the second between them.** *Staggering
requires setting them.* Nothing in any handler contract lets a prober set one, and the peer's own
value is neither declared nor necessarily finite — **one reference peer's reentry sender blocks
with no timeout at all.**

⇒ **The missing half:** a normative MUST naming a capability, with **no declared site a peer can
carry and no conformance check**, on a rule that has been normative for revisions. The scaffold is
the declared site; this proposal is the carrier.

### 2.2 ✅ The parameter

**`system/validate/dispatch-outbound` gains one optional param:**

```
deadline_ms: uint        ; optional. The per-request deadline to apply to THIS invocation's
                         ; outbound reentry EXECUTE, in milliseconds.
```

| | |
|---|---|
| **absent** | the peer applies whatever it applies today. **Unchanged behaviour, so every seat that ships the handler today is still conformant on the day this lands** |
| **present** | the peer **MUST** apply it as the per-request deadline on the reentry origination, enforced **at the request layer** (§6.11(c)) |
| **`0`, or a non-integer** | `400 invalid_params`. A zero deadline is degenerate, not a request to disable one |

### 2.3 ⭐ No silent substitution — the rule the negative control depends on

> **A peer that cannot honour the requested `deadline_ms` MUST refuse the invocation with
> `400 invalid_params`. It MUST NOT apply a different value.**

**This is the load-bearing half and it is easy to leave out.** The filing seat's requirement carries a
**mandatory anti-vacuity control** — one call, deadline `D`, never answered, MUST time out near `D` —
and branches on it: *if this arm fails, the run is SKIP (deadline input not honoured), not FAIL.*

**A peer that silently caps or floors the value fails that control for a reason the control cannot
name**, and the seat is then choosing between SKIP and FAIL with no way to tell which is right. A
refusal is a fact the prober can read; a substituted value is not. ⇒ **the peer says it cannot, or it
does what it was asked.**

⚠ **A peer MAY still have a ceiling** — it just has to say so by refusing, rather than by quietly
clamping. Nothing here obliges a peer to honour an arbitrarily large deadline.

### 2.4 ✅ What a timed-out sub-dispatch surfaces — the code is pinned, the shape is not

**Exactly §7a.1a's treatment, applied to the neighbouring outcome.** When the reentry sub-dispatch
expires on its deadline rather than being refused:

> **The surfaced code is `recv_timeout` (`ENTITY-CORE-PROTOCOL` §6.12, status `503`).** The status
> **shape** is not pinned — **relayed** (the handler propagates `503` as its own outer status) and
> **wrapped** (outer `200`, the timeout carried as the inner status) are both conformant, as §7a.1a
> already holds for refusals. **A generic or transport-catch-all code is non-conformant.**

**Why this belongs in the scaffold section and not only in §6.12.** §7a.1a's own answer transfers
verbatim: the scaffold is where the outcome is **caught and re-emitted**, and that re-emission is a
code path §6.12's authors were not describing. **A handler that wraps every unsuccessful
sub-dispatch in one generic failure launders a deadline expiry into a transport fault**, and the
resulting observable is indistinguishable from *"the route was broken"* — which is precisely the
`connection_broken` row sitting one line below `recv_timeout` in the same table.

⭐ **This upgrades something on the filing seat's side, and the change there is theirs to make.** Their
requirement records `call_1_code` as a **witness** rather than an assertion, correctly, because
**§9.4 makes the in-process representation implementation-defined.** At the **scaffold's wire
boundary** it is not implementation-defined — it is the code this section pins — so that witness can
become a scored arm. **We are not editing their file and not asking them to; we are saying the
constraint changed.**

### 2.5 ⭐ What we are NOT adding: elapsed-time reporting

**The ask names the handler as *"reporting per call: outcome and time-to-outcome."* We are granting
the outcome half and declining the timing half, and the decline makes the check stronger.**

**The suite is the counterparty and already holds both endpoints of the interval.** It sends the
invocation; it receives the response the handler returns when the deadline fires. `t(response) −
t(invocation)` is measured **externally, on one clock, by the party being lied to if the peer is
wrong** — and the requirement already carries a `clock_tolerance` posture axis that absorbs the
transit terms.

⇒ **A self-reported duration is the peer grading its own timing**, on the one axis the check exists to
measure. **A peer whose request-layer deadline is broken is exactly the peer whose self-report cannot
be trusted**, so adding the field would put the measurement inside the thing under test. **Adding
nothing is the better contract**, and the existing `{ status, result }` shape is untouched.

---

## §3 The edits

| # | file | change |
|---|---|---|
| **E1** | `guides/GUIDE-CONFORMANCE.md` §7a.1 | `deadline_ms` added to the `dispatch-outbound` params contract, marked optional, with the absent/present/invalid rows |
| **E2** | `guides/GUIDE-CONFORMANCE.md` new §7a.1b | the deadline contract: no silent substitution · `recv_timeout`/503 with shape unpinned · why no elapsed-time field · and the §4.8 composition that makes N invocations a valid §6.11(a) driver |

**Additive: no existing behaviour changes, and a peer that ships the handler today stays conformant**
until a probe sends the new param.

---

## §4 Who this is a delivery to

| seat | why | what changes for them |
|---|---|---|
| **the reference-peer generator** | implements the scaffold in three peers; one of them has a reentry sender that **blocks with no timeout**, which is the instance | accept `deadline_ms`; refuse rather than clamp; surface `recv_timeout` |
| **the lead reference implementation** | builds the validator side and ratified the §7a.2a params shape | same, plus the driver side |
| **the other reference implementations** | ship the scaffold; reached through the lead seat | same |
| **the conformance instrument** | the filing seat | `X6` withdraws with its reason; `X5` lands; §2.4 changes a witness into a scoreable arm **in their file, on their call** |

⚠ **The divergence unit is the params set, and it is all-or-none-adjacent to an existing rule.** §7a.1
already declares the `reentry_*` triple *"all-or-none: supplying the three selects the presented arm,
omitting all three selects the ambient arm, and a partial set is `400 invalid_params`."*
**`deadline_ms` is independent of that triple** and composes with either arm — stated here because a
reader meeting a fourth optional param beside a three-member all-or-none set will otherwise ask.

---

## §5 Open

| | |
|---|---|
| ⚠ **`GUIDE-CONFORMANCE` carries no `**Version**:` header** | `SPECIFICATION-FORMAT` §5.3 marks the field **required**, and this document — which every implementer reads to learn what it must pass — declares only `**Status**: Draft`. **So the rule ruled today has nothing to move here.** Found while applying it; **not fixed in this fold**, because choosing a starting number for a long-lived document is its own decision and folding it silently into a params change is how a version line comes to mean nothing |
| **The fan-out** | §1.3. Declined as filed and as re-derived; the evidence that would reopen it is a real run where §6.11(a) and §4.8 cannot be told apart |
| **Whether a peer's default deadline should be declarable** | the ask's *"or the peer's configured value, declared"* half. Not granted: a declared default is a second surface for the same fact, and the explicit param makes it unnecessary for the probe. **Named rather than dropped** |
