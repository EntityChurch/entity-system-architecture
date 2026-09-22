# PROPOSAL — REGISTRY peer-issued registration: authorization, the 202 carrier, and manual approval

**Status:** **DRAFT — reference proposal, written after the fold. See §0.** All rulings below are
already in `specs/extensions/EXTENSION-REGISTRY.md` (v1.2 → v1.5).
**Target:** `EXTENSION-REGISTRY.md` §6a.9, §6a.9.1, §6a.9.2, §6a.9.3, §6.5, §3b.3.1.
**Tier:** `extensions/` — validated by go · rust · py.
**Scope:** the peer-issued live-registration lifecycle — who may write, what the result type is, and
what happens when a registrar queues rather than decides. **Not** the service-advertisement half
(§3b), which has its own proposal.
**Source:** the 2026-08-11…14 cohort round, driven throughout by implementations building the
surface. Six of the eight rulings were routed by a seat, not found here.
**Cohort review:** each ruling was answered in-cycle against the packet that produced it. **Not
reviewed as a package**, and §7 lists what that leaves open.

---

## 0. Process deviation — recorded, not hidden

**This arc landed across seven commits with no proposal.** It includes new `[MUST]`s on
authorization, a changed result type, and a new normative subsection (§6a.9.3), which is not
wording-only hygiene.

Written now as the design record the folds should have had, per the reconstruction ledger
(`docs/status/PLAN-2026-08-15-spec-provenance-reconstruction.md`, arc E). **Not back-dated, not prior
authorization** — if review rejects a shape, that spec edit comes back out.

**First-pass reconstruction, now reviewed.** The body was assembled from the landed spec text and the
commit record. **The diligence pass against each peer's own document ran 2026-08-15 and found four
attribution defects and two open design items — see §10.** Corrections are folded into the body; §10
records what was checked and against which document.

## 1. The arc, in the order it happened

**Attribution corrected 2026-08-15 against each filing seat's own document — see §10.**

