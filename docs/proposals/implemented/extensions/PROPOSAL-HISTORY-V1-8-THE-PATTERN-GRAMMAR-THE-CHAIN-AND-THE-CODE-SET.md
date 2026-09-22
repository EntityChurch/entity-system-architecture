# PROPOSAL — `EXTENSION-HISTORY` v1.8: three of its own examples are not patterns, the prune algorithm rewrites an audit chain, and its code set does not exist

**Status:** FOLDED (2026-09-08) — `EXTENSION-HISTORY` **v1.8**, plus the §6a sweep corrections in `ENTITY-SYSTEM-REFERENCE`, `EXTENSION-TYPE` and `EXTENSION-REVISION`. Moved to `implemented/`.
**Target:** `specs/extensions/EXTENSION-HISTORY.md` v1.7 → **v1.8**
**Provenance:** four defects filed by the seat generating this extension, from emitting code
rather than from reading; plus one they did not report, found while verifying the first.
**Scope:** one extension. **No wire change. No core change.** The core pattern grammar is
confirmed correct as landed and is not touched.

---

## 0. The shape of this, in one paragraph

**Four of the five items below were found by building the extension, and none of them is a
disagreement about intent.** Three are places where the document's own worked examples use
pattern spellings the core grammar does not admit — so the examples cannot be constructed, and
one of them is a `REQUIRED` conformance vector. One is an algorithm that describes mutating
immutable entities. One is a missing error-code table whose absence leaves a security-relevant
refusal undefined.

**The fifth was found here, verifying the fourth, and it is the same defect one clause over.**

---

## 1. The pattern grammar — what the core actually admits `[confirmed, not changed]`

Read from the landed core specification §5.4 rather than from memory. `matches_pattern`
recognises exactly four cases, in order: bare `*`; a leading `/*/` peer wildcard; a trailing
`/*` subtree; **otherwise exact string comparison.** Its own enumeration of pattern types in
entity data lists `*`, `pattern/*`, `pattern`, `/*/*`, `/*/pattern/*`, `/{P}/pattern/*` — and
nothing else. `canonicalize` **rejects a bare leading `*/` outright**, with the error
*"ambiguous: use /\*/rest for peer wildcard patterns."*

> **The wildcard is in the peer position or at the tail. There is no mid-path wildcard, and
> there is no multi-wildcard form.** `[ruled 2026-09-08]` This is not a deprecation and not a
> statement that the grammar can never widen — it is what the grammar is today, and every
> document in this corpus is to be read against it.

**A mid-path `*` is therefore not a wildcard at all.** It falls through to the exact-match
branch and matches only the literal string containing an asterisk. The failure is silent in
the worst way: the pattern is well-formed, it stores, it canonicalizes, it resolves, and it
never selects.

---

## 2. Three examples in this document are not patterns

| § | Spelling | What it is |
|---|---|---|
| §2.2 pattern table | `*/project/*` | **rejected by `canonicalize`** — the exact spelling the core errors on |
| §2.2 type comment | `*/project/*` | same |
| §6.2 key-3 justification | `a/*/c` | mid-path — matches one literal string |
| §6.2 worked pair | `a/*/c/*/e` | mid-path, twice |
| §6.2 peer-ID note | `*/project/*` | rejected spelling |

### 2.1 §2.2 — the table and the pseudocode disagree, and the pseudocode loses

§2.2's table gives `*/project/*` meaning *"peer wildcard — `project/*` subtree in any peer's
namespace,"* which is the right **intent** and the wrong **spelling**. Its
`canonicalize_pattern` then passes that spelling through unchanged:

```
first_segment = first_segment_of(pattern)
if first_segment == "*":
  return pattern                          ; peer wildcard — already cross-peer
```

So the pattern reaches the matcher in a form the matcher does not recognise, falls through to
exact comparison, and matches nothing. **A peer that configured cross-peer history records
none and reports no error.**

**Fix:** `canonicalize_pattern` emits `/*/` + remainder for the leading-star case, and the
table shows the canonical spelling. The intent is unchanged; only the spelling that reaches
the matcher changes.

---

## 3. The specificity rule — the MUST is right, both of its justifications are not

### 3.1 What v1.7 says

v1.7 landed a `[MUST]` that `pattern_specificity` is a **two-key** comparison and selection
**MUST NOT** depend on enumeration order, on the reasoning that *"a scalar cannot carry a
two-key order"* — an implementation collapsing the keys into one number manufactures ties and
resolves them by store order. It adds a third key (lexicographic) justified as making the
order **total**, *"which keys 1–2 are not."* And it pins a `REQUIRED` vector,
`HIST-CONFIG-SPECIFICITY-1`, on the worked pair `a/b/c/d` against `a/*/c/*/e`.

### 3.2 The vector cannot be constructed

`a/*/c/*/e` is not a pattern. Under §5.4 it is an exact string, so it matches only itself,
while `a/b/c/d` is exact at depth 4. **No path matches both, so the vector's instruction to
"write at a path both match" has no satisfying input.**

