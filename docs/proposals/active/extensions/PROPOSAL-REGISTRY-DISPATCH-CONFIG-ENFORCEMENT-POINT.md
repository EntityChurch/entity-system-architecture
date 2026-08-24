# PROPOSAL — the privacy MUST binds the write, not the load; and four corrections it exposed

**Status:** DRAFT
**Tier:** extensions — `EXTENSION-REGISTRY` §4.1, §6a.6, §6a.9.1, §11.1
**Answers:** `entity-core-py` **SA-PY-21 Q1 / Q2**, **SA-PY-22** · `entity-core-rust` **R-4** and the row-(d) finding · `entity-core-go`'s consolidated reply `2026-08-19-e`
**Corroboration (NOT evidence):** `entity-browser-rust` `5e2297c` — see §1.0 on why a cohort implementation cannot validate a cohort ruling
**Read at:** arch `fc27873` · go `1797754` · rust `e6fb500` · py `f4e8fc2` · browser-rust `5e2297c`

---

## §0 Summary

Four seats, one question: **what does a resolver do at load with a `resolver-config` that would
leak?** Today the spec answers twice, in two places, incompatibly:

| Section | Says | Subject |
|---|---|---|
| **§4.1 step 2** | *"a **distribution's shipped** `resolver-config` MUST NOT make a name-transmitting backend eligible for an unscoped name"* … *"An operator MAY override this on their own peer; a distribution MUST NOT ship it."* | **the act of shipping** |
| **§11.1 `REG-DISPATCH-CATCHALL-LOCAL-1`** | *"MUST be **refused or normalized at load**"* | **the act of loading** |

**A loading resolver sees bytes. It cannot tell whether the operator wrote them or a distribution
shipped them** — §6a.9.2's store-first rule puts both in the same entity at the same path, and an
out-of-band seed is byte-identical to an operator edit. So §11.1 makes the resolver enforce a rule
whose *subject it cannot identify*, which necessarily over-enforces and **kills the `MAY` that §4.1
grants in the same breath.**

**Ruled: the MUST binds the write, not the load.** Enforcement moves to the two points where an
*actor* is identifiable, and §11.1's load-time clause — the sentence that created the contradiction —
is withdrawn.

**This is L17 pointed at a `MAY`:** a rule whose subject the enforcing party cannot observe is a
rule two conformant peers cannot both implement, for the same reason a value with no config site is.

---

## §1.0 First — what does NOT decide this `[corrected 2026-08-19]`

**This proposal originally led with *"it is what the only implementation does,"* citing
`entity-browser-rust`'s `name_dispatch.rs`. That is not an argument and it has been struck.**

A cohort implementation is **our own work**. It is one seat's choice, made under delivery pressure,
inside a proof-of-concept, without the question having been posed to them — browser-rust never
argued for kind-scoped, they wrote a function and it happened to take one parameter list. **Treating
it as validation is circular**: we would be ratifying our own guess and calling the echo a second
opinion, which is exactly the *"cohort-consistent, not independent convergence"* error
`AGENTS-STANDARD` names for conformance numbers, one layer up. **The same objection applies to
`entity-core-go`'s and `entity-core-py`'s converged push** — three seats agreeing is three seats
agreeing.

**What a shipping implementation *is* good for**, and it is not nothing: evidence that a reading is
**implementable**, that it costs little, and that a practitioner under real constraints reached for
it without being told to. That is a sanity check on a conclusion reached elsewhere. It is never the
conclusion.

**And a related correction: the argument all three seats made — and that this proposal published —
is weaker than it looked.** See §1.2.

## §1 Q1 — eligibility is **kind-scoped**, derived

**The question:** a broad rule naming `did-web` when no `did-web` entry exists in `resolver_chain` —
does it violate the MUST (kind-scoped) or not (chain-scoped)?

### §1.1 The deciding argument — who the MUST binds, and what they can evaluate

**§4.1 step 2's MUST binds a *distribution*:** *"a distribution's shipped
`system/registry/resolver-config` MUST NOT make a name-transmitting backend eligible for an unscoped
name."* The obligated party is whoever **ships** the artifact. That fact decides the reading.

