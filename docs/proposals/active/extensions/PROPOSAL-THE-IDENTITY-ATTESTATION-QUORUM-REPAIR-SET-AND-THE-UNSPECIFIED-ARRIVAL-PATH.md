# PROPOSAL — the identity / attestation / quorum repair set: 27 findings, and the one place the extension stack composes is the one place it is unspecified

**Status:** DRAFT (2026-09-09) — **HELD deliberately.** The extension tier is deferred behind core
convergence; this document exists so the repair set is enumerated, sized and sequenced rather than
re-derived when the tier reopens. **Nothing here is folded.**
**Target:** `EXTENSION-IDENTITY.md` (v3.10) · `EXTENSION-ATTESTATION.md` (v1.3) ·
`EXTENSION-QUORUM.md` (v1.2).
**Raised by:** a formal-methods track running machine-checked models against pinned copies of the
three specifications, plus a source census of three independent implementations. **27 distinct
decisions**, graded by the filing track into severity bands.

---

## 0. What this is, and what has been re-verified here

**The findings are a peer's measurement; the ones this document asserts have been re-checked
against our own text.** A true sentence about someone else's artifact carries none of its
verification onto ours — the party that moves it owns re-checking it. So §1 states what was
re-derived here, with our citations; §3 carries the rest as filed, cited, and explicitly **not yet
re-verified**.

**What the filing track's evidence is.** Each finding rests on three separable things, and they are
not interchangeable: a **census** of pinned spec text (machine-checked); a **field state** read from
three implementations' source (source reads, explicitly not behaviour — no peer was run); and a
**grade**, which is judgement and is ours to overturn. They separate the three themselves and say
so.

**One property of this set makes it different from every other cross-implementation finding we
carry.** On other tracks, the three implementations independently derived the same missing rule and
that unanimity was the argument for writing it down. **Here they disagree with each other.** All
three added a kind branch ahead of `IDENTITY §6.3` phase 1 that no document contains, and **no two
did the same thing.** So *"the implementations work around it"* is not available as a mitigation —
they work around it differently, and the differences are observable across a peer boundary.

## 1. Re-verified here — three findings, and one of them is a live authority-denial

### 1.1 An unsigned revocation can strip authority from any cert — no key compromise required

**This is the finding to act on first, and it is confirmed in our own text.**

Four statements, three of them in one document:

| Site | Says |
|---|---|
| `ATTESTATION` §3.3 | *"The attestation primitive validates the signature on the revocation."* |
| `ATTESTATION` §4.3 | *"Signature validation is not part of liveness… consumer-supplied per topology"* |
| `ATTESTATION` §6 (`:create` handler) | *"The handler does NOT validate the signature"* |
| `ATTESTATION` **TV-A8** | resolves the conflict by naming the rejection point: *"identity's `identity_verify_cert` rejects A at topology-dispatch step"* |

**§3.3's sentence is the outlier** — it contradicts §4.3 and §6 in the same specification, and the
v1.1 history entry records the ratified intent as *"substrate stays signature-agnostic."*

**TV-A8's justification does not hold for the kind it is about.** `IDENTITY`'s
`identity_verify_cert` step 1 reads:

```
if att.properties.kind != "identity-cert" and \
   att.properties.kind not in identity_lifecycle_kinds():
  return error("not_identity_attestation")
```

and `identity_lifecycle_kinds()` returns exactly `{identity-rotation-handoff,
identity-rotation-recovery, identity-retirement}`. **`"revocation"` is a universal kind and is in
neither set**, so it is rejected at step 1 — *for the wrong reason, and before topology dispatch,
which is reached later in the same function.* **The rejection point TV-A8 delegates to does not
exist for revocations.**

**What that leaves standing.** `IDENTITY` §3.6 step 3 honours a revocation on
`is_attestation_live(rev)` — structural state only, by §4.3's own definition — plus
`identity_is_authorized_revoker(rev.attesting, …)`, whose body is `return revoker == quorum_id`.
**Both are field comparisons. Neither is a signature check, and no identity-layer path ever
verifies one for this kind.** `ATTESTATION` §8 permits a raw `tree:put` to attestation paths.

> **⇒ An unsigned revocation naming the quorum as `attesting` marks any cert whose chain roots
> there `authority_revoked`. Denial of authority, no signature, no key compromise.**

**Compounding it in the useful direction:** a *legitimate* revocation arriving over sync is
unbound by §6.3 phase 2a before it can take effect (§1.2 below). **The mechanism is unusable in the
direction it is meant to work and usable in the direction it is not.**

