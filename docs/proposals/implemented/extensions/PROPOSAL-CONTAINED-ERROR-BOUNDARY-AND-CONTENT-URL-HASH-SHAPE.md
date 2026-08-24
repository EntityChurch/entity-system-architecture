# PROPOSAL — the contained-error boundary, and the content-URL hash shape

**Status:** FOLDED 2026-08-21 — all five deltas verified against the tree before this marker (L3)
**Specs:** `EXTENSION-COMPUTE` 3.25 → 3.26 (§2.3, §3.5, §4.1) · `EXTENSION-SUBSTITUTE` 1.2 → 1.3 (§7)
**Origin:** `entity-core-go` spec-issues **`2026-08-20-f`** and **`2026-08-20-g`**, both raised while
building against landed rulings. `-g` was independently found by `entity-workbench-go` from its own
consumer.
**Class:** two reconciliations. **Part A is arch's defect** — a v3.25 ruling invalidated a v3.23
premise and did not carry the consequence. **Part B is a definition that exists in one spec and was
omitted by the spec that restates its construction.**

---

## Part A — how a CONTAINED `compute/error` materializes (`2026-08-20-f`)

### §A1 The gap, and it is exactly what the discriminator was built to find

v3.23 **ruling B** removed `compute/error` from §4.1's construct-materialization set and from §2.3 N1's
list, with this reasoning:

> *"an error reaching any of them **short-circuits** … so **it is never placed into the data** and N1
> never applies to it. The listing was unreachable."*

**That premise was true at v3.23 and v3.25 made it false.** C-4 created **data positions** — `assoc`'s
`value`, `concat`'s elements, `group-by`'s `members` — where an error is **contained in a value** rather
than consumed. So an error now legitimately reaches materialization *inside an array*, and go's
`materialize()` guard — correct for its era — rejects it:

```
capture scope: internal: compute/error reached materialize() —
a §4.1 is_error short-circuit was missed (COMPUTE v3.23 ruling B)
```

**Arch wrote both rulings and did not reconcile them.** `GUIDE-CONFORMANCE` §7c.6's CV-4a and CV-5 are
the two vectors that sit exactly on the new positions, and they are unemittable — **the C-4
discriminator surfaced a hash-determining gap a uniform rule would have sailed past**, which is what it
was built for, one level earlier than expected.

### §A2 The ruling — code-only, by bare hash, and ruling B is scoped rather than reversed

**(1) Form: the contained error materializes CODE-ONLY, referenced by a bare `system/hash`.** Both
halves are landed text applied, not new policy:

- **Code-only** — §2.4 and `GUIDE-CONFORMANCE` §7c.3: *a materialized `compute/error` is content-hashed
  over `code` alone*; `message`/`at`/`expression` are in-flight diagnostics and **not** part of the
  materialized error boundary. **This is the load-bearing half:** if the contained element carried
  `message`, two conformant peers whose diagnostics differ would produce **different bytes for the
  containing array**, so CV-4a and CV-5 would fork cross-impl on a string neither spec pins. go's
  question 3 asks for exactly this confirmation and the answer is yes, for exactly that reason.
- **By bare hash** — §2.3 N1 and §4.1's materialization rule: an entity-valued thing placed into another
  entity's data is *"referenced by a **bare `system/hash`** (the value recursively materialized and
  stored)"*. A contained error is an entity-valued array element. **No new mechanism**, and no `kind`
  tag (those stay confined to `compute/scope`).

**(2) Reconciliation: go's reading is right and is adopted.** Ruling B's guard is a **consumption-site**
invariant. Its true content is *an error never enters the data **by way of a consuming expression***,
because every consuming expression short-circuits first. **A contained data-position materialization is
a new, legitimate path**, so ruling B is **scoped, not reversed**: `compute/error` returns to the
materialization set **only** for the §3.5 v3.25 data positions.

