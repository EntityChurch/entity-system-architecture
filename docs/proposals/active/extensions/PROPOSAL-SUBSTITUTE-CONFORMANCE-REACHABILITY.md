# PROPOSAL — SUBSTITUTE: a conformance section that required what nothing could reach

**Status:** **DRAFT — reference proposal, written after the fold. See §0.** Folded as v1.1 and v1.2.
**Target:** `EXTENSION-SUBSTITUTE.md` §5.2b.1 discipline, new §9.1, §6 (E2 refusal code), §10.
**Tier:** `extensions/` — validated by go · rust · py.
**Scope:** what SUBSTITUTE's §9 conformance list may require, and the refusal contract at §6.
**Source:** `entity-core-go`, 2026-08-13/14, on writing **the first behavioural checks this extension
has ever had in any implementation.**
**Cohort review:** ruled in-cycle; both halves now built (§4).

---

## 0. Process deviation — recorded, not hidden

**Folded across two commits (`f955200`, `77fb392`) with no proposal**, including a new normative
subsection and a `[MUST]` on vector assertions. Written now per the reconstruction ledger (arc F).
Not back-dated. **Diligence pass run 2026-08-15 — see §7. It found a third filed item (E3) that
neither the fold nor the first pass carried**; it is now §5.3 and is excluded from ratification.

## 1. The gap — an extension that read as covered and asserted nothing

**`EXTENSION-SUBSTITUTE` was built in all three implementations and had ZERO behavioural checks in
any of them, for its entire life.** It read as covered on every board.

§9 listed roughly **sixteen `TV-SS-*` vectors as "required for v1"**, and every one exercises the §3
chain — **which no conformance client can enter over the wire in any implementation, by design**:
`claimed_source_peer_id` is local dispatcher context and explicitly not a wire field, and §10 already
recorded that deferral. All three impls deferred the driver and left a comment saying so.

**So a conformance section required what nothing could reach, and the absence of coverage was
invisible because the requirement existed.** This is `GUIDE-CONFORMANCE` §5.2b's shape — a surface
the suite cannot reach — in its most complete form: not one unreachable check, but an entire
extension.

## 2. v1.1 — §9.1 rules them in-process, with the discipline attached

New **§9.1** rules the §3 vectors **`in-process` for v1** under the **§5.2b.1 declared-exclusion
discipline**: satisfied-by, plus **an executed mutation**.

**The mutation requirement is the whole point** — a declaration without one is what lets a category
read as covered while asserting nothing, which is the failure this section is fixing. The wire driver
is **named as owed** and is explicitly **not this extension's to build**.

## 3. v1.2 — a code pinned one day, asserted by nothing the next

**E2 pinned `400 wrong_substitute_type`. The reference check gated on `status != 400` and never read
the code — so the pinned value was a value nothing asserts**, one day old. `GUIDE-CONFORMANCE`
§5.2b.2's shape exactly, and the third member of the family §6a.9 opened.

**It was hiding a live divergence.** Source-read 2026-08-14: `entity-core-py` `14775ce` **refuses
correctly and before any fetch** — their test asserts the fetcher was never called, which is the
load-bearing half — **and answered `invalid_entry`**, the generic code they use for three different
refusals. **Python's behaviour conformed and its code did not, and no instrument could see that.**

v1.2 **MUSTs the vector read the code.** `400` is shared by every malformed-entry refusal on this
handler, so a status-only assertion cannot distinguish *refused for the right reason* from *refused
for any reason*.

## 4. Build state — closed both ways

**Verified at source 2026-08-15, in both trees rather than taken from either report:**

- **go asserts the code** — `cmd/internal/validate/substitute.go`, whose declaration says *"asserts
  the CODE, not merely the 400"*.
- **py answers the code** — `substitute/http.py`, at `bd252b0`.

**This row is closed.** Worth keeping from py's own note: their source comment still described the
refusal as *an unpinned question answered in the safe direction*, written the day before the ruling
landed the same way — **a stale comment framing a now-pinned MUST as local preference, sitting in the
file next to the code.**

## 5. Open questions for the cohort

1. **rust owes its in-process declaration** for the §3 vectors (§9.1), and the executed mutation with
   it. Ruled for all three; declared by one.
2. **The wire driver for §3 is still owed and unowned.** §9.1 makes the gap honest; it does not close
   it. Until a driver exists, SUBSTITUTE's §3 chain is exercised in-process only.
