# PROPOSAL — de-version the test-vector corpus

**Status:** **FULLY EXECUTED / IMPLEMENTED 2026-08-31** — read §9 (the amendment) and §10
(the execution record). The `crypto-agility` half landed 2026-08-22; the `ecf-conformance`
half was blocked on a core-protocol normative edit §3 never scoped, and §9 is the amendment
that covers it. `spec corpus` is **0 errors**, down from 4 → 1 → 0. Two of §8.3's four
carried rows remain open and live on the cohort ledger, not here.
**Target:** `GUIDE-CONFORMANCE.md` §5.1 (normative rewrite, this repo) ·
`entity-core-protocol/specs/test-vectors/` (layout) · `entity-core-go/cmd/` (three
command names) · `entity-core-keystone` (vendor relation) — **rename + rule change, no
vector value changes, no wire change.**
**Provenance:** `docs/status/STATUS-2026-08-13-version-stamp-audit.md` (the artifact
audit) + `docs/status/STATUS-2026-08-13-b-conformance-tooling-ownership.md` Finding F.
**Sibling of:** `PROPOSAL-CONFORMANCE-ORACLE-CONTRACT.md` — this one fixes the corpus's
*identity*, that one fixes who specifies the machinery. They can land independently.

---

## 0. The argument in one paragraph

`GUIDE-CONFORMANCE` §5.1 makes the corpus version part of the conformance citation and
requires a bump whenever a vector is added. **Every corpus change since the v0.8.0 release
has violated it** — ECF went 69 → 71 vectors at `be54baf` (2026-07-12), crypto-agility
added the three Phase-2 matrix vectors, and both files are still named `-v1` in all four
repos that carry them. There is no `-v2` anywhere in the polyrepo. So the rule is not
merely unused; **it is normative text the corpus has been silently breaking for a month,
and no gate could see it.** The choice is to start enforcing it — a rename cascade across
four repos on every vector addition, forever — or to retire it. This proposes retiring it,
because the corpus is a set of canonical vectors whose identity is *what it tests*, and
because the version stamp is what forced the two-copy structure that produced four separate
defects in one week.

---

## 1. What is wrong today

### 1a. The stamp does not describe the artifact

`v767` is spec revision **v7.67**, the revision that birthed the crypto-agility corpus. The
corpus has since been re-stamped under v7.77 rules (§4.5a item 1a) and the public release
is v0.8.0. **The name was true when written and is now false** — the same defect class as
every other item this cycle.

`-v1` on the artifact filenames has never been incremented (§0). And
`v767/conformance-vectors-v1.*` is misnamed twice: it is the **agility** corpus wearing the
**conformance** corpus's filename, two directories from the real
`ecf-conformance/conformance-vectors-v1.*`.

### 1b. The stamp is the *only* reason there are two copies

`specs/test-vectors/crypto-agility/` exists because it is the de-versioned publish form of
`specs/test-vectors/v767/`. Remove the stamp and there is no second file to keep.

That two-copy structure has cost, measured, in one week:

- the `.cbor` byte-identity invariant between copies, and the sweeps that maintain it;
- the **de-versioning rule** (`SEEDS.md` §5) and its failure mode — it may touch only what
  the encoder does not see, so a de-versioned `"description"` is a guaranteed `.cbor`
  divergence at the next rebuild;
- the **5-of-13 description drift** core-go found at rebuild-planning time — two copies
  still carrying a pre-1a claim, three diverged by de-versioning alone;
- the `-both` flag, and half of `v767-corpus-build`'s reason to exist;
- a **third** state in keystone, one rebuild stale, whose `.diag` still carries the F16
  widths (routed 2026-08-13).

### 1c. Nothing could see any of it

`test-vectors` sits in `exclude_dirs` for every prose analyzer scope. The corpus was
unreachable by the toolkit, not merely unchecked. **Fixed:** arch-tools `6bba52a` adds
`spec corpus`, which gates artifact naming, fixture widths, placeholders, pair agreement,
and vendor drift. This proposal is what turns that gate green.

---

## 2. The design

### 2a. A corpus is identified by what it tests

One directory per corpus, named for its subject. No revision stamp, no artifact version:

```
specs/test-vectors/
  crypto-agility/
    README.md
    SEEDS.md
    CHANGELOG.md          <- new
    agility-vectors.diag
    agility-vectors.cbor
  ecf-conformance/
    README.md
    CHANGELOG.md          <- new
    conformance-vectors.diag
    conformance-vectors.cbor
```

