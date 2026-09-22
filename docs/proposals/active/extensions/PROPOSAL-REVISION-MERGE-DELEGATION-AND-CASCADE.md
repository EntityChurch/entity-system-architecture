# PROPOSAL — REVISION custom-merge delegation + the strategy cascade

**Status:** **DRAFT 2026-08-24 — partially folded ahead of this document. See §0.** Items R1–R3, R11–R13 are already in
`specs/extensions/EXTENSION-REVISION.md` (**v3.9 → v3.11**), and **R14–R15** landed at **v3.12**
(§13–§15, added 2026-08-18); R4–R10 are proposed and unfolded.
**Target:** `EXTENSION-REVISION.md` — §2.3, §4.4.18, §5.1, §5.3. No V7/wire change, no new entity type, no new
capability, one new error-code *use* (an existing code on an existing op).
**Tier:** `extensions/` — validated by go · rust · py.
**Source:** the 2026-08-14/15 cohort round, plus the v3.9/v3.10 back-fill. Every item below arrived from an
implementation seat or the corpus coherence gate; none was
invented here.
**Cohort review:** NOT YET HELD for R4–R10. R1–R3 were folded without one — that is the deviation in §0.
**Diligence pass run 2026-08-15 — see §11.** Two items carried claims their filing seat had retracted or
that were narrower than the defect; corrections are folded into the body.

---

## 0. Process deviation — recorded, not hidden

**`EXTENSION-REVISION` v3.11 was folded directly into the spec with no proposal in existence.**
`AGENTS-STANDARD` is explicit: *"Significant or normative/protocol changes are proposal-first, not a direct
edit (wording-only hygiene can go direct)."* Two new MUSTs — one of them a **capability-authority rule** —
are not wording-only hygiene.

There is a narrower carve-out in `AGENTS.md` — *"Cohort impl findings fix the spec in place (no rev bump)
once landed"* — and the fold was outside it in both directions: it **added new MUSTs** rather than fixing a
landed decision, and it **bumped the rev**, which that clause explicitly excludes.

**Two consequences, and the second is the one worth keeping:**

1. The spec edit landed with no reviewable design record, so the cohort had nothing to disagree with.
2. **Because no proposal existed, the rationale had nowhere to live — so it was written into the spec.**
   v3.11's text named implementation repositories, quoted which seat argued what, and carried a dated
   ruling stamp. **A specification is implementable by someone who has never heard of this cohort;
   provenance belongs in a proposal.** The missing proposal *caused* the contaminated spec text. They are
   one failure, not two, and R11 is the cleanup.

**A second, larger process failure, and it is the reason this document covers more than v3.11 did.** The
v3.11 rulings were made from `entity-core-py`'s secondhand summary of a `entity-core-go` finding **without
opening go's spec-issue**, which was on disk at the time:
`entity-core-go/docs/validation/spec-issues/2026-08-14-revision-5-3-the-merge-request-shape-disagrees-three-ways.md`.
That document raises **E1–E5 plus a §0**; v3.11 ruled **two of them**. So the fold was not merely
unproposed, it was **partial in an area with an open, formal, unread spec-issue** — and it bumped the
version, which reads to every downstream consumer as *this area has been dealt with.*

This is the same shape the corpus already catalogs and it is now instanced against arch a seventh time:
*reading one claim at source does not cover the next claim in the same packet.*

**This document is not back-dated and is not prior authorization.** If cohort review rejects a shape, the
corresponding spec edit comes back out; the fold does not stand on having happened first.

---

## 1. Cycle inventory — everything the peers returned, with its disposition

The point of this section is that nothing routed in this cycle is tracked only in a dated ROUTING doc.
Items outside this proposal's target are listed with their real owner rather than silently dropped.

**Pins, read live 2026-08-15:** go `2df96f8` · rust `462f2c2` · py `f33526f` · keystone `ec69f56`.

### 1a. In scope here — `EXTENSION-REVISION`

