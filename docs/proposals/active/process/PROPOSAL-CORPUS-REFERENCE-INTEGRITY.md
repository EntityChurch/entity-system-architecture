# PROPOSAL: Corpus reference integrity — a reference nothing can follow is not a citation

**Status**: DRAFT 2026-08-24
**Targets**: `entity-system-arch-tools` (`spec standards` rules) · `docs/proposals/` layout ·
`SPECIFICATION-FORMAT.md` §11 · two owed conformance seeds
**Depends**: nothing landed blocks this; §5 sequences behind `PROPOSAL-DEVERSION-TEST-VECTOR-CORPUS`

---

## 1. The diagnosis, because these look like five problems and are one

In one cycle this corpus produced: a retraction escalation over a document that existed, four
conformance checks citing a proposal's *test-vector list* as though it were normative, five
citing a proposal section whose spec edits landed under a different number, four cross-references
written as literal `§1.X`, and a superseded term surviving in three tiers after being corrected
in one.

**Every one is the same defect: a reference that cannot be followed, in a corpus where nothing
checks whether references resolve.** They differ only in *what* is unfollowable — a filename, a
section number, a term. They share the failure mode that makes them expensive: **an unfollowable
reference is indistinguishable from a sound one at review time.** It has the right shape. It sits
in a sentence that parses. The reader who would catch it is the reader who tries to follow it,
and prose review does not.

**The gate learned to check one of the three kinds** (`proposal-citation-dangling`, arch-tools
`04d9452`) and that is the whole reason the first class is now closed. This proposal finishes
the job rather than fixing more instances by hand.

> **This is the cross-peer-seam lesson pointed inward.** We already know a claim that holds
> locally and collapses at a seam is invisible to review and needs a mechanism. A citation is
> the same shape: it holds for the author, who knows what it meant, and collapses for every
> reader who does not.

## 2. Class A — the rule exists, is not gated, and cannot be gated yet `[corrected]`

> **Correction, before anything else in this section.** The first draft of this proposal said
> intra-corpus section references were unchecked and proposed building a `dangling-section-ref`
> rule. **That was wrong, and wrong in this cycle's signature way: it asserted an absence
> without running the tool sitting next to it.** `spec address` has validated exactly this
> since before the split — `stale-section`: *"`DOC.md §N.M` whose doc resolves but whose
> section does not."* Fourth instance of the same defect in one cycle, so it is recorded here
> rather than quietly edited out.

**What is actually true is more useful.** `spec address` reports **1,129 deviations** and is
**not wired into `check`** — `cli.py check` runs `style` + `standards` only. It is a reader, not
a gate.

**And it cannot simply be switched on, because it has the defect `proposal-citation` had:**

| Class | Count | Real? |
|---|---|---|
| `dangling` | 574 | **~523 are not dangling.** 495 target `ENTITY-CORE-PROTOCOL`, 24 `ENTITY-NATIVE-TYPE-SYSTEM`, 4 `ENTITY-CBOR-ENCODING` — all in the **sibling `entity-core-protocol` repo**, which the analyzer was never given. The rest target the legacy archive. |
| `bare-internal` | 462 | judgment-tier by design |
| `drift` | 45 | mechanical (`DOC` → `DOC.md`) |
| `nickname` | 24 | mechanical where the section exists |
| **`stale-section`** | **22** | **real, in-corpus, and the Class-A findings this proposal wanted** |
| `leak` | 2 | judgment |

**Gating it today would fire 523 false errors** — *"the target is not a file in the corpus"*
reported for documents that are simply in the other repo. **That is could-not-look rendered as
a verdict, which is the exact failure this whole proposal is about, sitting inside the analyzer
built to catch it.**

**Proposed — the fix is the one already written, applied a second time.** `standards` learned
three-valued resolution at `04d9452`: resolvable → silent, unresolvable-with-nowhere-to-look →
**warn**, configured-root-missing → **exit 2**. `address` needs the same shape and the same
plumbing: **multiple corpus roots** (`--corpus` repeatable, or `$SPEC_CORPUS` as a path list),
so `entity-system-architecture` + `entity-core-protocol` are analyzed as the one corpus they
actually are. Then `dangling` means dangling, `stale-section` stays honest, and the gate can be
switched on.

**The 22 `stale-section` findings are real and fixable now** — they include three citing
*line numbers* as section numbers (`EXTENSION-ROLE §215`, `§239`, `§433`) and several to
`EXTENSION-SUBSCRIPTION §1.3`, which does not exist (§1.1 and §1.2 do).

**Placeholders remain a genuine gap in a different way.** A literal `§X` is not a *number*, so
`stale-section`'s parser does not see it at all — `EXTENSION-CONTENT` §7.1's *"content extension
§X"* and `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4.X are invisible to every analyzer. That is a
small regex, and it is the only genuinely new rule this class needs.

## 3. Class B — superseded-terminology residue `[closed this cycle; keep the lesson]`

"Chain root" survived in `EXTENSION-SUBSCRIPTION`, `EXTENSION-COMPUTE`, both SDK specs, and two
sites inside `EXTENSION-CONTINUATION` — **the very file whose §3.1a performed the correction.**
Now zero (`7a71dea`, `f6bf697`).

**What made it durable is worth naming: the pseudocode was right everywhere and only the prose
was wrong.** Every implementation built from the normative pseudocode was conformant, so no run
ever went red, while the `§1.1` concept text and the `§11.1` MUST list — the two places an
implementer actually reads for a rule — said the opposite. **A spec can be wrong in exactly the
places no test can reach.**

