# PROPOSAL — rotation authority parity (`identity-rotation-handoff`)

**Status:** DRAFT (2026-08-13) · **security-relevant** · extension-scoped, no core change.
**Target:** `specs/extensions/EXTENSION-IDENTITY.md` — §3.6 (`identity_topology_for`,
`identity_verify_cert`), §4.1 (kind table), §4.3 (`identity-rotation-handoff`). Target version
v3.11.
**Provenance:** established by trace against the live spec during the rotation / key-lifecycle
audit — `docs/research/explorations/EXPLORATION-ROTATION-AUTHORITY-AND-KEY-LIFECYCLE.md` §3. Not
derived from a cohort finding or an external report.
**Pins:** `entity-system-architecture` `bf88be0` · `entity-core-protocol` `3042bd8`.
**Scope:** identity-layer signature topology only. No new entity type, no new `properties` key, no
wire change, **no core protocol change** (§7).

---

## 1. Problem — issuance authority is dropped at rotation time

`identity_topology_for` (§3.6) dispatches signature requirements on
`(properties.kind, properties.function, is_quorum(attesting))`. Every `identity-cert` arm requires
the **issuing** authority. The handoff arm does not:

```
("identity-rotation-handoff", _, _):
  ; Dual-sig from old + new key
  target = lookup_target_cert(att, ctx)
  return {mode: "dual", signers: [target.attested, att.attested]}
```

It is a **wildcard on function**, and it overrides all four issuance arms above it:

| Cert / event | Authority to **issue** | Authority to **rotate** |
|---|---|---|
| `identity-cert` controller, top-level | **K-of-N from the quorum** | dual-sig (old + new) |
| `identity-cert` controller, sub | issuing controller | dual-sig (old + new) |
| `identity-cert` agent | controller (3-key) / identifier (4-key) | dual-sig (old + new) |
| `identity-cert` identifier | issuing controller | dual-sig (old + new) |
| `identity-rotation-recovery` | — | K-of-N from the quorum |
| `identity-retirement` | — | K-of-N from the quorum |
| `revocation` of a cert | — | quorum at the chain root (`identity_is_authorized_revoker`) |

**The authority required to grant a function is not required to move it.** Recovery, retirement
and revocation are all quorum-gated; only handoff is not. In the severe case, K-of-N from the
quorum grants the controller function and a dual signature relocates it.

### 1.1 The chain walk does not close this

`identity_verify_cert` step 5 walks `attesting` back to the quorum and re-verifies each link, so
it is fair to ask whether that catches the case. It does not:

1. An attacker holds stolen key `K_a`, which holds
   `cert(attesting=K_c, attested=K_a, function=agent)`.
2. The attacker mints `handoff(attesting=K_a, attested=K_thief, target_cert=cert)`, signing with
   `K_a` and `K_thief`. Step 4's dual-sig requirement is satisfied.
3. Step 5 walks from `handoff.attesting = K_a` and reaches `cert(K_c → K_a)` — genuinely valid,
   signed by the real controller, live, chain intact to the quorum.
4. Every link validates. **`identity_verify_cert` returns OK.**

The walk establishes **that the outgoing key legitimately held the function**. It never
establishes **that anyone authorized moving it**. The identical trace runs with `K_a` a controller
key, where the bypassed authority is the quorum.

Per §4.3's function-inheritance rule *"No fresh `identity-cert` issuance is required after a
handoff for the new key to be chain-walkable"* — so the attacker's key becomes a first-class
holder of the function immediately, with no further event.

### 1.2 The current design is deliberate — and that is the actual defect

This is not an oversight arm nobody thought about. §13.3 states the intent explicitly:

> *"The quorum doesn't sign this — the old handle's signature alone authorizes (proves the user
> holds `K_old`; new key's signature proves possession). Compromise-recovery is the K-of-N path;
> routine rotation is dual-sig."*

And §4.3 / §14.2's summary table partition the two kinds the same way: handoff is *"graceful cert
rotation (old key still available)"*, recovery is *"compromise-recovery cert rotation (old
unavailable)"*.

**The partition is by operator intent, and intent is not verifiable.** "The old key is still
available" is a fact about the world, not a property a verifier can check — and "available to
whom" is exactly what is in question under compromise. The two events are structurally identical
on the wire, so **the attacker selects which path to use, and will always select the cheap one.**
Recovery's K-of-N is not a gate on the attacker; it is a gate on the legitimate user, who is the
only party that would ever voluntarily use it.

