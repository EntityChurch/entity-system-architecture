# PROPOSAL — the conformance oracle is a specified thing, not a program

**Status:** DRAFT (2026-08-13) · **amended 2026-09-08** (§5a — more than one oracle gets built) ·
**amended 2026-09-11** (§3(h) fixture posture · **§3(i) a cross-peer check must not fix the language
of the other side** · §4 the six absent verdict fields · §5a's builder assignment superseded, a
dedicated seat · §5b the requirement-keyed consumer seam · **§5c two is a floor, not a target** ·
§6 · §7)
**Target:** a new `specs/SPEC-CONFORMANCE-ORACLE.md v1.0` (this repo) · consequential
rewrites in `entity-core-protocol/specs/ENTITY-CORE-PROTOCOL.md` (five normative
citations) · `GUIDE-CONFORMANCE.md` §3.2, §8 (this repo).
**Provenance:** `docs/status/STATUS-2026-08-13-b-conformance-tooling-ownership.md`,
Findings A–E.
**Scope:** specifies existing machinery. **No wire change, no protocol-behaviour change,
no vector change, and no change to what any peer must do to conform.** It changes *who
says* what conformance is.
**Sibling of:** `PROPOSAL-DEVERSION-TEST-VECTOR-CORPUS.md`. Independent; either can land
first.

---

## 0. The problem, stated once

**`validate-peer` decides whether any peer in this ecosystem conforms, and it is a program
in `entity-core-go`.** Referenced by 619 files in keystone, 181 in core-go, 49 in py, 36 in
rust, 30 here, 11 in formalization, 5 in core-protocol. The V7 core spec names it in
normative text — *"Gated by the validate-peer `resource_bounds` category"*,
*"`validate-peer`'s `security` and `authz` categories assert these exact (status, code)
tuples."*

So **conformance to the protocol is, in normative text, defined by the behaviour of one
implementation's binary.** `AGENTS-STANDARD` says the spec is upstream and implementations
implement rather than define it. On this surface that is inverted, and it has been for as
long as there has been a cohort.

**This is not a Go problem and the fix is not to take the oracle away from Go.** They built
it because it was needed, it works, it has found real defects, and it is why the cohort
converges at all. The defect is that **adoption never became specification** — so when
`GUIDE-CONFORMANCE` §8 correctly diagnoses that the oracle *"encodes the Go reference
peer's shape as the conformance contract"* and sets the goal of making it *"spec-ordained
and language-neutral,"* **there is no spec for it to be ordained by**, and each of the five
defects gets repaired against the judgement of the party whose judgement is the thing being
repaired.

Same shape, one layer down: the crypto-agility corpus builder's contract is real, correct,
and lives in `specs/test-vectors/v767/SEEDS.md` §4 — an appendix to a vector file that
`GUIDE-CONFORMANCE` does not reference and no gate reaches.

---

## 1. The principle

> **Arch specifies the requirements. Implementations implement them. No implementation's
> behaviour is the contract — including the reference oracle's.**

Three consequences, each of which is a MUST below:

1. Normative text names **categories and requirements**, never a binary.
2. **Anyone may implement an oracle.** A second implementation is the only real check that
   the first one is testing the spec; today one cannot be written, because the contract is
   only readable as Go.
3. **An oracle's verdict is evidence, not authority.** `SEEDS.md` already says this about
   corpora — *"an oracle produced and checked by a single implementation is that
   implementation's output with extra steps"* — and the same sentence applies to the oracle
   that scores them.

---

## 2. The normative delta: `specs/SPEC-CONFORMANCE-ORACLE.md v1.0`

A new spec here, not a guide section, for two reasons: `guides/` is user-facing how-to per
`AGENTS.md`, and — measured live at arch-tools `6bba52a` — the `standards` and `topology`
analyzers root at `specs`, so **a normative contract placed in `guides/` is ungated.** The
existing normative conformance text (V7 §9.0, `ENTITY-CBOR-ENCODING` App E) already sits in
specs; this joins it.

Section skeleton, with the load-bearing MUSTs drafted. **Everything below either generalizes
text that already exists somewhere, or promotes a rule out of a place nothing reads.**

### §1 Scope · oracle neutrality `[MUST]`

> Normative text **MUST NOT** name a particular oracle implementation as the conformance
> gate. A conformance requirement cites the **category** that exercises it
> (`the security category`), never the program that runs the category.
>
> Any party **MAY** implement a conformance oracle. A peer's conformance claim is against
> this specification and the §2 category register, **not** against a particular oracle's
> output. Where two conformant oracles disagree, the cited normative section arbitrates —
> not the older oracle, and not the one with more users.

*Generalizes: V7 §9.0's "A conformance oracle scoring a peer under `--profile core` (or
equivalent)", which is already written this way and is the only place that is.*

### §2 The category register `[normative]` — **and it is generated, not walked**

**Superseded by evidence, 2026-08-13.** This proposal called the register *"the largest
single piece of work"* and assumed each category would be walked and pinned by hand.
**That was wrong, and core-go built the counter-example in one cycle.**

