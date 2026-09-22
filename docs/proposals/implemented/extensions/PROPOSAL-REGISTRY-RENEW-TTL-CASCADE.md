# PROPOSAL — REGISTRY `renew-request` TTL: the second mint site, and the cascade that closes it

**Status:** **DRAFT — folded at authoring.** The §7 delta is in
`specs/extensions/EXTENSION-REGISTRY.md` (**v1.8 → v1.9**), and **§9 corrects it to v1.10 the same
day** — §3's totality claim was wrong and produced a null mint in the first implementation of it.
Every row was verified against the tree before each version moved (L3). §5 is filed and **not** folded.
**Read §9 before implementing §3.**
**Target:** `EXTENSION-REGISTRY.md` §6a.9 — the null-`ttl` rule set beside D11 / D12.
**Tier:** `extensions/` — implemented by go · rust · py.
**Scope:** how `renew-request` resolves a `ttl` it was not given. **Not** the register-time rules
(D11 / D12), which stand unchanged, and **not** a TTL ceiling, which §5 files as a separate question.
**Source:** `entity-core-py` found it (SA-PY-13) and **declined to fix it unilaterally**, to avoid
manufacturing a cohort split. `entity-core-go` confirmed the identical hole in their own tree, closed
the MUST violation the safe way, and routed the disposition rather than picking it
(`entity-core-go` `docs/validation/spec-issues/2026-08-18-c-*`, at `6ed6f95`).
**Cohort review:** the *defect* is three-way confirmed. The *disposition* below is arch's and has not
been reviewed by a seat — §8 is the ask.

---

## 1. The defect, and why it survived a fold aimed straight at it

A `kind: "peer-issued"` binding **MUST** carry a non-null `ttl` (§6a.3). The reason is narrow and
load-bearing: a hostile byte-server cannot forge a signature or move the consumer's clock, but it
**can withhold a revocation indefinitely**, and `ttl` is the *only* bound on that withholding. A
null-`ttl` peer-issued binding is therefore not merely irregular — it is **permanently unrevokable**,
and §6a.4 requires a conformant resolver to refuse it outright.

Three rules were built to keep that from happening, and all three point at `register-request`:

| Rule | Site | What it catches |
|---|---|---|
| D3 | §6a.3 / §6a.4 | the invalid binding, at *resolve* time |
| D11 | `set-issuer-policy` | a policy whose `default_ttl` is null, at *policy-write* time |
| D12 | `register-request` | a null resolution from a policy that reached the tree store-first |

**`renew-request` is a second producer of peer-issued bindings and appears in none of them.** On a
curated registry whose stored policy carries no `default_ttl` — reachable store-first, exactly the
case D12 exists for — a renew that omits `ttl` resolved to null and **minted the successor**. All
three implementations had it.

**Why the fold missed it is the part worth keeping.** D11/D12 were written as a *policy* fix — the
insight was that `default_ttl` is the operator's field, so the gate belongs at the operator's
operation. That framing is correct and it is also what hid the second site: a sweep organised around
*where the bad input is authored* does not enumerate *where bindings are minted*. This repo's own
ratchet states the general form — **a refuse-this-shape rule's blast radius is every producer of the
shape** — and this is that rule's own blind spot, one op over.

## 2. The unambiguous half — already closed

Minting a null-`ttl` binding violates a landed MUST. No ruling was needed for that half and go closed
it before routing: refuse `403 policy_rejected`, publish nothing. Mutation-tested — deleting the guard
makes the no-`ttl` renew mint a null binding and return `200`, and the test fails.

**That half is not in question and is not what this proposal changes.** What it changes is what
happens *instead* of the refusal.

## 3. The ruling — a three-step cascade, total by construction

Renew differs from register in exactly one respect, and it is decisive: **a renew always has a valid
`ttl` available that a register does not** — the superseded binding's own, non-null by §6a.3.

