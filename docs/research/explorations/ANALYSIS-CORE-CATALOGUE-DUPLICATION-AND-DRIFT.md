# ANALYSIS — ~~the core type catalogue duplicates 38 types it does not own, and 13 have drifted~~ **[VOID]**

> # ⛔ WITHDRAWN IN FULL — 2026-08-01, same day. Do not act on anything below.
>
> **Every finding in this document is a comparison against `ENTITY-CORE-MACHINE-SPEC.md`, which is a generated
> secondary artifact of this repo — nothing is sourced from it and it is not kept in sync.** A stale generated
> copy differing from its source is the expected state of a stale copy. It is not drift, not a defect in any
> spec, and **no implementation was ever at risk from any of it.**
>
> **The audit was mechanically correct and substantively worthless.** It compared specs against their own stale
> output and reported the difference as a finding — including three items escalated as "interoperation-breaking."
> Retained unedited as the record of a real methodological failure, not because the content is useful.
>
> **The lesson, which is the only thing here worth keeping:** *an audit is exactly as good as its premise about
> which documents are authoritative, and that premise was never checked.* The tooling was fine. Asking "who
> actually sources from this document?" before diffing it would have cost one grep and saved the entire cycle.
> **Establish authority before measuring conformance to it.**
>
> **The one item that was genuinely in this repo's lane is fixed** and did not depend on the machine spec:
> `SDK-OPERATIONS.md` restated the core-owned `system/type/field-spec` informally; it now carries a canonical
> pointer and is marked an abridged sketch.
>
> **Consequent recommendation:** delete `ENTITY-CORE-MACHINE-SPEC.md` upstream — see
> `PROPOSAL-INBOX-TYPE-NAMESPACE-CORRECTION` §6.

**Date:** 2026-08-01 · **Status:** ⛔ **VOID — withdrawn same day; see banner.** *(Original status line: audit
complete, mechanical, reproducible.)*
**Asked for:** *"one more pass over all the extensions to make sure we don't have any more oversights like this;
review core protocol for all types/etc that are mentioned to make sure everything is clean."*
**Method:** every `type/path := {` declaration extracted from both repos' published surface (`specs/` + `guides/`,
239 markdown files, 299 distinct types), field sets and `type_ref`s diffed pairwise. Script:
`typeaudit.py` / `drift.py` (scratch). Every finding below was **re-verified by reading both sources**; the
parser found candidates, it did not decide them.

---

## 0. Headline

**The inbox namespace problem was one symptom of a larger one.** `ENTITY-CORE-MACHINE-SPEC.md` is a complete
type catalogue that **re-declares 38 types owned by extension specs**, and **13 of those declarations have
drifted from their owner.** Three of the thirteen would break interoperability outright, not cosmetically.

**Nothing found is in the V7 §9.5 Core Type Floor** (except one reverse-direction case, §4), **and keystone has
adopted none of the drifted types** — every one is reported `absent / not-a-FAIL-if-absent` across all 22
generated peers. **The blast radius is bounded and the cost is bounded**; that is the good news and it is
temporary, because it is bounded only while nobody implements them.

## 1. The three that break interoperation `[severity 1]`

### 1.1 `system/tree/snapshot` — two different types sharing one name

| | Fields |
|---|---|
| **core catalogue** | `prefix: primitive/string`, `bindings: map_of system/hash` |
| **`EXTENSION-TREE.md`** (owner) | `root: system/hash` |

**They have no field in common.** This is not drift in a field's type — it is a **different entity**. And the
owner spec's prose says the divergence is deliberate: *"The snapshot does not carry location metadata (prefix);
location is operational context… This separation ensures snapshots are purely content-addressed. Two peers with
identical content under different prefixes produce the same trie root hash, the same snapshot entity, and the
same snapshot content_hash."*

**So the redesign was intentional and the catalogue kept the old shape.** An implementation built from the
catalogue emits a structurally different entity with a **different content hash**, which destroys exactly the
property EXTENSION-TREE relies on — and content-hash divergence is the least debuggable failure this system has.

### 1.2 `system/tree/merge-request` — a required field the owner made optional

| | `source` | `source_envelope` |
|---|---|---|
| **core catalogue** | **required** | **absent** |
| **`EXTENSION-TREE.md`** | optional — *"mutually exclusive with `source_envelope`"* | present |

An implementation built from the catalogue **rejects a valid merge-request** — one that supplies
`source_envelope` instead of `source`, which is the continuation-chain path the owner spec added. A validator
generated from the catalogue fails conformant traffic.

### 1.3 `compute/error` — required vs optional, and the owner explains why

| | `message` |
|---|---|
| **core catalogue** | **required** |
| **`EXTENSION-COMPUTE.md`** | **optional** — annotated *"In-flight diagnostic; NOT materialized"*, with `code` marked *"the ONLY materialized field"* |

The owner's materialization rule (§9.1) makes `message` deliberately droppable. The catalogue would force it onto
a materialized error — and error materialization is already a determinism-sensitive surface here
(`PROPOSAL-COMPUTE-ERROR-MATERIALIZATION-DETERMINISM`).

## 2. The other ten `[severity 2]`

The same pattern throughout: **the owner spec evolved and the catalogue did not.**