### 3.3 And the divergence it defends against is unreachable — enumerated, not argued

Fix an absolute path `/P/s₁…sₙ`. **Every pattern §5.4 admits that can match it** is one of:

| Form | literals | depth | scalar (`literals + depth`) |
|---|---|---|---|
| exact `/P/s₁…sₙ` | `n+1` | `n+1` | `2n+2` — even |
| `/P/s₁…s_k/*`, `0 ≤ k < n` | `k+1` | `k+2` | `2k+3` — **odd** |
| `/*/s₁…s_j/*`, `0 ≤ j < n` | `j` | `j+2` | `2j+2` — even |

*(`/P/*` is the `k=0` row — it is what bare `*` canonicalizes to. `/*/*` is the `j=0` row.)*

**A scalar tie that the tuple would break needs equal scalars and unequal literal counts.**
Parity kills every cross-family case: the `/P/…/*` family is odd, the other two are even. The
one same-parity pair is *exact* against `/*/…/*`, which needs `2n+2 = 2j+2`, so `j = n` — and
`j ≤ n−1`, because a trailing-`/*` prefix match requires the path to be strictly deeper than
the prefix. **So no scalar-vs-tuple divergence is constructible.**

**The same enumeration settles key 3, and this is the item that was not filed.** A key-1 and
key-2 tie needs two rows agreeing on both columns. `/P/…/*` gives `(k+1, k+2)` and `/*/…/*`
gives `(j, j+2)`; equality on the second forces `j = k`, which contradicts the first. *Exact*
ties neither (`n+1 = k+1` forces `k = n`; `n+1 = j` and `n+1 = j+2` are inconsistent). **Keys
1–2 are already total over the patterns §5.4 admits — so key 3's stated justification is
false, and its example `a/*/c` is not a pattern either.**

*(Verified by exhaustive enumeration of the candidate set for path depths 1–6: zero key-1+key-2
ties, zero scalar-vs-tuple divergences.)*

### 3.4 What to do about it — keep the rule, fix what it claims

**The two-key MUST stays.** A rule that is right for a reason that has not arrived yet is
cheap, and the grammar is explicitly not closed against widening. **What changes is that the
document stops claiming a live failure it cannot exhibit.**

- The two-key comparison and the order-independence MUST are **retained**, restated as
  **forward-looking**: under today's grammar the scalar and the tuple agree on every
  constructible pair, and the tuple is required so that they still agree if the grammar gains
  a form where they would not.
- **Key 3 is retained and its justification is corrected.** It is a defensive total-order
  tiebreak that no pair §5.4 admits reaches — not the thing that makes the order total, which
  keys 1–2 already do.
- **The `REQUIRED` vector is replaced, not withdrawn.** The property worth gating is real and
  a naive implementation genuinely fails it: **selection is by specificity rather than by
  enumeration order.** Two constructible vectors replace the unconstructible one.

### 3.5 The replacement vectors

**`HIST-CONFIG-SPECIFICITY-1` (REQUIRED) — key 1, and first-match-wins fails it.**
Configure `a/b/*` and `a/*` with distinguishable settings; write at `a/b/c`. Both match; key 1
is 3 against 2, so `a/b/*` is selected. **Write the two configs in both insertion orders and
assert the same selection both times** — an implementation returning the first match passes
one order by luck.

**`HIST-CONFIG-SPECIFICITY-2` (REQUIRED) — key 2 is load-bearing.**
Configure `*` (→ `/{local}/*`, 1 literal, depth 2) and `/*/a/*` (1 literal, depth 3); write at
`a/b`. **Key 1 ties at 1 and only key 2 separates them**, selecting `/*/a/*`. Both insertion
orders. This is the pair that fails an implementation which compares literal counts alone.

---

## 4. §3.3's pruning algorithm describes mutating immutable entities

§3.3 says the chain is severed such that *"the old transition keeps its previous field
(immutable in content store), but it's no longer reachable from the head."*

**Both clauses cannot hold.** Reachability from the head **is carried by** the `previous`
fields. Making the `(N+1)`-th transition unreachable requires a version of the `N`-th without
its `previous` — a different entity with a different content hash — which changes what the
`(N−1)`-th points at, and so on to the head. **Truncating a hash-linked chain means rewriting
all of it, and rewriting an audit chain is the opposite of what an audit chain is for.**

**Resolution: pruning is GC-side and the chain is never rewritten.** `max_depth` is a
retention *policy*, not a mutation. The chain stays intact and fully linked; transitions
beyond `max_depth` cease to be **retained** and become collectable under the peer's ordinary
collection policy, exactly as any other unreferenced content is. The head pointer is **never**
re-pointed by pruning — re-pointing it would orphan the *newest* entries, which is the
opposite of the intent.

This keeps `max_depth` implementable at `SHOULD` (§9.1) and is what a peer that declines to
implement it already does: retain everything. **A peer that implements no collection at all is
conformant** — `max_depth` bounds what must be kept, never what must be destroyed.