**Proposed rule — `superseded-term` (warn).** A short configured map of retired term → the
correction's canonical home (`chain-root check` → `V7 §5.5` / `CONTINUATION §3.1a`). Warn, not
error, because a correction banner legitimately quotes the term it retires — the gate must not
punish the sentence doing the fixing. **Populated only when a correction lands**, from the
correction itself; this is not a style dictionary.

## 4. Class C — the proposal corpus is not in this repo `[the real decision]`

**104 proposal citations in landed specs. 12 resolve here; 76 resolve only in
the internal legacy corpus; 16 only under filename-prefix match. Zero resolve nowhere.**

For a reader of this repo alone, **88 of 104 are unfollowable** — including every
`proposals/implemented/…` path a landed spec offers as its rationale. Three options, and this
proposal picks one:

| | Option | Cost | Leaves |
|---|---|---|---|
| 1 | Import the 76 into `docs/proposals/implemented/` | one bulk move | corpus self-contained; +76 historical docs on the published surface |
| 2 | Re-cite all 76 to their landed spec sections | high, one at a time | best corpus; each is a real fold audit |
| 3 | Declare the archive an external normative-history root; ship the gate configured to it | trivial | reader still cannot follow a citation |

**Recommended: 3 now, 2 incrementally, 1 never.** (3) makes the gate honest today. (2) is where
the value is — **every citation walked this cycle found something**: `convergent_mirror` found
an unfolded gate section plus a contradicting `§4.4`; the COHERENT-CAP walk found the chain-root
residue in two specs. **The re-cite is not bookkeeping, it is the audit.** (1) is rejected: it
publishes 76 superseded documents as though they were current, which is the drift
`AGENTS-STANDARD`'s one-canonical-home rule exists to prevent.

**Sub-rule this cycle earned — `PROPOSAL §N` is not a normative home.** Four checks cited
COHERENT-CAP §10, which is *"Test vectors / cross-impl agreement"*; five cited RELAY-SOURCE-ROUTE
§4, which is *"Conformance vectors (cohort)"*. **A conformance check citing a proposal's own
vector list is circular** — it asserts the vector by pointing at the vector. Re-cites under (2)
MUST land on the spec section, and `SPECIFICATION-FORMAT.md` §11 should say so.

## 5. Class D — two owed conformance seeds `[arch owes; not gate work]`

1. **Replacement positive SHA-384 vector.** Inverting `hash-format-sha-384.2.rehash`
   (`4cf0990`) removed the corpus's only affirmative SHA-384 content-hash assertion; `0x01` is
   now exercised **only by refusal**. The replacement belongs on a **non-`system/peer`** type —
   any entity that legitimately carries a home format. `SEEDS.md` §4.4 ruled against retargeting
   the inverted vector, so this is a new seed, and it is arch's.
2. **Third-party delivery has no conformance coverage at all.** `EXTENSION-SUBSCRIPTION`
   §6.1/§6.2 — A subscribes on B, delivery to C — is **the seam that makes the in-chain rule
   load-bearing** (§3), and core-go reports nothing exercises it (`3e8361f`). It needs a
   three-peer topology, so it is a work item, not a line edit. **This is the highest-value item
   in this proposal:** the rule we just corrected in four documents is the rule no test asserts,
   which is precisely how it stayed wrong.

## 6. Not in scope

- **Corpus version stamps** (`corpus-version-stamp` ×4 on `entity-core-protocol`) — owned by
  `PROPOSAL-DEVERSION-TEST-VECTOR-CORPUS`, already open. **Do not silence it here.**
- **The editorial warn backlog** — 192 `impl-team-ref`, 148 `proposal-citation-unresolved`, 95
  `date-in-body`, 94 `amendment-provenance`. Publication hygiene, separately owned, deliberately
  untouched: they gate nothing and folding them into a correctness proposal buries the four
  items that do.

## 7. Sequencing

| # | Item | Owner |
|---|---|---|
| 1 | **Multi-root corpus resolution in `address`** (+ three-valued verdicts, per `standards` `04d9452`) | arch-tools |
| 2 | Fix the 22 real `stale-section` findings | arch |
| 3 | `placeholder-section-ref` rule (`§X` is not a number, so nothing sees it) | arch-tools |
| 4 | Wire `address --gate` into `check` once (1) makes it honest | arch-tools |
| 5 | Configure the archive root in the gate invocation (Class C option 3) | arch-tools |
| 6 | `superseded-term` rule, seeded with `chain-root` | arch-tools |
| 7 | Third-party-delivery conformance seed (§5.2) | **arch → cohort** |
| 8 | Replacement SHA-384 seed (§5.1) | arch |
| 9 | Re-cite the 76, incrementally, as fold audits | arch, ongoing |

**1–6 are one arch-tools cycle, and (1) is most of it. 7 is the one that matters and the one with a real dependency**
(a three-peer harness). Nothing here is release-critical for 08-21.

## 8. The test of whether this worked

**Not "the corpus is clean."** It is: *when the next reference goes stale, does a machine say so
before a reader trips on it?* Class C's first kind now passes that test. Class A has an analyzer that
would pass it and is not allowed to, for a reason that is itself an instance of the bug. Class B
does not, and this cycle demonstrated both — at the cost of one false escalation, one unfolded gate
section, and a superseded rule that survived its own correction in three tiers.