| Type | Owner | Catalogue is missing / stale |
|---|---|---|
| `system/subscription` | SUBSCRIPTION | `include_payload` absent |
| `system/subscription/request` | SUBSCRIPTION | `include_payload` absent |
| `compute/apply` | COMPUTE | `capability`, `resource` absent |
| `compute/arithmetic` | COMPUTE | `constraints` absent; `op` is a bare string vs `one_of [add,sub,mul,div,mod]` |
| `compute/compare` | COMPUTE | `constraints` absent; `op` bare vs `one_of [eq,neq,lt,gt,lte,gte]` |
| `compute/logic` | COMPUTE | `constraints` absent; `op` bare vs `one_of [and,or,not]` |
| `system/tree/extract-request` | TREE | `paths`/`prefix` typed `primitive/string` vs `system/tree/path` |
| `system/tree/snapshot-request` | TREE | `prefix` typed `primitive/string` vs `system/tree/path` |
| `system/protocol/inbox/delivery` | INBOX | `result` typed `primitive/any` vs `core/entity` *(already routed)* |

**The `op` cases deserve a note:** a bare `primitive/string` where the owner defines a closed enum means the
catalogue **admits values the owner forbids**. Two implementations disagreeing on whether `op: "pow"` is valid is
a cross-peer divergence that no local test finds — and per this corpus's own convention, an unknown enum value
should fail loud rather than be ignored (`SDK-OPERATIONS` §11.6.9's precedent).

## 3. What is *not* wrong — four false positives, checked and dismissed

The parser flagged these; reading them shows they are **illustrative example instances**, not competing
definitions, and they are fine:

| Flagged | Why it is fine |
|---|---|
| `system/capability/grant-entry` in `DOMAIN-LOCAL-FILES.md` | An example grant (`handlers: {include: ["local/files"]}`), not a type definition. |
| `system/config` in `EXTENSION-SUBSCRIPTION.md` | Example instance (`concern: "subscription"`). |
| `system/handler` in `EXTENSION-REVISION.md` | Example handler entity (`name: "version"`). |
| `system/signature` in `EXTENSION-ENCRYPTION.md` | Placeholder shape (`<bytes>`, `<Ed25519 / Ed448 / …>`). |

Plus `SPECIFICATION-FORMAT.md`'s teaching examples (`grant-entry`, `my-handler/operations`, `system/example`),
which are identical in both repos by design.

**21 of the 38 duplicated declarations are byte-identical to their owner** — including all of
`system/content/*`, most of `system/tree/diff*`, and `system/protocol/inbox/notification`. **The catalogue is
mostly right**, which is precisely why nobody noticed the parts that are not.

## 4. The one reverse-direction case `[severity 3]`

`system/type/field-spec` is **core-owned** (it *is* in the §9.5 floor) and **`SDK-OPERATIONS.md` restates it
informally** — an unwrapped sketch (`type_ref: system/type/name`, *"; exactly one of:"*, *"; plus modifiers"*)
missing seven of core's fields and all optionality markers.

It is visibly a summary rather than a competing definition, so the interop risk is low — but it is the same
restatement pattern, in the same file where `system/protocol/error` had already drifted and been fixed this arc.
**Fix: add the canonical pointer** (`SPECIFICATION-FORMAT.md` §8.4.2) rather than restate.

## 5. Why this happened, and the structural fix

**A catalogue is a cache, and this one has no invalidation.** The machine spec's completeness is a real feature —
one document listing every type is genuinely useful to an implementer. But a cached copy with no link to its
source and no gate comparing them **will** drift, and the surprising thing is not that 13 drifted but that 21 did
not.

**Every instance drifted in the same direction: the owner moved forward, the catalogue stayed.** That is the
signature of a copy nobody owns — the extension author edits their spec and has no reason to know a second copy
exists.

**Two viable fixes:**

| | Pros | Cons |
|---|---|---|
| **(a) Reference, don't restate.** The catalogue lists type *names* + owner + section link; field definitions live only in the owning spec. | Eliminates the class permanently. Consistent with `SPECIFICATION-FORMAT.md` §8.4.2, just folded. | The catalogue stops being a single readable document — its main value. |
| **(b) Keep the copies, add a gate.** A corpus linter check: any type declared in two specs must have identical field sets, or the build fails. | Keeps the catalogue's usefulness. Mechanical, and the script in this analysis is most of it. | Requires the gate to actually run in CI, and a canonical-owner annotation per type. |

**Recommended: (b), with (a) for cross-*repo* cases.** Within one repo a gate is cheap and preserves the
catalogue. Across repos — which is exactly the core↔arch boundary where all 13 of these live — a gate is
awkward (two repos, two CI runs, no shared build), so the cross-repo copies are the ones that should become
references. **The linter check is worth adding regardless**: it is ~40 lines, it found 13 real defects on first
run, and it cannot be fooled by prose review.

## 6. Cost and timing

- **Keystone: zero.** None of the drifted types is implemented in any generated peer — all report
  `absent / not-a-FAIL-if-absent`. `system/tree/snapshot`, `merge-request` and `extract-request` are not even
  mentioned in the reports.
- **Core Type Floor: untouched.** No drifted extension-owned type is in the 53-type floor. No opcode, no framing,
  no wire-core renumber.
- **Three impls:** unknown and **theirs to size**. The relevant question is which shape each built from — the
  owner spec or the catalogue. `system/tree/snapshot` is the one to check first, because the two shapes are not
  reconcilable and whichever an impl chose, the other is wrong.
- **Timing:** the same argument as the inbox rename, and stronger. **These are cheap now precisely because
  keystone has adopted none of them.** Every one that gets implemented before the fix converts a spec edit into a
  migration.

---

**Recommendation:** fold this into the existing upstream handoff
(`PROPOSAL-INBOX-TYPE-NAMESPACE-CORRECTION`) as a second part. Same repo, same section area, same reviewers, and
the inbox `result` drift is literally item 9 in §2's table — routing them separately would send the same people
the same file twice.