---

## 5. There is no code set, and one missing code guards a security boundary

The handler emits six codes. Four resolve to the core's enumerated rows. Two do not:

| Code | Status | Defined |
|---|---|---|
| `unexpected_params` | 400 | core, enumerated |
| `capability_denied` | 403 | core, 403 default |
| `unsupported_operation` | 501 | core, 501 default |
| `storage_error` | 500 | core, enumerated |
| `path_required` | 400 | **nowhere in a code set** — see the separate item on that code |
| `not_in_history` | 404 | **§4.3.2's pseudocode only** |

**`not_in_history` is the one that matters.** §7.5 makes it the refusal that stops `rollback`
being an unrestricted write primitive — *"you can only restore entities that were previously
at that path."* The spelling is unambiguous because §4.3.2 writes it out; what is missing is a
**table**, so the core's default-code rule leaves it a more-specific 404 that no code set
defines. The core's 404 default is `handler_not_found`, which would be **actively wrong** for
this condition.

**Fix: an Appendix A on the same model as the content extension's**, carrying the six rows,
each with the authority for its status named.

**`path_required` is deliberately listed and deliberately not resolved here.** It is the
second extension to emit a code its own specification `MUST`s and no code set defines; that is
a corpus-level question about whether the core's 400 row is an open category or a closed set,
and it is tracked and answered separately. This proposal's Appendix A carries the row and
names that item as its authority, so this extension is not blocked on it.

---

## 6. The edit list

| # | § | Change |
|---|---|---|
| A | §2.2 | Table + type comment: `*/project/*` → `/*/project/*` |
| B | §2.2 | `canonicalize_pattern` emits `/*/` + remainder for the leading-star case |
| C | §6.2 | Key-3 justification corrected — keys 1–2 are already total; key 3 is defensive |
| D | §6.2 | Unconstructible worked pair replaced; the MUST restated as forward-looking |
| E | §6.2 | `HIST-CONFIG-SPECIFICITY-1` replaced; `-2` added; both REQUIRED and constructible |
| F | §3.3 | Pruning is retention policy, GC-side; the chain is never rewritten or re-pointed |
| G | new Appendix A | Six-row error-code table |
| H | §9.1 | Rows for the code set and for specificity selection |
| I | header | v1.7 → v1.8 |

**Nothing here changes what a conforming peer must *do*** except in the two places where v1.7
asked for something impossible — the unconstructible vector, and the prune algorithm. Both are
replaced with the implementable form of the same intent.

---

## 6a. The corpus sweep — this extension was not the only home `[added 2026-09-08]`

**A rule has every home it is stated in, not the one the finding names.** Sweeping the whole
published surface for both invalid spellings — a `*` in a middle segment, and the bare leading
`*/` the core rejects — returns hits in five further documents. **Three are defects and two are
not, and the difference is whether the string is a pattern or a placeholder.**

**Defects — each is a capability or grant pattern, so it is fed to `matches_pattern`:**

| Document | Was | Now |
|---|---|---|
| the system reference's capability-pattern table | `*/system/type/*`, labelled *"Peer wildcard"* | `/*/system/type/*` |
| the type extension | *"The type handler's grant must cover `*/system/type/*`"* | `/*/system/type/*`, with the spelling rule cited |
| the revision extension | *"SHOULD NOT hold direct `put` grants for `system/revision/*/config`"* | restated as intent — *"reaching any prefix's `config` binding"* |

**The first two are the same defect this proposal fixes in §2.2, in a document that shares none
of this extension's vocabulary** — and one of them is the capability-pattern reference table,
which is where a reader goes to learn the spelling. **A grant using the rejected form fails
closed**, so nothing was silently over-authorized; what was broken is that the documented grant
for cross-peer type operations cannot be expressed. The third is a `SHOULD NOT` whose pattern
matches only a literal string, so the advice was unstatable rather than wrong.

**Not defects, recorded so the sweep is not re-run:** the group extension and its guide
describe *"walks all `system/group/*/members/*` entries"* — a store traversal in prose where
`*` stands for an id, not a pattern handed to the matcher — and a browser deployment runbook
uses `*/manifest/*` as a **service-worker URL matcher**, a different matching domain entirely.
**`*` as an informal placeholder for a variable segment is ordinary prose.** The test is
whether the string reaches `matches_pattern`.

## 7. What this does not do

- **No core change.** §5.4's grammar is confirmed as landed and correct. The mid-path form is
  absent by design, and this proposal records that rather than adding it.
- **Does not resolve the `path_required` code-set question.** Named, carried, tracked
  elsewhere.
- **Does not touch the recording scope** — that a peer configured with `*` records its own
  protocol writes is a real finding and is a separate proposal, because the fix is a choice
  between two surfaces and this one is unambiguous.
- **Does not add `max_depth` enforcement.** It stays `SHOULD`.