**The scope boundary is exact and narrow, and stating it is what keeps ruling B's teeth everywhere it
had them.** `compute/construct`, `field`, `arithmetic`, `compare`, `logic`, `apply`'s consumed operands
and `if`'s condition **all still short-circuit** — §7.2 is untouched. **The contained set is exactly
three positions**: `assoc.value`, `concat` elements, `group-by` members. **A `compute/error` reaching
`materialize()` from anywhere else is still the bug ruling B named**, and an implementation should keep
that guard and add a carve-out rather than delete it.

**(3) Anti-vacuity: confirmed**, and it is why (1) is code-only.

### §A3 What go did, which is the disposition arch would have asked for

They seeded the **7 emittable arms** (CV-1, CV-2×2, CV-3×2, CV-4b, CV-6), held CV-4a and CV-5 in
`v325BlockedOnMaterialization` **unseeded** — *"seeding a vector go cannot emit would break the emit
stage"* — and pinned the blocked state as teeth (`TestV325ContainedErrorBoundaryIsBlocked`) so it
**flips when this ruling lands**. They did **not** unilaterally change `materialize()`, on the grounds
that it is a hash-determining path with rust/py not yet on v3.25 to cross-verify.

**That is the correct posture and it is worth naming as the reference one:** a blocked vector pinned as
a failing test that flips on the fix is a **measurement**, where a comment saying "arch owes a ruling"
is a claim.

---

## Part B — the content-URL hash shape (`2026-08-20-g`)

### §B1 The disagreement, and where the 64-char form actually comes from

A content hash `H` renders two ways in a content-fetch URL:

- **66-char, format byte included** — `hex(H.Bytes())`, e.g. `006a22fe…` (`00` = ECFv1-SHA-256).
- **64-char, digest only** — `hex(H.EffectiveDigest())`.

go serves and dials **66** on the Amendment-5 poll surface, and `core/types.BuildContentURL` builds
**64** for the substitute surface, citing *"STORAGE-SUBSTITUTE-HTTP §3-RES.2 and the §10.1 examples."*
Both seats are internally consistent; **a fleet-wide helper cannot span them.**

**Checked before ruling, and it changes the weight of one side.** `STORAGE-SUBSTITUTE-HTTP` **is not a
spec in this corpus.** Searched by filename and by content across the whole checkout: no such spec
document exists anywhere. The real artifact is
**`PROPOSAL-EXTENSION-STORAGE-SUBSTITUTE-HTTP.md`** — an **implemented proposal**, living only in the
**legacy** tree, and our own `RUNBOOK-CDN-BROWSER-DEPLOYMENT` §150 cites it correctly *as a proposal*.

**So the 64-char form's basis is examples in a folded proposal that did not cross the V8 split.** Per
this repo's lifecycle a proposal is **rationale**; the normative result is what was folded. And
`EXTENSION-SUBSTITUTE` §1 already carries the precedent for exactly this move: *"Earlier draft text in
the source HTTP proposal conflated the two; that text is **superseded by this spec**."*

*(Method note: the recursive grep that first searched for this returned **zero hits on a file this
session had already read**, because the tool silently filters. The absence was re-established with
`find | xargs grep` before any of the above was written. A zero from a search that cannot see the tree
is could-not-look, not evidence — L7's standing rule, and it nearly produced a confident wrong claim
that the document did not exist **at all**, when in fact it exists and is a proposal.)*

### §B2 The ruling — ONE convention, 66-char, and NETWORK already said so

**`EXTENSION-NETWORK` §6.5.3.1 settles it in landed text, explicitly and with a universality clause:**

> *"Sharded `content_layout` … slices the **same `{hash}` hex defined above**. Slices are positions in
> that string, counted from the left … `[0:2]` = the **format-code byte** (`00` for ECFv1-SHA-256, `01`
> for ECFv1-SHA-384) = the **algorithm partition** … **One definition of `{hash}` everywhere — no
> separate digest-sliced layout family.**"*