**One copy per corpus. `v767/` and `crypto-agility/` collapse into `crypto-agility/`,
taking `v767/`'s content** — it is the source, and `SEEDS.md` §5 already rules that where
the two disagree on an encoded field, `v767/` wins by definition. `agility-SEEDS.md` (the
06-21 publish copy, untouched for seven weeks and five commits behind) is superseded by
`v767/SEEDS.md`, renamed `SEEDS.md` in place.

### 2b. Change history goes in a CHANGELOG

Each corpus directory carries `CHANGELOG.md`: dated entries naming vectors added, vectors
whose `input`/`canonical` changed (and the spec revision that changed them), and the
cohort round that re-blessed the result. **This is what the version stamp was standing in
for, and it holds strictly more information** — `-v1 → -v2` says something changed; a
changelog entry says what, when, why, and who confirmed it.

### 2c. The conformance citation stops carrying a corpus version

§5.1 today makes the citation `(spec-version, corpus-version)`. **Replace it with
`(spec-version, corpus-name, artifact sha256)`.** The sha is the honest identifier: it is
exact, it is already how keystone's MANIFEST pins the corpus, it is already what
`corpus-verify` checks, and unlike an integer it cannot be forgotten. `ADR-0012` already
requires oracle-pinned numbers; this makes the corpus pin the same shape.

### 2d. Vendors take a byte copy

A vendored corpus is byte-identical to the source, both members, with no de-versioning
transform on the way in. Where a vendor needs local framing, it goes in the vendor's own
`MANIFEST.md`, never in the artifacts.

**This is the part that fixes the live keystone defect at the root.** Its `.diag` carries
58-byte Ed448 seeds because the vendored copy was a hand-de-versioned derivative rather
than a copy, so arch's `8d38e62` width correction had nothing to flow through. A byte copy
has no such gap, and `spec corpus --vendor` checks it mechanically.

### 2e. Release snapshots are unaffected

`entity-core-keystone/protocol-generator/shared/{spec-data,test-vectors}/v0.8.0/` and
`entity-core-formalization/spec-data/v0.8.0/` **keep their version directories.** A
point-in-time vendored snapshot of a released spec is a different thing from a corpus named
after a revision: there the version *is* the identity. `spec corpus` exempts
`vN.N.N`-shaped directories for exactly this reason.

### 2f. `v7.67` as a citation stays

*"This vector landed in v7.67"* is a true historical fact. **307 such citations in keystone
alone, 95 in core-go, 78 in rust, 68 in py, 24 in core-protocol, 10 in formalization.** None
of them move. Only the path/command/filename token goes.

---

## 3. The normative delta

**`GUIDE-CONFORMANCE.md` §5.1 — replace in full.** Current text is quoted in
`STATUS-2026-08-13-b` Finding F.

> ### §5.1 Corpus identity `[MUST]` `[revised 2026-08-13]`
>
> **A corpus is identified by its name, never by a version stamp.** The directory is named
> for what the corpus tests (`crypto-agility`, `ecf-conformance`); the artifacts are
> `<subject>-vectors.{diag,cbor}`. A corpus directory or artifact name **MUST NOT** carry a
> spec-revision stamp (`v767`, `v7.67`) or an artifact version (`-v1`, `_v2`). A corpus
> artifact's stem MUST name the same subject as its directory.
>
> *Rationale, and why the previous rule is retired rather than enforced: the prior §5.1 made
> an integer corpus version part of the conformance citation and required a bump on every
> vector addition. Between 2026-06-21 and 2026-08-13 the ECF corpus went 69 → 71 vectors and
> the crypto-agility corpus gained three Phase-2 matrix vectors; neither filename moved, in
> any of the four repos carrying them, and no `-v2` was ever created. A rule broken by every
> change it governs is not a weak rule, it is the wrong rule: a corpus is a growing set of
> canonical vectors, and vectors are never removed (below), so the "version" was only ever
> an opaque restatement of "something changed."*
>
> **The conformance citation is `(spec-version, corpus-name, artifact sha256)`.** The sha256
> of the `.cbor` is the exact identifier; an implementation citing conformance names the
> artifact it ran against, per `ADR-0012`'s oracle-pinning rule.
>
> **Change history lives in `CHANGELOG.md` beside the artifacts** — dated, naming vectors
> added, any vector whose `input` or `canonical` changed together with the spec revision
> that changed it, and the cohort round that re-blessed the result.
>
> **Vectors are never removed.** A landed vector stays a conformance criterion. A vector
> that is wrong is corrected in place through the §4.4 sequence and recorded in the
> changelog, never deleted.
>
> **Changing a vector's `input` or `canonical` remains a spec-changing event** and goes
> through the normal proposal cycle — the previous canonical bytes were *the* correct bytes
> under the prior spec.
>
> **One corpus, one copy.** A corpus has exactly one authoring location. A vendored copy is
> **byte-identical in every member**; no de-versioning, date-stripping, or citation-shortening
> transform is applied on the way in. Vendor-local framing belongs in the vendor's manifest,
> never in the artifacts. *(The two-copy structure this replaces produced, in one week: a
> 5-of-13 description drift, a seven-week `SEEDS.md` gap, a de-versioning rule that could
> silently invalidate the next rebuild, and a vendored `.diag` carrying widths its own
> `.cbor` had corrected.)*
>
> **Gated by** `spec corpus` (`entity-system-arch-tools`): `corpus-version-stamp`,
> `corpus-name-mismatch`, `corpus-pair-incomplete`, `corpus-placeholder`,
> `corpus-fixture-width`, `corpus-pair-disagree`, and `vendor-drift` under `--vendor`.