| Step | Source | Why here |
|---|---|---|
| 1 | the request's `ttl` | as for `register-request` |
| 2 | the issuer policy's `default_ttl` | **current operator intent outranks history.** An operator who lowers `default_ttl` must see renewals pick it up; if the predecessor's value ranked higher, every existing name would renew forever at the old duration and the knob would be inert on precisely the population it matters for |
| 3 | the superseded binding's `ttl` | non-null by §6a.3 — **step 3 cannot resolve null**, so the cascade never falls through |

Because step 3 cannot be null, **there is no fourth step and no refusal path.** The op stops being a
producer of invalid bindings without becoming a producer of denied renewals.

## 4. Why inherit and not refuse

Both dispositions are conformant to D3 today. The argument is not that refuse is unsafe — it is
strictly safer — but that it is the wrong side of a distinction this subsection has already drawn.

**(a) The spec's own placement rationale points here.** §6a.9.2 puts the register-time gate at
`set-issuer-policy` *"because this is where the missing input lives,"* and refuses to bill the
requester for the registry's misconfiguration — *"it would teach requesters to send `requested_ttl`
defensively."* At renew **the input is not missing.** The argument that forces a refusal at register
does not reach the op where the registry already holds a valid answer.

**(b) It is a recovery, not an invention — and that is what §6a.9.2 actually forbids.** The
neighbouring paragraph bans substituting *"an implementation-chosen default,"* on the ground that two
registries would then answer identically-stored policies with different lifetimes (a §5.10
determinism split). The superseded binding's `ttl` is not implementation-chosen: it is **one value,
already signed and published by this registry for this exact name**, byte-identical at every
conformant peer. Inherit is the only fallback available that has this property, which is why the
general ban does not extend to it.

**(c) Refusing revokes a name by inaction, for a defect its holder cannot see.** The registrant sent a
well-formed, layer-1-signed renew. The policy defect is the registry's. Refusal means the binding
lapses — on the one operation whose entire purpose is to prevent that — and the registrant has no
diagnostic path to a policy they cannot read.

**(d) It grants nothing new.** Step 3 re-grants the duration the registry itself chose, to the same
`target_peer_id` that proved layer-1 control, on a binding the registry can still revoke. Renew is
defined as *"extends a binding's lifetime"*; inheriting the prior window is that definition's minimum,
not an expansion of it.

**What decides it must be pinned either way.** The successor binding carries `ttl` in its content, so
refuse-vs-inherit is **binding-hash-determining** and directly cross-peer observable — one peer holds
a live name, another does not. go was right not to pick it unilaterally.

## 5. RULED `[2026-08-18, operator directive to decide it — folded v1.11]` — bounded on both sides, and the resolver's side is the one that matters

Verifying the cascade surfaced a second defect underneath it, and it is larger than the one that was
routed.

**Step 1 accepts the requester's own number, and `system/registry/issuer-policy` carries `default_ttl`
with no `max_ttl`.** So a layer-1-valid requester may register or renew with a `ttl` of any magnitude,
and a conformant registry has nothing to clamp it with. §6a.9.2 names the hazard in as many words —
*"handing TTL selection to the party §6a.1a treats as untrusted"* — and uses it as an argument for
where to put a gate, while the schema hands that selection over unbounded through the front door.

This matters more than an ordinary missing knob because of what `ttl` **is** on this surface: per
§6a.3 it is the *only* bound on a withheld revocation. An unbounded requester-chosen `ttl` reproduces
the exact condition §6a.3 was written to prevent — a binding that is, in practice, unrevokable —
without ever setting the field to null.

### The ruling, and it is not an invention — the corpus had already settled the pattern

**`docs/research/explorations/EXPLORATION-NON-INTERACTIVE-FRESHNESS-AND-ANTI-REPLAY.md` is the design
record for this exact family, and it was not read before §5 was first filed as "undecided."** Its Knob 2
states the posture in as many words: *"This is the §4.10 move: mandate the bound **exists and is
declared/enforced**, leave the **value** to the deployment."* It also names **REGISTRY specifically** as
the extension that leaves the propagation question open. The question was not open for lack of an
answer; it was open because the study that answers it went unopened (L11).