| # | Item | Source | State |
|---|---|---|---|
| **R1** | §4.4.18/§2.3 — the `handler` sentinel with no companion path MUST be refused at config-write | py ask | **FOLDED v3.11** |
| **R2** | §5.3 E1 — `merge-request` is `{base, local, remote}`; the EXECUTE example's `path` struck | go E1 · py pinned | **FOLDED v3.11** |
| **R3** | §5.3 E4.1 — delegated dispatch runs under the **caller's** capability | go E4 · py matched | **FOLDED v3.11** |
| **R4** | §5.3's opening sentence still uses the **open-value encoding retracted in v3.9** | go E2 | **OPEN** |
| **R5** | §2.3's v3.10 disposition table points at **§5.2** for arms that live in **§5.1** | go E3 | **OPEN** |
| **R6** | Disposition when the handler path is absent, unbound, or exposes no `merge` operation | go E4.2 | **OPEN** |
| **R7** | Disposition on a malformed `merge-response` | go E4.3 · py enumerated | **OPEN** |
| **R8** | §5.1 `pattern: "*"` — peer-wide vs single-segment, a **go-only** narrowing | go E5 (**framing retracted by go**) · py + rust both conformant | **OPEN — ask survives, basis corrected §5** |
| **R9** | §5.1's commutativity `SHOULD` is **unimplementable as written** | go, noted | **OPEN — needs a decision, not a fix** |
| **R10** | §5.1 cascade step 1 (per-type) — producer-with-no-consumer in one impl | go §0 / G-21 | **no spec change; recorded** |
| **R11** | Strip implementation provenance out of v3.11's spec text | this document §0 | ✅ **FOLDED** `1f210f4` — **v3.11 only; the arc's v3.9/v3.10 text still carries it, see §12** |
| **R12** | §2.3/§5.1 — the merge-strategy vocabulary disagreed with itself three ways (v3.9) | corpus coherence gate | **FOLDED** `b91a4e8` — §8a |
| **R13** | §2.3/§5.1 — `lww` is offered by the table and resolvable by nothing (v3.10) | go, while building R12 | **FOLDED** `a1832df` — §8b |

### 1b. Returned this cycle, owned elsewhere — tracked, not folded here

| Item | Source | Owner / disposition |
|---|---|---|
| **GROUP published at M5 while built in no impl** | go spec-issue `2026-08-14-group-is-…` | **arch — DONE.** ADR-0012 false-conformance claim; corrected in `STATUS-RELEASE-SURFACE-AND-MATURITY-CANONICAL.md`, re-verified in all three trees. Not a spec change |
| **§5.3's resolving half is not wire-drivable cross-impl** | **go and py, independently** | **arch — DECISION OWED.** Driving it needs a surface that installs an application handler on the peer under test. Until then the honest disposition is a declared exclusion with its mutation executed (`GUIDE-CONFORMANCE` §5.2b.1), never a proxy. See §6 |
| **A-6 — `register-result` 202 carrier** | go | **rust owes the build.** Already ruled (arch `81e73ae`); no arch action |
| **Seed-policy `bounds` divergence** | py §4, go §5 | **keystone owns the format.** Flagged: go silently drops a declared bound, py refuses the file. A silent authority *upgrade* vs fail-closed — both must not ship. The one item in that group with a security direction |
| **Seed-policy `_comment` vs `additionalProperties: false`** | py §4, go §5 | keystone — ordinary schema work |
| **`hash_wire_format` scanner collision** | rust routed, go fixed (`d0ed41f`) | **closed.** A byte-scan treating any `0x58 0x21`/`0x58 0x31` hit as a hash; collides on high-entropy content for *any* impl. Sighted twice and closed as "a flake" both times |
| **Cohort numbers exclude the `convergence` category; restrictive posture drops cross-peer delivery checks** | py | **arch — DONE.** Both caveats now travel with the published numbers |
| **E5 `pattern: "*"` is ~~two-language, not three~~ **one**-language** | py corrected go to two; **go then retracted to one** | Folded into **R8**. ⚠ The first pass stopped at py's intermediate correction. go's own retraction goes further — *"one language, ours"* — and rust is **verified conformant at source**. The note must not push python **or rust** toward the segment reading |
| **`t1_1` wall-clock ratio** | py | Not a finding. Noisy on that box at both commits (0.77–1.44); named so it is not re-derived |

---

## 2. R4 — §5.3 opens with the encoding §2.3 retracted

§5.3's first sentence reads *"When the merge strategy names a handler path (e.g., `app/merge/text`), the
revision handler delegates:"* — the **open-value encoding**, where the strategy value *is* the path. §2.3
retracted that in v3.9: §5.1's `apply_strategy(strategy, …, handler_path)` takes the two as separate
arguments and branches on `strategy == "handler"`, and the open-value reading cannot coexist with §2.3's
`400 invalid_strategy` at config-write, which requires a closed enumeration.

**The retraction landed in §2.3 and §5.3 was never swept.** The section the cohort is told to build from
contradicts the ruling that defined the build, and §5.3 is the *natural first read* because v3.10's
disposition row points there.

**Proposed:** rewrite the opening to the sentinel encoding. Wording only; no behaviour in question.

## 3. R5 — the disposition table's pointer is wrong

§2.3's v3.10 table cites **§5.2** for both the `lww` and `handler` arms. Both are in **§5.1**
(`apply_strategy` / `dispatch_merge_handler`); §5.2 is `three_way_merge` and contains no dispatch at all.