3. **E3 — §6 and §9 contradict each other about whether the http convention is optional. Filed
   2026-08-13, never ruled, and the first pass did not record it. `[found by review, 2026-08-15]`**

   go's spec-issue is titled *"the vectors have no driver, and **two** unpinned rows"* — E2 **and
   E3**. This proposal folded E1 and E2 and **omitted E3 entirely**; what stood here previously was
   the adjacent *fact* (rust's crate is unwired) framed as **rust's posture decision**, when go's
   actual ask was **for arch to resolve a contradiction in its own spec.**

   **Both halves are still in the landed text, unreconciled:**

   - **§6** — installing a convention makes its handler discoverable, uninstalling makes it
     unavailable, and an entry whose type has no installed handler *"yields `not_found` and the chain
     advances."* **A peer with no http convention is a normal, supported deployment.**
   - **§9** — *"Required for v1 (cross-impl convergent): … §7 HTTP convention Mechanism A."*

   **`entity-core-rust` sits exactly in the gap** — the `storage-substitute-http` crate exists and
   the default peer does not depend on it, so it answers `404 handler_not_found`. **Conformant under
   §6, non-conformant under §9**, and go's suite reports a category skip, **which under ADR-0012
   counts as a failure.** A seat is being scored against a contradiction.

   **The ask, in go's words:** does §9's *"required for v1"* bind the **implementation** (the crate
   exists — rust satisfies it) or the **default peer** (it does not)? go reads it as the former and
   says the resulting ask to rust is *"please install it on the default peer so the surface is
   measurable,"* **not** *"you have a bug."*

   **RULED, AND IT WAS ALREADY RULED WHEN THIS SECTION WAS WRITTEN.** `EXTENSION-SUBSTITUTE` **§9.2**
   answers go's question in go's own terms: *"required for v1"* binds the **implementation** (the
   module exists, is conformant, and is installable); **a deployment MAY omit it**, leaving §6's
   uninstalled-handler path conformant; and **a peer offered for conformance measurement MUST have it
   installed**, because otherwise the category reports a skip and a skip counts as a failure
   (ADR-0012) — so the suite would be scoring a deployment choice as a defect. It closes with the
   correct ask to the affected seat being *"install it on the peer under test,"* **never** *"you have
   a bug"* — which is the sentence this section asks for, already in the landed text.

   **Correction, and it is this arc's second instance of its own failure mode.** The section above was
   written asserting an unfolded state **without opening §9**. §9.2 landed at arch `f955200`, a
   verified ancestor of the review commit that declared it unfolded. The review was created to catch
   claims made from summaries rather than sources, and it made one — about *this repo's own spec*,
   which is the cheapest artifact in the corpus to open. **A claim that something is unruled is a
   claim about a document, and it is checked by reading that document.**

## 6. Fold plan

**Already folded** (v1.0 → v1.2). Ratification means §9.1's in-process ruling and the code-assertion
MUST stand. **This arc is the canonical instance of the oracle-coverage axis** — built in all three,
asserted by none — and it is why `WORKSTREAMS.md` grew that column at all.

**E3 is folded too** — §9.2, landed before this proposal's review pass ran and missed by it (§5.3).
It ratifies with the rest. **Nothing in this arc is outstanding against arch**; what remains is that
the affected seat installs the convention on the peer it offers for measurement, which §9.2 states is
an ask about the peer under test and not a defect report.

## 7. Review — the diligence pass `[2026-08-15]`

**Both axes run**, per `PLAN-2026-08-15-spec-provenance-reconstruction.md` §8.

**Source opened:** `entity-core-go`
`docs/validation/spec-issues/2026-08-13-e-substitute-has-no-driver-and-two-unpinned-rows.md` —
**§0 + E1 + E2 + E3, counted.** Build state read live in go's and py's trees.

### What the review changed

| # | Axis | Finding | Disposition |
|---|---|---|---|
| **F1** | A | **E3 was filed and is missing from the proposal.** The document's own title says *two* unpinned rows; the first pass folded one. What stood in its place recast arch's spec-contradiction as **rust's posture decision** | **New §5.3; excluded from ratification** |

**The item-count check is what caught it, and it is the cheapest check in the pass** — go's title
states the number. `REVISION` v3.11 ruled two of seven; this folded one of two. **Same failure, and
both times the filing document announced its own count in the first line.**

### What the review confirmed as correct

- **§1's E1 account is exact** — sixteen `TV-SS-*` vectors required for v1, all exercising the §3
  chain, which no conformance client can enter because `claimed_source_peer_id` is local dispatcher
  context and not a wire field. go's document says the same, and §10's deferral is real.
- **§3's v1.2 account is exact**, including the live divergence it was hiding: py refused correctly
  and *before any fetch* while answering a generic code.
- **§4's build state, re-verified at source rather than carried from either report:** go's
  `substitute.go` declares `wrong_substitute_type_refused` with *"Asserts the CODE, not merely the
  400"* in the check's own declaration; py's `substitute/http.py` returns `wrong_substitute_type` at
  the type-mismatch branch, with `invalid_entry` retained for the other refusals — **which is exactly
  the split the ruling asked for.** Row closed, correctly.

### One corroboration worth carrying to arc B

go's **§0** — out of scope here, since it is their own non-conformance and not a spec issue — records
that their new check *"was written the wrong way round … it **FAILed `entity-core-py` for the
conformant behaviour and PASSed go for the non-conformant one**."*

**That is a second, independent instance of the inversion `PROPOSAL-CONFORMANCE-COVERAGE-FAILURE-
TAXONOMY` §3 was corrected to describe** — a positive-only or wrongly-oriented check scoring the
correct peer as the defect, with the same seat on the receiving end both times. **Two instances in
two days makes it the family's most common outcome, not an edge case**, and it is worth stating that
way in §2.4a rather than as a consequence.

### Not covered by this pass

- **rust's in-process declaration for §9.1** (§5.1) is still owed and was not chased.
- **The wire driver for §3** (§5.2) remains owed and unowned — unchanged.