**Prior art, since it decides the shape rather than merely supporting it.** DNS splits this in two: the
authority sets a record's TTL, and the **resolver** caps what it will honor (`max-cache-ttl`), because
the resolver is the party holding stale data. TUF sets expiry per role. X.509/ACME put the ceiling in
policy, not in the protocol. **Not one of them lets the requesting party choose an unbounded lifetime,
and not one writes the maximum into the wire format.**

**Applied here, and the asymmetry is the finding:**

| Side | Rule | What it actually buys |
|---|---|---|
| **Issuer** — `max_ttl` REQUIRED on any policy reaching *approve*; `default_ttl <= max_ttl`; requests above it **clamped** | operator hygiene | stops a careless registrant asking for a decade |
| **Resolver** — MAY declare a local maximum; when it does, effective lifetime is `min(binding.ttl, local_max)`, computed at resolution, never written back | **the security property** | the only bound a consumer controls |

**Why the registry-side ceiling is the weaker half, which is the part that was not obvious.** §6a.3's
entire argument is about the **consumer**: a hostile byte-server withholds a revocation and `ttl` bounds
the exposure. **A ceiling the registry enforces cannot protect a consumer from that registry** — a
hostile or compromised issuer just sets `max_ttl` high. Only the party bearing the risk can bound it.
The first framing of this question treated it as a registry-configuration gap; it is primarily a
resolver obligation, and a proposal that had only added `max_ttl` would have shipped the half that does
not defend anyone.

**Clamp, not refuse**, for §6a.9.2's own reason: refusing bills a well-formed request for a policy the
requester cannot read, and teaches requesters to probe for the ceiling. DNS does not `NXDOMAIN` a record
with a long TTL; it caps it.

**No number is written.** Same reason §4.10 writes none and the same reason the protocol-wide TTL floor
was rejected for register: there is no defensible constant, and choosing one makes every unconfigured
deployment look configured.

### The three options as originally filed, kept for the record



| Option | Cost |
|---|---|
| (a) `max_ttl` on `issuer-policy`; requests above it are clamped | a new field; clamping silently alters a signed request's intent |
| (b) `max_ttl` on `issuer-policy`; requests above it are refused `400` | explicit, but a policy change starts refusing renewals that used to work |
| (c) Decide layer-1 proof is sufficient and say so | honest and free, but leaves §6a.9.2's stated concern unaddressed in the schema |

**Not ruled here.** It is separable from the cascade, it changes the policy entity's shape, and this
session found it rather than being routed it — the register-time half is affected identically, so it
is not a renew question at all. **Operator decision requested.**

## 6. Rejected alternative

**A protocol-wide TTL floor or default.** Already rejected in this subsection for register, and the
reasoning transfers unchanged: *"there is no defensible number, and choosing one makes every
unconfigured registry look configured."*

## 7. Spec delta — every row verified against the tree before the version moved

| # | File | § | Change | Verified |
|---|---|---|---|---|
| D1 | `EXTENSION-REGISTRY.md` | §6a.9 | New `[MUST, v1.9]` — the three-step renew cascade table | ✅ present |
| D2 | `EXTENSION-REGISTRY.md` | §6a.9 | The recovered-vs-chosen distinction against the neighbouring "implementation-chosen default" ban | ✅ present |
| D3 | `EXTENSION-REGISTRY.md` | §6a.9 | The refuse-side rebuttal from §6a.9.2's own placement rationale | ✅ present |
| D4 | `EXTENSION-REGISTRY.md` | §6a.9 | Scope limits — no lifetime extension, D3 preserved, revocation untouched | ✅ present |
| D5 | `EXTENSION-REGISTRY.md` | §6a.9 | `REG-RENEW-TTL-CASCADE-1`, three rows (a)/(b)/(c) | ✅ present |
| D6 | `EXTENSION-REGISTRY.md` | §6a.9 | §5's ceiling gap recorded as open, pointing here | ✅ present |
| D7 | `EXTENSION-REGISTRY.md` | header | **1.8 → 1.9** | ✅ present |