`cmd/conformance-register` (core-go `73d9007`, `01c86f1`) walks the oracle's AST, reads out
every declaration — category, name, citation, and whether the check contacts the peer — and
resolves each citation against the live spec trees. **Verified independently from this seat
at core-go `01c86f1`:** `go test ./cmd/conformance-register/` passes (14 tests), `-check`
exits 0 at 1165 checks / 212 findings against its pinned baseline, and two `-md` runs are
byte-identical. The committed `docs/validation/CONFORMANCE-REGISTER.md` regenerates
byte-for-byte against this repo at `3bb9303`, including three documents added after it was
generated.

**This changes the shape of §2, not its content.** The register is a *generated artifact*
whose input is the oracle's own declarations; arch's job is to rule on which citation is
**correct**, not to transcribe which citation **exists**. The spec text should therefore
require the register to be generated and reproducible rather than maintained:

> The category register **MUST** be mechanically derived from the oracle's declarations and
> **MUST** be reproducible — the same oracle source and the same spec trees produce a
> byte-identical register. A register maintained by hand is a second copy of the oracle's
> declarations and will diverge from them.
>
> Every declared check **MUST** produce a register row, including checks whose declaration
> cannot be read from source; the derivation **MUST** fail rather than emit a register
> covering fewer checks than the oracle declares.
>
> A citation that does not resolve **MUST** be reported, never inferred. Where exactly one
> candidate document carries the cited section, the candidate **MUST** be reported as a
> candidate and **MUST NOT** be applied.

*The last rule earned itself immediately: of 21 single-candidate rows, one was
`origination.reference_connect`, whose citation reads "harness precondition (not a spec
vector)" — it has no normative home by construction, and the heuristic wanted to give it
one.*

**What remains genuinely arch's**, and it is the smaller half: ruling on the 212 rows that
do not resolve. Those rulings are in
`docs/status/ROUTING-2026-08-13-b-every-ghost-document-resolved.md`.

### §2a Profile membership is data, not prose `[MUST]` `[added 2026-08-13]`

> The category register **MUST** carry per-check profile membership, **including
> carve-outs**, in machine-readable form. A profile whose exclusions exist only as prose
> forces every consumer to re-derive them from a paragraph, and a consumer that does not
> re-derive them over-counts silently.

*V7 §9.0 specifies three per-check carve-outs under `--profile core` — a check routing
through an extension-specific handler or expecting an extension-specific code (ROLE §5.5
`capability_revoked`/401) is skipped, and the equivalent core-handler-routed check runs
instead. The reference oracle implements the pair correctly. Its register still reports
`authz_revoked_1` as a core-profile check resting on an extension tier, because the only
machine-readable thing available is `coreProfileCategories` — a **category** set — and the
carve-outs are a sentence in §9.0. **The register is not wrong to over-count; it has nothing
to read.** Any second implementation would over-count identically. This is the first concrete
case of the register surfacing a place where the spec's shape and the oracle's shape can only
be reconciled by hand, which is the argument for §2 being generated from data rather than
transcribed from prose.*

### §2b What the categories are `[normative]`

> Each category names: what it asserts, its normative home, and its profile membership
> (`core` / `full` / extension-scoped). A category exists in this register or it is not a
> conformance category.

*Generalizes V7 §9.0's enumerated core set (`connectivity`, `encoding`,
`universal_address_space`, `peer_canonicalization`, `format_agility`, `crypto_agility`,
`negotiation`, `multisig`, `type_system`, `handlers`, `tree_operations`, `capability`,
`authz`, `security`, `concurrency`, `resource_bounds`) plus the extension-scoped categories
that `GUIDE-CONFORMANCE` §7 already maps. **This is the largest single piece of work in the
proposal** — the register has to be written from the oracle's existing categories and each
one pinned to its normative home.*

### §3 Oracle obligations `[MUST]`