### 1.2 A declared dispatch row is unreachable, and compromise recovery is satisfied vacuously

`IDENTITY` §6.3 phase 2 declares seven handler rows, one of them
`(quorum-publish, *) → seed_contacts_cache`. Phase 1 runs `identity_verify_cert` unconditionally,
and by the step-1 gate quoted above `quorum-publish` is rejected — §4.4 confirms the kind is
**owned by `EXTENSION-QUORUM`**, with identity as a consumer that *"does not duplicate the kind
definitions."* **So the row that fills the cache never runs**, and on the cross-peer arrival path
phase 2a unbinds the entity on the way past.

**§9.4 keys compromise recovery on exactly that cache**: an arriving `identity-rotation-recovery`
MUST validate K-of-N against the cached `quorum-publish` at
`contacts/{old_handle_hex}/quorum-publish`, and MUST reject fail-closed when none is cached.

> **⇒ A compromise-recovery signed K-of-N by the identity's *real* quorum, delivered to a contact
> that has already received that identity's genuine `quorum-publish`, is rejected.** §9.6 names
> compromise recovery as the only remedy for a stolen controller key; §11.3 calls the
> configuration that provides it the recommended default.

**And the prohibition beside it passes vacuously.** *"Recovery rotations cannot be accepted on the
strength of arbitrary signatures"* holds trivially on a peer that can never accept one. **A check
that exercises only the fail-closed rejection passes on a peer where recovery is impossible** —
which is why the positive witness has to be part of what any check set discriminates.

### 1.3 Two normative statements that cannot both be satisfied in the recommended shape

`IDENTITY` §9.2 is a MUST — reject attestations under `system/identity/public/` carrying signatures
from any currently-live controller — while §4.2's valid-modes table gives `agent` **any of the four
modes**, §4.2a and §5.1 place `identity-cert (function=agent, mode=public)` under `public/cert/`,
and §2.3 makes an agent cert's `attesting` the issuing controller in the three-key shape.

**In the three-key default — §11.3, the recommended shape — those produce exactly what §9.2
forbids.** The four-key shape is unaffected, and that asymmetry is the tell: §9.2 reads as written
with only the four-key shape in view. **This is a choice between two landed normative statements
and is a design decision, not an editorial one.**

## 2. The decision that gates most of the rest — who owns the arrival path

**`IDENTITY` §6.3 phase 1 is identity's; quorum's kinds arrive through it; no document says what
happens when they do.** All three implementations invented an answer:

| | An arriving `quorum-publish` | `quorum-update` | `revocation` |
|---|---|---|---|
| implementation A | survives, **not cached** | survives, no effect | **survives and validates** |
| implementation B | **cached, unvalidated** | unbound | unbound |
| implementation C | **validated, then cached** | validated | unbound |

**§9.4 keys its fail-closed rule on the first column, so the peers do not interoperate on
compromise recovery.** Every cell there is a different answer to a question no document asks.

**Deciding this is upstream of at least nine of the 27**, and it is one ruling rather than nine. It
is also the general lesson the set carries: *the one place the extension stack composes is the one
place it is unspecified.*

## 3. The full set, as filed — carried, not yet re-verified here

**Band S — structural (the design does not do what the document says).** The trust anchor cannot be
stored through the path that stores it (§1.2 above) · nothing states what may *become* that anchor,
so there is no validation step to omit · a revocation's signature is checked nowhere (§1.1 above) ·
`QUORUM` §4.2 may trust unvalidated tree state, and the closure it needs is **write-side**, not the
read-side one first proposed — the filing track corrected its own remedy here and says so.

**Band A — high (a wrong answer on ordinary input, on a path that decides authorization).**
`QUORUM` §4.2 returns the creation-time roster while an update is in force, on a plain chain of
three · `ATTESTATION` §5.3 `find_live_head` filters successors by the full liveness predicate and
so cannot traverse a chain of three · identity may unbind an arriving `quorum-update`, a second
independent route to the same end state · `ATTESTATION` §4.3's `is_self_revoked` and `not_expired`
are used in normative pseudocode and defined in no section, and their two readings disagree about
liveness · §4.3's liveness equation needs a **joint** order over the supersedes and revocation
relations, since the recursion alternates and `visited` guards one hop — per-relation acyclicity is
not sufficient.

**Four more arriving after the triage, and they are not editorial:** a routine privacy rotation may
disable compromise recovery, because §5.1 writes the anchor under `published_handle` and §9.4 reads
it under `old_handle` · `IDENTITY` §6.3's `update_handle_cache_to` is undefined, **and the census is
the finding: all seven handler names in §6.3's dispatch table occur exactly once in the document,
in the table, and none is defined anywhere** · §3.6 step 3 may look revocations up in the tree that
§6.3 phase 2a deleted them from · an `identity-retirement` may be undone by the retired cert
arriving again, since phase 2 dispatches on `(kind, function)` and consults no state.

