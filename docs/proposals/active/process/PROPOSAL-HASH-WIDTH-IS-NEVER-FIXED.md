# PROPOSAL — a hash has no width, and a derived hash has no automatic format

**Status:** **DRAFT — reference proposal, written after the fold. See §0.** Folded as
`SPECIFICATION-FORMAT` **§8.4.5 and §8.4.6**, plus corrections across nine specs.
**Target:** `SPECIFICATION-FORMAT.md` §8.4.5 (the width rule) **and §8.4.6 (the disposition rule —
`hold-and-fetch` vs `derive-to-meet`)**; corrections in `EXTENSION-NETWORK` §6.5.3.1,
`EXTENSION-ENCRYPTION` §7.3, `EXTENSION-RELAY`, `EXTENSION-TREE`, `EXTENSION-REGISTRY`,
`EXTENSION-CONTENT`, `EXTENSION-ROLE`, `EXTENSION-REVISION`, `EXTENSION-IDENTITY`, and others.
**Tier:** `process/` — the rules are authoring standards; their instances are extension text.
**Scope:** the corpus MUST NOT state a hash width as a requirement, **and every hash it introduces
MUST state which of two dispositions it has.** **Not** a change to any hash algorithm, format code,
or the wire — **but §8.4.6 does pin `system/peer` authoring to the floor**, which is a real narrowing
(§5).
**Source:** `entity-core-go`, 2026-08-10 — two spec-issues, one filed on top of the other's ruling.
**Cohort review:** both rules were ruled in-cycle; the width sweep was corrected by core-go the same
week (§3).

> **The title of this document was half the arc, and so was the first pass. `[corrected 2026-08-15]`**
> Arc C landed in two commits, and the second one's own subject line reads *"the width sweep had nine
> survivors — **and a second class underneath it**."* The first-pass reconstruction carried the nine
> survivors and **omitted the second class entirely** — §8.4.6, the `derive-to-meet` ruling, which is
> the more consequential of the two. §5 is that half. **The reconstruction was working from commit
> messages and missed a ruling named in the subject of the commit it cites.**

---

## 0. Process deviation — recorded, not hidden

**Folded across two commits (`38f3bfd`, `140a1bf`) with no proposal.** A corpus-wide prohibition
binding every spec is not wording-only hygiene.

Written now per the reconstruction ledger (`docs/status/PLAN-2026-08-15-spec-provenance-reconstruction.md`,
arc C). Not back-dated; if review rejects the rule, §8.4.5 comes out and the instances revert.
**Diligence pass run 2026-08-15 — see §8. It found that this arc had a second, larger ruling (§5) the first pass omitted entirely.**

## 1. The gap

`entity-core-go` turned on `--hash-type sha384` for the first time and found **31 failing checks and
a live cohort split**: go and rust answered `400` where python answered `200` on a 98-hex
`CONTENT_GET`. **Each implementation had implemented a different half of one sentence.**

`EXTENSION-NETWORK` §6.5.3.1 **pinned the served hash hex at 66 characters in the same bullet that
justified the format byte as crypto-agility.** It reserved a partition it then forbade anyone from
entering.

## 2. The ruling

**Ruled python's way: the length is implied by the leading format byte and is never assumed.** The
strictness test gets *stronger*, not weaker — reject any hex whose length disagrees with **its own
format byte**, rather than against a constant.

**Fixed generally rather than per-instance**, per the corpus's recurring-failure-shape rule.
`SPECIFICATION-FORMAT` **§8.4.5** now bars a fixed hash width corpus-wide. The applications layer had
already learned this independently (`CHARTER` rule 6, after `APP-CONVENTION-EMBED` v0.1 shipped a
fixed width), which is the argument for hoisting it to the authoring standard rather than patching
the network spec.

**Why a width is not a harmless documentation detail:** a stated width is read as a validation
constant, and a validation constant on a variable-length value is a **silent interop split** — the
peers do not error, they disagree about what is well-formed.

## 3. The sweep failed, and the failure is the more useful half

**§8.4.5 shipped as a corpus-wide prohibition enforced by a documented grep.** core-go then read the
corpus and **found nine sites the sweep had missed.**

**The miss was arch's, and the mechanism is worth stating plainly: the grep was run piped through
`head -25` and acted on the truncated list. The full sweep returns 41 hits.** That is the *prove a
negative before you claim it* rule, failed by the party that had just written it down — in the same
change.

**Two survivors were normative and on the SHA-384 path**, and one of them decides a key:

- **`EXTENSION-ENCRYPTION` §7.3 is the one that mattered.** It is not descriptive — it **pins the
  HKDF `info`**, so it determines the derived key, and it named `0x00` and 33 bytes explicitly. **A
  SHA-384-home recipient's pubkey hash is 49 bytes, so a conformant reader of that line derives a
  different key**, and the decryption failure gives no hint why.