**Consequential edits in this repo:** `EXTENSION-ENCRYPTION.md` §1142 and §1185 cite
`v767/SEEDS.md` as the pattern to follow; both become `crypto-agility/SEEDS.md`.
`GUIDE-CONFORMANCE` §3.2 is corrected by the sibling proposal (it documents one of the two
builders — Finding A).

---

## 4. Migration

Measured 2026-08-13; `v767` path/command tokens only, excluding `v7.67` citations.

| Repo | Tokens | What moves | Owner |
|---|---|---|---|
| `entity-core-protocol` | 31 | `v767/` → merge into `crypto-agility/`; drop `-v1` from both corpora; add two `CHANGELOG.md` | **arch** |
| `entity-core-go` | ~~160~~ **46 lines / 11 files** | `cmd/v767-corpus-{build,verify}` → `cmd/corpus-{build,verify}`; `cmd/v767-phase2-pins` → `cmd/agility-phase2-pins`; `scripts/validate-complete.sh`, `.gitignore`, `cmd/internal/validate/crypto_agility.go`, four stray constants. **Half a day.** | core-go |
| `entity-core-rust` | 22 | `cohort_compare_v767_phase{1,2}.rs` → `cohort_compare_agility_phase{1,2}.rs` | rust |
| `entity-core-py` | 2 | two comment lines | py |
| `entity-core-keystone` | 1 | one findings-log line naming `regen-v767-cbor.py`; **re-vendor as a byte copy** | keystone |
| `entity-system-architecture` | 33 | 2 in `EXTENSION-ENCRYPTION.md`; the rest are dated status snapshots — **immutable, do not sweep** | **arch** |
| `entity-core-formalization` · `browser-rust` · `arch-tools` · `content` · `workbench` | 0 | — | — |

**Impl-repo test filenames are the impl teams' call.** Arch owns `specs/test-vectors/`, the
§5.1 rule, and the two `EXTENSION-ENCRYPTION` citations; everything else is routed, not
mandated.

**Correction, 2026-08-13 — the core-go row was measured wrong and in the wrong direction.**
It counted 160 `v767` **tokens** and presented them as work. Measured live in their tree:
144 `v767` lines, **98 of them in `docs/status/`** — dated snapshots, immutable once
published, **the same carve-out this table correctly gave arch's own 33 and silently
withheld from theirs.** Applying one rule to both repos gives 46 lines across 11 files. The
schedule does not change; the item should not be weighed as if it were large.

---

## 4a. The hazard this proposal did not name

**Raised by core-go, and it is right: collapsing to one copy deletes the `.cbor`
byte-identity invariant — not by satisfying it, but by removing the second copy it compared
against.** `-both` goes, and `SEEDS.md` §5's de-versioning rule goes with it. That is most of
the point (§1b), but it has a consequence this proposal skipped:

**`corpus-build -check` becomes the sole structural protection on the corpus.** It is the
gate that proves *source produces artifact* — the second of `SEEDS.md` §4.2's two gates,
whose absence cost two months. So:

1. **`-check` MUST survive the rename and stay wired in PASS 0.** A rename that quietly
   drops it trades one invariant for none.
