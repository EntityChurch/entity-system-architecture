# ANALYSIS — is strong non-interactive freshness (W7 Knob 1) critical for v1? A first-principles + landscape assessment

**Status:** Analysis — 2026-07-21. **NOT** a proposal; it *decides* an open question inside
`PROPOSAL-NON-INTERACTIVE-FRESHNESS-KNOBS.md` (the Knob-1 "async two-message bridge: promote to v1, or defer?").
Companion to `EXPLORATION-NON-INTERACTIVE-FRESHNESS-AND-ANTI-REPLAY.md`. Author: arch workspace.

**The question (operator-posed).** W7's Knob 1 is strong freshness for a *relayed* capability (challenge-response
instead of just TTL+revocation). The strong form (§4.6 handshake over a relay circuit) waits on deferred
machinery; a cheaper **async two-message challenge-response** could ship in v1. Should it? Is Knob 1 *critical*,
or is TTL + revocation (Knobs 2/3) the correct floor with strong freshness deferred to an extension?

**The decision in one line.** **Defer Knob 1; land Knobs 2/3; do not promote the async form to v1** — but make
the deferral *informed* with one v1 security-consideration note. The absence of strong non-interactive freshness
is **not a divergence risk** (it's a uniform verify floor); it is **not critical** (the architecture already
assigns replay defense to handler idempotency + short TTL); and the **cost is asymmetric** — adding it *creates*
interop surface for a niche, while deferring costs ≈0 because the workaround is application-level and needs zero
protocol change. Every comparable stateless-credential system made the same call.

---

## Part A — First-principles analysis

### A1 — What is the actual residual threat?