**Band B — moderate (interop divergence, availability, or a two-way-readable conflict).** §9.2 vs
§4.2 (§1.3 above) · `QUORUM` §4.2's tie-break is unstated and the three implementations break it
three ways · which of §4.2's two texts is normative, its sentence or its pseudocode, since they do
not compute the same thing · `as_of` is rested on and never defined · `QUORUM` §6.2 may validate a
threshold against the pre-resolution array length, permitting a quorum that passed every check to
be permanently unable to reach its own threshold · §5.1's retention floor expires *at* accept, so a
duplicate delivery of one recovery gets a second, different verdict · nothing enforces §3.6's
`att.attesting = target.attested`, asserted in a parenthetical · §4.2's `supersedes` graph is
assumed acyclic in prose, validated nowhere — **not constructible under content addressing, and the
unstated argument is the finding.**

**Band C — editorial.** Two word-level fixes: `ATTESTATION` §5.2 passes a hash to §4.0's path-keyed
accessor, and §3.6's handoff arm dereferences an unresolved lookup its own sibling caller guards.

**The validation surface — not about any spec's design.** A validation matrix with 136 vector IDs
and a migration-fixes proposal are stranded outside the corpus a reader gets, while **20 citations
across 13 documents point at them — seven normative specs and one inside `EXTENSION-ATTESTATION`
§9's conformance clause** *(re-measured here: exact)* · `EXTENSION-IDENTITY` carries **zero** inline
test vectors against 16 in attestation and 13 in quorum *(re-measured here: exact — identity's only
two vector references sit in its document history and cite attestation's)* · and a granularity
ruling is owed, because a vector over a composite asserts nothing about a named helper it consumes.

## 4. The rule the set proposes, and it is the ratchet aimed at authoring

**For each normative MUST, name the operation that enforces it and the check that exercises that
operation.** Nine of these findings share one shape — an obligation stated in one place whose
enforcement is assumed to happen somewhere that does not do it — and **three of the nine could not
have been written under this rule, because there is no operation to name.**

**Adopt it.** It is `SPECIFICATION-FORMAT` §8.5a's addressable-inventory work pointed one level
deeper, it is mechanically checkable once a requirement row has an id, and §1.1 and §1.2 above are
both exactly what it catches: a MUST whose named enforcement point rejects the kind it was supposed
to enforce, and a MUST satisfied vacuously because its positive witness is unreachable.

## 5. Why this is held, and what reopens it

**Sequencing, not doubt.** The core protocol tier converges first; these three specifications sit
above it and the seat that generates implementations from them is building a different extension
next. Folding twenty-seven decisions into three specifications while the tier below them is moving
would put every one of them at risk of a second pass.

**What reopens it, in order:** ① the arrival-path ruling (§2), because it is one decision that
resolves nine findings and none of the others should be written before it · ② §1.1's
signature-check repair, which is a security fix and is the one item that has a claim on being taken
out of order · ③ Band S and Band A as one fold per specification · ④ Band B and C as a hygiene
pass with the rest.

**§1.1 is the one to argue about.** It is a live authority-denial reachable without key compromise,
and the case for pulling it forward is that it does not depend on the arrival-path ruling: §3.3's
outlier sentence and TV-A8's false justification can be corrected against `ATTESTATION`'s own
ratified intent, and identity's step-1 gate can admit `revocation` into topology dispatch, without
deciding who owns the arrival path.

## 6. What is not claimed

- **No remedy is proposed for most of these**, deliberately. The filing track declined to write
  remedies on its own record of having a sound census and a corrected remedy twice, and that
  restraint is right. §1.3 in particular is a choice between two normative statements.
- **No behaviour was measured.** Every finding is structural, derived from spec text and source
  reads. **No peer was run and no exploit was built**, including for §1.1 — its reachability is a
  reading of our own pseudocode, and a peer that happens to reject unsigned revocations for some
  other reason would not exhibit it.
- **No signature is verified in any of the underlying models.** The filing track states outright
  that none of this is an assurance statement about the three specifications; it is a defect
  report, and every extension result there assumes signatures work.
- **The identity findings assume `QUORUM` §4.2 works**, which the same track separately measured as
  defective. They did not re-file those findings wearing identity section numbers, which is the
  correct call and is worth preserving when this is folded.