2. **The rename MUST be provably byte-neutral.** Re-run `-verify-legacy` against the frozen
   `56d4de4` pair afterwards and show the sha unchanged. **A path rename that moves a sha is
   a defect nobody would find for months** — and this corpus has already demonstrated that a
   `.cbor`/`.diag` disagreement can sit unnoticed for seven weeks.
3. **`spec corpus` (arch-tools `6bba52a`) is the third leg** and is independent of both — it
   checks fixture widths, placeholders, and `.diag`/`.cbor` agreement without needing a
   second copy, which is precisely the protection the byte-identity invariant was standing in
   for.

---

## 4b. Two live conformance checks depend on the rule being retired

The generated conformance register (core-go `01c86f1`) found, mechanically, that
**`conformance.corpus_version_agreement` and `conformance.spec_version_agreement` both cite
`GUIDE-CONFORMANCE §5.1`** — the exact rule §3 rewrites. Verified from this seat in their
register at that commit; both rows are classified `NON_NORMATIVE` because §5.1 lives in a
guide.

**So the corpus-version rule is not merely unenforced (Finding F) — it is asserted by two
live checks that have been passing over it.** This is a dependency, not an obstacle, and
naming it is the point: when §5.1 is rewritten those two checks change with it, and the
ratification MUST route that delta to core-go rather than leaving two checks asserting a
retired rule.

**What replaces them:** under §3's citation form the agreement being checked is
`(spec-version, corpus-name, artifact sha256)`, so `spec_version_agreement` survives
unchanged in intent and `corpus_version_agreement` becomes a corpus-**sha** agreement check —
which is what `corpus-verify` already computes.

---

## 5. Sequencing

1. **Fold `hash-format-sha-384.2.rehash` first** (ruled 2026-08-12, still unfolded at
   `da6baa8` — verified live: `kind` is still `content_hash_under_format`, pin still
   `012e64bbde…`). It is a `.diag` field change and it forces a rebuild. **Landing the
   rename before it means folding into paths that are about to move; landing it after
   means one rebuild instead of two.**
2. **Ratify this proposal**, then arch executes the core-protocol layout: merge `v767/` into
   `crypto-agility/`, drop `-v1`, write both changelogs, rewrite §5.1.
3. **Route to core-go** — three command renames and four path constants. `corpus-build`
   loses `-both`; there is one target.
4. **Route to keystone** — re-vendor as a byte copy. This supersedes the 2026-08-13 routing
   packet's item 1: the byte copy fixes the `.diag` widths as a side effect of being a copy.
5. **Wire `spec corpus` into `check`** once there is one corpus per name and the gate is
   green. It is deliberately standalone until then — shipping a red gate into CI is what
   core-go correctly refused to do with the corpus verifier.

**Rust and py are unblocked at any point** — their tokens are test filenames and comments.

---

## 6. What this does not do

- **No vector value changes.** No `input`, no `canonical`, no expectation moves. The `.cbor`
  content is unchanged by the rename; only the path and stem change.
- **No wire change, no spec-behaviour change.** ADR-0002 is untouched.
- **No change to keystone's `v0.8.0/` snapshot directories**, or formalization's.
- **No retraction of `v7.67` citations** anywhere.
- **It does not renumber or re-tag the v0.8.0 release.** The tag is the archival record and
  contains `crypto-agility/` in full; correcting the working tree changes what the *next*
  release ships, which `SEEDS.md` §5 already ruled is an ordinary fixture correction and not
  a history edit.

---

## 7. Open question for the cohort

**Does `ecf-conformance` also drop `-v1`, or only `crypto-agility`?** This proposal says
both, because the §5.1 rule is corpus-wide and ECF is the corpus that actually violated the
bump rule (69 → 71). But ECF is the more widely vendored of the two — keystone, rust
(`conformance/vectors-v1.cbor`), and py (`test-vectors/v1/`) all carry copies or
derivatives — so the rename touches more consumers for a corpus that has no `v767` problem.

**Recommended: both, in one pass.** Doing ECF later means running the split rule twice and
keeping `spec corpus` red in the interim, which is how the first two-copy structure was
justified too.

**Answered 2026-08-22 — and the recommendation was overruled, with the reason recorded in
§8.2. `spec corpus` is red in the interim exactly as this section predicted.** That cost is
accepted rather than argued away.

---

## 8. Execution record — 2026-08-22

### 8.1 What landed