With Knobs 2/3 landed, a relayed capability verified at peer C is checked: signature + chain + attenuation +
caveats + `TTL ± δ` + revocation (bounded by C's declared `revocation_propagation_bound`). The residual attack
is: **an authentic, authorized, in-window message captured on/around a relay and replayed within
`min(TTL_granularity, revocation_bound) ± δ`.**

Three properties bound its blast radius *before* any freshness mechanism:

1. **It is not escalation.** Replay re-executes an *already-authorized* operation; it grants no new authority
   (attenuation + the cap chain are unchanged). The damage is confined to the effect of re-running that one op.
2. **Idempotent ops are immune.** Reads and content-addressed puts are replay-safe by construction. The threat
   is only non-idempotent state mutations.
3. **The architecture already assigns this to the handler.** `GUIDE-MULTISIG.md` §"Why caps can't enforce
   at-most-once": V7 *deliberately* keeps the cap layer stateless and lets "handlers own idempotency where they
   can express it natively." So replay defense for non-idempotent ops is **already a handler-layer
   responsibility**, not a gap awaiting a cap-layer freshness primitive.

**Residual niche after those three:** a non-idempotent, relayed operation, whose handler implements no
idempotency, in a deployment whose minimum acceptable TTL is longer than its threat tolerance. Real, but narrow
and deployment-controllable.

### A2 — Divergence risk (the load-bearing lens for this repo)

The repo's primary correctness concern is cross-impl interop: a MAY/SHOULD two conformant peers read differently
across a peer boundary (`AGENTS.md` "pin the cross-impl-observable surface"). Apply it both ways:

- **Absence of Knob 1 → no divergence.** Every conformant peer verifies a relayed cap with the *same* floor
  algorithm (sig+chain+TTL±δ+revocation). There is nothing optional to read two ways. The freshness *posture*
  (how short you set TTLs, how fast you sync revocations) is a per-deployment value, not a wire-observable
  behavior that can split a peer boundary. → **not an interop hazard.**
- **Presence of Knob 1 → new divergence surface.** An optional challenge-response is, by construction, a
  MAY/SHOULD handshake: peer C *may* demand it, origin A *may* support answering it, with matching semantics
  (what counts as a valid response, the challenge's own freshness, negotiation when unsupported) to pin across
  the boundary. This is precisely the "a MAY/SHOULD that diverges across a peer boundary → pin it" hazard the
  guide names — i.e. *adding* Knob 1 manufactures the very interop surface the repo works to avoid. (The
  real-world witness: DPoP's optional server-nonce negotiation is a known integration-complexity source — Part B.)

**Conclusion:** the divergence lens argues *for* deferral. The genuine cross-peer-observable freshness surface is
Knobs 2/3 (the TTL/revocation/skew accounting) — which is exactly what v1 pins.

### A3 — Cost of adding (v1) vs. cost of deferring (extension)

| | Cost of **adding** the async two-message form to v1 | Cost of **deferring** to a future extension |
|---|---|---|
| Spec surface | new entity type(s) (challenge + reply), verifier "when to demand" logic, reply-correlation binding | one security-consideration note (Part D) |
| Interop | new MAY/SHOULD handshake to pin (A2) | none added |
| Conformance | new vectors + a "supports-freshness-challenge" tier split | none |
| Maintenance | a second, weaker freshness path that the real mechanism (session/DPoP-style) later supersedes → drift/dual-path | none |
| **Benefit foregone by deferring** | — | **≈ zero: the workaround is application-level and needs no protocol change** — an app that wants a nonce puts it *in the operation payload* and its *handler* rejects duplicates (exactly the "handlers own idempotency" path A1.3 already sanctions) |

The asymmetry is stark: adding buys a niche a defense it can already build itself at the app layer, at the
price of a new interop/conformance/maintenance surface and a soon-superseded second path. Deferring forgoes
almost nothing.

### A4 — Why the *strong* form (not the async bridge) is the right eventual shape

When Knob 1 does land, it should be the §4.6-handshake-over-a-circuit form (reuse the nonce; zero new cap
machinery), not the async two-message exchange. The async form is a *new* weaker mechanism that the handshake
form and the planned `EXTENSION-ENCRYPTED-SESSION` both dominate. Shipping the weak form in v1 would create a
dual path we later deprecate — the worst of both. So the async bridge is rejected outright, not merely deferred.

---

## Part B — Landscape (how comparable stateless-credential systems decided)

Every system in the adjacent design space treats strong non-interactive freshness as an **optional, layered,
later** concern — never a mandate baked into the credential core.

- **OAuth 2.0 / DPoP (RFC 9449).** The dominant modern answer to bearer-token replay over intermediary-heavy
  routes. Two facts are directly on-point: (1) DPoP is *"an extension to OAuth 2.0 rather than modifying the
  core bearer token specification, allowing for broader compatibility"* — i.e. **freshness/PoP is deferred out
  of the token core by design**; (2) its **nonce challenge-response is itself the optional *stricter* tier** on
  top of base key-binding (*"to provide stricter protection… DPoP supports a challenge-response mechanism using
  a nonce"*). Two nested "keep it optional / keep it out of core" decisions. Best-practice guidance places PoP
  exactly on "flows that traverse browsers / intermediary-heavy routes" — our relay case — but as a
  **deployment-selected** option, not a floor.
- **Biscuit / Macaroons** — V7's own family (stateless, offline-verifiable, attenuating caps). They rely on
  **short TTL + attenuation**; they do **not** carry per-message freshness. Nonce identifiers appear only for
  *revocation* (Fly.io's nonce-macaroons), and the literature states the rule plainly: *"the strongest replay
  defenses often require memory such as a used-token set, but stateless designs frequently [substitute]
  self-expiring tokens or challenge-response."* A stateless cap system buys freshness only by **abandoning
  statelessness** — which V7 has explicitly refused (A1.3).
- **Kerberos** — the one adjacent system with a *mandatory* freshness proof (the authenticator: timestamp +
  nonce + replay cache). It is the exception that proves the rule, because it requires exactly the two things V7
  deliberately rejects: a **stateful verifier replay cache** (the stateless-cap refusal) and **synchronized
  clocks** — *with a declared ±5-minute skew tolerance*. That last detail independently **validates Knob 3**:
  the canonical mandatory-freshness system reconciles cross-clock verification with a *declared tolerance window*,
  precisely Knob 3's `δ`. Kerberos is also KDC-mediated/interactive, not store-and-forward.

**Landscape verdict:** deferring strong non-interactive freshness to an optional extension, while shipping
short-TTL + revocation as the floor, is the **mainstream, well-trodden** choice for stateless bearer credentials.
The mandate-it path (Kerberos) demands the two architectural properties V7 has already ruled out.

---

## Part C — The decision

1. **Defer Knob 1** (strong non-interactive freshness) to a future extension, DPoP-pattern: named, composed, and
   documented now; specified when RELAY **Mode C** + `EXTENSION-ENCRYPTED-SESSION` land. Not v1.
2. **Reject the async two-message bridge for v1** outright (A4) — it is a weaker, soon-superseded second path
   that manufactures interop surface (A2) for a benefit already available at the app layer (A3).
3. **Land Knobs 2 + 3 in v1** unchanged — they *are* the cross-peer-observable freshness surface, they carry the
   real divergence risk, and Knob 3 is landscape-validated (Kerberos's declared skew tolerance).
4. **Make the deferral informed, not silent** — add the one v1 security note in Part D (the DPoP discipline:
   when you defer PoP, you state the residual risk + the mitigation ladder).

---

## Part D — What v1 adds instead (near-free, no divergence surface)

A **security-consideration note** at the core spec's relay/freshness discussion (§210 / the §4.6 boundary),
making the residual window explicit and giving the mitigation ladder — so a deployment chooses its posture
*informed*:

> **Non-interactive replay window (security consideration).** A capability delivered over a relay or other
> non-interactive path (one-way re-verification, store-and-forward) is authenticated and authorized but **not
> proven fresh** (§210): within `min(TTL_granularity, revocation_propagation_bound) ± δ` an authentic, in-window
> message can be replayed. This is bounded, not eliminated. Deployments reduce it, in order of leverage:
> (1) **handler idempotency** for non-idempotent operations (the protocol's primary replay defense — caps are
> stateless by design; handlers own at-most-once where they can express it natively); (2) **short TTLs** on
> relayed caps; (3) a **tight `revocation_propagation_bound`** (Knob 2); (4) a direct (non-relayed) connection
> for the highest-value operations, where the §4.6 handshake nonce gives interactive freshness. Strong
> non-interactive freshness (challenge-response over a relayed circuit) is a **future extension**
> (DPoP-analogous; composes with RELAY Mode C + `EXTENSION-ENCRYPTED-SESSION`), deliberately not mandated in the
> stateless cap core.

This costs one paragraph, adds **no** wire/type/conformance surface, and carries **no** divergence risk (it is a
non-normative consideration, not a MAY/SHOULD behavior). It converts a silent deferral into an engineered one.

---

## References

- Decides the open question in: `entity-core-protocol/docs/proposals/PROPOSAL-NON-INTERACTIVE-FRESHNESS-KNOBS.md` (Knob 1).
- Design record: `EXPLORATION-NON-INTERACTIVE-FRESHNESS-AND-ANTI-REPLAY.md`; seed: `REVIEW-gaps-in-keystone-analysis.md`.
- Architecture anchor: `guides/GUIDE-MULTISIG.md` §"Why caps can't enforce at-most-once" (stateless-cap decision, handler idempotency).
- Landscape (accessed 2026-07-21): OAuth DPoP [RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449) +
  [Auth0 DPoP overview](https://auth0.com/blog/protect-your-access-tokens-with-dpop/); Kerberos
  [RFC 4120](https://datatracker.ietf.org/doc/html/rfc4120) +
  [MIT replay cache](https://web.mit.edu/kerberos/krb5-1.12/doc/basic/rcache_def.html); Biscuit
  [notes](https://petermalmgren.com/biscuitsec-0/) + Macaroons [Fly.io](https://fly.io/blog/macaroons-escalated-quickly/);
  bearer-token replay best practice [Duende](https://duendesoftware.com/learn/best-practices-managing-token-expiration-refresh-revocation-in-web-apis).