**The layout enum and the hash shape are coupled, and that coupling is the decisive argument** — not a
preference between two renderings. Under the 64-char form, `sharded-2-flat`'s `[0:2]` slices **the first
digest byte** instead of the format code, and the **algorithm-partition property evaporates** — the
stated rationale (*"it diversifies the instant a non-SHA-256 hash ships"*) becomes false, and a
deployment that adds SHA-384 silently collides two algorithms into one bucket family. **Choosing the
64-char form does not merely pick a different string; it breaks a property NETWORK spent a paragraph
establishing.**

**Ruled: one convention, fleet-wide — `{hash}` is `hex(H.Bytes())`, format byte included.**

### §B3 Why it was ambiguous, and the fix is the omission rather than a contradiction

**`EXTENSION-SUBSTITUTE` §7 restates the URL construction and never restates the definition.** It gives
`flat`/`sharded-2-flat`/`sharded-2-4` in terms of `{hash}` and `{hash[0:2]}` — **and never says what
`{hash}` is.** So a reader working from SUBSTITUTE alone has no definition and reaches for the intuitive
one (the digest), which is precisely what happened.

**Neither document contradicts the other. One defines and the other omits**, and the omission is
invisible because the construction reads as complete. **Same shape as `EXTENSION-REGISTRY`'s F3** — a
unit declared in §3, never restated in §6a, so a reader working from §6a misses it — which is a second
instance of one failure mode and is recorded as such.

---

## §5 Deltas

| # | Spec | Section | Delta |
|---|---|---|---|
| **A1** | `EXTENSION-COMPUTE` | §2.3 N1 / §4.1 | `compute/error` returns to the materialization set **scoped to the §3.5 v3.25 data positions only**; materializes **code-only**, referenced by a **bare `system/hash`** per N1 |
| **A2** | `EXTENSION-COMPUTE` | §4.1 (ruling B) | Scope ruling B: *"never reaches `materialize()`"* is a **consumption-site** invariant. Name the three contained positions as the carve-out; everything in §7.2 still short-circuits |
| **A3** | `EXTENSION-COMPUTE` | §3.5 | The flow-through table's `contain` rows gain the boundary form, so an implementer meets it where the rule is stated |
| **B1** | `EXTENSION-SUBSTITUTE` | §7 | Define `{hash}` as `hex(H.Bytes())` — format byte included — citing NETWORK §6.5.3.1's *"one definition everywhere"*; state that the source proposal's 64-char examples are superseded |
| **B2** | `EXTENSION-SUBSTITUTE` | §7 | State the coupling: the shard slices are positions in **that** string, so `[0:2]` is the algorithm partition. A digest-only rendering breaks it |

**Versions: `EXTENSION-COMPUTE` 3.25 → 3.26 · `EXTENSION-SUBSTITUTE` 1.2 → 1.3.** Both change bytes a
conformant peer produces or fetches, so the cohort must see the text moved.

## §6 Open items

**None.** Recorded explicitly per L9.

## §7 Cohort impact

| Seat | Owed |
|---|---|
| `entity-core-go` | **A:** carve out `materialize()`'s guard, emit CV-4a/CV-5, re-seed, **re-freeze + re-pin the golden SHA** (their own stated sequence). `TestV325ContainedErrorBoundaryIsBlocked` flips. **B:** no change owed — both surfaces are correct against their own text today; `BuildContentURL` becomes fleet-wide once it renders 66, and `outbound.go`'s bypass comment can then collapse as it predicts |
| `entity-core-rust`, `entity-core-py` | v3.26, not v3.25 — the contained-error boundary is part of the corner set they are building this cycle |
| `entity-workbench-go` | **B closes their finding** — `TestContentURLUsesWireHexNotDigestHex` is now the spec's rule, not a local pin. Their consumer was right |
| arch | Pull `PROPOSAL-EXTENSION-STORAGE-SUBSTITUTE-HTTP` into the corpus or retire the citation — **T2 corpus item**, filed |

**Routing: one packet, to `entity-core-go`**, with the rust/py and workbench-go sections written to be
relayed.