> **(a) Score against the spec, not against a reference implementation.** An oracle **MUST
> NOT** derive an expected value by executing a reference implementation and asserting
> equality with it. Expected values come from the cited normative section or from the
> conformance corpus. *(The privileged-encoder prohibition — `ecf_key_ordering` validated
> against Go's `ecf.Encode` rather than the spec.)*
>
> **(b) Assert the negative half.** Every check of a guarded operation **MUST** assert both
> that the operation succeeds when permitted and that it is refused when not, and **MUST**
> assert the refusal's `code`, not its status alone.
>
> **(c) Four verdicts, and a SKIP is not a pass.** `PASS` / `WARN` / `FAIL` / `SKIP`. A
> `SKIP` **MUST** name the reason and the surface it did not reach. For release purposes a
> skip counts as a failure (`ADR-0012`).
>
> **(d) Scope to what the peer declares.** Assertions **MUST** be scoped to the peer's
> declared conformance level and extension set. An absent optional extension is a `SKIP`,
> never a `FAIL`; the presence of an extension in any reference implementation **MUST NOT**
> make it expected of others.
>
> **(e) Fail closed on could-not-look.** An oracle that could not reach a surface **MUST
> NOT** report a pass for it. *(The three-valued discipline the spec toolkit already runs:
> "could not look" and "looked and found nothing" are different results.)*
>
> **(f) The oracle's own checks are testable.** Each check **MUST** be verifiable
> independently of the peers it scores — a check that cannot fail against a deliberately
> broken peer is not a check. Where the surface allows it, mutation-test and record the
> date the mutation was executed.
>
> **(g) A self-check is not a peer result.** An oracle **MUST** distinguish checks that
> contact the peer under test from checks that do not, **MUST** label them in the verdict
> document, and **MUST NOT** report a self-check as a peer result. A conformance claim about
> a peer is made from the peer-attributable subset.
>
> *The operative test is not "does the check take the peer client" — that criterion,
> audited against the reference oracle's 976 rows, produced 336 false positives, because a
> staged suite passes the client to a runner rather than into every check; keying on the
> enclosing runner instead produced zero, wrong in the other direction. **The test is: could
> a sibling implementation ship none of this rule and the row not move?** If so the row is
> not about that peer.*

*Added 2026-08-13 on core-go's evidence, which is why it is a MUST and not a SHOULD. The
whole `crypto_agility` category takes a peer client and discards it — four offline
known-answer tests against the oracle's own libraries on pinned fixtures, declared as
ordinary checks. **No expected value came from a reference implementation, so §3(a) is
satisfied and does not catch it.** From the v0.8.0 release until 2026-08-12 every run
against rust or py counted four rows as peer-attributable that measured Go — **inside
`--profile core`.** Verified live in the generated register at core-go `01c86f1`:
`crypto_agility` is a core-profile category with 4 checks and **0 peer-attributable**.
§3(g)'s consequence for §4: the verdict document carries the peer-attributable split, not
only per-category `P/W/F/S`.*

> **(h) An oracle declares its fixture posture, per check, as data `[MUST]` `[added 2026-09-11]`.**
> A check **MUST** declare the preconditions it assumes of the peer under test — granted scopes,
> seeded identities, installed handlers — **in machine-readable form**. A precondition stated only
> as English inside a skip message is not declared. The verdict document (§4) **MUST** record the
> posture the run was performed in, and a conformance figure **MUST NOT** be published without it.

*Added 2026-09-11 on a measurement, which is why it is a MUST. The reference oracle was run
against one peer twice — once as the ecosystem has always run it, once with the peer launched on
the specification's own bootstrap default instead of a wide debug grant. **The check set changed
size**, 778 → 717, and 54 severities moved, every one downward: 41 PASS→SKIP, 7 PASS→WARN, 6
PASS→FAIL. Under `ADR-0012` a skip counts as a failure, so that is **47 failures**. What is lost is
**core** surface — every `universal_address_space` check, every `core_register_*` check (the §6.2
five-write contract, itself a floor MUST), and most of `capability`. Re-measured independently in
the oracle's own source: the posture is referenced across **21 files** and **13 checks are written
to skip without it**, two of them in core-profile categories.*

*Three things make this §3's most consequential clause rather than a disclosure nicety.* **First,
the posture is an input that decides which checks exist**, so two runs under different postures are
not a better and a worse number — they are **not comparable measurements**, and nothing in either
report says which was which. **Second, every published conformance figure in the ecosystem was
produced in one posture and no document states it** — not because anyone was careless, but because
**the verdict document has no field for it** (§4). **Third, and this is why it binds a second
builder hardest: the posture is invisible from inside the seat that chose it.** It was found by a
seat re-running someone else's instrument, not by the seat that wrote it, and not by any review.
*An oracle that does not declare its fixture posture cannot be audited, cannot be compared to a
second oracle, and cannot have its own coverage questioned — because the coverage is what stops the
asking.*

> **(i) A cross-peer check MUST NOT fix the language of the other side `[MUST]` `[added 2026-09-11]`.**
> Where a check requires a second peer — as a dialling target, a convergence partner, or any other
> counterparty — **the peer pair is a parameter of the run, not a property of the oracle.** Any peer
> that clears the floor is a candidate for either end. The verdict document **MUST** record which
> two peers produced a cross-peer result; where the full matrix is not run, the pairs that were run
> **MUST** be declared.

*This is §3(h)'s defect one level up — an input that decides the outcome, chosen once, by one
party, and never written down — and it is the more expensive of the two. The reference oracle is
not only a scorer: for its `origination` checks it **dials a second peer**, and that peer is always
the same implementation as the oracle. **So one implementation's defect in the dialled peer scores
every other implementation in the cohort, and no run can distinguish that from a finding about the
peer under test.** The cohort has dozens of peers across dozens of languages; there is no reason for
the counterparty to be fixed, and fixing it concentrates in one seat exactly the authority this
proposal exists to distribute.*

*The failure this prevents is the one §1 names and is worth stating concretely: **a defect that
sits on the privileged side of every cross-peer check is not found, it is propagated.** Every other
implementation is adjusted until it interoperates with it, at which point the defect is the
cohort's observed behaviour — and observed behaviour is what everybody reads the specification
against. **That is how an implementation bug becomes the protocol**, and the check that should have
caught it is the mechanism that spreads it.*

*Generalizes: `GUIDE-CONFORMANCE` §2.4a (b), §5.2b/§5.2b.1 (e, f), §5.2c, the §8 S1/S3/S5
defects (a, d), and `ADR-0012` (c). §3(h) generalizes §2a one layer out — profile membership and
fixture posture are the same defect, a machine-readable contract whose exclusions live in prose —
and §3(i) generalizes both once more, to the choice of counterparty.*

### §4 The verdict document `[normative shape]`

> A conformance run emits a machine-readable verdict carrying: oracle name + commit; corpus
> name + artifact sha256; spec version; profile; **the fixture posture the run was performed in
> (§3(h))**; **the executed check-set digest**; peer identification; per-category
> `P/W/F/S` counts; **the peer-attributable split (§3(g))**; and per-check
> `(id, requirement_id, verdict, spec_ref, message)`.
>
> `ADR-0012`'s citation form (`N·0F @ <digest>` with the P/W/F/S breakdown) is
> **derived from this document**, not typed by hand.

*Closes Finding E: the anchor ships 5,000-line conformance report files whose shape is
specified nowhere. This is mostly ratifying what those files already do.*

> **Re-measured 2026-09-11, and "mostly ratifying what those files already do" was too generous —
> this is the section with the largest gap between what it specifies and what exists.** The
> reference oracle's report structure carries peer address, peer id, peers, timestamp, summary,
> checks and declared exclusions. **It carries no oracle identity, no check-set digest, no profile,
> no spec version, no requirement id and no posture.** Six of the fields above do not exist.
>
> **Two consequences, and the second is the one that generalizes.** ① The sentence *"`ADR-0012`'s
> citation form is derived from this document, not typed by hand"* **is not true today and cannot
> be** — the inputs are absent. ② **The consuming seat built the producer's contract in its own
> tree**: the executed-check-set digest and the gate that refuses to place two peers in one column
> unless both reports carry it are the anchor's own tooling, manufacturing the identity the oracle
> should emit. *That is the correct response to a missing contract and it is also the tell — when a
> consumer has to reconstruct a producer's identity to compare two of its outputs, the missing field
> is a specification defect, not a downstream inconvenience.* **§4 is therefore the cheapest
> high-value thing in this proposal to land first**, and it is a prerequisite for two oracles being
> comparable at all.

### §5 The corpus builder contract `[MUST]`

> **(a) Arch sets fields; the encoder settles bytes.** No hand-derived, hand-edited or
> hand-inspected bytes. A claim about an artifact's bytes is made by running a tool and
> quoting it.
>
> **(b) Two gates, both required.** `verify` = the artifact is what is expected.
> `check` = the source produces the artifact. **The second's absence cost two months.**
>
> **(c) Deterministic and idempotent.** Rebuilding from unchanged source **MUST** produce a
> byte-identical artifact. A builder that cannot reproduce its own output **MUST** refuse to
> write.
>
> **(d) The legacy-encoder proof stays pinned to a frozen source.**
>
> **(e) A corpus change follows the sequence:** arch edits the fields → the builder rebuilds
> → `check` green → `verify` green → arch commits the artifacts.

*This is `SEEDS.md` §4.1–§4.4 promoted verbatim out of a vector-file appendix into normative
text, generalized from one corpus to all of them. **It is already ratified; it is just
filed somewhere nothing reads.***

### §6 Ownership `[normative]`

| Layer | Owns |
|---|---|
| **Architecture** | this spec; the §2 category register; corpus field content and vector rulings; the §5 build contract; **adjudicating a disagreement between two oracles (§5a)** |
| **The cohort (any impl)** | implementations of the oracle and of the builder |
| **The reference oracle** | conformance *to this spec*, like any other implementation |
| **The dedicated conformance seat** `[2026-09-11]` | the independent check set, core and extension; its own oracle implementing this spec. **Ships no peer** — §5a |
| **Vendors (the anchor &c.)** | byte-copies; **no vendor authors canonical bytes** |

> **No single implementation's derivation is authoritative.** A value derived by one
> implementation is that implementation's output; it becomes a corpus pin when the cohort
> confirms it byte-equal. This is existing practice (Phase-1 was byte-pinned 3-of-3
> precisely for this reason) — it becomes a rule.

---

## 3. What changes in existing documents

**`entity-core-protocol/specs/ENTITY-CORE-PROTOCOL.md`** — five normative citations become
oracle-neutral. Mechanical; no requirement changes:

| Where | From | To |
|---|---|---|
| §4.9a | "Gated by the validate-peer `resource_bounds` category." | "Gated by the `resource_bounds` conformance category." |
| §5.2 | "`validate-peer`'s `security` and `authz` categories assert…" | "The `security` and `authz` conformance categories assert…" |
| §6.9a | "The validate-peer harness tests the peer-as-shipped…" | "The conformance harness tests the peer-as-shipped…" |
| §6.11 | "The validate-peer oracle's `origination` category…" | "The `origination` category…" |
| §6.13 | "Three additions to `validate-peer --profile core`…" | "Three additions to the `--profile core` category set…" |

**`GUIDE-CONFORMANCE.md`** §3.2 currently documents one of the two corpus builders and does
not mention the other (zero occurrences of `corpus-build`, `agility-vectors`, or
`crypto-agility` in the whole file). It is rewritten to point at §5 of the new spec for the
contract, and to name both builders as implementations of it.

**`GUIDE-CONFORMANCE.md` §8** stays — it is a good defect list and the remediation is real
work. What changes is its framing: **S1–S5 become non-conformances of the reference oracle
against §3 of the new spec**, rather than items on a Go to-do list with no external
referent. S3 in particular is exactly §3(a); S5 and S1 are §3(d).

---

## 4. Cost, honestly — **revised down 2026-08-13**

**The original estimate was wrong and is corrected here rather than quietly dropped.** This
section said the register was the expensive part and would have to be walked by hand. It is
generated (§2), it took core-go one cycle, and the generator is deterministic and
mutation-tested.

**What the estimate got right is that it would surface real gaps — it did, and they are
worse than expected.** 306 of 1,165 citations did not resolve, and **nothing had ever
checked**. The residue after core-go's first repair pass is 212. The gaps are not formatting:

- `format_agility` (**core profile**) — 10 checks, 10 peer-attributable, **0 resolved**.
- `handlers` (**core profile**) — 9 checks, **1 resolved**.
- `crypto_agility` (**core profile**) — 4 checks, **0 peer-attributable** (§3(g)).

> ⚠ **Two of those four rows were re-measured 2026-09-11 and neither means what it says.** The
> figures above are from 2026-08-13; the register was regenerated at a current tip and the totals
> moved from `1165 / 66 categories` to **`1239 / 70`**, so re-take before quoting any of them.
>
> **`handlers` "1 of 9 resolved" is a READABILITY limit, not an authoring gap.** Eight of the nine
> are declared inside a loop or a helper, so the static walk cannot read their names and reports
> them `<computed>`. **They do carry citations** — `V7 §4`, `V7 §6.2 N3`, `V7 §6`. Read as written,
> this row sends someone to author eight requirements that are already cited. *A derived artifact's
> "unresolved" is three states — uncited, cited-but-unresolvable, and unreadable-by-the-derivation —
> and collapsing them is the same could-not-look defect this proposal's own §3(e) prohibits, here in
> the instrument that measures the instrument.*
>
> ⭐ **`format_agility`'s unresolved rows cite a RETIRED VERSION SCHEME.** Every one reads
> `v7.66 §4.4 surface N (AGILITY-CANONICAL-1: …)`. They do not fail because the requirement is
> missing; they fail because **`v7.66` is a document version that no longer exists.** **That is the
> same defect as the `v7.75` retirement clock still sitting in the core spec, and it makes two
> instances** — a citation into an abandoned numbering scheme is indistinguishable from a genuinely
> missing normative home, and is one lookup from being neither. **Before recording a requirement as
> unwritten, check whether its citation is merely stale.**
>
> *The transferable half, and it is why this sits in the proposal rather than in a status note:
> **the register is evidence about the ORACLE, never about the SPECIFICATION.** §2 already says it
> is derived from the oracle's own declarations. These two rows are what that limit looks like when
> a reader forgets it.*
- `convergent_mirror` — 4 checks asserting a `≤ 1.5 × N` amplification bound that **has no
  normative home anywhere**, citing a proposal that exists in neither this repo nor the
  archived pre-split monorepo (searched by filename and by content).

**The remaining cost is arch ruling on those rows**, which is real but bounded and is the
part that was always ours. Everything else in this proposal is existing ratified rules being
moved somewhere they can be cited and gated.

Everything else is cheaper than it looks: §1, §3, §5 and §6 are almost entirely **existing
ratified rules being moved somewhere they can be cited and gated**. §4 mostly ratifies the
report shape keystone already emits. §3's rewrites are five sentences.

**This does not need to land before 08-21 and should not try to.** It is not on the release
path — the release ships against the oracle that exists, which is the same oracle that
would exist afterwards.

---

## 5. What this does not do

- **Does not replace, rewrite, fork, or reassign the existing reference oracle.** It stays
  where it is, owned by the seat that built it, and it remains the backstop.
- **Does not change what any peer must do to conform.** Not one requirement moves.
- ~~**Does not require a second oracle to be built.**~~ **Superseded by §5a** — the call was
  made and it is directive. §5c then settles the shape: not *a second* oracle, but a neutral
  requirement set with **as many independent implementations over it as it takes**.
- **Does not block the S1–S5 remediation** — that work proceeds, and lands against a
  written contract instead of against nothing.

---

## 5a. The call §5 deferred has been made — **more than one oracle gets built**

> **Amendment, 2026-09-08.** §5 said this proposal *"does not require a second oracle to be
> built. It makes one possible, which is the point; whether anyone builds one is a separate
> call."* **That call is now made, and it changes this proposal from permissive to
> directive.**

**The target state, in one paragraph.** The conformance check set is **specified here, in
neutral language, so that any party can implement it** — and more than one party does. The
**core** check set is built independently by the conformance anchor. The **extension** check
sets are built, extension by extension, by the generation repository, as it works through the
corpus. The existing reference oracle **stays**, is not forked and is not reassigned; it
remains the standing independent implementation, which is the failsafe that makes the others
checkable. Where a second builder finds a requirement the reference oracle does not cover,
**the gap is ported back into it** as well as being covered in the new one. A third and fourth
builder are anticipated and are not scheduled.

> ### ⛔ Amendment, 2026-09-11 — **the builder assignment above is superseded. The rest of §5a stands.**
>
> **Everything in the paragraph above survives except the two sentences that say *who builds*.**
> The call that more than one oracle gets built is unchanged and is the reason the rest of this
> section is still correct. What changes is that the second instrument is built by **a dedicated
> seat that neither implements peers nor scores the roster**, rather than by the two consuming
> seats.
>
> **Why the assignment changed — and the reason is workload and focus, not disqualification.**
> The original §5a assignment was **deliberate and defensible on its own terms**, and it came with
> a guide amendment attached: `GUIDE-CONFORMANCE` §7.0's rule that the anchor *"authors none of
> these"* was **going to be relaxed**, on the reasoning that **the reference oracle remains the
> backstop** — so an anchor-built check set would not control both the exam and the runner. It
> would be bounded by the specification, by architecture's rulings, and by an independent
> implementation that already exists and already passes. **That is a sound argument and it is not
> being overturned here.** What changed is simpler: **authoring test suites is a second discipline
> bolted onto seats whose job is generation**, and the overlap is too much to carry. Giving the
> instrument its own seat lets the anchor and the generation repository do the thing they are good
> at, and lets one seat own the contracts end to end.
>
> ⚠ **The one real defect, and it is narrower than it looks: the amendment existed only in
> conversation.** §5a was written against a §7.0 that was going to change, and **nothing on disk
> said so** — so for a month the corpus carried an assignment and a prohibition that disagreed,
> with no record that one was scheduled to move. **Both consuming seats read the disk state
> correctly and drew the only conclusion available from it**, one concluding it must not author,
> the other that it had found a contradiction. **Neither was wrong about what was written; what was
> written was incomplete.** *This repository's own standing rule is that a decision made in
> conversation is tracked on disk in executable form before any deferral — and the cost here was
> not confusion, it was that a live workstream looked blocked on a conflict when it was waiting on
> an edit nobody had written.*
>
> **The corrected assignment:**
>
> | | Builds the check set | Why |
> |---|---|---|
> | **The reference oracle's seat** | ✅ its own, as today — **permanent, not deprecated, not forked** | it is the backstop, and the backstop is what makes every other instrument checkable |
> | **The dedicated conformance seat** | ✅ the independent sets — core **and** extension | its only job; owns the contracts end to end |
> | **The conformance anchor** | ⛔ | **focus.** Generating and scoring a peer roster is a full discipline; authoring suites on top of it is a second one |
> | **The generation seat** | ⛔ | **focus**, same argument — and it reached it first about itself |
> | **Architecture** | ⛔ | it specifies the requirements and **adjudicates disagreements**; scoring is not its role |
>
> **The new seat carries one constraint the others do not, and it is structural rather than a
> matter of focus: it must never ship a peer.** The other seats are subjects of the exam because
> they build the things being examined, which is fine — they are not writing it. This seat writes
> it, so the moment it implements the protocol it is writing its own exam, and the reason for its
> existence collapses.
>
> **Three constraints bind the new seat from birth, and they are the reasoning that created it
> applied to it:**
>
> 1. **It must not also be a subject.** The moment it ships a peer it is writing its own exam.
>    It consumes peers built elsewhere; a behaviour no available peer provides is a **finding**,
>    never a peer built in-house.
> 2. **It must not be the only scorer either**, or the audit problem returns one repo over. Two
>    instruments are worth their cost only if *both* run and disagreements are adjudicated by a
>    third party. **That party is architecture, and it is designed in rather than discovered.**
> 3. **It inherits §3(h) from day one.** A second oracle that bakes its own undeclared fixture
>    posture is a second unexaminable artifact, and the ecosystem will have paid for the privilege
>    of having two.
>
> **What the two consuming seats keep is not a consolation prize and is on the critical path:**
> the declarative, requirement-keyed check format the generation seat has already built and proven
> on live checks is **the format the new seat starts from** (§5b), and the anchor keeps the roster,
> the matrix and the pin — gaining a second instrument to score against.

**Why more than one, stated as the mechanism rather than as a preference:** a single oracle
cannot distinguish *"the peer is wrong"* from *"the oracle is wrong."* Every check it runs is
scored by the same judgement that wrote it, so its own errors are invisible **by
construction** — which is the argument §1 already makes about corpora, applied one layer up to
the thing that scores them. **A second implementation is the first real measurement of the
first.**

**Verified state, 2026-09-08 — read from each seat's own tree, not inferred.** Both consuming
seats already record that they do not author the check set: the anchor's build file says the
suite *"is validate-peer and which we do not author"*, and the generation repository's own
guidance names it *"our gate"* and *"our conformance axis."* **So this is not a correction of
anyone's practice — it is a capability that no seat has claimed and that no document has ever
asked for.** The reference check set is 116 source files and roughly 68,000 lines; **nobody
should read that number as the size of the specification**, which is the point of §2 being
generated rather than transcribed.

### 5a.1 The extension half is blocked on something this proposal does not own

**The §2 register is derived from the oracle's own declarations.** That is right for the core
half and it is the reason §2 got cheap — but it means the register can say *what the reference
oracle tests* and **structurally cannot say what the specification requires.** For the core,
the gap is small: the core spec's normative sections are cited by the checks themselves.

**For the extension half there is no addressable source at all.** An extension's conformance
inventory is its `Conformance` section, and across the 26 extension specs those sections
have **no declared shape and no stable row identifiers** — measured: 18 use level-named
subsections, 2 a table, 5 something else each, and **one has no conformance section at
all.** A requirement row is addressable today only by its English, so a check cannot name the
requirement it measures, only the section the requirement lives in — and a section routinely
carries four separate rows.

**So: an addressable conformance inventory is a hard prerequisite for specifying the extension
check sets, and it is tracked as its own open item rather than absorbed here.** This proposal
can land its core half without it. It cannot land the extension half.

**One thing that is cheaper than it looks:** the identifier scheme does not need inventing.
Three extension specs already carry `SUBJECT-CONDITION-N` vector ids in their own conformance
sections. **The house convention exists; extending it beats minting a second one.**

## 5b. The consumer seam — **a consumer keys to REQUIREMENTS, never to an oracle's check names** `[MUST]` `[added 2026-09-11]`

> A document, gate or baseline outside an oracle's own tree **MUST** identify a conformance
> obligation by its **requirement id** (`SPECIFICATION-FORMAT` §8.5a's `<PREFIX>-R<n>`), never by
> an oracle's check name. **Each oracle publishes its own check → requirement map**; that map is
> the only place an oracle's internal names appear outside it.

**This is the clause that makes §5a's second instrument worth building rather than merely
permitted, and it is nearly free today and expensive to retrofit.** Without it a second oracle is a
source of noise; with it, it is a source of findings.

**The cost of not having it, measured in the most oracle-coupled consumer tree: 431 pinned check
names** — 344 as gate baselines across seven compositions, 87 more in coverage maps across three
contracts. **If a second instrument names the same assertion differently, all 431 go red at once.**
That is a false red across an entire tree, and *the reasonable response to a false red across an
entire tree is to delete the baselines* — so the failure mode is not a bad migration, it is the
instrument being discarded and the coverage with it.

```
        today                           with this clause
   consumer -> oracle check name    consumer -> REQUIREMENT id  <- each oracle declares
               (431 strings)                    (COMP-R7, …)       its own check -> requirement map
```

**Three consequences, and the third is the whole argument for a second oracle:**

1. **A consumer stops naming any oracle's checks.** A baseline pins *"requirement `COMP-R7` is
   measured and passing"*; which check established that is the oracle's business. This pays for
   itself before any second oracle exists — today a re-pin that renames a check silently
   invalidates a baseline.
2. **Coverage becomes comparable, and jointly countable.** *"Oracle A reaches 24 of 38 binding
   rows, oracle B reaches 19, together 31"* becomes a sentence that can be written. Today it
   cannot be written at all.
3. **A disagreement localizes to a requirement, which is what makes it adjudicable.** *"One passes
   and one fails — is it the specification, the implementation, or the check?"* is **structurally
   unanswerable** when two oracles share no vocabulary, and is a **well-posed question for
   architecture** when both say *"we disagree about `COMP-R7`."* **Requirement-keying is the
   mechanism by which two oracles produce findings instead of two defensible numbers for one
   peer** — which §7's closing paragraph already names as the failure to avoid, without saying how.

**The identifier scheme needs no inventing and the prerequisite is already on the board.** §8.5a
landed 2026-09-08 with a worked reference spec and a ratcheting gate. **The rule is landed; the
sweep is not — 1 of 26 extension conformance inventories are conformant.** That sweep is
architecture's, it is the same blocker §5a.1 names, and it gates the **extension** half of both the
new seat's work and this clause. **The core half is unblocked**: core requirements are cited by the
checks themselves.

---

## 5c. **Two is a floor, not a target** — the check set is built N times, and convergence is the exit condition `[added 2026-09-11]`

**§5a settled that more than one instrument gets built. This settles the shape, and it is not "a
second oracle" — it is a neutral requirement set with as many independent implementations over it as
it takes for a new one to stop finding anything.**

> The requirement set and its implementations **MUST** be separable: a requirement is stated once,
> in neutral language, and **any number of suites may implement it.** A suite declares which
> requirements it covers; coverage is reported **jointly across suites**, not per suite.

**Why this is the shape rather than an ambition.** The ecosystem has run this pattern twice and both
times it converged: one protocol specification → dozens of generated peers; one extension
specification → implementations across many languages. **Each round of independent construction
consumed ambiguity out of the neutral statement**, and the statement got sharper because building
against it is what exposes what it failed to say. There is no reason to expect conformance
requirements to behave differently, and good reason to expect they will behave the same — *it is the
same mechanism, not an analogy to it.*

**The economics run the right way.** Early suites are expensive and productive: they surface most of
the ambiguity, and every disagreement they raise is either a specification gap, an implementation
bug, or a bad check. Later suites get cheaper and find less. **A new suite that finds nothing is the
exit condition** — at that point the requirements are unambiguous enough that independent
construction no longer adds information, and a further suite is not worth building. *The value of
this model is that the stopping point is measured rather than asserted.*

**And it puts a number on the thing nobody can otherwise estimate:** how much of the ecosystem's
conformance confidence rests on one author's judgement. Today the answer is *all of it*. With N
suites it is a ratio, and the ratio is reportable.

**Corollary — suites do not share assertion code.** Sharing the substrate (peer launch, transport,
reporting) is expected and cheap. **Sharing assertions defeats the purpose**: a shared assertion is
a shared judgement, and a shared judgement is the single point of failure two suites exist to not
have. *Two suites that share their assertion layer are one suite with two front ends.*

---

## 6. Open questions — **answered 2026-09-08**

1. **Does this spec live here or in `entity-core-protocol`?** Conformance to the *core* is a
   core concern, and V7 §9.0 already carries oracle text. But the register spans core and
   extension categories, and `GUIDE-CONFORMANCE` — the operational half — is here.
   **Recommended: here**, with V7 §9.0 reduced to a pointer, because a register that
   stopped at the core boundary would leave the extension categories exactly as unspecified
   as they are now.

2. **Do the `system/validate/*` conformance handlers (§7a) fold in?** They are already
   specified here and already have an ownership table — the only real question is whether
   §7a moves into the new spec or stays in the guide and is cited from it.
   **Recommended: cite, don't move**, so this proposal stays reviewable.

3. **Does the register pin category *names*?** Renaming a category breaks every published
   conformance citation, which argues for pinning them. But the current names are the
   reference oracle's Go identifiers, and adopting them wholesale is a soft form of the
   coupling this proposal exists to remove. **Recommended: pin the names, and say plainly
   that they are inherited** — the alternative is invalidating every conformance number in
   the ecosystem to win a naming argument.

**All three recommendations are adopted, 2026-09-08, and the reasoning that settles each is
the same one: a second implementation now has to be written from this document.**

| # | Answered | Because |
|---|---|---|
| **1** | **Here.** The core spec's §9.0 reduces to a pointer | A register stopping at the core boundary leaves the extension categories exactly as unspecified as they are today — and the extension half is now the half with two builders queued behind it |
| **2** | **Cite, don't move.** The conformance handlers stay where they are and are referenced | Moving them makes this proposal a re-organisation as well as a contract, and it is already the larger of the two |
| **3** | **Pin the inherited names, and say they are inherited** | A second builder needs the names to mean the same thing on day one. Renaming to win the coupling argument would invalidate every published conformance number in the ecosystem, and the coupling it removes is cosmetic where the coupling that matters — expected values derived by running an implementation — is what §3(a) already prohibits |

**What is still genuinely open, and it is the ruling work rather than a design question:** the
**212 unresolved register rows**, and specifically the four groups §4 names — a core-profile
category with 10 checks and 0 citations resolved, another with 9 checks and 1, a third whose
checks are peer-attributable 0 of 4, and four checks asserting an amplification bound **with
no normative home anywhere**, citing a proposal that exists in neither this repository nor the
pre-split archive. **Those are arch's to rule and they were always ours.** They are bounded,
they are enumerated, and none of them blocks the two builders from starting on the categories
that do resolve.

## 7. Sequencing — what a second builder can start on today

**Nothing in §5a waits for this proposal to land.** Stated explicitly because the failure mode
this repository keeps measuring is an implementer holding off while a document is drafted, and
the standing rule is that implementations discover by building.

**Revised 2026-09-11 for §5a's corrected assignment.** Two rows changed owner and one is new; the
sequence and the blockers did not move.

| Phase | What | Blocked on |
|---|---|---|
| **now** | **Consuming seats key their gates and baselines to requirement ids** (§5b) — no ruling needed, pays for itself before any second oracle exists, and it is the prerequisite for every row below | nothing |
| **now** | **The new seat** implements the **core** categories that already resolve, from the generated register plus the cited normative sections — **starting from the declarative, requirement-keyed check format the generation seat has already built and proven**, not from a blank design | its repo existing |
| **now** | The generation seat continues extension-by-extension, recording what it had to *author* rather than transcribe — **and hands its check format and schema to the new seat**. It does **not** author a wire oracle (§5a) | nothing |
| **now** | **§4's verdict fields land in the reference oracle** — posture, digest, profile, spec version, requirement id. Cheapest high-value item in this proposal and a prerequisite for two oracles being comparable | nothing |
| **next** | Arch rules the 212 unresolved rows | arch only |
| **next** | The conformance inventory gains a declared shape and stable row ids across the 26 extension specs | `GI-11` — **the rule landed 2026-09-08; the sweep is at 1 of 26** |
| **then** | The **extension** check sets are specified in neutral language, per extension | the row above |
| **then** | This spec lands, §2's register becomes normative, and the five core-spec citations go oracle-neutral | the rulings |
| **then** | **The first differential** — both instruments run against one peer, and architecture adjudicates what disagrees | all of the above |

**The direction of the port is one-way and worth pinning:** a gap found by a second builder is
fixed in **both** its own check set and the reference one. A divergence that is left standing
in either direction turns two oracles into two dialects, which is worse than one oracle,
because it produces two defensible conformance numbers for the same peer.