**The generalized lesson, which is why this arc is process-tier rather than a set of edits:** a
prohibition enforced by a discipline is exactly what fails — the same conclusion `GUIDE-CONFORMANCE`
§5.2b had reached in the same packet. §8.4.5 is now gated by `hash-width-pin` in
`entity-system-arch-tools`, with self-test invariants covering both the shapes that shipped and the
exemptions that must stay quiet (a width guarded by *"length follows the format byte"*, a worked
instance pinned to a stated floor, text arguing *against* a width).

## 4. The exemption shape, stated because it is load-bearing

The rule cannot be *"never write 33"*. Legitimate text includes a worked example under a named
format, a floor-pinned derivation, and prose explaining that the width is not fixed. The gate's
discriminator is a nearby acknowledgement that **the length follows the format byte**; the exemption
window is deliberately a few lines, since CDDL comments wrap, and deliberately not a whole section.

## 5. The second class — §8.4.6, and it is the bigger ruling `[added by review, 2026-08-15]`

**`entity-core-go` filed a second spec-issue the same day, on top of this arc's own fix:** *"the width
class has a second class underneath it: **which format** a derived hash uses"*
(`docs/validation/spec-issues/2026-08-10-b-which-format-a-derived-hash-uses-is-unruled.md`). Their
framing is why it belongs in this document rather than its own:

> *"The width rule and the format rule are two halves of one thing: **a hash has no width, and a
> derived hash has no automatic format.**"*

**And this arc's fix sharpened the ambiguity it exposed.** §8.4.5 said never pin a *width*, so the
corrected sections came out reading *"66 under `00`, 98 under `01`"* — **which reads as *follows the
home format* without ever saying so.** The width fix made the format question louder and answered
none of it.

**What was ruled (§8.4.6, `140a1bf`): every hash in the corpus has one of two dispositions, and a
specification introducing one MUST say which.**

| Disposition | Rule | Because |
|---|---|---|
| **hold-and-fetch** | You already hold the hash. Use it verbatim at whatever width its own format byte implies. **A spec MUST NOT state its format or width** | it is not re-derived, so there is nothing to disagree about |
| **derive-to-meet** | You *compute* it from an agreed input so another party can independently compute and match. **Pinned to the ECFv1-SHA-256 floor (`0x00`); MUST NOT follow the deriving peer's home format** | two peers deriving under different home formats construct different values and **never meet, with nothing failing loudly** |

**The one-question test:** *would a second party, holding only the agreed input, have to know your
home format to reproduce this value?* If yes, it is derive-to-meet. **A home format is not
discoverable from a peer-id or a path**, so any rule requiring it is already broken.

**This is the equivalence-collapse shape again, and §8.4.6 says so:** the two dispositions produce
identical bytes on a single-format network — every network anyone has run — so both readings pass
every test and the distinction is invisible until two peers run different home formats. **Same reason
§8.4.5's width lock survived three reviews.**

### 5.1. The two rulings inside it, both narrowing

- **`{peer_id_hex}` is derive-to-meet, and the *definition* was the half that gave.** `ROLE` §1.5.1
  called it *"the `content_hash` of their `system/peer` entity"* **and** used it as a path segment
  every peer constructs for every *other* peer. Both cannot hold: home-format derivation means A and
  B compute different `{peer_id_hex}` for the same third peer C, so **role assignments, identity
  certs and quorum paths never meet.** Ruled pinned; the coincidence with the stored entity's
  `content_hash` holds only for a SHA-256-home peer and is the local collapse, not the definition.
- **`system/peer` entities are authored under the floor unconditionally** — the correction to the
  *first* half of that ruling, made in the same fold. Pinning only the segment leaves **two
  `content_hash`es for one identity**, which is the state V7 §1.8 exists to prevent. An entity whose
  every field is recoverable from a public peer-id has no author-chosen content and cannot be
  hold-and-fetch.

### 5.2. The named cost, and why it is a decision

**Pinning derive-to-meet values makes ECFv1-SHA-256 effectively mandatory for any peer participating
in role, identity, quorum, revision or signaling paths** — stronger than V7 §1.2's *"SHOULD support."*
The spec states this rather than leaving it to be re-discovered per extension, and records that the
alternative — promoting it to a `[MUST]` on peers — was **rejected**, because it would retire
`content_hash_format` negotiation as a live wire surface and make any non-floor conformance arm
illegal by construction rather than merely divergent.

**That is a significant, deliberate narrowing of the crypto-agility axis this very arc was opened to
unblock, and until now it had no proposal at all.** It is the single strongest argument in the
reconstruction for why proposal-first exists: **the width prohibition got a document and the ruling
that constrains every peer's home format did not.**

## 6. Open questions for the cohort

1. **The width class is discharged in `specs/` and the gate now says so.** `hash-width-pin` is
   configured `error` in `entity-system-arch-tools` and the full corpus run is **0 errors**
   (2026-08-15) — so the 41-hit sweep is closed *as far as a gate can see*. **Unchanged: the gate
   reads `specs/` and not `guides/`** (W-CORPUS C1), so the guides are still unaudited for this class.