**This one is not cosmetic and go's reason for filing it is the argument:** they **copied it** — the wrong
pointer propagated verbatim into two source comments and two status docs before they caught it. *A pointer
in a normative disposition table is the thing implementers follow rather than verify.*

> **Scope corrected 2026-08-15 — it is five wrong pointers, not two.** The first pass proposed correcting
> "both citations." A full enumeration of `§5.2` in `EXTENSION-REVISION.md` finds the same defect in three
> further places, all of them in text **this arc itself folded**:
>
> | Site | Reads | Actually |
> |---|---|---|
> | §2.3 table, `lww` row | "§5.2's arm read `lww_resolve(…)`" | the `lww` arm is §5.1's `apply_strategy` |
> | §2.3 table, `handler` row | "§5.2 has always carried a working dispatch arm" | §5.1's `handler` arm → `dispatch_merge_handler` |
> | §2.3 retraction note (v3.11) | "§5.2's `apply_strategy(strategy, …, handler_path)`" | `apply_strategy` is §5.1 |
> | §2.3 vocabulary note (v3.9/R12) | "§5.2's dispatch branched on `field-level`" | §5.1's dispatch |
> | §4.4.18 pseudocode comment | "§5.2 branches on `strategy == \"handler\"`" | §5.1 |
>
> **Structurally verified, not pattern-matched:** §5.1 *Merge Resolution Cascade* contains
> `apply_strategy` and `dispatch_merge_handler` and every strategy arm; §5.2 *Built-in Strategy: Three-Way
> Merge* begins after all of them and contains only `three_way_merge`. Three other `§5.2` citations in the
> file (the default-strategy note, the type-agnostic note, and the in-pseudocode "field-level comparison"
> comment) **are correct and must not be swept** — they genuinely refer to three-way merge. The file also
> already cites "§5.1's `dispatch_merge_handler`" correctly elsewhere, **so the document contradicts
> itself**, which is what makes the wrong pointer survive a read.

**Proposed:** correct **all five** citations to §5.1, leaving the three correct §5.2 references untouched.
**This is exactly the class the coherence gate should own** — a cross-reference to a section that does not
contain the named symbol is mechanically checkable, and R12 was found by that gate. Routed as a gate ask.

## 4. R6 / R7 — the two unpinned dispositions

§5.3 specifies the request/response contract and stops. Three states are reachable and none is pinned:

- **R6** — the handler path is absent, unbound, or exposes no `merge` operation.
- **R7** — the response is malformed: `resolved: true` with no `entity`; an `entity` naming a hash the peer
  does not hold; a response of the wrong type; the handler raises; a non-2xx.

**Proposed for both: the conflict entity**, matching what every other unresolvable arm does, and matching
what **go shipped and py independently enumerated** (py's list is the more complete of the two and is the
one to fold).

**The load-bearing case is the dangling hash, and it deserves its own sentence** — py's framing is the right
one: it is *"the disposition that protects state rather than recording it."* A handler claiming a merge it
cannot produce must not bind an unreadable hash into the merged trie. A conflict entity records the failure;
binding the hash corrupts the tree.

**Why pin rather than let it converge:** "obvious" is how three implementations diverge, and the divergence
is observable to a *caller* (a conflict entity and a merge-level error are different results).

## 5. R8 — `pattern: "*"` scope

§5.1 says a `*` config *"matches all paths within any merge, regardless of prefix"* and core v7.70 A1 names
`"*"` the peer-**wide** footgun. The spec sentence is unambiguous. **`entity-core-go` shipped a
single-segment narrowing (G-22)** — `path.Match` is segment-aware — **and their pre-existing vector could
not catch it**, because its one `pattern: "*"` row drove a single-segment key and passed for the wrong
reason. **That is the whole of the measured defect.**

> **Corrected 2026-08-15 — the first pass carried a claim its own filing seat had already retracted.** §5
> previously read *"every stdlib pulls the other way: Go's `path.Match` and the common Rust glob crates
> treat `*` as single-segment."* **go retracted the three-language framing on 2026-08-14** (`2115e2e`,
> reported in `HANDOFF-2026-08-15-both-peers-green-…` §3): *"The Rust half was never measured either, and
> rust's cycle reported no `*`-scope fix. **It was a one-language trap and the language was ours.**"*
>
> **Verified independently in rust's tree rather than taken from the retraction**
> (`path_pattern_matches`, `extensions/revision/src/merge.rs`, `entity-core-rust` `462f2c2`, source-read
> 2026-08-15): rust special-cases `pattern == "*"` → `return true`. It is **peer-wide and conformant**,
> hand-rolled, and **depends on no glob crate at all.** The proposal asserted a non-conformance about a
> sibling that does not exist.
>
> **`entity-core-py` is conformant too, and by the stdlib rather than by choice** — `fnmatch` translates
> `*` to `(?s:.*)\Z`, which has no concept of path segments and crosses `/`. py flagged this themselves and
> **explicitly declined to claim credit** for it. **An implementer reading a segment-flavoured note would
> "fix" python toward the segment reading and break a correct implementation** — py's framing, and it now
> applies to rust identically.