`entity-core-protocol`, one commit, **artifact byte-neutral**:
`b5484e84dd2cddfa7d3cc8a041deba92cb29615aedb2180e31d8b6910ac5b648` before and after,
measured on both sides of the rename.

- `specs/test-vectors/v767/` **deleted**. The corpus collapses to one copy.
- `agility-SEEDS.md` → `SEEDS.md`; `agility-vectors-v1.{diag,cbor}` → `agility-vectors.{diag,cbor}`.
- `CHANGELOG.md` written (§2b), carrying the corpus history back to the first byte-pin. **No
  git SHAs in it** — `--series` re-authors every published commit, so an internal hash in a
  published file is a dangling citation by construction. Three of core-protocol's four
  dangling SHAs were inside `v767/` and are closed by its deletion.
- `.diag` edits are **comment-only** — verified by diffing the pre-rename blob against the
  new file: every changed line sits inside a `/ /` span, and no `id`, `kind`, `input`,
  `description` or `h'…'` value moved. The encoding therefore cannot have changed. **Byte
  re-verification is still the build owner's step under §5.1d and is routed, not assumed.**

`entity-system-architecture`:

- `GUIDE-CONFORMANCE` **§5.1 replaced** per §3, plus **§5.1a–§5.1d** — the corpus-process
  rules that had been living in the working copy's `SEEDS.md` §§4.1–4.4 and §5. §5.2's
  citation form updated to `(spec-version, corpus-name, artifact sha256)`.
- `EXTENSION-ENCRYPTION` §1162's `v767/SEEDS.md` citation → `crypto-agility/SEEDS.md`.

### 8.2 What did NOT land, and why

**`ecf-conformance` keeps `-v1`.** §7 recommended doing both corpora in one pass; that is
overruled for this cycle on a dependency §7 did not name:

**`ENTITY-CBOR-ENCODING` Appendix E is normative and requires the corpus version in the
citation** — *"Conformance reports MUST cite the version of `conformance-vectors-v{N}.cbor`"*,
plus three further `v{N}` references. De-versioning ECF therefore requires a **core-protocol
normative edit**, and **§3 of this proposal never included one**: it scoped the rule change to
`GUIDE-CONFORMANCE` §5.1 and missed that a core spec states the same rule independently. L1
holds — no normative spec edit without a proposal covering it — so the ECF half waits for an
amendment that does.

*This is worth more than a scheduling note: **the proposal identified §5.1 as the rule's home
and there were two homes.** §4b caught the same shape one layer out (two live conformance
checks citing §5.1) and the core spec was still missed. A rule restated in a second normative
document is invisible to a proposal that greps for the first one.*

Consequence, stated plainly: **`spec corpus` reports 1 ERROR against this corpus and will
until the ECF half lands.** It was 4 before this pass. The gate is a ratcheting worklist, not
a green light.

### 8.3 Carried to the ledger, not closed here

Per L9 — a proposal's open items do not fold with the proposal — these are rows on
`docs/COHORT-OPEN-ITEMS.md`, not sentences that expire in this file:

1. **ECF de-versioning + the `ENTITY-CBOR-ENCODING` §E amendment** (§8.2).
2. **The build owner's rename + registry collapse** — one `Copy` entry deleted, three command
   directories renamed, and the byte re-check under §5.1d.
3. **A positive SHA-384 content-hash vector is owed.** Inverting `hash-format-sha-384.2.rehash`
   removed the corpus's only affirmative SHA-384 digest assertion; `0x01` is now exercised only
   by refusal. The replacement belongs on a non-`system/peer` type.
4. **§4a's byte-neutrality re-check** — the encoder proof against the frozen pair, run in the
   build owner's tree after the rename.

---

## 9. Amendment — the `ENTITY-CBOR-ENCODING` Appendix E delta `[2026-08-31]`

**This section is the L1 instrument §8.2 said was owed.** It scopes the core-protocol normative edit
that §3 missed, so the ECF half can land. **EXECUTED the same session** — see §10.

### 9.1 What §3 should have said

§3 named `GUIDE-CONFORMANCE` §5.1 as the rule's home. **`ENTITY-CBOR-ENCODING` Appendix E is a
second home**, normative, in the core spec, stating the corpus-version citation rule independently:

