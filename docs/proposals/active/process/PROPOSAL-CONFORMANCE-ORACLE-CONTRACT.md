# PROPOSAL — the conformance oracle is a specified thing, not a program

**Status:** DRAFT (2026-08-13)
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

*Generalizes: `GUIDE-CONFORMANCE` §2.4a (b), §5.2b/§5.2b.1 (e, f), §5.2c, the §8 S1/S3/S5
defects (a, d), and `ADR-0012` (c).*

### §4 The verdict document `[normative shape]`

> A conformance run emits a machine-readable verdict carrying: oracle name + commit; corpus
> name + artifact sha256; spec version; profile; peer identification; per-category
> `P/W/F/S` counts; and per-check `(id, verdict, spec_ref, message)`.
>
> `ADR-0012`'s citation form (`N·0F @ <oracle-commit>` with the P/W/F/S breakdown) is
> **derived from this document**, not typed by hand.

*Closes Finding E: keystone ships 5,000-line `CONFORMANCE-REPORT.json` files whose shape is
specified nowhere. This is mostly ratifying what those files already do.*

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
| **Architecture** | this spec; the §2 category register; corpus field content and vector rulings; the §5 build contract |
| **The cohort (any impl)** | implementations of the oracle and of the builder |
| **The reference oracle** | conformance *to this spec*, like any other implementation |
| **Vendors (keystone &c.)** | byte-copies; **no vendor authors canonical bytes** |

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

- **Does not replace, rewrite, fork, or reassign `validate-peer`.** It stays the reference
  oracle, in `entity-core-go`, owned by the Go team.
- **Does not change what any peer must do to conform.** Not one requirement moves.
- **Does not require a second oracle to be built.** It makes one *possible*, which is the
  point; whether anyone builds one is a separate call.
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

| Phase | What | Blocked on |
|---|---|---|
| **now** | A second builder implements the **core** categories that already resolve, from the generated register plus the cited normative sections | nothing |
| **now** | The generation repository continues extension-by-extension, recording what it had to *author* rather than transcribe | nothing |
| **next** | Arch rules the 212 unresolved rows | arch only |
| **next** | The conformance inventory gains a declared shape and stable row ids across the 26 extension specs | `GI-11` |
| **then** | The **extension** check sets are specified in neutral language, per extension | the row above |
| **then** | This spec lands, §2's register becomes normative, and the five core-spec citations go oracle-neutral | the rulings |

**The direction of the port is one-way and worth pinning:** a gap found by a second builder is
fixed in **both** its own check set and the reference one. A divergence that is left standing
in either direction turns two oracles into two dialects, which is worse than one oracle,
because it produces two defensible conformance numbers for the same peer.