Under **chain-scoped**, the proposition *"this shipped config is safe"* is not a property of the
shipped artifact at all. It is a property of **the artifact paired with whatever the downstream
operator later adds to `resolver_chain`** — a pairing the distribution cannot evaluate, cannot
re-review, and (per §1's bootstrap-with-precedes model, where users inherit by accepting the build)
cannot reach again once shipped. **You cannot hold a party to a property they are structurally unable
to evaluate**, and holding shippers to this property is the entire purpose of the sentence.

Under **kind-scoped**, validity is a function of `name_format_dispatch` alone. A reviewed artifact
stays reviewed. The safety property is **monotone under extension**: adding chain entries can never
invalidate a config that was valid when shipped.

*Stated generally, because this is a recurring shape and not a registry fact: **a configuration
predicate that binds the author of an artifact MUST be decidable from that artifact**, or it binds
someone who cannot check it.*

### §1.2 The argument three seats made, and why it is not the one to rely on

All three engine seats — and the first version of this proposal — argued from **silent arming**: a
chain-scoped config transmits nothing today, and *"the day an operator adds that backend the
pre-existing row arms itself, through an edit that touches nothing the MUST mentions."*

**It does not arm silently.** §4.1 already binds *"the configuration as a whole, not one row,"* so
adding that chain entry is itself a write, and §4.3's write-time check evaluates the **whole
resulting config** — under which a chain-scoped implementation refuses at exactly that moment. The
hole the silent-arming argument describes **is closed by whole-config validation under either
reading.**

**Three seats and arch published a rationale that its own neighbouring paragraph refutes.** The
ruling survives; that argument does not, and it is recorded rather than quietly replaced because a
correct conclusion resting on a wrong reason is the thing that gets cited later.

### §1.3 The tie-breaker — asymmetric failure cost

Kind-scoped's false positive: a presently-harmless config is refused, the operator deletes a kind
from one row, sees the diagnostic immediately, loses nothing, and can override deliberately (§2).
Chain-scoped's false negative: a config that discloses **every bare name a user types** the moment an
unrelated chain entry appears — silently, on the happy path, and by §4.1's own word *irreversibly*.

**A visible recoverable edit against silent irreversible disclosure is not a close call**, and the
conservative reading wins even at the cost of refusing configurations that are harmless today.

### §1.4 Corroboration, at its proper weight

Consistent with §1.0: `entity-browser-rust`'s `validate_rules`/`eligible_backends` take no
`resolver_chain`, so a practitioner building this under delivery pressure landed on the kind-scoped
shape without the question being posed. **That is evidence the reading is cheap and natural to
implement. It is not evidence that it is correct**, and it is listed last for that reason.

**No existing check can settle this empirically**, which is why it needed a derivation:
`REG-DISPATCH-CATCHALL-LOCAL-1`'s observable is *the absence of a request*, and a kind absent from
the chain transmits nothing under **either** reading. §3 adds the check that can.

---

## §2 Q2 — the enforcement point moves to the write; the load-time clause is withdrawn

**Ruled, three parts:**

**(a) Publication / packaging time — a distribution MUST NOT ship a violating config.** This is
§4.1's sentence, unchanged in meaning, now with its enforcement point named: a **packaging lint**,
not a runtime check. `entity-browser-rust`'s `validate_rules` is exactly this shape and is the
reference form — it returns **every** violation rather than the first, *"because an operator fixing a
chain wants the whole list."*

**(b) Config-write time — a peer MUST refuse to store a violating `resolver-config` `[MUST, v1.17]`.**
When a config is written through the `system/capability/registry-configure` surface, the writer is a
present, identifiable actor, and a refusal is actionable, non-destructive, and reaches a human. This
is the §6a.9.2 `set-issuer-policy` shape applied to the neighbouring entity, and it is the door the
old load-time clause was reaching for.

**(c) At load — surface it; do NOT normalize, do NOT refuse to start `[MUST, v1.17]`.** A resolver
that finds a violating config MUST surface the condition as a diagnostic and MUST NOT silently alter
its behaviour or its bytes. Two reasons, and the second is the one that decides it:

- **Silent normalization makes the operator's config lie** — the bytes say one thing and the peer
  does another, with no diagnostic. That is SA-PY-22's defect (§4 below).
- **Refusing to start would delete the `MAY`.** §4.1 grants the operator the override *on their own
  peer*; a peer that will not boot on a config the operator deliberately wrote has revoked the grant.
  **The out-of-band seed path (§6a.9.2) means a violating config can always be present at load, so
  load-time is exactly where the two acts become indistinguishable and enforcement must not live.**

**The `MAY` was never in conflict with the MUST.** An operator editing their own peer and a
distribution shipping a default are **different acts with different actors**. §11.1 collapsed them by
moving enforcement to a point that cannot tell them apart, and the contradiction was manufactured
there — not in §4.1.

**No provenance field.** `name_format_dispatch[]` rows carry no opaque slot, so adding one is a
type-hash change to a content-addressed type for a distinction that the write-time point does not
need. Both engine seats flagged this; it is declined for their reason.

---

## §3 The vector that can actually settle both — `REG-DISPATCH-CONFIG-REFUSED-1` `[v1.17]`

Both seats correctly observed that whichever way Q1 is ruled, **it is unenforceable without a
config-read-back vector** — straight back to L17. This is that vector, and it discriminates Q1 and
Q2 at once:

| Row | Offer to the config-write surface | MUST |
|---|---|---|
| (a) | broad rule (`*`) naming `did-web`, **and** a `did-web` chain entry present | **refused** |
| (b) | broad rule (`*`) naming `did-web`, **no `did-web` chain entry at all** | **refused** — this is the Q1 discriminator. Chain-scoped would accept |
| (c) | no `name_format_dispatch` at all, `did-web` in the chain | **refused** — §4.1's second door |
| (d) | scoped rule (`did:web:*`) naming `did-web` | **accepted** |
| (e) | after any refusal, **read the stored config back** | **byte-identical to before the offer** — nothing partially written, nothing normalized |

**Row (e) is the one that closes SA-PY-22** and the reason the vector is read-back rather than
status-only: a peer that accepts-then-rewrites and a peer that refuses both return a non-success on
some path, and only the stored bytes distinguish them.

---

## §4 SA-PY-22 — answered, and it dissolves

py asked whether *"normalizes at load"* narrows the in-memory view or rewrites the stored
`resolver-config` — **cross-impl observable, because the content hash moves under one and not the
other.** With §2(c) ruled there is **no load-time normalization at all**, so the question has no
subject.

**The general rule it earns, stated once `[MUST, v1.17]`: a resolver MUST NOT rewrite an operator's
stored configuration as a side effect of reading it.** py read §6a.9.1's *"a use bound, not a
re-issue"* as the general principle and implemented view-narrowing rather than assuming — **that
instinct was right and is now the rule.** Reading is not writing, at either surface.

---

## §5 R-4 — rust's refutation is correct; §3.1 is not §6a.6

**Closed as answered. No code owed by any seat.** rust landed the keyed reader as routed, then
reverted it (`cf570b2`) when the armed gate took
`registry.v6_meta_resolver_revocation_honored` from PASS to FAIL. Verified against the spec text
rather than the report:

- **§3.1** — the generic revocation rule — says *"`:resolve` MUST check for a
  `system/registry/revocation` targeting a candidate binding."* It **constrains no storage path and
  names no index.** It is the local peer's own revocations at the §3 layer.
- **§6a.6** is `revoked(registry, binding_hash)` — its parameter is a **registry**, its caller is
  §6a.4 (the peer-issued algorithm), and its whole argument is about *"the party §6a.1a names as the
  fourth actor"*, a remote byte-server. **It is scoped to a remote registry by construction.**
- **`entity-core-go` scans the same prefix at the §3.1 layer** (`revocationFor`). The divergence R-4
  described **does not exist between go and rust.**

**R-4 was a mis-routed finding, and the mis-routing was arch's** — it read a §3.1 scan as a §6a.6
violation because both touch revocations. **Delta:** one scoping sentence on §6a.6 so the next reader
does not repeat it. **The keyed form is not extended to §3.1**, and rust's revert stands.

---

## §6 Row (d)'s *"or pinned"* is unreachable — the same category error, made twice, 600 lines apart

`REG-TTL-RESOLVER-CEILING-1` row (d) reads *"a sticky binding (`local-name` **or pinned**, no
`ttl`)."* **A pin can never carry a `hints.max_ttl`**, because §4.1 **step 1** returns the
synthesized pin result *before* the step-2 filter and the chain are reached — so no chain entry, and
no `hints`, is ever in scope.