**So the trap is one-language, and R8's ask survives the correction intact.** go's own retraction keeps it:
*"§5.1's sentence still deserves a shared-corpus vector. The three-language framing does not."* Two
conformant implementations that arrived there by accident are not evidence the sentence is safe — neither
chose the reading, and either could lose it to a routine matcher swap.

**Proposed:** a normative note at §5.1 stating the peer-wide semantics *and* that a segment-aware matcher is
non-conformant, plus a shared-corpus vector driving a **nested, multi-segment** key — the row that the
existing vector's single-segment key could not fail. **The note MUST NOT name which languages get it wrong**
— that framing is what produced the retracted claim, and it would point an implementer at two conformant
trees.

## 6. R9 — a SHOULD nothing can satisfy

§5.1: *"implementations SHOULD detect and reject configurations that pair a known non-commutative handler
type with `caller-perspective` ordering."* **There is no way to know a handler is non-commutative** — no
declaration field exists on the merge-config, and the handler is arbitrary application code at a path.

This is the corpus's own named class: **a `MUST`/`SHOULD` may not name a referent the corpus does not
define** (`EXTENSION-REGISTRY` §6a.9, D10). Filed as a **decision, not a fix**, with three options and no
recommendation embedded in the spec:

| Option | Cost |
|---|---|
| (a) Add a `commutative: bool` declaration to the merge-config | A new field on a config type; the author self-declares, so it is a hint, not a proof |
| (b) Retract the SHOULD | Honest today; loses a real hazard the sentence was written about |
| (c) Restate as a deployment-posture note in `GUIDE-CAPABILITIES` | Keeps the warning, drops the false normative claim |

**Recommendation: (c) unless the cohort wants (a).** It is the only one that neither lies nor forgets.

## 7. R10 — recorded, no spec change

go found `merge-config` with `scope: "type"` writing a namespace **nothing in their tree read** — an
operator sets a per-type strategy, gets a `200 "set"`, and the config is never consulted. A
**producer with no consumer**, which passed every test its author wrote because the tests assert the write.

**No spec change is proposed: §5.1's cascade already specifies per-type as step 1.** It is recorded here
because (a) it is the mirror of the REGISTRY §3b shape arch routed *to* them the same day, and (b) their
initial framing — *"nothing in the cohort has built it"* — was an inherited claim about two repositories
nobody had opened, **which they corrected themselves after measuring**: py had built step 1 before go did,
and rust is *unmeasured, not failing*. Both halves of that are worth keeping.

## 8. R11 — remove implementation provenance from v3.11's spec text

v3.11's folded text names `entity-core-go` and `entity-core-py`, attributes arguments to seats, and carries a
`[ruled 2026-08-15]` stamp. **Proposed:** the normative rules stay exactly as they are; the surrounding
narrative is deleted from the spec and lives here instead. A rule reads *"the merge handler is dispatched
under the caller's capability"* — not who argued for it or when.

**This is an instance, and the class is corpus-wide:** 109 implementation-repo references across 17 spec
files, 58 backticked short-hash tokens, and 163 dates currently sit in `specs/`. **Every one passed a green
gate**, because the style and standards scopes have no rule for it. That is a separate, larger cleanup and
an `entity-system-arch-tools` gate; it is named here and **not** bundled into this proposal.

---

## 8a. R12 — the vocabulary disagreed with itself three ways (v3.9)

**Not reported by any seat — found by the corpus coherence gate on its first run**, on a surface
nobody had built hard enough to hit. Three lists, three answers, all normative-looking:

| Site | Vocabulary |
|---|---|
| §2.3 built-in table | `three-way` \| `source-wins` \| `target-wins` \| `lww` \| `keep-both` \| `manual` |
| §5.1 `apply_strategy` | branched on `field-level`; **no `three-way` arm, no `manual` arm** |
| per-type config block | `"field-level" \| … \| "handler"` — omitting **both** `three-way` and `manual` |

**So a config carrying `strategy: "three-way"` — this spec's own documented default, and the value
used in its own §4.4.18 and merge-params examples — reached the dispatch's `else` arm and silently
degraded to a conflict entity.** `field-level` is the name of the *algorithm* §5.1 runs, not of a
strategy an operator may select; it leaked into the selectable vocabulary and displaced the real one.

**Cross-impl-observable and dormant.** §2.3 pins `400 invalid_strategy` at config-write time, so
**which values a peer refuses depended on which of the three lists its implementer read.** All three
sites now read from the table.