**No V7 wire change, no new entity type, no new capability, no new error code** — `403
policy_rejected` is no longer reachable on this path, which removes a code use rather than adding one.

## 8. Cohort impact — **routed, not only tabled**

Per L13, a row here is a record; the packet is the delivery. All three seats are addressed in
`docs/status/ROUTING-2026-08-18-m-*`.

| Seat | State at read | Owed |
|---|---|---|
| `entity-core-go` | `6ed6f95` | **One branch changes.** The refuse guard becomes step 3. Keep `TestRenew_NullResolvedTTL_FailsClosed` as the mutation control, re-pointed: no-`ttl` renew now → `200` with the inherited value, and deleting step 3 must make it mint null |
| `entity-core-rust` | `a701e13` | Currently mints the null successor. Implement all three steps + `REG-RENEW-TTL-CASCADE-1` |
| `entity-core-py` | `9ba439b` | Currently mints the null successor. Implement all three steps + `REG-RENEW-TTL-CASCADE-1`. **You found this** and were right to hold |

**Row (c) is the one that separates a correct implementation from a plausible one** — a seat that
reads "inherit" as the fallback and wires it above the policy default passes (a) and (b) and fails
(c), and the failure is invisible until an operator lowers `default_ttl`.

---

## 9. Correction to §3 — the cascade was not total, and the defect was in this document first

**`[2026-08-18, same day; folded v1.9 → v1.10]`** **Found in `entity-core-go` `5b86b2b` while reviewing
their implementation of §3, and the implementation was faithful.** The defect is upstream of it.

§3 said step 3 *"cannot resolve null"* **because** §6a.3 requires a peer-issued binding to carry a
finite `ttl`. That is an inference from an invariant to a fact about stored bytes, and it does not
hold: a predecessor with `ttl: null` can be **already there** — seeded out-of-band, written directly
to the tree, or predating these rules.

**What it produced.** go's cascade reads:

```go
} else if existing.TTL != nil {
    // (guarding the deref keeps a corrupt out-of-band write from panicking)
    t := *existing.TTL
    ttlPtr = &t
}
successor.TTL = ttlPtr        // nil if all three steps yielded nothing
```

The guard is the correct instinct — do not dereference a possibly-null field — and because this
document told them the branch was unreachable, there was nothing to put in the `else`. **So the
fallthrough mints `ttl: null`: the exact D3 violation the entire arc exists to prevent, restored by
the change that closed it.** go removed the `403` that had been there, on our instruction.

**Why this is not a footnote.** §6a.9.2 already carries the general form one paragraph above the
cascade — *"the write-time refusal cannot be the only thing standing between a bad stored policy and a
bad outcome"* — and D12 exists **solely** to answer the stored-state case for `register`. The cascade
was written without carrying that lesson across to `renew`, in the same subsection, four paragraphs
down. **We reproduced the exact gap we were fixing, one operation over, in the fix.**

**The general shape, which is the part worth keeping: an unreachable branch that is asserted rather
than enforced is not a safe branch.** A conformant implementation must do *something* there, and if
the spec says the case cannot arise, the something is chosen by whoever is avoiding a panic. Stating
the invariant tells an implementer the branch is dead; **only a MUST tells them what dead means.**

**Folded:** the cascade table gains a terminal row — all three steps null ⇒ **`403 policy_rejected`,
publish nothing** — plus the fail-closed clause and **`REG-RENEW-TTL-NULLPRED-1`**, a two-stage vector
(write a null-`ttl` binding directly to the tree, then renew it) with `REG-RENEW-TTL-CASCADE-1` row
(b) as the required `200` control. **REGISTRY 1.9 → 1.10.**

**Cohort impact:** `entity-core-go` `5b86b2b` restores the refusal as the final `else` (they had it at
`93dbab9`; it becomes the terminal arm rather than the whole rule). `entity-core-rust` and
`entity-core-py` implement the cascade with the terminal refusal from the start — **they have not
landed the v1.9 shape yet, so they should build v1.10 and skip the intermediate.**