**The spec already says this, in a sentence I wrote:**

> **`pinned` is not a backend kind and cannot be reached from dispatch:** §4.1 step 1 returns a
> pinned match **immediately**, before the step-2 filter runs… **Naming it in `backend_kinds` was a
> category error that no configuration could act on.**

I then named `pinned` in a ceiling vector row, which is the same category error in a different field
— **the third time in two sessions that a vector row was written without checking that its input can
reach the mechanism** (row 3 of the name-constraints vector, then this). **Delta:** drop *"or
pinned"*; row (d) is `local-name` only.

**And the substantive answer, so nobody re-adds it:** a pin is **the user's own assertion**, not a
backend's answer. The resolver ceiling bounds how long *a backend's answer* is honored. §4.1.2's pin
carve-out already runs this way (*"a pin is the user's assertion, so reach is the user's problem"*).
**A pin is correctly outside the ceiling.**

---

## §7 The read-at-resolution MUST has no instrument — named, not silently trusted

go and py both flagged it and they are right: **`REG-TTL-RESOLVER-CEILING-1` cannot discriminate
latched-at-start from read-at-resolution**, because a fresh peer per check reads config once either
way. That is the **exact property v1.16's MUST is about** and the whole of rust's security argument.

**`REG-TTL-CEILING-REREAD-1` `[v1.17]`** — one peer process, three resolutions, the
`resolver-config` rewritten between them: `hints.max_ttl` absent → present → a lower value. The
effective lifetime MUST track the current stored config at **every** resolution. A peer that latches
at start passes rows 1 and fails 2 and 3.