**Why this one matters beyond its own fix:** it is the first defect found by a gate rather than by a
build or a review, and it was found on the run that gate was written. The rule it validates —
*a rule carried as prose recurs; a rule with a gate behind it holds* — is the argument for
`enum-value-not-declared` existing at all.

## 8b. R13 — `lww` is offered by the table and resolvable by nothing (v3.10)

Routed by go while building R12's write-time validation: §2.3's table offers `lww` and `handler`, and
neither resolved a conflict in any implementation. **Their pair is split here, because the two are
not the same kind of gap** — `handler` is an implementation gap (§5.3 fully specifies the delegation;
nothing had built it), `lww` is a **spec** gap.

§5.1's arm read `return {resolved: true, hash: lww_resolve(local_hash, remote_hash)}`, and
**`lww_resolve` is defined nowhere in this corpus** — over a comparison basis §2.3 *itself* says is
unavailable, in the very paragraph where it **rejects** `lww` as a `deletion_resolution` value
(*"real LWW requires commit-metadata not currently spec'd"*).

**So the table offered a strategy the same section says it cannot support, and the pseudocode
auto-resolved it through an undefined helper.** Three implementations reading that would have
produced three comparison bases and a convergence bug — **the divergence would have been in merge
outcomes, which is about as expensive as this corpus gets.**

**Ruled:** `lww` **stays in the vocabulary** (an operator may write it, `400 invalid_strategy` does
not fire, the pinned rejection contract does not churn) and **MUST degrade to a conflict entity.**
Deliberately **not** inventing a comparison basis — a guessed one *is* the convergence bug, and
application-layer LWW is available today through `handler`. The dependency is named rather than
resolved.

## 9. Open questions for the cohort

1. **R6/R7** — does anyone object to the conflict entity for both, and is py's malformed-response
   enumeration complete?
2. **R8** — is a normative note enough, or does this want a shared-corpus vector before rust builds?
3. **R9** — (a), (b), or (c)?
4. **R2 follow-on** — go noted a `path` field in `merge-request` is *defensible on the merits* (a text
   driver wants the file extension). v3.11 struck it from the example rather than adding it to the type,
   which is the conservative direction. **If any seat wants `path`, this is the moment** — it is a type-block
   amendment and all three should land it together.
5. **§5.3 cross-impl coverage** (§1b) — is arch expected to specify a handler-install surface, or is a
   declared in-process exclusion the accepted answer? Two seats have now asked.
6. **To `entity-core-go`, not a spec question `[raised by review, 2026-08-15]`** — your E5 retraction did
   not reach your own spec-issue. `2115e2e` reports it *"corrected in all four places"*; the spec-issue is
   not one of them, and at `2df96f8` it still reads *"a three-language trap … the common Rust glob crates
   default to single-segment"* with no banner. **That file is the one `AGENTS.md` sends us to first**, and
   it is where we picked the claim up. A one-line `RETRACTED` banner closes it.

## 10. Fold plan

R4, R5 fold on ratification (wording). R6, R7, R8 fold with vectors. R9 folds as whichever option is chosen.
R11 folds immediately — it removes text, adds no rule, and is not gated on cohort review.

R1–R3 are already in the spec; if review rejects any of them, that edit is reverted rather than defended.

## 11. Review — the diligence pass `[2026-08-15]`

**Both axes run**, per `PLAN-2026-08-15-spec-provenance-reconstruction.md` §8. This proposal already cited
go's spec-issue by path, so Axis A here was **counting the items and checking for supersession** rather
than discovering the document. **The supersession check is what paid.**

