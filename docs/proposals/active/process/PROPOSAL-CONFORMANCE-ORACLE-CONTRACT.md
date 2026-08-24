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

## 6. Open questions

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