2. **§8.4.6's own MUST is not gated.** *"A specification introducing a hash MUST state which
   disposition it is"* is exactly the checkable property §8.4.6 claims to create — and nothing checks
   it. `hash-width-pin` gates §8.4.5 only. **A `hash-disposition-undeclared` rule is the obvious
   follow-on, and it is ours to build in arch-tools.** Instances that state no disposition today
   include `ROLE`'s `{token_hash}` and `IDENTITY`'s cert-path `{att_hash}` — both named in go's
   suggested sweep, neither labelled.
3. **go's disposition list had four items; three are discharged in the landed text** (the general
   rule, `prefix_hash`, `{peer_id_hex}`) and **the fourth — "sweep for the rest" — is partial.**
   §8.4.6 names `QUORUM`'s `{quorum_id_hex}`/`{hash_hex}` and `IDENTITY`'s `{published_handle_hex}`
   as hold-and-fetch, which covers most of it; the two above are the remainder.
4. **`ENCRYPTION` §7.3's HKDF `info` is a derived-key input.** Any peer that shipped against the old
   text derives incompatible keys. Is that a live migration concern anywhere, or was it caught before
   any deployment?
5. **§8.4.6's "effectively mandatory" cost has never been put to the cohort.** It was ruled, and its
   rejected alternative recorded, in a fold no seat reviewed. **rust and py have not been asked
   whether pinning `system/peer` to the floor is acceptable in their trees**, and it is the one
   ruling in this arc that changes what a conformant peer must author.

## 7. Fold plan

**Already folded**, and §8.4.5 is mechanized. Ratification means **both** rules stand — §8.4.5's
width prohibition and §8.4.6's disposition MUST — along with the instance corrections. This is the
arc that most clearly justifies the gate-over-prose principle: the rule was written, swept by hand,
and still left nine survivors including one that derives an encryption key.

**§8.4.6 is the half that most needs cohort review and has had none** (§6.5). If it is rejected, the
`system/peer` floor pin and the derive-to-meet labels come back out together.

## 8. Review — the diligence pass `[2026-08-15]`

**Both axes run**, per `PLAN-2026-08-15-spec-provenance-reconstruction.md` §8.

**Sources opened:** `entity-core-go`
`docs/validation/spec-issues/2026-08-10-b-which-format-a-derived-hash-uses-is-unruled.md` (§1–§6,
including its **four**-item suggested disposition) · `docs/validation/reports/2026-08-10-sha384-home-format-cohort.md`
· `docs/status/HANDOFF-2026-08-10-encryption-v1-0-is-closed-and-sha-384-is-not-runnable.md`. Landed
text read at `SPECIFICATION-FORMAT.md` §8.4.5/§8.4.6 and the labelled instances in `ROLE`,
`REVISION`, `IDENTITY`, `QUORUM`.

### What the review changed

| # | Axis | Finding | Disposition |
|---|---|---|---|
| **C1** | A | **An entire ruling was omitted — §8.4.6 / `derive-to-meet`** — landed in `140a1bf`, one of the two commits this proposal cites, **whose subject line names it** | **New §5; title, target and scope rewritten** |
| **C2** | B | §8.4.6 is **more consequential than §8.4.5**: it makes the floor effectively mandatory for role/identity/quorum/revision/signaling participation and pins `system/peer` authoring unconditionally | **New §5.2; §6.5 opened** |
| **C3** | B | **§8.4.6's own MUST is ungated** — the disposition declaration is exactly the checkable property it claims to create, and nothing checks it | **New §6.2, an arch-tools ask** |
| **C4** | B | Open question 1 **answered**: the width class is 0-error under the `hash-width-pin` gate today | **§6.1 rewritten** |

**The mechanism of C1 is the one to keep.** The reconstruction was assembled *from commit messages* —
and the commit message said *"and a second class underneath it."* **The miss was not a missing source;
it was reading the first clause of a subject line.** Every other arc's misses came from reading the
wrong document or stopping at the main finding; this one came from stopping mid-sentence in the right
one.

### What the review confirmed as correct

- **§1's numbers** — 31 failing checks, and the go/rust `400` vs py `200` split on a 98-hex
  `CONTENT_GET` — match go's own report.
- **§3's account of arch's own sweep failure is honest and complete**, including the mechanism
  (`head -25` on a grep returning 41 hits) and the concession that it failed *prove a negative* in the
  same change that wrote it down. **Left exactly as it stands.**
- **§3's `ENCRYPTION` §7.3 survivor** is correctly identified as the one that mattered: it pins the
  HKDF `info`, so it determines a derived key, and a 49-byte SHA-384-home pubkey hash makes a
  conformant reader derive a different key with no diagnostic.
- **§4's exemption shape** matches the gate's implemented discriminator.

### Not covered by this pass

- **No cohort review of either rule** — and §8.4.6 has never been seen by a seat other than its filer
  (§6.5).
- **`guides/` remains unaudited for both classes** (C1 in W-CORPUS), unchanged.
