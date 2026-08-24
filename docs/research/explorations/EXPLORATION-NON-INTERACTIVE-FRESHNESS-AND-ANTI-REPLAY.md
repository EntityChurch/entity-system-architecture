# EXPLORATION — non-interactive freshness & anti-replay: the trade-off surface and the three knobs

**Status:** Exploration — 2026-07-21. **NOT** a proposal (a companion DRAFT proposal is warranted and named in
Part H). Maps the design space that `REVIEW-gaps-in-keystone-analysis.md` opened (Gap 1 + Gap 4) and that the
`PROPOSAL-KEYSTONE-CROSS-SUBSTRATE-HARDENING` bundle routed to **W7** from its §F1 (RT-6) and §F3 (RT-9).
Author: arch workspace.

**The finding in one line.** The protocol's freshness/anti-replay story is **complete for the interactive
handshake and deliberately absent everywhere else** — and that absence is *correct as a floor* but leaves three
un-declarable knobs a real deployment needs: (1) **no strong non-interactive freshness mechanism** for
relay/one-way/store-and-forward; (2) **no declarable bound** on revocation-propagation latency (the exposure
window is un-reason-about-able); (3) **no stated clock model or skew tolerance** at the temporal check (issuer
clock vs verifier clock silently perturbs every relayed TTL). All three are the **§4.10 pattern applied to
freshness**: provide the mechanism + the knob, mandate the bound *exists and is declared/enforced*, leave the
*value* to the deployment. None is a before-freeze wire change.

**Provenance (read-only, cited; two exhaustive absence-sweeps).** Core normative corpus
(`entity-core-protocol/specs/*`) and the arch extension/guide surface (`specs/extensions/EXTENSION-*.md` ×25,
`guides/GUIDE-*.md` ×40). Every "absent" below was run as a full-term sweep across **both** repos before being
claimed — the two sweeps and their term-sets are recorded in the session handoff. Citations pin `(section-or-symbol,
path)`; core §NNN anchors are current-state line numbers in `ENTITY-CORE-PROTOCOL.md` v0.8.0.

---

## Part A — The frame: substrate-broad, topology-narrow

Keystone proved **substrate** diversity (~45 language/runtime substrates reproducing the wire + authority
core) but exercised almost entirely **one communication topology: the interactive, direct, request/response
handshake** (`REVIEW-gaps-in-keystone-analysis.md` §"The frame"). The protocol supports several
**non-interactive** topologies the harness never drove, and every freshness gap lives there.

**The freshness model is bifurcated, and only one half is built out:**

- **Interactive path — complete.** The connect handshake carries a per-connection challenge: the responder
  issues a `nonce`, the initiator echoes it in `authenticate`, mismatch/absent → `401 invalid_nonce`
  (`ENTITY-CORE-PROTOCOL.md` §4.6, L1803). "The nonce is the anti-replay mechanism" (L1811). RT-6 (`§F1` of the
  hardening bundle) hardens single-use here and **scopes the MUST to this path** — correctly, because the nonce
  is the *only* freshness primitive and it cannot leave the connection.

- **Non-interactive paths — signatures + TTL + revocation only, by the spec's own statement.** §2.4 L210:
  *"Signatures prove authenticity… They do not prove freshness… Freshness requires re-fetching from the
  authoritative peer or receiving updates through a subscription."* There is **no challenge** in these flows —
  confirmed absent across the entire corpus (Part B).

**The four non-interactive topologies (each a real, specified flow):**

| Topology | What carries the cap | Where specified |
|---|---|---|
| **Relay / forwarded dispatch** | opaque, signed inner envelope the intermediary never decodes | §302 (relay carve-out concept), §1756 (format-fidelity carve-out); `EXTENSION-RELAY.md` §1/§9 |
| **One-way cap re-verification "away from its issuer"** | detached-signature cap chain; remote verifier cannot consult issuer's local tree | §922; §6.9a.0 detached-sig shape (L3568); cross-peer chain-construction registry (§5.8, L2829) |
| **Store-and-forward / async delivery** | queued signed envelope, TTL-bounded | `EXTENSION-INBOX.md` (TTL replay window); `EXTENSION-SUBSCRIPTION.md` |
| **Asynchronous revocation** | content-addressed revocation marker, sync-propagated | §2880 (convergent Layer-1 input); `EXTENSION-IDENTITY.md` §9.5 |

