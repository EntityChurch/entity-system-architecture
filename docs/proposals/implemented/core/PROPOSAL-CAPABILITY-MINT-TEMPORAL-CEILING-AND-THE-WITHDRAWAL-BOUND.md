# PROPOSAL — "revoked on leave" names a hash the granter never holds, and the TTL that would bound it is unenforced

**Status:** **FOLDED 2026-08-17** — landed as `0.8.1 CAP-4 / CAP-5 / CAP-6` in `entity-core-protocol` `30ca731`
(§6.2 withdrawal-via-revoke · §6.2 step 4 mint ceiling · §5.6 `MIN_DEFINED`). The Q7 wording correction
landed in `PROPOSAL-SHARE-AS-GRANT` §3.1/§3.1a. Validating half — §9 checks **(t)** and **(u)** — unbuilt.
*(Originally: a gap in a landed arch ruling (Q7), plus the core-protocol
defect that makes the honest answer unsound.**
**Target:** `entity-core-protocol` `specs/ENTITY-CORE-PROTOCOL.md` §6.2 (`request` evaluation contract) ·
`docs/proposals/active/applications/PROPOSAL-SHARE-AS-GRANT-AND-THE-AUDIENCE-CARRIER` §3 (Q7 wording).
**Filed by:** `entity-browser-rust`, `ROUTING-2026-08-17-c` §2 (A2). **The finding is theirs and it is
correct.** §3 below is a defect they did not find, met while verifying theirs.
**Read at:** core-protocol `f83c256` (spec **Version: 0.8.0**) · core-go `cc537ea` · core-rust `f23fb3b` ·
core-py `40d1df2` · browser-rust `588eb9c`. All impl reads are source reads taken here.

---

## §0 Summary

**A2 is confirmed against the landed text, and it is a gap in my own ruling rather than an
implementation defect.** `entity-browser-rust` accepted Q7 and built toward it; the two halves of the
mechanism Q7 named do not connect, and no implementation is at fault for that.

| | |
|---|---|
| **The ruling** | Q7: group-as-audience is *"per-member minted tokens, **revoked on leave**"* — `revoke` writes a marker, §5.2 step 4 MUST honor it, *"**with no step added to the check path**"* |
| **The gap** | `revoke` is keyed by **token hash**. In the policy flow the granter authors a *policy*; the **member** calls `request`; the handler mints and returns the token **to the requester**. **Nothing ever hands the granter the hash of what was minted from its own policy** |
| **Why no index rescues it** | §6.2: *"`request` and `delegate` operations return tokens inline — **no tree writes**."* A `request`-minted token is a **wire-only cap by construction**. §5.1's reverse index is `hash → path` for caps that *were bound*; it cannot enumerate caps that were never written, and it is not keyed by grantee in any case |
| **Ruling asked for** | **Model (b)** — policy change closes future mints; live tokens expire. Q7 gets a correcting clause |
| **The defect that makes (b) unsound today (§3)** | (b) says the exposure window is the TTL. **The spec never says whose TTL.** §6.2 step 4 mints with the *requester's* `ttl_ms`, unbounded by the policy entry's `ttl_ms` **or by the caller's own capability expiry**. **go and rust implement exactly that; py alone clamps.** So today the *withdrawn member* chooses how long withdrawal takes to take effect |

**The second finding is the reason this is a core-spec revision and not just a wording fix.** Ruling (b)
without §3 would ratify an unbounded window while describing it as bounded.

---

## §1 A2, verified from the landed text

`entity-browser-rust` reported this from `entity-core-rust`'s source. It reproduces from the spec alone,
which is the stronger form — **no implementation could close it.**

**The mint is wire-only by design.** §6.2, *Self-namespace authorization*:

> `request` and `delegate` operations return tokens inline — **no tree writes.**

**The revoke input is a token hash.** `system/capability/revoke-request` CDDL:

```
system/capability/revoke-request := {
  fields: {
    token:   {type_ref: "system/hash"}     ; Hash of capability token to revoke
```

and the marker lands at `system/capability/revocations/{cap_hash_hex}` — keyed by that same hash.

**The granter is never in possession of it.** §6.2's evaluation contract for `request`: the handler
resolves the caller's identity, matches the policy entry, subset-validates, then *"mints a token with the
request's `grants`… returns it per the result envelope shape."* The result envelope goes to the
**requester**. The policy author wrote a ceiling, not a token; it does not learn what was minted beneath
it, and no operation exists to ask.

**§5.1's index is the wrong index, twice over.** §5.1 describes `capability_path_for` as *"an
**observational reverse index (hash→path, built by recording bindings)**"* — it answers *"where is this
cap stored"* for a hash you already hold. It (a) requires the hash as **input**, which is the thing that
is missing, and (b) records **bindings**, and a `request`-minted token has none. §5.1's own text
anticipates the class: *"For caps with no recorded binding (**wire-only caps from `request`/`delegate`**),
`is_revoked` performs an explicit revocation marker check."* Correct — and it still needs the hash.

**So Q7's three consequences are exactly as browser-rust states them:**

1. *"Revoked on leave"* is **not expressible by a policy-based granter**. It can close future mints; it
   cannot name live tokens.
2. **Withdrawal never revoked anything already minted** — including on the path where the empty-grants
   write succeeds. That write shuts the future-mint door; a token already held stays valid to
   `expires_at`.
3. **"Stop sharing" is honest only up to TTL.**

**They asked for a ruling rather than proposing one.** That was the right call and it is answered below.

---

## §2 Ruling — model (b), and why not (a) or (c)

**RULED: (b). Policy change is the revocation of *future* mints; live tokens expire at their
`expires_at`.** The exposure window for a withdrawal is bounded by the TTL the granter's own policy entry
declares — which §3 makes true.

**Why (b) and not (a) — "the granter records what it minted".** (a) is a real addition and fights the
model in three places at once. It contradicts §6.2's *"no tree writes"* on `request`; it makes the
granter accumulate unbounded per-mint state, which is the accounting shape the methodology's D9 exists to
flag; and it does not survive its own success — the granter would hold hashes for tokens whose holders
have long since let them expire, with no signal to prune. **(a) is available to an implementation as a
local convenience and MUST NOT be assumed by any portable design.** If a deployment wants immediate
per-token revocation it already has the tool: mint *for* the member rather than letting the member
`request`, and keep the hash. That is a different flow, not a fix to this one.

**Why (b) and not (c) — "revocation keyed by grantee".** (c) is the shape that would deliver genuine
immediate revocation, and it is the one to build if (b) proves insufficient. It is rejected **now** for
two reasons. First, it **adds a step to the check path** — a second lookup keyed by the token's grantee,
on every verify — which is the precise property Q7 claimed to preserve, so adopting it silently would
make Q7's own sentence false in the other direction. Second, a grantee-keyed marker is a **new
convergent Layer-1 input**, and §5.10's cross-peer determinism MUST would need re-derivation for it
(*"the same set of **observed** revocation entries"* is stated for token-keyed markers). That is a real
piece of design work and it should be a proposal of its own, on evidence, not a rider on a wording fix.

**What (b) costs, stated plainly, because the UI has to say it out loud:** withdrawing a share stops new
grants immediately and **revokes nothing already issued**. A member who holds a live token keeps the
access until it expires. **The share's TTL is therefore a share-design parameter, not an afterthought** —
it *is* the withdrawal latency, and an operator choosing `ttl_ms` is choosing how long "stop sharing"
takes.

**This is not a new convention — it is the one the protocol already uses for revocation.** §5.10's
**revocation-propagation bound** (0.8.1, W7 Knob 2) already establishes that a peer *"MUST declare a
finite `revocation_propagation_bound`"* and that *"the revocation exposure window for a cap verified at
this peer is `min(TTL_granularity, B)` — a reason-about-able quantity."* **(b) is that same sentence
applied one layer up**, and it is why (b) is not a climbdown: bounded-and-declared is how this protocol
already treats revocation everywhere. What is missing is not the model. It is the enforcement of the
`TTL_granularity` half — §3.

### §2.1 The correcting clause for Q7

`PROPOSAL-SHARE-AS-GRANT` §3 currently reads:

> **Leaving revokes.** … This delivers the property Q7 wanted from check-time resolution — *revocable by
> leaving* — **without any step added to the check path.**

**As written this reads as immediate revocation, and `entity-browser-rust` built toward it on that
reading.** Replace with:

> **Leaving closes the door on the `request` path; it does not recall what is already out.**
> `[SCOPED 2026-08-17 per §3.3 — the original clause said "never holds the hash" and over-generalized to
> the §4.4 path, where the granter does hold it and `revoke` is available immediately.]`
> Removing a member's policy entry —
> or writing it empty per `PROPOSAL-CAPABILITY-EMPTY-GRANTS-AND-POLICY-WITHDRAWAL` — causes every
> subsequent `system/capability:request` from that member to fail subset-validation with
> `403 scope_exceeds_authority`. **It does not revoke `request`-minted tokens already issued.** Those are
> returned inline with no tree write (§6.2), so the granter never holds their hashes and
> `system/capability:revoke` — keyed by token hash — is not addressable by it. **On that path withdrawal
> is immediate for new access and bounded by `expires_at` for existing access**, which makes the policy
> entry's `ttl_ms` the withdrawal latency and a share-design parameter.
> **The §4.4 authenticate-response path is different and better:** that capability *is* recorded per-peer
> at mint time, so the granter can name it and `revoke` it immediately (§3.3). **A complete withdrawal
> from a connected peer is therefore two operations — the policy write, and a `revoke` of that peer's
> delivered capability.** No step is added to the check path, which is the part of the original claim
> that holds.

---

## §3 The defect (b) rests on — nothing bounds the minted TTL

**This is not browser-rust's finding.** It surfaced while verifying theirs, and it is the reason this
proposal targets the core spec.

Ruling (b) says the exposure window is the TTL. **The spec never says whose.**

**§6.2's evaluation contract for `request` bounds `grants` and nothing else.** Step 3 subset-validates
the request's `grants` against two ceilings — the caller's cap and the policy entry. Step 4:

> Mints a token with the request's `grants` (now validated as bounded by both ceilings); returns it per
> the result envelope shape.

**`ttl_ms` appears in neither step.** `policy-entry.ttl_ms` exists in the CDDL, annotated *"Optional;
null = no expiry (issuer can still revoke)"* — but **no normative sentence anywhere makes it a ceiling on
the minted token's `expires_at`.** Nor does anything bound the mint by the caller's own capability expiry.

**§5.6 does not rescue it.** Its attenuation rule (*"child expiration must not exceed parent's"*) governs
**parent → child chains**. A `request`-minted token has **`parent: None`** — it is a fresh root cap
signed by the granting peer. There is no parent link for `is_attenuated` to check.

**The consequence.** A peer whose own capability expires in an hour can call `request` with
`ttl_ms = 10 years` and receive a **root token valid for ten years**, carrying a subset of the scope it
held for one hour — *on an implementation that performs the subset check on grants alone.* Every scope
check passes, and nothing in the verify path objects, because the token is internally valid and
self-signed by the granter.

### §3.1 The cohort, measured — **CORRECTED 2026-08-17**

> **The original table said rust minted the ten-year token. It rejected. The row was wrong and it was
> published in a routing packet.** `entity-core-rust` caught it, `entity-core-go` validated both legs
> against source, and both are right. Corrected here at the point of claim rather than silently, with
> the original claim left visible above it. **See §3.1a for what produced the error.**

| impl | behavior at `request` | site |
|---|---|---|
| **`entity-core-py`** | **Clamps.** `expires_at = min(caller_cap.expires_at, now + policy.ttl_ms, now + request.ttl_ms)` over the defined values | `capability.py:499-512` @ `40d1df2` |
| **`entity-core-go`** | **Unbounded — the escalation is real here.** `requireAttenuation` builds `parentToken := CapabilityTokenData{Grants: parent}`, **stripping the caller cap's `ExpiresAt`**, so `IsAttenuated`'s §5.6 expiry check sees an infinite parent and is skipped. The subset check is grants-only; the mint then takes `request.ttl_ms` alone | `ext/capability/handler.go:872` + `:221` @ `cc537ea` |
| **`entity-core-rust`** | **~~No clamp~~ → REJECTED, and the over-long mint was unreachable.** `is_attenuated(&child, caller_cap, …)` is called with the **real `caller_cap`**, while the probe `child` was built `expires_at: None`. §5.6: null child under finite parent → `false`. **So every request from an expiring caller 403'd, whatever `ttl_ms` it asked** | `extensions/capability/src/lib.rs:199` @ `f23fb3b` |

**The two implementations diverged in *opposite* directions on one surface: go under-bound, rust
over-bound.** That is a sharper finding than the one originally filed, and it is the reason the vector
shape in §5 had to change — see §3.1b.

**py is correct and is alone in being correct.** Its comment — *"Policy entry `ttl_ms` further
constrains lifetime (if present)"* — states the rule the spec omits. **Neither go nor rust is refutable
from the text**: go implements step 4 literally, and rust implements §5.6 literally at a site step 4
never told it to apply. **The spec is what produced both.**

### §3.1a What produced the error, and what it cost

**The rust row was inferred from the assignment that computes `expires_at` without reading the
attenuation probe fifteen lines above it.** `let expires_at = req.ttl_ms.and_then(…)` is a true quotation
and the conclusion drawn from it — *"unbounded mint"* — was false, because control never reached that
line for the caller the claim was about.

**Cost, stated honestly.** The wrong row went into `ROUTING-2026-08-17-f` §3 and told rust to add a clamp
at `lib.rs:215`. **The outcome was still right — rust's `683a4f0` is titled *"clamp the request mint, and
fix the probe that made it unreachable"*, so they landed the clamp and found the real defect while
implementing it.** The reasoning I supplied was wrong and the seat corrected it. **A ruling that arrives
at the right instruction by the wrong route is not validated by the outcome**, and it would not have
survived a different implementation.

### §3.1b The vector shape changes — and this is the load-bearing consequence

`entity-core-rust` raised this and `entity-core-go` seconded it. **It is correct and it invalidates the
vector as originally specified in §5.**

**A vector asserting only `minted.expires_at ≤ caller_cap.expires_at` scores go and rust identically** —
go passes by clamping and minting a shorter token; rust "passes" by never minting at all. **Reject
behavior is indistinguishable from clamp behavior under a `≤` assertion.**

The ruling says *mint a clamped token*: a legitimate over-long request from an expiring caller MUST
return **`200` with a clamped expiry**, not `403`. So the vector MUST assert:

1. status is **`200`** — the request succeeds; and
2. `minted.expires_at` **equals** the exact `MIN(…)` value, not merely `≤` any one term.

**Note for `entity-core-rust`:** confirm the new `clamp_mint_expiry` is actually *reachable* — that the
clamp runs before, or instead of, the pre-existing whole-token `is_attenuated` expiry rejection. The
probe fix at `683a4f0` builds the probe with the clamped expiry, which is the right shape; the vector
above is what proves it.

**The construction py uses is already normative elsewhere in this document**, which is what makes this an
omission rather than a design question. §5.6's expiration nil-vs-finite rule names it verbatim for role's
synthetic cap:

> the conventional construction is `expires_at = MIN(parent.expires_at, role.ttl,
> caller_capability.expires_at)` **taking only the defined values into the minimum.**

**§6.2 needs the same sentence for `request`.** It was written for ROLE and never generalized.

### §3.2 Proposed edit

In §6.2, replace step 4 of the *Evaluation contract for `request`*:

> 4. Mints a token with the request's `grants` (now validated as bounded by both ceilings); returns it
>    per the result envelope shape (above).

with:

> 4. Mints a token with the request's `grants` (now validated as bounded by both ceilings) and with
>    **`expires_at` bounded by every applicable ceiling (normative)**:
>    `expires_at = MIN(caller_capability.expires_at, now_ms() + policy_entry.ttl_ms, now_ms() +
>    request.ttl_ms)`, **taking only the defined values into the minimum** (the §5.6 construction). If no
>    value is defined, `expires_at` is null. **A minted token MUST NOT outlive the capability that
>    authorized the request, nor the `ttl_ms` declared by the policy entry that ceilinged it.** Rationale:
>    `request` mints a **root** token (`parent: null`), so §5.6's parent-child attenuation rule does not
>    reach it — without this bound, temporal attenuation is the one dimension a requester can escape,
>    and the policy author has no handle on the lifetime of what is minted under its own policy. This is
>    also what makes policy withdrawal a bounded operation: the entry's `ttl_ms` is the withdrawal
>    latency for tokens already issued.

And to the `policy-entry` CDDL comment on `ttl_ms`:

```
    ttl_ms:       {type_ref: "primitive/uint", optional: true}
                                                          ; Optional CEILING on the lifetime of tokens
                                                          ; minted under this entry (§6.2 request step 4),
                                                          ; not a default. null = this entry adds no
                                                          ; temporal bound. It is the withdrawal latency
                                                          ; for this peer — see §6.2 writing-policy-entries.
```

**`delegate` needs no equivalent edit**: a delegated token has a real `parent`, so §5.6's `is_attenuated`
already binds its expiry, and both go and rust do clamp there.

### §3.3 F3 — §2.1's "never holds the hash" over-generalizes, and scoping it dissolves F2

**`entity-core-rust` found this and `entity-core-go` validated it in their own tree. Both are right, and
it is the most useful correction in the batch** — it changes the answer to F2 rather than qualifying it.

**§2.1 says the granter *never* holds the hash of a token minted from its own policy. That is true of
`request` and false of the §4.4 handshake path**, where the same policy entry's grants are delivered.
Verified in `entity-core-go` at two independent sites: `WriteMintedSession`
(`core/protocol/session.go:116`, called from `connect.go:689`) persists
`SessionData{MintedCapability: minted, RemoteIdentityHash: …}` — **a tree write, keyed by the grantee's
identity hash, carrying the minted cap's content hash.** That is precisely the grantee-keyed index §2
said does not exist. It does — for the connection cap.

**So the two paths carry *complementary* withdrawal mechanisms, and neither needs the other's:**

| path | mint is | granter holds the hash? | withdrawal mechanism |
|---|---|---|---|
| `system/capability:request` | **wire-only** (§6.2, no tree write) | **No** | **bounded by `expires_at`** — the §3.2 clamp |
| §4.4 authenticate-response | **tree-written** (session entity, keyed by grantee) | **Yes** | **`revoke`, immediately** — addressable by hash |

**F2 — RULED, and the ruling is "do not clamp the connection cap."** `entity-core-go` leaned toward
clamping at `connect.go:641` with the same helper, and offered to build whichever way this ruled. **It
rules the other way, for a reason that only became visible once F3 was on the table:**

1. **The connection cap is not policy-derived; it is a union.** §4.4: *"the initial grant scope delivered
   to A is the **union** of the SHOULD floor above and the matched policy entry's grants."* Clamping the
   token bounds **the floor as well** — and the floor is *"a core protocol contract independent of the
   capability extension"* by §4.4's own words. A policy entry would then be silently shortening a grant
   the capability extension does not own.
2. **It aims the knob at the wrong thing.** Grant #2 of the floor is
   `handlers: system/capability, operations: [request]` — the grant that lets a peer *renew*. Expire the
   connection cap and the peer must reconnect to get one, so `policy.ttl_ms` becomes a
   **connection-churn** setting rather than a withdrawal latency. It would work, in the sense that access
   ends; it would not mean what the operator wrote it to mean.
3. **The hole it patches is not open.** Withdrawal on this path is already immediate, via `revoke` —
   which is *better* than what the `request` path can offer, not worse. **F2 read as a missing bound
   because §2.1's over-generalization hid the mechanism that was already there.**

**What is owed instead is guidance, not a clamp.** Add to §6.2, with the withdrawal paragraph:

> **Withdrawal reaches live connections through `revoke`, not through the policy write.** Writing or
> emptying a policy entry governs *future* grants on both trigger points; it does not alter a capability
> already delivered at §4.4 authenticate-response. Because that capability is recorded per-peer at mint
> time, an implementation **MUST** be able to name it for a given remote identity, and **SHOULD** expose
> it so an operator withdrawing a peer can `revoke` it. **An operator withdrawing access from a connected
> peer performs two operations — the policy write, and a `revoke` of that peer's delivered capability.**
> A withdrawal that omits the second leaves the live connection's grant intact.

**The consequence for `entity-browser-rust`'s UI is a real improvement over §2.2:** on the connection
path *"access ended"* **is** sayable — provided the revoke is issued. It is the `request`-minted tokens
that remain TTL-bounded.

### §3.4 F4 — `ttl_ms == 0`, overflow, and which terms are durations

**F4a — `ttl_ms == 0`: RULED as `0` means expire immediately.** go treats it as *undefined → no bound*
(`*ttl > 0`); rust and py treat it as *defined → `expires_at = created_at`*. **go is the lone dissenter,
and `entity-core-go` explicitly declined to flip on cohort-count and routed it instead — which is the
correct handling of a genuine ambiguity and is why this is ruled rather than voted.**

Three reasons, and the cohort count is deliberately not one of them:

1. **`null` is already the "no bound" spelling.** The CDDL says *"Optional; null = no expiry."* If `0`
   also meant no bound there would be **two spellings for one meaning and none for immediate expiry** —
   and the value most likely to be written by accident would be the permissive one.
2. **Under this proposal's own framing it inverts.** `ttl_ms` is the withdrawal latency. *Zero latency →
   never expires* is backwards, and **fail-open on the exact axis the proposal exists to bound.**
3. **It is coherent with the withdrawal form.** An entry with `ttl_ms: 0` grants nothing that survives
   its own mint — the temporal analogue of the empty `grants` array in the sibling proposal.

**One thing MUST be pinned with it or the ambiguity returns one layer down: the boundary.** A token with
`expires_at == created_at` is only unambiguously dead if expiry is an **exclusive** upper bound. Add to
§5.6 / §5.2's temporal check: *a capability is expired when `now_ms() >= expires_at`* — so
`expires_at == created_at` is expired at every observable instant. **Without this, `ttl_ms: 0` is a
one-instant capability and the divergence reappears as a race.**

**F4b — overflow: pin it at the shared §5.6 construction, not at §6.2 step 4.** `entity-core-rust` raised
the placement and `entity-core-go` seconded it; both are right, because the `MIN(…)` construction is
named in §5.6 and *referenced* by §6.2.

The three impls do three different things today, and **the divergence is observable on the wire, not
merely internal**: `expires_at` is a token field, so two impls that encode a maximal lifetime differently
produce **different content hashes for the same request**.

- **rust** — `checked_add` → `None` → `.flatten()` **drops the term** from the `MIN`.
- **go** — also drops on overflow (`clampMintExpiry`).
- **py** — Python integers are arbitrary-precision, so it produces a **huge finite** `expires_at` where
  rust and go produce **no expiry at all**.

**Ruled: a term whose `created_at + ttl_ms` is not representable contributes no ceiling — it is treated
as absent from the `MIN`, exactly as a null term is.** That ratifies rust and go, and py must clamp to
the same rule rather than carrying an unbounded integer. **Implementations MUST NOT wrap**, and MUST NOT
saturate to a representable maximum either — saturation and absence differ in the encoded token.

**And the units, which `entity-core-go` flagged while validating F4b.** The `MIN(…)` construction mixes
**timestamps and durations**, and the spec never says which is which:
`caller_capability.expires_at` is an **absolute** wall-clock ms timestamp, while `policy_entry.ttl_ms` and
`request.ttl_ms` are **relative durations** requiring `created_at +`. §5.6's ROLE wording — `MIN(parent.
expires_at, role.ttl, caller_capability.expires_at)` — reads as three absolutes, and **`entity-core-go`
consumes `role.ttl` as an absolute** while treating `policy.ttl_ms` as relative. *(Whether go's
`role.ttl`-as-absolute is itself correct is an `EXTENSION-ROLE` question — flagged here, not chased, and
not ruled.)* **§5.6's wording MUST name the shape of each term**, or the construction it defines is only
portable by luck.

---

## §4 What this proposal does NOT claim

- **Not** that go or rust is buggy on §3 **as of today's text**. They implement step 4 literally. They
  become non-conformant only once this lands.
- **Not** that py is buggy anywhere here. On §3 py is alone and correct; the spec should say what py does.
- **Not** that model (c) — grantee-keyed revocation — is wrong. It is **deferred with a named trigger**:
  if a deployment surfaces a share whose withdrawal latency cannot be expressed as a TTL, (c) is the
  design to open, as its own proposal. **Per the standing deferral rule this cut is pinned to
  (`ENTITY-CORE-PROTOCOL.md`, §5.2 step 4 / §5.10, `f83c256`) and expires when that text moves.**
- **Not** a claim that any of this changes the check path. It does not — §2's ruling preserves Q7's
  no-added-step property, and §3 is a mint-time bound, not a verify-time one.
- **Not** a re-opening of the frozen §5.2 vector set. §6.2 mint semantics are the other side of that
  boundary.

## §5 Blast radius

| | |
|---|---|
| **Wire** | **None.** `expires_at` is already a token field; this constrains what the minter may put in it |
| **Frozen §5.2 vector set** | **Untouched** |
| **`entity-core-py`** | **The clamp: already conformant** — this ratifies it. **F4b: one change** — clamp an unrepresentable `created_at + ttl_ms` to *absent* rather than carrying an arbitrary-precision integer (§3.4) |
| **`entity-core-go`** | Clamp **landed** (`b5cecbf`, verified: `requireAttenuation` was grants-only, so this was the real escalation). **F4a** staged pending this ruling — drop both `> 0` guards. **F2: nothing** — ruled against clamping the connection cap (§3.3) |
| **`entity-core-rust`** | Clamp + probe fix **landed** (`683a4f0`). **Confirm the clamp is reachable** and not shadowed by the pre-existing `is_attenuated` expiry reject — §3.1b's vector is what proves it |
| **`entity-browser-rust`** | **The A2 answer, improved by §3.3.** On the §4.4 path the UI **may** say *"access ended"* once the peer's delivered capability is revoked; for `request`-minted tokens it may say only *"no new access"*. Withdrawal is **two operations** — policy write + `revoke`. Their share design still needs a deliberate `ttl_ms` for the `request` path |
| **`entity-workbench-go`** | Same, when they build the share side |
| **`entity-core-go` — as oracle author** | **Four `validate-peer` checks in the `capability` category** (`cmd/internal/validate/capability.go`), appended to the `GUIDE-CONFORMANCE` §9 register. **None exist today.** (1) **The clamp, in the shape §3.1b requires** — caller cap expiring at `T`, `request` with `ttl_ms ≫ T` → assert **`200`** AND `expires_at == MIN_DEFINED(…)` exactly. **A `≤ T` assertion scores reject and clamp identically and MUST NOT be used.** (2) `ttl_ms: 0` → token expired at every observable instant. (3) `created_at + ttl_ms` unrepresentable → term absent, not wrapped, not saturated. (4) §4.4 delivered capability is nameable per-peer and `revoke`-able |
| **`entity-core-keystone`** | **Authors nothing here.** It *consumes* the oracle: once the four checks land and the oracle is re-pinned, `check_set_digest` moves and the cohort census re-runs across the peer matrix. **Its work is triggered by the oracle bump, not by this proposal** |

## §6 Ask

1. **§2.1's clause folds into `PROPOSAL-SHARE-AS-GRANT` §3 here, in this repo** — it is a correction to
   an arch ruling and this is the filing seat. **Done, twice: corrected, then scoped per §3.3.**
2. **§3.2 + §3.4 route to `entity-core-protocol` as a core-spec revision** — the `MIN(…)` ceiling at §6.2
   step 4, and at **§5.6**: the `ttl_ms == 0` definition, the exclusive-expiry boundary, the
   overflow-is-absent rule, and **which terms are timestamps and which are durations**. §3.4's last item
   is the one most likely to be skipped and is the one that makes the shared construction portable.
3. **§3.3's guidance paragraph routes to §6.2** — withdrawal reaches live connections through `revoke`,
   not through the policy write.
4. **`entity-browser-rust` is unblocked on A2 by §2 + §3.3** and does not wait on the revision landing.
5. **`entity-core-go` owes no code until §3.4 lands**; F4a is staged and F2 is ruled away. **`entity-core-
   rust` owes the reachability confirmation** in §3.1b.