**Sources opened:** `entity-core-go` `docs/validation/spec-issues/2026-08-14-revision-5-3-*.md` (§0, E1–E4,
plus E5 appended in its status block — **E1–E5 + §0 confirmed, the proposal's count is right**) ·
`docs/status/HANDOFF-2026-08-15-b-*.md` · `docs/status/HANDOFF-2026-08-15-both-peers-green-*.md` §3 ·
commit `2115e2e`. `entity-core-py` `docs/status/HANDOFF-2026-08-14-both-routed-items-closed-*.md` §2–§5.
`entity-core-rust` `extensions/revision/src/merge.rs`. **Build state re-pinned 2026-08-15:** go `2df96f8`,
rust `462f2c2`, py `f33526f`, all clean; go's move off `d0ed41f` is docs-only.

### What the review changed

| # | Axis | Finding | Disposition |
|---|---|---|---|
| **R8-a** | A | **The proposal carried a claim go had already retracted.** E5's "three-language trap" became "one-language, and the language was ours" on 08-14. §5 still asserted rust's glob crates narrow `*` | **§5 rewritten** |
| **R8-b** | B | **Verified rust independently rather than accepting the retraction:** `path_pattern_matches` returns `true` for `"*"` — peer-wide, conformant, no glob crate. The proposal alleged a sibling non-conformance **that does not exist** | **§5 + §1b corrected** |
| **R5-a** | B | **R5 is five wrong pointers, not two.** Three more `§5.2`-for-§5.1 citations sit in text this arc folded; three *other* §5.2 citations are correct and must not be swept | **§3 rewritten, gate ask added** |
| **R11-a** | B | **R11 is partial with respect to its own arc.** It stripped provenance from v3.11; the v3.9/v3.10 text **this same proposal folded** (R12/R13) still names `entity-core-go` and carries `[ruled 2026-08-14]` stamps | **New §12** |

### The generalizable finding — reading the right document was not enough

**arch read go's formal spec-issue, which is the correct document, and still got E5 wrong — because go's
retraction never landed in it.** `2115e2e` says *"corrected in all four places, including the packet to
arch"*; that commit **does not touch the spec-issue**, and at go `2df96f8` the file still reads
*"a three-language trap … the common Rust glob crates default to single-segment."* No retraction banner.

> **Rule this earns.** `AGENTS.md` says *read the filing seat's own document*. That is necessary and, twice
> now, insufficient. **A spec-issue is a living document and a seat's retraction may not reach it** — so
> before acting on any item in one, check the seat's later status/handoff docs for a correction, exactly as
> a dated build-state pin is re-verified. Same failure shape as D8, one artifact over.

**Arch had both halves and merged them inconsistently:** §1b recorded py's intermediate "two-language"
correction while §5 kept go's un-retracted three-language text. **A proposal that disagrees with itself
about a fact is evidence neither half was checked.**

### What the review confirmed as correct

- **R1** — the ask is py's own (`HANDOFF-2026-08-14` §4, *"To ARCH — one ask, ours"*), correctly attributed;
  go built the same inference independently.
- **R2 / R3** — py *"pinned two readings, both matching go"*: `{base, local, remote}` with no `path`, and
  caller's-capability authority. Attributions in §1a are right.
- **R6 / R7** — py's enumeration is the more complete one and the proposal folds it; *"the disposition that
  protects state rather than recording it"* is py's phrase and is credited.
- **§1b's §5.3 wire-drivability row** — *"go and py, independently"* is exact: py's §5 says *"it is the same
  ask from two seats now."*
- **R10** — go corrected their own "the cohort has not built it" framing after measuring; py built cascade
  step 1 first; rust is **unmeasured, not failing**. All three halves are stated correctly.
- **R5's core claim is structurally true** — §5.1 holds every strategy arm; §5.2 holds only `three_way_merge`.

### Not covered by this pass

- **No cohort review of R4–R10.** Unchanged from §0; R5-a and R8-b are new since the first pass and both
  should travel in the same packet.
- **R9's three options were not tested against any seat's opinion** — still a decision, not a fix.

## 12. R11 is partial — the same arc's v3.9/v3.10 text still carries provenance `[found by review, 2026-08-15]`

R11 was scoped to *"v3.11's spec text"* and did that job (`1f210f4`). **But this proposal also folds v3.9
(R12) and v3.10 (R13), and that text carries the identical defect** — `EXTENSION-REVISION.md` §2.3 still
reads *"Routed by `entity-core-go` while building v3.9's write-time validation"*, *"Argument routed by
`entity-core-go` and adopted"*, and `[ruled 2026-08-14, v3.10]` / `[ruled 2026-08-14]` stamps.

**It does not fail the gate** — it is held in `.spec-baseline.json` as pre-existing debt, correctly. **The
finding is that the arc cleaned one of its three revisions and reported R11 as done.** Either R11's scope
extends to the whole arc, or its row should say v3.11-only. **Proposed: extend it** — the rationale already
has a home in this document, which is the condition R11 was waiting on and the reason the v3.11 strip was
safe to do.

---

## 13. R14 / R15 — the merge-config matcher was undefined twice `[2026-08-18, folded v3.12]`

**Status:** both folded in `EXTENSION-REVISION.md` **v3.11 → v3.12**; §14 is the verified delta.
**Source:** `entity-core-go` routed R14 as *"the §2.3/§2.4 scoping contradiction is arch's to
reconcile"* (`docs/validation/spec-issues/2026-08-18-d-*` §3, at `6ed6f95`), after `entity-core-py`
found their own merge matcher on `fnmatch` (SA-PY-12). **R15 was not routed — it was found while
verifying R14**, and it is the larger of the two.

### R14 — which matcher evaluates merge-config `pattern`

`1664a67` closed the exclude matcher to four forms and scoped it: *"that matcher is scoped to
`exclude` and `exclude_types` in this document **and nowhere else** … a `*` appearing outside these
two fields — **including elsewhere in this spec** — is §5.4's `*`."*

**§5.1's resolution pseudocode has always read `glob_match(config.data.pattern, path)`.** The document
contradicted itself the moment that sentence landed, and both readings were live in one file.

**Ruled: merge-config `pattern` is the four forms.** The deciding fact is in §5.1's own prose, not in
a preference — it motivates per-path config with *"all `.lock` files use source-wins."* That is a
**suffix** match: form 3, the single form §5.4 cannot express. Reading `pattern` as `matches_pattern`
would have made this section's own worked example unexpressible, and would have required go — the
only seat conformant under either reading — to regress.

> **How the error was made, because the method will be used again.** `1664a67`'s scoping clause was
> derived from a census of *`*`-bearing patterns* — 40 of them, classified prefix / suffix / infix /
> already-correct. That census is sound and its ruling is sound. **A pattern census cannot see a call
> site.** `glob_match(config.data.pattern, …)` carries no `*` and appears in no pattern inventory, so
> the field that most needed scoping was structurally invisible to the measurement that set the scope.
> **When a rule is scoped by an inventory, the inventory's unit must be the thing the rule binds** —
> here, callers of the matcher, not instances of its syntax.

### R15 — `pattern_specificity` is called by §5.1 and defined nowhere in the corpus

Step 2 computes `specificity = pattern_specificity(config.data.pattern)` to choose among matching
configs. **The corpus defines no such function** — searched by name across all files. This is the
`EXTENSION-REGISTRY` §6a.9 **D10** class the cohort already named (*a normative algorithm may not name
a referent the corpus does not define*), and it is the same class as R9 one section up.

**It is worse than R14 in consequence.** R14 decides whether a config matches; R15 decides **which
config wins when several do** — and the winner picks the strategy, which picks the merged bytes, which
are the version `root`. And the fallback is not merely unspecified but actively non-deterministic:
`specificity > best_specificity` keeps whichever config `list_entities` yielded first, **and
`list_entities` ordering is unspecified.** Two peers with identical configs and identical content can
resolve the same conflict differently, with nothing failing anywhere.

**Ruled: a total order, ties impossible** — exact (3) → subtree prefix (2) → trailing literal (1) →
match-all (0); within a rank, longer literal wins; the residual tie (two configs, same `pattern`,
different `{name}`) breaks on lexicographic `pattern` then `{name}`, both peer-independent.

### R15 is not hypothetical — three seats already diverge, source-read at named commits

Every seat invented a specificity function, because the algorithm called one. **They are not the same
function**, and no test could have caught it: each is self-consistent, and the conformance suite has
no two-config merge row.

| Seat | commit | symbol · path | Scores by |
|---|---|---|---|
| `entity-core-go` | `6ed6f95` | `patternSpecificity` · `ext/revision/strategy.go` | literal characters, excluding `*` and `?` |
| `entity-core-py` | `9ba439b` | `_pattern_specificity` · `packages/entity-handlers/src/entity_handlers/revision.py` | `len − count("*") − count("?")` — **identical to go** |
| `entity-core-rust` | `a701e13` | inline `pattern.len()` · `extensions/revision/src/merge.rs` (**two sites**: 420 `deletion_resolution`, 597 `strategy`) | total pattern length — **counts the `*`** |

**A two-config witness that diverges today.** Configs `pattern: "*"` → `source-wins` and
`pattern: "a"` → `target-wins`; conflict at path `a`. go and py score `*`→0 and `a`→1, so **exact
wins** — the correct answer under this fold. rust scores both **1**, ties, and keeps whichever
`location_index.list()` yielded first. **One merge, two outcomes, both peers reporting
`status: merged` with an empty `conflicts` array** — §5.1 already records that a config-resolved
conflict is byte-identical to a clean merge, so there is no audit signal either.

**And all three tie-break the same wrong way.** Every seat compares with strict `>` over an
unspecified enumeration order (`list_entities` / `location_index.list()` / the py equivalent), so even
where the scores agree, two configs of equal score resolve by whichever the store happened to yield.
That is the defect the lexicographic final tie-break exists to remove.

**Note what this says about R14 by contrast.** On the *matcher* question all three seats are
**already conformant** — go's `mergePatternMatch → globMatch`, rust's `path_pattern_matches →
engine::glob_match`, py's `_glob_match` (moved off `fnmatch` when they closed SA-PY-12) are all the
four forms. R14 ratifies what the cohort built. **R15 is where they actually split**, and it was
found only by chasing R14's routed question one level down.

**One rung is a genuine choice and is recorded as one.** Ranks 3 and 0 are forced. Rank 2 above rank 1
is not: for `docs/a.lock`, `docs/*` and `*.lock` both match and neither contains the other. Anchored
beats floating — a prefix names a location the operator laid out, a suffix names a kind that may
appear anywhere — and an operator who wants the kind rule to win inside a subtree writes a narrower
prefix or the exact path, which outranks both. **A seat that disagrees should say so before building;
this is the one row in the fold that is arch's judgment rather than the document's own logic.**

**Closing it was not deferrable, and `1664a67` already settled why.** Its own commit message records
the reasoning error it corrected: *"an earlier draft cited the hash-determinism as a reason to DEFER
this ruling. That is backwards. The divergence is caused by the surface being unpinned, so severity
raises the priority of closing it."* R15 is the same shape with a wider blast radius.

### Interaction with R8 — none, and that is worth stating

R8 asks whether `pattern: "*"` is peer-wide or single-segment. **Both candidate matchers answer
peer-wide** (form 1 = match-all; §5.4 bare `*` = match-all), so R14 neither resolves R8 nor disturbs
it. R8's ask — a shared-corpus vector driving a nested multi-segment key — **survives intact**, and
`MERGE-SPEC-ORDER-2` now supplies a second row that exercises `"*"` against a competing config, which
R8's single-config vector could not.

## 14. R14 / R15 spec delta — verified against the tree before the version moved (L3)

| # | § | Change | Verified |
|---|---|---|---|
| D1 | §2.4 cons. 3 | Scope extended from two fields to **three**, naming merge-config `pattern` | ✅ present |
| D2 | §2.4 | Correction box — the `.lock` argument and the pattern-census-vs-call-site mechanism | ✅ present |
| D3 | §2.4 | `glob_match` intro + `subject` definition carry the third field (trie-relative path) | ✅ present |
| D4 | §2.4 | Form-3 scope line reads "three matcher fields" | ✅ present |
| D5 | §2.4 | Write-time rejection now cites **§4.4.17 V6 and §4.4.18 V7** | ✅ present |
| D6 | §5.1 | `pattern_specificity` defined — rank table, tie-break, the anchored-vs-floating note | ✅ present |
| D7 | §4.4.18 | **V7** in the algorithm — `400 config/invalid-merge-pattern`, no binding lands | ✅ present |
| D8 | §4.4.18 | Six vectors: `MERGE-PATTERN-{REJECT,SUFFIX,SUBTREE}-1`, `MERGE-SPEC-{ORDER-1,ORDER-2,TIE-1}` | ✅ present |
| D9 | §6.1 | R11/R12 extension — the "Go and Rust already perform augment-then-dedup" sentence removed (L5) | ✅ present |
| D10 | header | **3.11 → 3.12** | ✅ present |

**One new error code use:** `config/invalid-merge-pattern` on an existing op. No V7 wire change, no
new entity type, no new capability.

**D9 is R12's outstanding half, closed here.** §12 above proposed extending R11's provenance strip to
the v3.9/v3.10 text; `entity-core-rust` independently reported this specific sentence, both because it
names two repos in normative text (L5) and because **it is inaccurate about one of them** — rust
augments on the commit path only. Removal rather than a fresher pin is the standing rule: a
build-state claim in a spec is re-read as a rule and cited long after it stops being true.

## 15. R14 / R15 cohort impact — **routed, not only tabled** (L13)

Delivered in `docs/status/ROUTING-2026-08-18-m-*`, addressed to all three seats.

| Seat | State at read | Owed |
|---|---|---|
| `entity-core-go` | `6ed6f95` | **R14: conformant, no change** — `mergePatternMatch → globMatch` is the four forms, verified at source. **R15: `patternSpecificity` → the rank order + lexicographic tie-break.** Add V7 + the six vectors |
| `entity-core-rust` | `a701e13` | **R14: conformant, no change** — `path_pattern_matches → engine::glob_match` is the four forms, verified at source. **R15: two sites** (`merge.rs` 420 and 597) move off `pattern.len()`; yours is the seat whose scores differ from the other two. V7 + vectors |
| `entity-core-py` | `9ba439b` | **R14: conformant, no change** — `_glob_match`, moved off `fnmatch` when you closed SA-PY-12. *(`entity-core-go`'s 08-18 (f) §3 reports py still on `fnmatch` at this site; that was true when you filed SA-PY-12 and is not true at `9ba439b` — your fix landed in between.)* **R15:** `_pattern_specificity` → rank order + tie-break. V7 + vectors |

**Nobody owes a matcher change; everybody owes a specificity change.** That inversion is the
correction to how this round was framed — R14 was the routed question and is the settled half; R15
was found underneath it and is the live divergence.

**`MERGE-SPEC-TIE-1` is the row all three seats fail**, and it fails silently: it needs two configs of
equal score written in both orders, which no existing vector sets up. **`MERGE-SPEC-ORDER-2` is the
row `entity-core-rust` fails on scoring alone** — `*` and `a` tie at length 1 there and do not tie
under literal-count.