| # | Ruling | Routed by | Landed |
|---|---|---|---|
| 1 | **Authorization** — all three write ops bind proof; operator authority is local-only | **py** found the unsigned-`revoke`/`renew` hole (08-10); **go** filed the separate *operator-proof-shape* ambiguity and supplied the interim reading (08-11) | `4dd07f5` |
| 2 | **`401 signature_invalid`** — a check of a pinned row MUST assert the *code*, not the status | **go** (§2.4a audit, 08-11) | `58d7e35` |
| 3 | **Status-only rows are corpus-wide**, not three named vectors | **go** (R-5's audit — it "found more than we named") | `bf88be0` |
| 4 | **Q-3 — `register-result`**, an undeclared third outcome | **go** (spec-issue, 08-12) | `81e73ae` |
| 5 | **§6a.9.3 manual approval** — so `pending_hash` names something (v1.3) | **arch**, from the three-way split (rust withheld, py built, go's oracle failed peers) | `f162953` |
| 6 | **Four gaps the builds found** (v1.4) — three from go, the fourth from rust | **go** I1–I3 · **rust** R4 (independently recorded by **py**) | `4d664a3` |
| 7 | **§3b.3.1 vector retracted** — arch does not author fixtures | **browser-rust** | `57c1ab6` |

## 2. Authorization — the hole two of three shipped

**`entity-core-go` and `entity-core-rust` shipped `revoke`/`renew` with no verification at all.** Any
peer able to reach either registry could permanently revoke any binding in it, and revocation is
monotonic — there is no undo.

**`entity-core-py` did not, and it is the seat that found it.** py enforced the layer-1 proof on both
ops, was therefore *failed* by the shared conformance checks (which sent unsigned requests and
asserted `200`/`202`), declined to match, and filed it — `entity-core-py`
`docs/status/HANDOFF-2026-08-10-catchup-inbox-cut-6a92-and-the-unsigned-revoke-python.md` §5,
2026-08-10. Verified at source, not from the report: `_handle_revoke_request` / `_handle_renew_request`
both gate on `_verify_proof_by(ctx, req_hash, target_peer_id)` at py `4cdf805` (2026-08-10), the
commit preceding the filing.

> **Correction, recorded because it is this reconstruction's own defect class.** The first pass wrote
> *"all three implementations shipped `revoke`/`renew` with no verification at all."* That sentence was
> inherited verbatim from `entity-core-go`'s spec-issue, which asserts *"Go, rust and py **all three**
> shipped … with no verification at all"* and then, five lines later, *"core-py declined to match,
> reported it, and that is the only reason it was found."* **The filing seat's own document was
> internally contradictory, and arch copied the wrong half** — and dropped py's credit, which go itself
> had given. **A seat's document is authoritative about that seat, not about its peers**; the third
> party's own tree is. That is the diligence rule one turn finer than "read the filing seat's document."

**§6a.9's operator proof is local-only.** The text said "signed by `target_peer_id` **or the
operator**" while naming an act the corpus gives no operator identity, key, inbound capability, or
request field. The operator's authority is defined *entirely* as a **local capability over a local
act** (§6a.9.1's `registry-issue-binding`), so the **wire surface accepts `target_peer_id` proof
exclusively.** Adopted from core-go's interim reading unchanged, on the asymmetry: *a registry that
widens later un-accepts nothing, while one that guessed wide has already handed out permanent
denial-of-name.*

**The sentence that explains how it survived three reviews: replay defence is not authorization.**
`renew`'s nonce stops a *captured* request while leaving a *fresh unsigned* one accepted.

**Why it was exactly these two operations.** §6a.9 named a proof vector for `register` and none for
`revoke`/`renew`, and **all three impls skipped layer 1 on exactly the two ops with no named
vector.** The vector list, not the prose, is what gets implemented against — which is the root this
arc shares with the conformance taxonomy (`PROPOSAL-CONFORMANCE-COVERAGE-FAILURE-TAXONOMY`).

**Generalized out of this arc, and folded there rather than here:** every check of a guarded
operation MUST assert the negative half (`GUIDE-CONFORMANCE` §2.4a). Recorded because this arc is
*where that rule came from*.

## 3. The code is the contract

**`401 signature_invalid` pinned; a check of a pinned row MUST assert the `code`, not the status
alone.** Live at the time: go and rust answered `signature_invalid` (rust having converged from
`invalid_signature`), **py answered `proof_failed`** at all three layer-1 sites.

**It survived a full cohort cycle because the instrument asserted layer 2 as
`status != 403 || code != not_entitled` and every layer-1 row as `status != 401`** — same file, two
rows apart, one checking the contract and the other half of it.

**A retraction belongs here, because the ratification rationale was wrong.** An earlier entry read
that the statuses were *"pinned as rust and py converged to them"* and that *"py's deliberate
divergence is conformant by rule."* **Neither was verified against py's tree.** The cohort converged
on the **status**; the **code** diverged. The causal error is worth keeping: **py's divergence was
read as a *string* divergence (conformant, by our own new rule) when it is a *code* divergence
(non-conformant) — the rule written to prevent drift is what made the drift look sanctioned.**

**§3's scope was then found too narrow (`bf88be0`).** **`entity-core-go`'s own R-5 audit** — *"found
more than we named"* — turned up two further pinned rows asserting status-only, §6a.9
`202 pending_review` and §6.5 `409 bind_already_exists`, so the code-assertion MUST is scoped to
**every row of every pinned status table**, not the three named vectors.

## 4. Q-3 — the result type that could not express its own outcome

**`register-request`'s signature read `→ binding_hash | rejection` — two outcomes — while step 5
introduced a third in the pseudocode without extending the return type**, and `pending_hash`
appeared nowhere in the specification at all.

**So three implementations invented three carriers, and not one of them was wrong, because there was
nothing to be wrong against.**

**Ruled: `system/registry/register-result` `{status, binding_hash?, pending_hash?}` on both 200 and
202.**

- **`system/protocol/error` MUST NOT carry the 202.** An error entity denotes a *failed* operation;
  `202` denotes accepted-pending. A client branching on result *type* reaches the opposite conclusion
  from one branching on *status*.
- **`system/protocol/status` rejected on structure, not taste** — the 200 must carry `binding_hash`,
  the 202 a poll handle; a carrier with room for neither pushes the payload one field down and
  re-opens the divergence.
- **Borrowing another operation's result type is prohibited** — payload-shape coincidence is not a
  type relationship.
- **Adopted on merits and explicitly not on the count.** rust's hold was the correct posture and the
  delay was arch's.

**The proximate cause was the operations table's `Code` column** — a category error on a 2xx row,
contradicted by §6a.9's own pseudocode, and **an implementation that trusted the table emitted an
error entity on a 2xx, faithfully.** The table now carries a fourth column naming what *carries* each
value: a status table that names a value without naming its carrier is under-specified by exactly the
column a wire implementer needs.

## 5. §6a.9.3 — manual approval, so the MUST names something (v1.3)

The 08-12 ruling made `pending_hash` a MUST pointing at `system/registry/pending-binding`, whose
schema was **reserved to §6a.9.3 and never written.** The three seats split: rust withheld the value
on principle, py invented the entity plus an `approve-request` op, and go's oracle **failed a peer
for returning no `pending_hash`.**

**The spec manufactured a conformance failure out of its own reserved section.** Rule recorded at
§6a.9 and now mechanized as `enum-value-not-declared`/`undefined-wire-referent` in arch-tools:
**a MUST may not name a referent the corpus does not define; if the referent waits, the MUST waits
with it.**

Ruled:

- **Body + by-request pointer**, following §6.3's existing body/pointer split rather than inventing a
  second pattern.
- **Pointer is `by-request/{target_peer_id}/{name}` — peer FIRST, normative**, because `name` is not
  guaranteed single-segment and a variable-depth value in a non-terminal position makes the path
  unwalkable. *(The identical defect was corrected in `EXTENSION-NETWORK` §4.1 the same day — worth
  noting as a shape, not a coincidence.)*
- **`pending_hash` is registry-local and MUST NOT be gated on cross-peer reproducibility.** Stated
  explicitly **because the corpus's other content-hash rulings run the opposite way**, and an
  implementer generalizing from §2.4 would strip `queued_at` chasing a determinism this surface does
  not need.
- **One pending head per `(target_peer_id, name)`; a new request supersedes.** Retries carry fresh
  nonces, so without this an operator queue fills with duplicates of one intent.
- **`deny` is not a delete** — a denied head stays reachable.

## 6. The four gaps the builds found (v1.4)

**Ruled 08-13; built by all three inside twenty-four hours** — go `7e0fb7c`, rust `4107c32`, py
`808d9e6`, with go's oracle **27/27, 0F against each.** Three ground-up codebases converging on one
ruling's shape is the strongest validation this method produces. **On three of the four, all three
seats reached the same reading before anything was ruled** — the signal that the text was
under-determined rather than misread. Ratified as built, not re-litigated.

- **I1 — `deny-request` returns `"denied"`** and the result type enumerated two values; now three.
  **The finding is the recurrence, not the fix:** §6a.9's own correction box stated the general rule
  and **the very next subsection reproduced it**, because the rule was written as prose beside one
  instance instead of as a check over the corpus. Routed to the gate.
- **I2 — the decision ops stay un-typed; handlers MUST decode by shape.** No impl may register
  `system/registry/approve-request`/`deny-request`: publishing a definition for a name the spec does
  not carry manufactures a type-census divergence out of a spec gap and makes it the publisher's.
  **The narrow exception, stated as such** — inventing two names now pins a wire surface no operator
  tooling consumes.
- **I3 — a decision on a superseded head answers `404 not_found`.** Approving one would issue a
  binding on terms the operator's queue no longer shows. *"One pending head per pair"* is a rule
  about what is **decidable**, not only about what is listed.
- **R4 — supersession observability. This one is `entity-core-rust`'s, not go's**, and the first pass
  numbered it "I4" into go's sequence, which credited the wrong seat. go's spec-issue carries exactly
  three items (its own title says so); rust's ask 4 is the fourth, and `entity-core-py` recorded the
  same observation independently while building. **`pending-binding` carries no nonce**, so two
  retries of one intent on identical terms inside a millisecond content-address to **one body**. That
  is correct — one head, one hash — but it makes supersession unobservable, so it becomes a
  conformance obligation rather than a spec change: **the supersession vectors MUST vary a field the
  schema carries, never the nonce, and no implementation may use "the `pending_hash` changed" as its
  supersession signal.** As py put it, `pending_hash` identifies *(pair, terms, millisecond)*, not a
  request.

## 7. §3b.3.1 — arch does not author conformance vectors

§3b.3.1 shipped a literal 33-byte `k`, two endpoint strings and two SHA-256 digests, presented as
*"the oracle in the spec."* **`entity-browser-rust` recomputed it on receipt and filed two defects.
Both are the lesser problem.**

> **Corrected 2026-08-15 against browser-rust's own routing.** The first pass — and arch's `57c1ab6`
> commit message before it — said browser-rust *"caught a wrong byte."* **browser-rust says the
> opposite about the digests:** it recomputed both SHA-256 values and the winner independently and
> found all three *"correct as published … Not a transcription slip — the digests are right."* The two
> defects it actually filed are: **(A)** the section claimed the winner is *not* the
> lexicographically-first endpoint, and in the published vector it **is** (`.` = `0x2E` sorts before
> `2` = `0x32`), so an implementation sorting by endpoint ascending **passes the vector that exists to
> catch it**; and **(B)** the 33-byte `k` opens with `01`, a **non-floor `content_hash_format` tag**,
> where `EXTENSION-SIGNALING` §3.1 pins a rendezvous key to the SHA-256 floor `0x00` precisely so two
> peers cannot derive different keys. Both live in the same key, and browser-rust notes one byte fixes
> both. **Defect A is the same shape one level down — a vector that cannot discriminate the
> construction it targets** — which is the argument the retraction rests on, so getting the finding
> right strengthens the ruling rather than reopening it.

**Specs pin rules; vectors are generated and pinned by the conformance oracle** and cited
`N·0F @ <oracle-commit>` (ADR-0012). **A digest typed into prose cannot be checked by reading it** —
no reviewer can verify it, which is precisely how a wrong one survives to be banked as a false green
by three implementations at once. **The failure mode that caught this is the argument against having
done it.**

Rewritten to state the *properties* the vector MUST have and author none of its bytes: two members in
one priority tier; endpoints chosen so a wrong operand selects a **different** member (a vector every
candidate operand passes tests nothing); a winner that is not the lexicographically-first endpoint;
and a member outside the lowest tier present. **The vector itself is owed from the oracle.**

browser-rust's underlying point drove the rewrite and is untouched: **a fixture authored by one
implementation cannot adjudicate its own operand.**

## 8. Open questions for the cohort

1. **§6a.9's local-only rationale is falsified by §6a.9.3 as built — and the ruling still looks right.
   `[raised by this review, 2026-08-15]`** The first pass asked *"if an operator-initiated wire path is
   ever wanted … is anyone building toward that?"* **One already exists and is conformance-green.**
   §6a.9 narrows `revoke`/`renew` to `target_peer_id` proof **on the ground that** the operator's
   authority "is defined entirely as a **local** capability over a **local** act —
   `system/capability/registry-issue-binding`, §6a.9.1." Two days later §6a.9.3 gated
   `approve-request`/`deny-request` on **that same capability**, and those are **inbound handler ops
   driven remotely**:

   - go's `handleDecision` takes a `*handler.Request` and gates on `types.CapRegistryIssueBinding`
     (`ext/registry/peerissued/register.go`, `entity-core-go` `2df96f8`, source-read 2026-08-15).
   - go's own oracle drives it over the wire — `issuerDispatchFull(ctx, client, uri, OpApproveRequest,
     …)` in `pending_approve_issues_and_leaves_head` (`cmd/internal/validate/registry_issuer.go`).
   - py removed its self-origin floor for exactly this reason and wrote down the rule: *"gating on
     origin means the operator must BE the registry process, so a capability whose whole purpose is
     handing operator authority to someone else could never be exercised by them … **Origin is not
     authority**"* (`entity-core-py` `808d9e6`).

   So `registry-issue-binding` is, as built and as ratified, a **delegable capability exercisable
   across the wire**. **The narrowing is still the right call** — `revoke` is unrecoverable and
   capability-gated approval is not the same surface as unauthenticated revocation — **but the
   rationale as written is now false**, and a false rationale is what the next implementer
   generalizes from. This is the local-collapse shape `AGENTS.md` names: the rationale assumes
   *operator == registry process*, and §6a.9.3 is the seam where that springs apart. **Owed: rewrite
   §6a.9's justification to rest on the op's irreversibility rather than on the capability being
   local, without touching the ruling.**

1a. **Consequently — is `registry-issue-binding` one capability or two?** It now gates both a local
   sign-and-publish act and a remotely-exercisable decision op. §6a.9.3 says "no new capability:
   approving a queued request *is* issuing a binding," which is sound; the question is whether an
   operator delegating *approval* can be prevented from also delegating *issuance*. Today it cannot.
2. **I2's un-typed decision ops are an explicit exception** to the corpus's own type-declaration
   discipline. It should be revisited the moment operator tooling wants to consume them.
3. **§3b.3.1's vector is owed from the oracle** and does not exist. Until it lands, §3b.3's
   byte-pinning is stated and unexercised.
4. **§3b.3.1 is missing a fifth property, and browser-rust's Defect B is therefore *not* discharged.
   `[raised by this review, 2026-08-15]`** The retraction removed the malformed `k` by removing all
   bytes, but the four properties the section now pins (pool size, operand discrimination, weight-vs-
   endpoint order, tier rule) **say nothing about the form of the rendezvous key**. The oracle can
   generate a vector with a non-floor key and reproduce the exact defect. **Owed: a fifth MUST — the
   vector's rendezvous key MUST be well-formed per `EXTENSION-SIGNALING` §3.1, `varint(format) ‖
   digest` with `content_hash_format` at the SHA-256 floor `0x00`.** Defect A *is* discharged: it was
   a false property claim about published bytes, and property 3 now states it as an obligation on the
   oracle instead.
5. ~~This document has not been read against each peer's routing docs.~~ **Done 2026-08-15 — §10.**
   Four attribution defects were found and corrected in place; the likely-defect-class prediction in
   §0 was right.

## 9. Fold plan

**Already folded** (v1.2 → v1.5). Ratification means the rulings stand and the reasoning is on disk
under review. §3b's service advertisement is `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT`
(`implemented/`); the conformance rules this arc generated are
`PROPOSAL-CONFORMANCE-COVERAGE-FAILURE-TAXONOMY` (`active/process/`). **Those two plus this one cover
the arc; none of the three subsumes another.**

## 10. Review — the diligence pass `[2026-08-15]`

**Both axes run.** Scope: `PLAN-2026-08-15-spec-provenance-reconstruction.md` §8 +
`HANDOFF-2026-08-15-c`. **Nothing here was taken from a summary, a report, or an arch commit
message** — every claim below was read in the filing seat's own document, and every build-state claim
in a live worktree.

**Sources opened (peer documents, not summaries):**

| Seat | Document |
|---|---|
| `entity-core-py` | `docs/status/HANDOFF-2026-08-10-catchup-inbox-cut-6a92-and-the-unsigned-revoke-python.md` §5 · commit message `808d9e6` · `packages/entity-handlers/src/entity_handlers/registry.py` @ `4cdf805`, `9ac97bb`, `f33526f` |
| `entity-core-go` | `docs/validation/spec-issues/2026-08-11-revoke-names-an-operator-with-no-proof-shape.md` · `.../2026-08-13-d-6a-9-3-three-gaps-found-by-building-it.md` · `docs/validation/reports/2026-08-11-2-4a-audit-*.md` · `.../2026-08-15-cohort-status-all-three-peers-green.md` · `ext/registry/peerissued/register.go` · `cmd/internal/validate/registry_issuer.go` |
| `entity-core-rust` | `docs/status/ROUTING-2026-08-11-b-*.md` · `docs/status/ROUTING-2026-08-13-q-*.md` (asks 1–4) |
| `entity-browser-rust` | `docs/status/ROUTING-2026-08-15-the-3b31-oracle-does-not-discriminate-and-its-k-is-not-a-key.md` |

**Build state re-pinned 2026-08-15** (D1/D8 — the first-pass pins were read the same day and expire
when a HEAD moves): go `2df96f8`, rust `462f2c2`, py `f33526f`, browser-rust `c5f89b5`, all clean.
**go moved from the `d0ed41f` this arc's pins cite; the move is docs-only** (its own cohort-status
report, two files, no source) — verified by diff, so the source-level claims here survive the move
rather than being carried across it.

### What the review changed

| # | Axis | Finding | Disposition |
|---|---|---|---|
| **R1** | A | §2's *"all three implementations shipped `revoke`/`renew` with no verification"* is **false** — **two of three**. py enforced layer-1 (`_verify_proof_by`, py `4cdf805`, source-verified) | **§2 rewritten** |
| **R2** | A | The hole was **found and filed by py** (08-10), a day before go's spec-issue; the proposal credited the arc to go. **go itself credited py; arch dropped it** | **§1 + §2 corrected** |
| **R3** | A | The fourth v1.4 item is **rust's ask 4**, not a go "I4" — go's spec-issue has exactly three items. py recorded the same point independently | **§1 + §6 corrected** |
| **R4** | A | §7's *"browser-rust caught a wrong byte"* misstates their finding — **the digests are right, by their own recomputation**; they filed a false property claim (A) and a non-floor key tag (B) | **§7 rewritten** |
| **R5** | B | **browser-rust's Defect B is not discharged.** §3b.3.1's four properties do not constrain the key's form; the oracle can regenerate it | **New open item §8.4** |
| **R6** | B | **§6a.9's local-only rationale is falsified by §6a.9.3 as built.** The same capability gates remotely-driven inbound decision ops, conformance-green. The *ruling* survives; the *justification* does not | **§8.1 replaced; §8.1a opened** |

### What the review confirmed as correct

- **§3 stands as written.** py did answer `proof_failed` at **all three** layer-1 sites on the ruling
  date — verified at py `9ac97bb` (2026-08-11), where all three sites return `401 proof_failed`. An
  earlier snapshot (`4cdf805`) has revoke/renew at `403 not_entitled`, which would have made the
  sentence wrong; it is right for the date it describes. **Checked before it was reported, which is
  the discipline that stopped this from becoming a seventh finding.**
- **§6's *"on three of the four, all three seats reached the same reading"* is accurate** — go
  (I1–I3), rust (asks 1–3) and py (`808d9e6`) each reached the `denied`-undeclared, untyped-decision-op
  and superseded-head-404 readings independently, before anything was ruled.
- **§4, §5 and the §7 retraction rationale are unchanged.** No defect found.

### Not covered by this pass

- **No cohort review of this document as a package** (§0's reversibility claim remains untested).
  R5 and R6 are the two items that need a seat's answer, and both are new since the first pass.
- **§6a.9.3's retention `[SHOULD]`** was not read against any peer's implementation.