So the proposal is not "you forgot a check." It is: **a security partition that depends on
operator honesty is not a security partition.** Rule 1 replaces it with one a verifier can
actually evaluate — the issuing authority's signature — and Rule 2 restores the cheap path for the
honest case in a form the attacker cannot forge.

This is the "cross-peer seam equivalence-collapse" shape from `AGENTS.md`: a distinction that
holds at issuance collapses at a later event. Every conformance vector asserting handoff behavior
asserts the *current* topology, so the suite certifies the hole.

## 2. Proposal

**The principle: rotation MUST NOT require less authority than issuance.**

### Rule 1 — authority parity (normative)

A handoff MUST satisfy the **issuance topology of the cert it rotates**, plus a signature from the
incoming key.

### Rule 2 — the pre-committed exemption (normative, conditional on pre-rotation)

A handoff MAY substitute a **redeemed pre-rotation commitment** for the issuer's signature. Where
the outgoing key published a commitment to the incoming key's digest before this event, the
dual-sig form (outgoing + incoming) remains sufficient.

**Rule 2 is not decoration — it is what keeps Rule 1 adoptable.** A top-level controller cert is
issued K-of-N, so parity alone collapses its handoff into `identity-rotation-recovery` and the
graceful roll disappears for the one key that most needs it. The commitment restores self-service
roll safely: an attacker holding the outgoing key cannot produce a key matching the pre-committed
digest.

| Target cert | Closed by | Why the other does not suffice |
|---|---|---|
| agent · sub-controller · identifier | **Rule 1** — the issuer co-signs | the issuer is a single reachable key; a commitment is unnecessary ceremony |
| controller (top-level) | **Rule 2** — pre-rotation | the issuer is the quorum; parity alone means a K-of-N ceremony per routine roll |

## 2a. Is this the right security level? — the practicality audit

The objection to write against is *"stricter rotation rules create operational stress."* Audited
against the spec's own frequency model, **that objection holds for exactly one configuration**,
and it is the opt-in one.

### 2a.1 Who is already in the loop

Rule 1 asks the **issuer** to sign. The relevant question is therefore not "is this stricter" but
**"is the issuer already present at rotation time?"** Per configuration:

| Rotation | How often (spec's own words) | Issuer of target | Rule 1 marginal cost |
|---|---|---|---|
| agent / device roll | device lifecycle — §13.4's stolen laptop is the documented case | controller (3-key) / identifier (4-key) — **one key** | **zero.** Issuing that agent cert already required exactly this signature (§3.6, single-sig from `attesting`). If you cannot reach the issuer to roll the key, you could not have reached it to enroll the device. |
| sub-controller roll | occasional | issuing controller — **one key** | **zero**, same argument |
| identifier roll (4-key) | *"Rotates rarely (only on compromise of the identifier itself)"* (§7.2) | controller — **one key** | **zero.** The controller signs the identifier cert that establishes the binding anyway. |
| controller roll (3-key — the controller **is** the handle) | privacy hygiene; occasional | quorum, K-of-N | **already paid.** §13.3 step 1: three-key routine handle rotation is *"a new controller key `K_ctrl_v2` **paired with quorum re-certification**."* The quorum is already convened; Rule 1 adds one signature to a ceremony that is happening. |
| controller roll (4-key) | ***"The controller rotates frequently (hygiene; per-incident; policy-driven)"*** (§7.2) | quorum, K-of-N | **REAL — the only stress case.** |

**Three-key is the RECOMMENDED configuration (§1.1 progression 3). Rule 1 is operationally free
across all of it.**

### 2a.2 The one stress case, and why it is not an argument against Rule 1

Four-key advanced exists for *"hygiene-rotation independence"* — its headline property is that the
controller rotates frequently without contacts re-validating. Under Rule 1 each of those rotations
needs K-of-N.

But **that cheapness is built on the defect itself.** Verified by trace: a handoff's `attesting` is
the outgoing controller key, so `identity_is_quorum_link` is false; the walk goes
handoff → `cert(quorum → K_ctrl)` → terminates, and the K-of-N on the *original* cert is what
satisfies step 5. **The handoff itself never carries a quorum signature**, and §4.3's function
inheritance means no fresh cert is minted afterwards. Frequent controller rotation is cheap today
precisely *because* it bypasses the quorum.

So the honest statement is not "Rule 1 adds friction to four-key" but **"four-key's frequent-rotation
story is built on the hole Rule 1 closes."** For that configuration Rule 2 is therefore not
optional — it is what preserves the configuration's reason for existing. Four-key is explicitly
opt-in (§7.2: *"Most users don't need this"*), so this is a bounded, named consequence rather than
a broad regression.

### 2a.3 What the strictness actually buys — stated without overclaiming

Today's exposure is **not** permanent takeover. `identity_verify_cert` step 3 runs the
authority-revocation check on every chain link, and `identity_is_authorized_revoker` lets the
quorum at the chain root revoke the original cert — which breaks the attacker's handoff chain.
§13.4 documents exactly that response for a stolen agent peer.

So the accurate framing:

| | Today | Under Rule 1 |
|---|---|---|
| Stolen key can relocate its own function | **yes**, silently, immediately chain-walkable | **no** |
| Remedy | detect → convene the quorum → revoke | not required |
| **Depends on detection** | **yes** | **no** |

That last row is the argument. Detection is the largest unclosed gap in this whole thread
(`PROPOSAL-IDENTITY-PRE-ROTATION.md` §5.2, and KERI concedes the same). **A fix that does not
depend on detection is worth substantially more than one that does** — and Rule 1 is preventive
where revocation is reactive.

### 2a.4 Why not pre-rotation alone

Pre-rotation protects only keys that **pre-committed**, requires every key-installing event to
carry a commitment, and brings the ratchet-vs-recovery tension (that proposal's decisions 2 and
13). Rule 1 protects **every** cert immediately, including keys that never committed to anything,
and introduces no new state.

Rule 1 is therefore the floor and Rule 2 the escape hatch — not two competing fixes. Rule 2 also
covers a case Rule 1 cannot: **scheduled or automated key rotation while the issuer is offline.**
A key that planned ahead can roll itself; a key that did not must involve its issuer. That is the
right default in both directions.

### 2a.5 Verdict

Right level. Rule 1 is free in the recommended configuration, costs nothing anywhere the issuer is
a single key, is already paid in three-key controller rotation, and its one real cost falls on an
opt-in configuration whose cheapness was never sound. It removes a dependency on detection, which
is the weakest link we have.

## 3. Normative changes

### 3.1 §3.6 `identity_topology_for` — replace the handoff arm

```
("identity-rotation-handoff", _, _):
  ; Authority parity (PR-AP-1): a handoff carries the ISSUANCE topology of
  ; the cert it rotates, plus the incoming key. The outgoing key's signature
  ; is retained — its absence is what distinguishes recovery from handoff.
  target = lookup_target_cert(att, ctx)
  if target is null:
    return error("target_cert_not_found")

  ; Rule 2 — pre-committed exemption. Only available where the commitment
  ; construction has landed; see §4.3a.
  if identity_redeems_commitment(att, target, ctx):
    return {mode: "dual", signers: [target.attested, att.attested]}

  ; Rule 1 — parity with issuance, plus the incoming key.
  return {mode: "issuance-parity",
          target: target,
          also: [target.attested, att.attested]}
```

### 3.2 §3.6 `identity_verify_cert` step 4 — add the dispatch arm

```
"issuance-parity":
  ; Satisfy the target's own issuance topology against THIS attestation's hash,
  ; then require the additional signers.
  inner = identity_topology_for(topology.target, ctx)
  match inner.mode:
    "k-of-n":
      if not QUORUM.verify_k_of_n_signatures(
          att.content_hash, inner.signers, inner.threshold, ctx):
        return error("k_of_n_failed")
    "single":
      if not ATTESTATION.verify_specific_signer(att, inner.expected_signer, ctx):
        return error("missing_issuer_sig")
  for signer in topology.also:
    if not ATTESTATION.verify_specific_signer(att, signer, ctx):
      return error(f"missing_sig: {signer}")
```

### 3.3 §4.1 kind table — replace the handoff row

| `properties.kind` | Category | Sig topology | Storage tier | Spec section |
|---|---|---|---|---|
| `"identity-rotation-handoff"` | Cert lifecycle | **issuance topology of `target_cert` + old + new** (or dual-sig old + new where a pre-rotation commitment is redeemed) | same audience tier as target | §4.3 |

### 3.4 §4.3 — replace the signing line

Replace *"**Signed dual-sig**: both `attesting` (old) and `attested` (new) keypairs sign"* with:

> **Signed (normative, PR-AP-1).** A handoff MUST satisfy the signature topology that
> `properties.target_cert` required at issuance, **and** carry signatures from both `attesting`
> (outgoing key) and `attested` (incoming key). Where the outgoing key published a pre-rotation
> commitment that `attested` redeems (§4.3a), the issuance topology is **not** additionally
> required and the outgoing + incoming signatures suffice.
>
> Rationale: rotation must not require less authority than issuance. Without this, a compromised
> key relocates its own function — including a top-level controller function whose issuance
> required K-of-N from the quorum.

## 4. What this does not solve

- **Detection.** Nothing here shortens the window before a compromise is noticed. Unchanged and
  unclosed, identically to `PROPOSAL-IDENTITY-PRE-ROTATION.md` §5.2.
- **A compromised issuer.** If the controller key is stolen, Rule 1 does not protect the agent
  certs it issued — the attacker signs as the issuer. That case escalates to the quorum, which is
  what `identity-rotation-recovery` and `identity_is_authorized_revoker` already require.
- **Availability.** Rolling an agent key now requires the controller to be reachable. This is the
  same trade ATProto makes by keeping `rotationKeys` apart from the PDS's signing key, and it is a
  real operational cost, stated rather than discovered.
- **App-defined functions.** Their issuance topology defaults to single-sig from `attesting`
  (§3.6, pinned in v3.3), so parity inherits that default. Apps with custom topology already wrap
  `identity_verify_cert`; they inherit the wrapper here too.

## 5. Interaction with `PROPOSAL-IDENTITY-PRE-ROTATION.md`

**Not superseded, and the relationship is now sharper than "two security fixes."**

That proposal frames pre-rotation as the answer to *"a stolen key can sign a handoff to the
thief."* Rule 1 answers that directly for every cert whose issuer is a single key — without a
commitment, without a new `properties` field, without a construction ruling. What pre-rotation
uniquely buys is **Rule 2**: preserving self-service roll for the top-level controller, whose
issuer is a quorum.

So the two are one design, and this proposal supplies the reason pre-rotation is *needed* rather
than merely *useful*. Consequences for its open decisions:

- **Decision 12 (which commitment is in force)** becomes a precondition of Rule 2 rather than an
  internal detail — the exemption is only as sound as the resolution rule.
- **Decision 2 (§4.4 sticky commitment as MUST)** is strengthened: under Rule 2 the commitment is
  an *authority substitute*, so a silently-disarmed ratchet is an authority downgrade.
- **Decision 9 (the §3.7 option-B fallback — quorum co-signs every handoff)** is largely subsumed:
  Rule 1 **is** that fallback, scoped correctly to each cert's actual issuer rather than
  uniformly to the quorum.

**Recommended sequencing:** Rule 1 is ratifiable alone and closes the wider hole. Rule 2 lands
with the pre-rotation construction. If Rule 1 lands alone, top-level controller handoff collapses
into recovery until Rule 2 arrives — an accepted, temporary ceremony cost that should be ruled on
knowingly (decision 3 below).

## 6. Interaction with the conformance surface

The ruled scope of `PROPOSAL-IDENTITY-PRE-ROTATION.md` decision #4 (RULED 2026-08-11) already
requires rotation-predicate vectors with `GUIDE-CONFORMANCE` §2.4a's **state-probe** rule on the
negative half. This proposal lands inside that scope and sharpens the negative case:

> a handoff lacking the issuer's signature (and redeeming no commitment) MUST be refused **and
> MUST produce no state change** — the incoming key MUST NOT become chain-walkable for the
> target's function.

Per §2.4a a positive-only matrix would certify exactly the defect this proposal closes.

## 7. Impact

- **Core protocol: none.** The change is confined to identity's own validators. Confirmed three
  ways: `EXTENSION-ATTESTATION` §3.2 (`properties` is consumer-extensible, the primitive does not
  interpret keys); `EXTENSION-IDENTITY` §3.6 (the reject-on-unknown gate ranges over
  `valid_functions()` and `identity_lifecycle_kinds()`, not signature topology); and
  `EXTENSION-IDENTITY` §1, *"chains stay pure V7."* It also passes `ENTITY-CORE-PROTOCOL` §5.10 —
  the predicate runs in `identity_verify_cert` over an attestation, never in `verify_request` over
  a capability chain, so it introduces **no Layer-1 predicate**.
- **Wire: none.** No new entity type, no new field, no encoding change.
- **Behavior: yes, and deliberately.** A handoff valid today becomes invalid. Per `AGENTS.md`
  ("no backward compatibility, no installed base") this lands as the only design, with no
  migration window.
- **Cohort:** three reference impls plus keystone regeneration; new vectors in the IDENTITY
  behavioral category that decision #4 already scopes.
- **`Depends:`** unaffected by this proposal. (The stack-wide stale-floor finding —
  IDENTITY/ATTESTATION/QUORUM/REGISTRY all declaring `v7.40+` against a v7.77 corpus — is recorded
  in the exploration §6.1 and is separate work.)

## 7a. Change scope — the full sweep, not one dispatch arm

The validator edit is the small half. `dual-sig` is **pinned in prose across eleven sites in three
documents**, and a swept-incompletely change leaves the spec self-contradicting. (This is core's
`213a2ac` lesson applied here: *when a pinned primitive moves, sweep the prose that pins it, not
only the artifacts that failed.*)

**`specs/extensions/EXTENSION-IDENTITY.md` — 9 sites**

| Site | What it says now | Change class |
|---|---|---|
| §3.6 `identity_topology_for`, handoff arm | returns `{mode: "dual", …}` | **normative — the mechanism** |
| §3.6 `identity_verify_cert` step 4 | `"dual"` arm; `missing_dual_sig` error | **normative — new arm + error surface** |
| §4.1 kind table row | "dual-sig (old + new)" | normative summary |
| §4.3 signing line | "**Signed dual-sig**: both … sign" | **normative — the rule** |
| §3.3 kind list | "graceful key roll (dual-sig)" | descriptive |
| §6.3 dispatch table | `(identity-rotation-handoff, *) → handle_dual_sig_handoff` | **normative — routine name + routing** |
| §10.1 signing-pattern enforcement bullet | "`identity-rotation-handoff` is dual-signed (old + new)" | normative impl requirement |
| §10.1 topology-first dispatch MUST | enumerates "dual-sig via `verify_specific_signer` per signer" | normative impl requirement |
| §13.3 worked example + rationale | *"The quorum doesn't sign this…"* | **rationale — must be rewritten, not deleted** (§1.2) |
| §14.2 summary table row | "dual-sig (old + new)" | normative summary |

**`guides/GUIDE-IDENTITY.md` — 1 site:** the `identity_verify_cert` walkthrough enumerating
"K-of-N (top-level) / single-sig / dual-sig (rotation-handoff)".

**`specs/sdk/SDK-IDENTITY-INFRASTRUCTURE.md` — 1 site:** `RotateController` "finalizes via
dual-sig handoff" — and this one is **not** a wording change (below).

### 7a.1 The caller-side change is the part that is easy to under-price

Per §6.3, *"the local op's caller is responsible for providing the necessary signatures via
`envelope.included`"* (core §6.5 / SPEC-25). So the signature set is assembled **before**
submission, and Rule 1 changes who must participate in that assembly:

| Target cert | Signatures to assemble today | Under Rule 1 (no commitment) | Under Rule 2 |
|---|---|---|---|
| agent · sub-controller · identifier | old + new (2 parties) | old + new + **issuing controller** (3 parties) | old + new |
| controller, top-level | old + new (2 parties) | old + new + **K-of-N of the quorum** | old + new |

**This is a coordination change, not just a validation change.** `RotateController` and the
equivalent agent-roll flow must gather an additional signer — for the top-level controller under
Rule 1 alone, that means a quorum signing session, which is the interim cost decision 3 asks about.
The SDK surface (`SDK-IDENTITY-INFRASTRUCTURE` §11) needs the extra signer threaded through, and
the error path (`missing_issuer_sig`) must be retryable per §6.3's "re-submit with corrected
signatures" contract rather than rolling back.

## 7b. Review and validation scope

**What prose review can settle** — the topology rules, the §13.3 rationale rewrite, the sweep
completeness. This is a single-reviewer pass over the eleven sites above.

**What prose review cannot settle, and the meta-rule says so** (`AGENTS.md`: *a normative claim
about what a verifier accepts is not validated until a cross-impl conformance test exercises it*):

1. **Positive** — a handoff carrying the target's issuance topology plus both keys is accepted,
   for each of the four target functions. Four vectors; the `controller`-top-level one needs a
   K-of-N fixture.
2. **Negative, with the §2.4a state probe (MUST)** — a handoff lacking the issuer's signature and
   redeeming no commitment is **refused AND produces no state change**: specifically the incoming
   key MUST NOT become chain-walkable for the target's function. Asserting the `401`/error alone
   would certify the defect, since today's failure mode is precisely that the entity binds and the
   function transfers.
3. **The exact §1.1 trace as a named vector** — a valid outgoing-key-held cert, a self-authored
   handoff to an attacker key, chain intact to the quorum. This is the case that passes today; it
   is the regression anchor.
4. **Rule 2 exemption** — accepted with a redeemed commitment, refused with a mismatched one.
   Blocked on the pre-rotation construction; do not write these until decision 12 is ruled.

**Cohort cost.** Three reference impls (Go/Rust/Python) plus keystone regeneration (15 generated
peers). `EXTENSION-IDENTITY` is M5 🟢 in the v1 locked set, so this de-matures it until the vectors
are green — that is the real schedule item, not the code.

**Sequencing note.** Items 1–3 are independent of pre-rotation and can be written as soon as
Rule 1 is ruled. Item 4 waits. This is why decision 3 (does Rule 1 land alone) is the load-bearing
one: it determines whether the conformance work can start now or blocks on a second proposal.

**Rough size.** Per impl: ~25 lines of validator change, ~1 new error code, the caller-side signer
threading (larger and impl-specific), and the vector fixtures. The validator is a day; the
coordination change and the cross-impl vector convergence are the schedule.

## 8. Decisions requested

1. **Adopt Rule 1** — a handoff carries the issuance topology of its target. *(Recommended: yes.
   It is the defect's direct closure and requires no new mechanism.)*
2. **Retain the outgoing key's signature** under Rule 1? *(Recommended: yes — its absence is what
   distinguishes handoff from `identity-rotation-recovery`, and dropping it would merge the two
   kinds.)*
3. **If Rule 2 is not yet available, does Rule 1 land alone?** *(Recommended: **yes**, and §2a
   settles it rather than leaving it to judgment.* Rule 1 is operationally **free** in the
   RECOMMENDED three-key configuration and everywhere the issuer is a single key — the same
   signature issuance already required. The only new cost is **four-key advanced controller
   rotation**, which is opt-in (§7.2 *"Most users don't need this"*) and whose cheapness today
   comes from bypassing the quorum — i.e. from the defect. Interim consequence, named: four-key
   controller rotation becomes as expensive as three-key until Rule 2 lands.)*
3a. **Do we say so in §7.2?** A one-line note that four-key's frequent-controller-rotation
   property is restored by the pre-rotation commitment, so operators do not read the interim cost
   as permanent and quietly stop rotating — *a rotation people avoid is worse than an expensive
   one.* *(Recommended: yes.)*
4. **Ratify Rule 2's shape** — a redeemed commitment substitutes for the issuer's signature, and
   nothing else about the handoff changes. *(Recommended: yes, conditional on the pre-rotation
   construction landing; the seam is `identity_redeems_commitment` in §3.1.)*
5. **Confirm the negative-vector requirement of §6** as part of decision #4's already-ruled scope.

## 9. Sources

**Internal (read live, 2026-08-13)**
- `EXTENSION-IDENTITY.md` v3.10 — §1 (what the extension does not define), §3.6
  (`valid_functions`, `identity_lifecycle_kinds`, `identity_topology_for`,
  `identity_is_authorized_revoker`, `identity_verify_cert`), §4.1 (kind table), §4.2, §4.3
  (function inheritance), §4.4, §4.5 — `entity-system-architecture` `bf88be0`
- `EXTENSION-ATTESTATION.md` v1.3 — §3.1 (`system/attestation` fields), §3.2 (`properties` /
  `kind` convention)
- `ENTITY-CORE-PROTOCOL.md` 0.8.0 — §1.5 (`peer_id = f(public_key)`), §5.10 (Layer 1 / Layer 2,
  the extension contract) — `entity-core-protocol` `3042bd8`
- `GUIDE-CONFORMANCE.md` §2.4a (state-probe rule on the negative half)
- `docs/research/explorations/EXPLORATION-ROTATION-AUTHORITY-AND-KEY-LIFECYCLE.md` §3 (the trace)
- `PROPOSAL-IDENTITY-PRE-ROTATION.md` (staged internally) — decisions 2, 9, 12

**External (design precedent, re-audited 2026-08-12)**
- AT Protocol DID spec — https://atproto.com/specs/did (`#atproto` signing key)
- did:plc specification v0.1 — https://web.plc.directory/spec/v0.1/did-plc
  (`rotationKeys` vs `verificationMethods`; *"Cannot control the DID"*)
- KERI pre-rotation (KID0005) — https://identity.foundation/keri/kids/kid0005Comment.html