**Recorded honestly:** no current harness drives one process across a config rebind, so this vector
**lands specified and unimplemented**, and it goes on `COHORT-OPEN-ITEMS.md` as owed rather than
counted as coverage. A ruled MUST with no instrument is the L17 gap in its second form — the value
had no site, this has no check — and pretending otherwise is what a green suite that is not looking
buys you.

---

## §8 Deltas

| # | Section | Change |
|---|---|---|
| D1 | §4.1 step 2 | Eligibility is **kind-scoped** `[MUST, v1.17]` — the violation is naming the kind in a broad rule, independent of the chain's current contents |
| D2 | §4.1 step 2 | Name the **enforcement points**: packaging lint (distribution) + config-write refusal `[MUST, v1.17]`. Keep the `MAY`, and state that the two acts have different actors |
| D3 | §4.1 step 2 | **At load: surface, never normalize, never refuse to start** `[MUST, v1.17]` |
| D4 | §4.1 step 2 | **A resolver MUST NOT rewrite stored configuration as a side effect of reading it** `[MUST, v1.17]` (SA-PY-22) |
| D5 | §11.1 `REG-DISPATCH-CATCHALL-LOCAL-1` | **Withdraw** *"refused or normalized at load"*; re-scope the runtime half to *a conformant config produces no third-party request for a bare name* |
| D6 | §11.1 | **New `REG-DISPATCH-CONFIG-REFUSED-1`** — five rows incl. the byte-identical read-back |
| D7 | §6a.6 | Scoping sentence: §6a.6 binds the **remote-registry** path (§6a.4's caller); §3.1's local check constrains no storage path. Closes R-4 |
| D8 | §6a.9.1 `REG-TTL-RESOLVER-CEILING-1` | Row (d): drop *"or pinned"*; add why a pin is outside the ceiling |
| D9 | §6a.9.1 | **New `REG-TTL-CEILING-REREAD-1`** — one process, three configs |
| D10 | header | `1.16` → `1.17` |

---

## §9 Cohort impact

| Seat | Owed |
|---|---|
| `entity-core-go` | Relay this. `REG-DISPATCH-CONFIG-REFUSED-1` (5 rows) + `REG-TTL-CEILING-REREAD-1`. **Their `runRegDispatchFilter` oracle installs row (b)'s shape and now asserts kind-scoped** — the ruling moves that oracle, as they predicted |
| `entity-core-py` | **SA-PY-21 Q1/Q2 and SA-PY-22 all close.** Their view-narrowing instinct is ratified; the load-time normalizer they built comes out and moves to the write surface |
| `entity-core-rust` | **R-4 closes, revert stands, no code owed.** Row (d) loses *"or pinned"* — re-check the ceiling vector. New vectors as above |
| `entity-browser-rust` | **Their implementation is the ratified shape** — `validate_rules` / `eligible_backends` are kind-scoped and return the full violation list. Owed: nothing on Q1/Q2. Still R-8 (the fourth ceiling key), arch's direct conversation |
| `entity-workbench-go` | Unchanged — R-9 (`hints` round-trip) only |