| # | Site | Was |
|---|---|---|
| 1 | **§E.2** fixture format | *"The normative fixture is `conformance-vectors-v{N}.cbor`"* |
| 2 | **§E.2** source | *"Human-editable source: `conformance-vectors-v{N}.diag`"* |
| 3 | **§E.2** canonical location | `test-vectors/ecf-conformance/conformance-vectors-v1.cbor` |
| 4 | **§E.3** harness step 1 | *"Load `conformance-vectors-v{N}.cbor`"* |
| 5 | **§E.6** compliance reporting | **`[MUST]`** — *"Conformance reports MUST cite the version of `conformance-vectors-v{N}.cbor` … A report of 'passes v1' means …"* |

**Site 5 is the load-bearing one.** It is a live core `[MUST]` requiring a citation form that the
retired rule invented. Renaming the artifact without amending it would leave a MUST demanding a
version stamp that no longer exists anywhere — **L17**: a normative MUST naming a value with no
declared site a peer can carry.

### 9.2 The delta

Sites 1–4 drop the stamp. **Site 5 is rewritten to §5.1's landed form:**

> Conformance reports MUST cite the **corpus name and the sha256 of the `conformance-vectors.cbor`
> artifact** the implementation passes — the citation form is `(spec-version, corpus-name, artifact
> sha256)`. A report means every vector in the artifact with that digest returns pass under §E.3
> semantics.

Plus the vendor-verification sentence — **verify a vendored copy by digest, never by filename** —
because a filename mismatch makes an automated vendor check report *could-not-look* rather than a
failure, which is the specific way the last rename went unnoticed for six weeks.

**Scope:** citation-rule and filename only. **No vector value changes, no wire change, no encoding
change, no new code or status.** The `.cbor` artifact is byte-identical across the rename.

### 9.3 Why this was missed the first time, kept because it is the corpus's own lesson

§3 asked *"where is this rule written?"* and answered from where the rule was **found**. §4b caught
the same shape one layer out — two live conformance checks citing §5.1, correctly routed — **so the
proposal did ask "who else depends on this rule?" and asked it of code. It never asked it of the
specs.**

This is the **first incident of L23** (*a rule has every normative home it is stated in, not the one
the proposal names*), which this corpus ratified on 2026-08-30 and which fired a **third** time on
2026-08-31 during the FM-1 fold. The rule now exists because of this proposal; landing the amendment
closes the loop that opened it.

---

## 10. Execution record — 2026-08-31: the ECF half landed

`entity-core-protocol`, amendment **0.8.2.1**.

**Artifact byte-neutral where it counts:**

| | sha256 |
|---|---|
| `conformance-vectors.cbor` | `9695b1f1d939cfdfdd4297f8ad32122d424b1ec180cfae74c92d509d88f7c6dc` — **identical before and after**, measured on both sides |
| `conformance-vectors.diag` | `71015b72…` → `da521d67aa8193a3bf9acd232088d8515d50333f0294c65a5df3b45f46a1a87b` |

The `.diag` moved by **exactly two lines**, both inside its opening `/ … /` comment span, both naming
the old filename. Full diff verified against the pre-rename blob; no `id`, `kind`, `input`,
`description` or `h'…'` value moved, and the unchanged `.cbor` digest is the proof the encoding did
not change. **71 vectors, unchanged.**

- `conformance-vectors-v1.{cbor,diag}` → `conformance-vectors.{cbor,diag}`.
- `specs/test-vectors/ecf-conformance/CHANGELOG.md` written — the corpus's version now lives there,
  mirroring the crypto-agility half.
- Appendix E sites 1–5 amended per §9.2.
- **`spec corpus` is now 0 errors** against `entity-core-protocol`, down from 4 → 1 → 0. The gate
  that stayed red for nine days as §7 predicted is green.

**§8.3's rows 1 and 2 are discharged; rows 3 and 4 remain open** and stay on
`docs/COHORT-OPEN-ITEMS.md`: a positive SHA-384 content-hash vector is still owed, and §4a's
byte-neutrality re-check is the build owner's step under §5.1d.

**Routed, not assumed — this rename makes vendor gates blind, not merely stale.** Every tree
carrying a copy is named in the same session with `(seat, path, expected sha256)`, per the ratified
enforcement point from L8's sixteenth form: **`entity-core-keystone`, `entity-core-rust`
(`conformance/vectors-v1.cbor`), `entity-core-py` (`test-vectors/v1/`), `entity-core-go`
(`-corpus conformance-vectors-v1.cbor`)**. A vendored copy under the old filename reports
`vendor-unmatched`, which is a could-not-look and **must not be read as a pass**.

**This proposal is now fully executed and moves to `implemented/`.**