**Why the monoculture hid all three gaps:** the generator produces interactive peers; the oracle drives them
interactively; a relayed / one-way / store-and-forward flow is **not a shape the harness emits**. Substrate
diversity cannot surface a topology it never exercises — *every* substrate was tested in the same one.

---

## Part B — Red-team, per topology (what breaks, and why the nonce can't save it)

Each topology below is walked as an adversary would. The common root: **a signature proves *who and what*, never
*when* — and the nonce, the only "when" primitive, is connection-scoped and absent here.**

### B1 — Relay / forwarded dispatch

The relay is **byte-opaque transport by hard design**: it "carries opaque, signed, capability-bearing
envelopes… the capability chain passes through unchanged; the intermediary is transport"
(`EXTENSION-RELAY.md` §1, L18); the destination "verifies the inner signature **exactly as on a direct
connection**" (L135/L146), a byte-identical-to-direct promise strong enough that it resolved the Rust-vs-Python
raw-frame divergence in favor of raw-frame (L144). The relay's only anti-replay-adjacent field is `ttl_hops`, a
**hop-count loop bound, explicitly not a freshness bound** (L97).

**Attack.** A relay (or a passive observer of one) captures a validly-signed inner envelope carrying a
still-in-TTL capability and **re-injects it later** (or to a different destination reachable through the mesh).
The destination verifies signature + chain + TTL and accepts — it has no per-message challenge to bind the
message to *this* exchange, because the origin issued none and the relay cannot inject one (it is byte-opaque).
Within the TTL window the message is replayable with no recourse.

**Why the nonce can't help.** The origin never spoke to the eventual verifier; there is no connection nonce to
echo. And the relay *cannot* mint one on the origin's behalf — it never decodes the envelope (§9 opacity).

### B2 — One-way capability re-verification away from its issuer

The detached-signature cap is *designed* to be verified with no contact to the issuer: "a remote verifier
cannot consult the issuer's local tree" (§6.9a.0, L3572), so freshness-by-re-fetch (§210's first escape hatch)
is **structurally unavailable** on exactly the path that most needs it. The cross-peer chain-construction
registry (§5.8, L2829) enumerates the sites that build such chains — SUBSCRIPTION, CONTINUATION, ROLE, COMPUTE —
and requires identity slots + signature discovery + chain-inclusion, but **no freshness/recency term appears**.

**Attack.** A cap minted with a generous TTL, collected into a bundle, and re-verified at a peer the issuer has
never contacted: the verifier can confirm authenticity and structural validity but has **no way to know the
issuer hasn't since narrowed/rotated/retired the authority** except the two escape hatches §210 names — re-fetch
(unavailable here) or subscription (async, Gap 2).

### B3 — Store-and-forward / async delivery

`EXTENSION-INBOX.md` bounds replay only by TTL ("TTL bounds the replay window", per the sweep);
`EXTENSION-ENCRYPTION.md` is explicit that it "does NOT provide replay or freshness protection… per-message
freshness/ordering is a session-extension concern (§20), not a stateless single-shot one." So a queued message
is replayable within its TTL, and the layer that *could* add per-message freshness (a session extension) is
named but not the one in play for single-shot cap-bearing envelopes.

### B4 — Asynchronous revocation (the Gap-2 seam)

