# PROPOSAL — HISTORY config selection is a two-key order, and a scalar cannot carry it

**Status:** **DRAFT — folded at authoring.** `specs/extensions/EXTENSION-HISTORY.md` **v1.6 → v1.7**;
§6 is the verified delta.
**Target:** `EXTENSION-HISTORY.md` §6.2 (`find_history_config`), against the order §2.2 already states.
**Tier:** `extensions/` — implemented by go · rust · py.
**Scope:** how `find_history_config` picks among matching configs. **No change to §2.2's stated
order**, to what history records, or to the config schema.
**Source:** found by arch on 2026-08-18 while reviewing `entity-core-go` `5b86b2b`'s implementation of
`EXTENSION-REVISION` R15 — **not routed by any seat.** go's revision fix left an unrelated
`patternSpecificity` in `ext/history/config.go`, which is what prompted the corpus sweep.
**Cohort review:** none. The divergence below is source-read, not reported.

---

## 1. The finding — R15's defect, in a second document, already three-way live

`EXTENSION-REVISION` R15 pinned `pattern_specificity` because §5.1 called it and the corpus defined it
nowhere. **A sweep for that shape returns exactly two sites**, and the second was not fixed:

```
$ grep -rn "best_specificity\|pattern_specificity" specs/
specs/extensions/EXTENSION-HISTORY.md:652   best_specificity = -1
specs/extensions/EXTENSION-HISTORY.md:658     specificity = pattern_specificity(pattern)
specs/extensions/EXTENSION-HISTORY.md:659     if specificity > best_specificity:
specs/extensions/EXTENSION-REVISION.md:2961  best_specificity = -1        (pinned v3.12)
```

**HISTORY is better off than REVISION was in one respect and identical in the other.** §2.2 *does*
define the order — *"(1) number of literal (non-wildcard) segments, then (2) total segment depth"* —
so this is not an undefined referent. But §6.2 compares it as **a single value with `>`**, over
`list_entities` output whose ordering is unspecified. **A scalar cannot carry a two-key order.**

## 2. Measured — go and rust collapse the keys; py does not

Source-read 2026-08-18 at the commits named:

| Seat | commit | symbol · path | Returns |
|---|---|---|---|
| `entity-core-py` | `9ba439b` | `pattern_specificity` · `.../entity_handlers/history.py` | **`(literal_count, total_depth)`** — a tuple; faithful to §2.2 |
| `entity-core-go` | `5b86b2b` | `patternSpecificity` · `ext/history/config.go` | `int` — 2 per literal segment, 1 per wildcard |
| `entity-core-rust` | `a701e13` | `pattern_specificity` · `extensions/history/src/engine.rs` | `u32` — **byte-identical scoring to go** |

**The witness:** `a/b/c/d` (4 literal, depth 4) against `a/*/c/*/e` (3 literal, depth 5).

- **py:** `(4,4)` vs `(3,5)` → key 1 decides, **`a/b/c/d` wins.**
- **go and rust:** `4×2 = 8` vs `3×2 + 2×1 = 8` → **tie**, resolved by whichever the store listed first.

**py is conformant to §2.2 as written and the other two are not** — but §2.2 never said "compare as a
tuple," and a reader who wants one comparable number reaches the go/rust scoring naturally. The spec
under-specified and two of three took the available reading.

**Two of three agreeing is not evidence here.** go and rust score identically because 2-per-literal /
1-per-wildcard is the obvious scalarization, not because they validated it against each other.

## 3. Severity — lower than R15, and worth stating honestly

This is **not** hash-determining. History is opt-in, per-peer recorded state; a differently-selected
config changes what a peer records about its own transitions, not the content of any entity or trie
root. **It is a determinism defect, not a convergence defect.**

It still gets pinned, for the reason `1664a67` recorded: *"the divergence is caused by the surface
being unpinned, so severity raises the priority of closing it and lowers the acceptability of leaving
it open."* Low severity lowers the priority; it does not make an unpinned selection acceptable. And
the cost here is one comparison.

## 4. The fix — complete the order §2.2 chose; do not invent one

**This is deliberately not R15.** R15 had to choose an order because none existed, and it recorded
anchored-above-floating as arch's judgment. **Here §2.2's keys stand exactly as written** and the fold
adds only what makes them usable:

| Key | Value | Source |
|---|---|---|
| 1 | count of literal (non-`*`) segments — higher wins | §2.2, unchanged |
| 2 | total segment depth — higher wins | §2.2, unchanged |
| 3 | lexicographic byte order on the canonicalized pattern — lower wins | **new — totality only** |

**Key 3 is the whole normative addition.** Keys 1–2 are not total: `a/*/c` and `a/b/*` are both 2
literal segments at depth 3. Lexicographic order is peer-independent, so every conformant peer picks
the same config.

**§2.2's peer-ID sentence needs no key and is not changed.** *"An explicit peer ID is more specific
than a wildcard peer ID at the same depth"* is already implied by key 1 — an explicit peer segment is
literal, a `*` peer segment is not. Verified against both scoring schemes: `/{peerA}/project/*` ranks
above `*/project/*` under the tuple **and** under go/rust's scalar. **That sentence was never the
defect and is not touched.**

## 5. Rejected alternative

**Share one `pattern_specificity` with `EXTENSION-REVISION`.** They are different functions over
different domains — tree paths with segment structure here, merge patterns in four closed forms there
— and REVISION v3.12 now carries a note saying so. An implementation that shares one is wrong at
whichever site it did not come from. **The shared *name* across two specs is a real footgun** and is
the spec-level twin of the two-`globMatch` collision `entity-core-go` flagged in their own tree; it is
recorded in both documents rather than renamed, because renaming a called referent in a landed spec is
a larger change than the defect warrants.

## 6. Spec delta — verified against the tree before the version moved (L3)

| # | § | Change | Verified |
|---|---|---|---|
| D1 | §6.2 | `[MUST, v1.7]` — two-key tuple comparison; selection MUST NOT depend on enumeration order | ✅ present |
| D2 | §6.2 | Key 3 (lexicographic) for totality, with the `a/*/c` vs `a/b/*` justification | ✅ present |
| D3 | §6.2 | The peer-ID sentence is subsumed by key 1 — stated, not re-specified | ✅ present |
| D4 | §6.2 | The `a/b/c/d` vs `a/*/c/*/e` worked pair that separates tuple from scalar | ✅ present |
| D5 | §6.2 | `HIST-CONFIG-SPECIFICITY-1`, driven in **both** insertion orders | ✅ present |
| D6 | header | **1.6 → 1.7** | ✅ present |

**No wire change, no new entity type, no new capability, no new error code.**

## 7. Cohort impact — **routed, not only tabled** (L13)

Delivered in `docs/status/ROUTING-2026-08-18-n-*`.

| Seat | State at read | Owed |
|---|---|---|
| `entity-core-py` | `9ba439b` | **Keys 1–2 already correct** — your tuple is the faithful reading. Add key 3 and `HIST-CONFIG-SPECIFICITY-1` |
| `entity-core-go` | `5b86b2b` | `patternSpecificity` · `ext/history/config.go` → tuple compare + key 3, + the vector |
| `entity-core-rust` | `a701e13` | `pattern_specificity` · `extensions/history/src/engine.rs` → same |

**`HIST-CONFIG-SPECIFICITY-1` must be driven in both insertion orders.** A peer that ties resolves by
enumeration and passes one order by luck — which is exactly how this survived three implementations
and a full conformance suite.