§2880 makes revocation a **convergent** Layer-1 input: "observation is asynchronous (a peer may not yet have
synced a marker another peer has)." This is *correct* — markers are content-addressed and designed to converge —
but the **propagation latency is nowhere bounded**. `EXTENSION-IDENTITY.md` §9.5 names the resulting window an
**"adversarial surface"**: "A peer that has observed a controller-cert but not yet its revocation will validate
caps the controller can no longer authorize. The window's bound is the convergence latency" (L1165), "a real
exposure that the architecture **bounds rather than eliminates**" (L1290). GROUP echoes it four times as
"sync-latency-bounded" (L711/L888/L1012/L1056), REGISTRY leaves it an **open question** ("how fast does the
revocation propagate…? Per-backend", L921).

**The seam:** the property is *canonical and cross-referenced* but **un-quantified** — a high-threat deployment
is told the window "matters" (IDENTITY L1165) and given no protocol handle to bound it.

### B5 — The clock composition under all of the above (Gap 4)

Temporal validity is a **bare comparison against the verifier's own clock, with no tolerance term**:
`t = now()` sampled once (§5.10, L2481); `if t < not_before: DENY` / `if expires_at < t: DENY` (L2569–2570).
Timestamps are **issuer-stamped wall-clock epoch ms** — "set by the handler from **server wall-clock**" (§2965,
L2965). So across any B1–B4 hop, an **issuer-clock** `expires_at` is compared to a **verifier-clock** `t` with
**no reconciliation clause anywhere in the corpus** (Part C confirms the absence). If the two clocks are skewed
by δ, the effective TTL is silently lengthened or shortened by δ. §5.10 correctly handles the *single-verdict*
case (two peers near a boundary may differ; `t` is a declared input, not a leak) — but it says nothing about the
**relationship between the issuer's TTL-origin clock and the verifier's `t` across a hop**, which is exactly the
non-interactive case.

---

## Part C — What is genuinely absent (the proven negatives)

All four claims below were run as full-term sweeps across the **core spec + 25 extension specs + 40 guides**.
Each holds; the design-shaping constraints found alongside are what make the knobs in Part D non-naive.

1. **No strong non-interactive freshness mechanism.** No challenge-response / nonce-back-through-relay /
   recency-proof exists beyond the interactive §4.6 nonce. `EXTENSION-ATTESTATION`'s `is_attestation_live` is
   **structural only** (expiration + supersession + self-revocation), explicitly *not* a recency proof.
   **Design constraint found:** `GUIDE-MULTISIG.md` §"Why caps can't enforce at-most-once" (L284/L324) — "Caps
   are stateless by construction"; V7 **deliberately** rejects nonce-caps (unbounded consumed-nonce registry,
   domain-specific pruning, doesn't compose with handler idempotency). *Any* Knob-1 design that adds state to
   the cap layer contradicts a settled decision.

2. **No clock-sync assumption or skew bound.** `EXTENSION-CLOCK.md` makes synchronization an explicit
   **non-goal** ("NTP, PTP — external"; "no global clock — peers maintain local clocks", L31–32/L48;
   "Monotonicity is not guaranteed", L74) and points to logical/vector clocks for ordering (L594). The temporal
   check carries **no leeway/grace/tolerance term** (Part B5). **Design constraint found:** Knob 3 cannot add
   wall-clock sync — it must be a declared *tolerance*, not a *synchronization*.

3. **No revocation-propagation-latency bound.** Every treatment is qualitative "sync-latency-bounded" (IDENTITY
   §9.5, GROUP ×4) or an open per-backend question (REGISTRY L921). No numeric/SLA/declarable maximum, and no
   "declared exposure window," anywhere. **Design pattern found:** the convention *already exists* and is
   cross-referenced — it just isn't declarable. This is the §4.10 "ratify observed convergence" setup exactly.

4. **Relay adds no freshness by design; attestation is not a freshness primitive.** `EXTENSION-RELAY` delegates
   all freshness/replay verification to the destination "byte-identical to a direct connection" (L146) and
   cannot see inside the opaque inner envelope (§9). Confirmed: freshness for forwarded caps must be an
   **end-to-end origin↔verifier** property, never a relay feature.

---

## Part D — The trade-off spectrum and the three knobs

**The spectrum (per topology, the operator's real choice):**

```
STRONG / EXPENSIVE  ─────────────────────────────────────────────►  CHEAP / ASYNC-TOLERANT / WEAKER
end-to-end challenge-response          TTL + revocation-convergence + content-addressed dedup
(origin re-signs against a verifier    (no round-trip; works store-and-forward / offline;
 nonce carried opaquely through the     exposure window = min(TTL granularity,
 relay) — extra round-trips, needs                       revocation-propagation latency ± clock skew))
 the origin reachable
```

The right posture is the one the spec already uses for resource bounds (§4.10 L1862: "have and enforce finite
bounds… the *values* are deployment-dependent", which itself "ratifies observed convergence") and bootstrap
(§3036, implementation-defined): **provide the mechanisms and the knobs; let the deployment set its security
threshold.** The protocol should not mandate one freshness point for all non-interactive traffic — but it
should make each point *expressible and reason-about-able*. Today none of the three is.

### Knob 1 — reuse the interactive handshake over a relay circuit (the strong end; Gap 1)

**Shape (revised after the 2026-07-21 relay/encryption review — supersedes the earlier "new freshness-assertion
type" sketch).** The strong end needs **no new mechanism**: the §4.6 handshake — whose nonce is *already* the
protocol's freshness primitive — runs **end-to-end origin↔verifier over a relay circuit**, with the relay as
byte-opaque transport. The nonce is the freshness proof; the cap layer stays stateless because the nonce is
**connection** state, which it always was (fully respects the MULTISIG stateless-cap decision — nothing is added
to the cap). This is the "extra round-trips, origin must be reachable" end of the spectrum, sourced from
existing machinery rather than invented.

**What it composes with (both currently deferred/sibling — this is the "analysis to get right"):**
- **RELAY Mode C (Circuit)** — "a virtual circuit between two peers that can't dial each other directly"
  (`EXTENSION-RELAY.md` §1/§3.4), **deferred from v1** (§11.1); its normative text is a follow-on. Mode C is the
  vehicle that lets a live bidirectional §4.6 handshake tunnel through the relay. (Mode F/S, the v1 modes, are
  single-envelope + async INBOX/CONTINUATION reply correlation — §6.2 — not a live session, so end-to-end they
  share **no connection and thus no nonce**; that is exactly why B1 breaks under v1 relaying.)
- **Confidentiality over an untrusted/public relay** — the relay forwards plaintext ECF bytes unless encrypted,
  so a public relay can read a tunneled handshake. `EXTENSION-ENCRYPTION.md` §1 delivers "end-to-end through
  relays; untrusted intermediaries can't read content" (peer-mode, **stateless single-shot**, no PFS, §35/§510).
  A *live interactive* handshake is stateful-interactive, whose planned home is the sibling
  **`EXTENSION-ENCRYPTED-SESSION`** (Noise XK / MLS; §20/§35), out of ENCRYPTION v1. So Knob 1's confidentiality
  is peer-mode-ENCRYPTION now / session-sibling later.

**v1-available fallback (if a live circuit is unavailable):** an **async two-message challenge-response** — the
verifier forwards a nonce (Mode F) and the origin replies via INBOX `deliver_to`/`deliver_token` + a CONTINUATION
at the reply path (§6.2). This works in v1 but *is* a small new application-level exchange (a freshness-challenge
type + reply), and still needs the origin reachable-before-the-challenge-expires. Prefer the handshake-over-circuit
(1b) shape once Mode C lands; the async form is the bridge.

**Not before-freeze; not a MUST; interactive-only.** A deployment that accepts the cheap end never asks for it,
and the genuinely origin-offline async case **cannot** use Knob 1 (no live origin to answer a nonce) — it falls
to knobs 2 + 3.

### Knob 2 — a deployment-declarable revocation-propagation bound (makes the cheap end reason-about-able; Gap 2 / RT-9)

**Shape (ratifies C3):** promote the already-canonical "sync-latency-bounded" property (IDENTITY §9.5) into a
**declared** value: a deployment publishes a `revocation_propagation_bound` (a max convergence latency it
commits to via its sync/subscription cadence). The exposure window then becomes
`min(TTL granularity, declared-bound)` — a **number a high-threat operator can reason about**, which is exactly
what IDENTITY L1165 says "matters" but leaves unbounded. This is the §4.10 move: mandate the bound *exists and
is declared/enforced* (fast sync to honor it), leave the *value* to the deployment. It unifies the scattered
GROUP/IDENTITY/REGISTRY "sync-latency-bounded" language into one place instead of re-deriving it per extension.

### Knob 3 — a declared clock model + skew-tolerance term (closes Gap 4)

**Shape (constrained by C2):** state, once, at the temporal check, the clock model CLOCK deliberately leaves
external — **wall-clock timestamps are compared across uncoordinated clocks** — and provide a
deployment-declarable **skew-leeway** `δ` folded into the comparison: `expires_at + δ < t → DENY`,
`t + δ < not_before → DENY`. This is a *tolerance*, not a *synchronization* (so it does not reopen CLOCK's
non-goal), and it makes the relayed-TTL composition (Part B5) reason-about-able: the effective window is
`TTL ± δ`, declared rather than silent. For deployments needing strong ordering, CLOCK L594 already routes to
logical/vector clocks — Knob 3 is the *floor* that makes bare wall-clock TTLs honest, not a replacement for that.

**Interaction:** Knobs 2 + 3 compose into a single declarable **exposure-window budget** the operator sets;
Knob 1 is the escape hatch for flows that need better than that budget and can pay the round-trips.

---

## Part E — What lands now vs. what is W7

- **Landed (this pull-in):** RT-6 nonce single-use, **scoped to the interactive handshake** (hardening bundle
  §F1). The interactive half is done; it was never the gap.
- **W7 (this exploration → the companion proposal):** the three knobs. None before-freeze; none a wire
  renumber; each is the §4.10 "provide the knob, declare the bound, leave the value" pattern. RT-6 and RT-9 are
  revealed as **two symptoms of this one surface** — the interactive symptom is fixed, the non-interactive
  surface is the knobs.

---

## Part F — Conformance-mode sketch (cohort-side; not a spec change here)

The monoculture is a *test-harness* gap as much as a spec one. A conformance mode that exercises the knobs must
**drive the non-interactive topologies at both ends of the spectrum**, which pairwise-connect never does:

- A **relay/one-way re-verification** matrix: mint at A, relay through B (byte-opaque), verify at C with A never
  having contacted C — assert the detached-sig path (§6.9a.0) and the Knob-1 challenge round-trip when elected.
- A **revocation-window** probe: revoke at A, measure observed-vs-declared propagation against a deployment's
  `revocation_propagation_bound` (Knob 2) — the exposure window becomes a *measured* quantity, not a hope.
- A **skew** probe: verify a relayed TTL with the verifier's clock offset by ±δ; assert the Knob-3 leeway makes
  the verdict boundary declared rather than clock-dependent.

This rides the same meta-limit as `REVIEW-gaps-in-keystone-analysis.md` Gap 2 (the Go-derived oracle) and Gap 3
(no keystone peer ships compute): **the corpus only tests shapes the oracle emits.** Route to the cohort as a
new `validate-peer` topology category, not before-freeze.

---

## Part G — Honest ledger

- **All three knobs are DRAFT-track, not ratified.** Nothing here touches the locked wire core; nothing is a
  renumber. Each is additive and deployment-elected.
- **Knob 1 is a *composition*, not a new mechanism (revised).** The 2026-07-21 relay/encryption review found the
  strong end is the existing §4.6 handshake tunneled over a relay circuit + encryption — reusing proven
  machinery, not a new `freshness_assertion` type. That de-risks it (no stateful cap surface), but moves the
  dependency onto **two deferred/sibling pieces**: RELAY **Mode C** (deferred, §11.1) and, for public relays,
  ENCRYPTION peer-mode now / the **`EXTENSION-ENCRYPTED-SESSION`** sibling later. So Knob 1 is the *slowest* of
  the three (it waits on Mode C), even though it is the cleanest. Its honest boundary: **interactive-only** — it
  serves the both-peers-online case, never the origin-offline async case.
- **Knobs 2 + 3 are the higher-confidence, higher-leverage pair.** They *ratify and quantify* conventions the
  spec already states qualitatively (IDENTITY §9.5; the §4.10 pattern), which is the safer and more immediately
  useful move. If W7 ships incrementally, ship 2 + 3 first.
- **Evidence class:** every absence is sweep-proven across both repos (Part C); every design constraint is
  pinned verbatim (C1–C4). This exploration is prose-reviewed only — per the CDN-corridor meta-rule, the knobs
  are **not validated until the cohort exercises the Part-F topology mode**. State that in the proposal.
- **Scope discipline:** this stays in the spec lane — it records the spec delta + the MUST/SHOULD shape + the
  owning surface. It does **not** track what impl teams owe.

---

## Part H — Recommendation & next unit

**A companion DRAFT proposal is warranted** — the three knobs are concrete and the design constraints are
resolved. Recommended unit: **`PROPOSAL-NON-INTERACTIVE-FRESHNESS-KNOBS`** (arch `docs/proposals/`, or directly
in `entity-core-protocol/docs/proposals/` per the "manage core directly" pattern), structured as:

1. **Knob 2 + Knob 3 first** (ratify-and-quantify; higher confidence) — the declarable
   `revocation_propagation_bound` and the temporal-check skew-leeway `δ`, each written as the §4.10 pattern.
2. **Knob 1 — deferred (decided 2026-07-21).** The "promote the async bridge to v1, or defer?" question was
   assessed from first principles + landscape in the companion
   `ANALYSIS-NON-INTERACTIVE-FRESHNESS-CRITICALITY.md`: **defer** strong non-interactive freshness to a future
   extension (DPoP-pattern), **reject** the async two-message bridge outright, and land only an informed-deferral
   security note in v1. Absence is not a divergence risk (it's a uniform floor); it is not critical (handler
   idempotency + short TTL + revocation already cover it); the cost is asymmetric (app-level workaround, zero
   protocol change). The eventual mechanism is the §4.6-handshake-over-a-Mode-C-circuit form.
3. A **Part-F conformance-mode** note routed to the cohort as the validation gate.

## References

- Companions: `REVIEW-gaps-in-keystone-analysis.md` (Gap 1 + Gap 4, the seed), `REVIEW-keystone-retrospective-and-red-team.md`.
- Routed from: `entity-core-protocol/docs/proposals/PROPOSAL-KEYSTONE-CROSS-SUBSTRATE-HARDENING.md` §F1 (RT-6, scoped) + §F3 (RT-9 → W7).
- Tracker: `docs/status/WORKSTREAMS.md` (W7, enlarged 2026-07-21).
- Core spec anchors (`entity-core-protocol/specs/ENTITY-CORE-PROTOCOL.md` v0.8.0): §210 (freshness), §302/§1756
  (relay), §922 (re-verify away from issuer), §2569–2570/§2481 (temporal check vs `t`), §2829 (§5.8 cross-peer
  chain registry), §2876/§2878 (`t` per verdict), §2880 (async revocation), §2965 (issuer wall-clock stamp),
  §3568/§3572 (§6.9a.0 detached-sig), §1862 (§4.10 bounds pattern), §3036 (impl-defined bootstrap).
- Extension/guide anchors (`entity-system-architecture/specs/extensions/`, `guides/`): `EXTENSION-RELAY.md`
  §1/§9 (byte-opaque transport, L18/L97/L135/L146), `EXTENSION-CLOCK.md` §Non-goals (L31–32/L48/L74/L594),
  `EXTENSION-IDENTITY.md` §9.5 (L1165/L1290 convergence window), `EXTENSION-GROUP.md` (L711/L888/L1012/L1056
  sync-latency-bounded), `EXTENSION-REGISTRY.md` (L921 open question), `EXTENSION-ATTESTATION.md` (structural
  liveness, L24/L296), `EXTENSION-INBOX.md` (TTL replay window), `EXTENSION-ENCRYPTION.md` (L936 session-freshness
  home), `GUIDE-MULTISIG.md` §"Why caps can't enforce at-most-once" (L284/L324 stateless-cap decision).
