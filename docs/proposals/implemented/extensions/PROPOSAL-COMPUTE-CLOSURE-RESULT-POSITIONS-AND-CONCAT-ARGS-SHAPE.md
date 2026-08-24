# PROPOSAL — the closure-result positions, `concat-args`' evaluated shape, and entity-key equality bytes

**Status:** IMPLEMENTED — folded 2026-08-21, `EXTENSION-COMPUTE` **3.26 → 3.27**, D1–D8
(D7 is a rollout ruling and carries no spec text; every other row verified against the tree before this
marker, per L3) · **Spec:** `EXTENSION-COMPUTE` §3.5, §2.3 N1, §2.4, §7.1, §7.2, §10.1, §11.6
**Raised by:** `entity-core-go` spec-issue `2026-08-21-a`, concurring with items `entity-core-rust`
and `entity-core-py` both routed on their v3.24–v3.26 landings. **All three seats measured; none
converged unilaterally** — each of these forks a boundary hash.
**Ruled by arch 2026-08-21.** Extended the same day with **§8** (the eval-limit dispositions),
**§9** (D3's rollout), **§10** (§7.1's walk) — and **§3.2's derivation is RETRACTED**, which is what
turned up §10.

> **Two of the corrections in this document are arch's own defects, and they are marked at the point
> of claim rather than tidied:** §2.3's *"propagate"* — one word with two readings in a disposition
> column, which cost `entity-core-go` a wrong `fold` — and §3.2's §7.1 derivation, which reasoned
> from this spec's pseudocode to a conclusion about three implementations' behaviour without opening
> any of the three trees.

---

## §1 What the v3.26 "exactly three" sentence did not settle

v3.26 §3.5 pins the contained set as **exactly three** — `assoc.value`, `concat` elements,
`group-by.members`. That sentence was written about the **four v3.25 collection primitives**, whose
positions the §3.5 table enumerates. **`map` / `filter` / `fold` are not in that table**, and they
predate it. They also place closure results into outputs, so the "exactly three" count was taken
over an incomplete enumeration.

The cohort found this the way this class is always found — by three seats measuring their own
behaviour and disagreeing about whether it was intentional.

---

## §2 Corner 1 — RULED: the defect is **provenance-dependence**, and it is already forbidden

### §2.1 What all three seats measured

go's behaviour (`ext/compute/builtins.go`, `builtinMap` / `invokeClosure`), which rust and py both
independently flagged:

| Closure result | Carried as | Outcome |
|---|---|---|
| **value-form** error (SA-1, or a lookup onto a stored error) | a normal return value | **appended to the output** — contained |
| **minted** error (a failing op inside the closure) | a language-level error return | **short-circuits the whole `map`** |

**That asymmetry is the finding, and it is not a `map` question.** Two closures that produce a
`compute/error` with the **same `code`** give different results depending only on *how the error came
to exist*.

### §2.2 The ruling, and it needs no new rule

**A `compute/error` behaves identically regardless of how it was produced. `[MUST]`**

This is not new spec. §2.4 already states it: *"only the materialized form is normative; the
in-flight representation is implementation-private."* §2.3 N1's construct-materialization note says
the same. **"Minted" versus "value-form" is exactly an in-flight representation** — in go it is the
difference between a `*ComputeError` Go-error and a value in the return slot — and the asymmetry lets
that private representation decide a **boundary-hash-determining** outcome. §2.4 forbids that today.

**So the seats exhibiting the asymmetry are non-conformant against v3.26, not against a new rule.**
Stated that way deliberately: it means no seat gets to treat this as a change of direction, and it
means the fix is not gated on this proposal landing.

### §2.3 Which way each position goes — §7.2's own words, applied

§7.2's short-circuit `[MUST]` binds *"all expression types that **consume values**."* Its `store`
worked example gives the other side: *"a builtin's write payload is not a consumed operand."*

**The distinction that resolves all three primitives: handing a value to a closure is a
*binding*, not a consumption. A primitive consumes only what it reads itself.**

| Primitive | Position | Does the primitive read it? | Class |
|---|---|---|---|
| `map` | element (into the closure) | no — bound into closure scope | pass through; the closure's own operators short-circuit *inside* the closure |
| **`map`** | **output element (closure result)** | **no — placed into the output array** | **contain** |
| `filter` | element (into the closure) | no — bound | pass through |
| **`filter`** | **predicate result** | **yes — read for truthiness (§4.5) to decide inclusion** | **short-circuit** |
| `fold` | element / accumulator (into the closure) | no — bound | pass through |
| **`fold`** | **final accumulator (the result)** | **no — returned** | **contain** |

- **`map`'s output element contains.** `map` never reads the closure's result; it places it. That is
  structurally the same position as `concat`'s element and `assoc`'s `value`, and it is the
  §7.2 `store`-payload case. It is also what §1.5's model requires: *"the same model as NaN
  propagation in IEEE 754"* is **element-wise** — `map(f, [1,2,3])` where `f` fails only on element 2
  yields `[a, E, c]`. Short-circuiting the whole array is exception semantics, which is the model
  §1.5 explicitly declined.
- **`filter`'s predicate result short-circuits.** It is read for truthiness, which makes it a
  consumed operand by §7.2's plain terms. It is the **identical** case to `group-by`'s derived key,
  and it fails the same way if contained: an error has no truth value, and coercing it to false
  **silently drops the element** — a well-formed wrong answer carrying no error, which is precisely
  the reasoning §3.5 already used to refuse clamping a negative `range(n)` to `[]`.
- **`fold`'s accumulator is CONTAINED, and a closure that ignores it RECOVERS.** `fold` binds the
  accumulator into the next closure invocation and never reads it. So an error accumulator — whether
  it arrived as `initial` or as a previous step's result — is passed onward as an ordinary bound
  value, and a closure that does not consult its accumulator returns a non-error and **the fold
  recovers**. `fold` **never aborts on an error accumulator.** That is correct under the
  error-as-value model, where errors are ordinary values a program may inspect (§4.1 `is_error`).
  *Recorded as the position of the three with the widest blast radius if wrong: it is the only one
  where the alternative reading (abort) produces a **different value**, not merely a different cost,
  and `CV-8d` is what pins it.*

  > **The word "propagate" is struck from this bullet and from every disposition column in this
  > proposal, and the reason is a measured defect `[2026-08-21]`.** The bullet above read *"fold's
  > accumulator **propagates**"* and the §2.1 table's disposition column read *"contain / propagate"*
  > — meaning *threads onward into the next call*. **`entity-core-go` read "propagate" as
  > *short-circuit*, which is the word's other standard meaning in exactly this domain**, and
  > implemented the abort. `entity-core-rust` and `entity-core-py` read it the other way. One word,
  > two readings, three seats, and the two readings differ in the produced **value**.
  >
  > **The routing packet that carried this bullet said, in its own §6, *"do not implement from this
  > packet's prose."* This is the sentence that warning was about, and the warning did not save it** —
  > a seat that is going to implement reads the disposition column, because that is where a
  > disposition is supposed to live. `entity-core-py`'s framing of the rule is better than the
  > incident: ***a one-word disposition column is where a two-reading word loses its second
  > reading.***
  >
  > **The defect is arch's, in arch's text, and it is fixed at the source rather than annotated.**
  > The vocabulary is now **contain / recover** and **short-circuit** — `propagate` names no
  > disposition in this proposal.

### §2.4 The count sentence is wrong and is replaced, not patched

**"The contained set is exactly three positions"** becomes **five**: `assoc.value`, `concat`
elements, `group-by.members`, **`map`'s output element**, **`fold`'s final accumulator**.

**Why the sentence is replaced rather than incremented.** *"Exactly three"* was an enumeration taken
over one table, and it read as a closed structural claim about the language. It will be wrong again
the next time a primitive with an output position lands. The replacement states the **rule** and
gives the enumeration as its current extension:

> A position is **contained** when the primitive **places** the value without reading it, and
> **consumed** when the primitive reads it to decide control flow, ordering, membership, or a write
> location. Today that is five contained positions: *(list)*. **A new primitive adds rows to this
> table by applying the rule, not by amending the count.**

*(This is L14's lesson on a spec rather than on a discipline: a rule written at the width of the
incident that produced it passes the case it was not written for.)*

---

## §3 Corner 2 — RULED: the **evaluated** shape is normative, and §7.1 is why

### §3.1 The divergence

§3.5 declares `system/compute/concat-args := { collections: {array_of: {type_ref: "system/hash"}} }`
— an **array of hashes**. **All three seats evaluate `collections` as a single hash resolving to an
array of arrays**, and go's own `ComputeConcatArgsData.Collections []hash.Hash` matches the
declaration while `builtinConcat` does not. The two shapes produce **different IR bytes for the same
logical concat**.

### §3.2 The ruling: tighten the declaration to the evaluated shape

```
system/compute/concat-args := {
  fields: { collections: {type_ref: "system/hash"} }   ; Hash of an expression evaluating to an array of arrays
}
```

> ### RETRACTED — the derivation this section shipped with was false, and it was arch's `[2026-08-21]`
>
> **The ruling below stands. The argument it was published with does not, and it is withdrawn at the
> point of claim rather than quietly replaced.** The original derivation read:
>
> > §7.1's `walk` descends into scalar `system/hash` fields and does not enter arrays. Under the
> > declared array-of-hashes shape, **every `compute/lookup/tree` inside every sub-collection of
> > every `concat` goes unregistered** — a reactive expression containing a `concat` silently never
> > re-fires.
> >
> > …and: **`concat-args` is the only array-of-hashes in the entire expression grammar.**
>
> **Both sentences are false, and each is false in its own way.**
>
> **(1) No implementation has that defect.** Read at `entity-core-go` `c1b0708` (`walkDepValue`,
> `ext/compute/engine.go`), `entity-core-rust` `2ee6bf7` (`walk_hash_fields`,
> `extensions/compute/src/walker.rs`) and `entity-core-py` `f09ae70` (`_walk_deps`,
> `entity_handlers/compute.py`): **all three descend into containers.** go and rust recurse over
> arrays and maps to unbounded depth; py handles the container shapes the current grammar actually
> uses. **The predicted silent-never-re-fires does not happen at any seat**, and it never would have
> — because the implementations are written against §7.1's **prose** rule, not its pseudocode.
>
> **(2) `concat-args` is not the outlier.** `compute/apply.args` is `{map_of: {type_ref:
> "system/hash"}}` — a **map** of hashes. `compute/let.bindings` is an array of maps each carrying
> `value: system/hash`. Both are in §2.1, both predate `concat` by many revisions, and **both are far
> more common in real expressions than `concat` will ever be.** The claim was made from a grep for
> `array_of` and published as a claim about the whole grammar — a negative asserted from a partial
> search, which `AGENTS-STANDARD` forbids by name.
>
> **The shape of the error, because it is worth more than the correction.** The derivation was
> reasoned from **the spec's own pseudocode**, and the conclusion drawn was about **implementation
> behaviour**. Those are different things, and the gap between them is exactly what §7.1's
> *"Conservative static collection"* paragraph occupies: the prose states *"all `compute/lookup/tree`
> paths reachable in the expression graph are registered,"* which is strictly stronger than the
> pseudocode beneath it. **Three implementations read the prose. Arch read the pseudocode and called
> it a defect in the cohort.** `AGENTS.md` **L8** — an artifact is not a conclusion about the thing
> it names — with the artifact being normative text this repo wrote, which is the one artifact class
> an arch session trusts without checking.
>
> **The cost had it shipped as written:** a routing packet telling three seats their reactive
> dependency registration was broken, when the actual defect is in the spec's pseudocode and is
> **D8** below.

**The derivation, re-taken: `concat` is the only collection primitive whose collection argument is
not an expression, and there is no reason for it to be the exception.**

Every sibling in §3.5 declares its collection input as **one scalar `system/hash` — a hash of an
expression that evaluates to an array**: `map-args.collection`, `filter-args.collection`,
`fold-args.collection`, `group-by-args.collection`, `assoc-args.collection`. `concat-args.collections`
alone declares a **literal array of hashes**, which is a different kind of thing: not "an expression
yielding the operand" but "the operand, written out."

That difference has a consequence, and it is the ruling's ground: **the declared shape freezes
`concat`'s arity at authoring time.** `concat` applied to a computed number of collections —
`concat` over the output of a `map`, over a `group-by`'s `members`, over anything whose length is not
known when the IR is written — **is inexpressible.** The evaluated shape costs nothing and admits it,
and it is the shape that makes `concat` compose with the four primitives it was adopted alongside.
A primitive introduced to make k-way joins possible should not be the one primitive whose k is a
constant.

**And it removes an outlier from a traversal that is about to be tightened.** D8 rules §7.1's walk to
descend into any container, which makes the declared shape merely redundant rather than dangerous —
but a grammar in which every reference field is a scalar hash is one whose traversal obligations can
be **stated and checked**, and D8 makes that statement. D3 is what keeps it true.

**What this does NOT rest on:** that three implementations agree. Three seats of one cohort agreeing
is cohort-consistency, not evidence (L18) — and it is worth saying plainly here, because the
convergence *looks* like the argument and is not. It is, however, legitimate corroboration that the
evaluated shape is implementable and cheap: no seat had to be persuaded into it.

### §3.3 What is preserved, stated so nobody reads a change into it

The §3.5 table's **`concat` | each `collection` | short-circuit** row is unchanged: the outer array's
items are the collections (consumed — go's **CV-7c**), and items *of* a sub-collection are elements
(contained — **CV-5**). **Both shapes support that distinction**, so it does not discriminate between
them; recorded because it is the first thing a reader will reach for.

`concat()` → `[]` and `concat(a)` → `a` are unchanged.

### §3.4 Cost

**The cheap direction and the correct direction are the same one here, which is worth noting because
it usually is not.** No vector re-keys — go reports CV-5 / CV-7c are already built the evaluated
way. The edit is one line of declaration plus go's `Collections []hash.Hash` field, which is the one
place the declared shape survived into a type.

---

## §4 Corner 3 — RULED: entity-key equality is over the **materialized** form (go is right)

§3.5 already says key equality is *"byte-identity over the canonical ECF encoding of the derived
key."* For a **primitive** key that is unambiguous. For an **entity-valued** key it was not stated,
and rust and py both asked.

**Ruling: `materialize(key) → ECF encode` — the materialized bare-entity bytes.** `entity-core-go`'s
`canonicalKeyBytes` (`ext/compute/builtins_v324.go`) is correct.

**Derivation:** the alternative is to encode the **in-flight** value, which for compute is
kind-tagged (§2.3 N1 confines kind-tagging to `compute/scope`). Encoding that would make an
implementation-private representation **byte-load-bearing in a group's identity** — the exact thing
§2.4's *"the in-flight representation is implementation-private; only the materialized form is
normative"* forbids, and the exact failure mode v3.26 ruled for contained errors one paragraph over.
**This corner is the same ruling as C-9, reached from the key side instead of the value side**, which
is why it needs no independent argument.

Filed by go as a data point rather than a routed item; arch is confirming it as the rule so it stops
being one seat's undocumented choice.

---

## §5 Deltas

| # | File | § | Delta |
|---|---|---|---|
| D1 | `EXTENSION-COMPUTE` | §3.5 | **Provenance-independence `[MUST]`** — a `compute/error` behaves identically whether minted or value-form. Stated as a **restatement of §2.4**, not a new rule, with the map-asymmetry named as the defect it forbids |
| D2 | `EXTENSION-COMPUTE` | §3.5 | Extend the consumed/contain table with `map` / `filter` / `fold` rows per §2.3 above, and **replace** *"the contained set is exactly three positions"* with the **rule** (places-without-reading = contain; reads-to-decide = consume) plus its current five-position extension |
| D3 | `EXTENSION-COMPUTE` | §3.5 | `system/compute/concat-args.collections` → `{type_ref: "system/hash"}`, one hash resolving to an array of arrays. **Reason in the spec is uniformity with the four sibling collection args + arity;** the §7.1 argument is retracted (§3.2) and must not be restated anywhere |
| D4 | `EXTENSION-COMPUTE` | §3.5 | `group-by` key equality for an entity-valued key is byte-identity over the **materialized** encoding |
| **D5** | `EXTENSION-COMPUTE` | §3.5 | **RULED (§8)** — the eval-limit dispositions. `budget_exhausted` and `cascade_limit` **short-circuit** in every position including the contained ones; `depth_exceeded` **contains** like any other error. The discriminator is **whether the counter is restored on unwind** (§5.1), not "limit-ness" |
| **D6** | `EXTENSION-COMPUTE` | §3.5 | **RULED (§8.3)** — the eval-limit disposition is keyed on the **`code`**, in **both** arms. A value-form `budget_exhausted` short-circuits exactly as a minted one does. D1 admits no provenance split, and §2.4 already declares the two to be the same entity |
| **D7** | *(rollout, no spec text)* | — | **RULED (§9)** — D3 lands by a named order with a declared transient `type_system` FAIL, not by a lockstep and not by a tolerance in the checker |
| **D8** | `EXTENSION-COMPUTE` | §7.1 | **RULED (§10)** — the `walk` pseudocode is weaker than §7.1's own prose rule and weaker than all three implementations. It descends into **any container value** to find hash references; the prose rule (*"all reachable paths are registered"*) is the normative property and the pseudocode is its illustration. **Every reference field in the expression grammar is a scalar `system/hash`, and that is now a stated, checkable invariant** on new args types |

---

## §6 Conformance (L19 — class and satisfaction mode declared)

All four are **compute differential-corpus vectors (`GUIDE-CONFORMANCE` §7c)** — shape
`(IR, root bindings, budget) → { boundary-hash | error{code} }`. **Not** §7.0's fixture corpus.
Per §7c.5: **arch supplies IR + expected outcome, the cohort emits and cross-blesses, and arch
supplies no boundary hashes** (§7c.4(3) — no implementation is privileged).

| Vector | Discriminates |
|---|---|
| **CV-8a** | `map` whose closure yields a **minted** error on one element vs. **CV-8b** the same shape yielding a **value-form** error. **The pair is the point** — a provenance-dependent implementation gives two different answers and fails exactly one arm |
| **CV-8c** | `filter` whose predicate returns an error → short-circuit *(the arm that fails if a seat contains it and silently drops the element)* |
| **CV-8d** | `fold` whose closure **ignores** its accumulator, run with an error accumulator → recovers. **This is the arm that pins §2.3's widest-blast-radius call**, and it is deliberately the shape where contain and short-circuit produce different **values** |

**These three seats already hold the discriminator discipline** — go's CV-4/CV-7 series is exactly
this shape, and CV-4 caught the C-9 spec defect one level earlier than it was designed for.

### §6.1 The eval-limit rulings (§8) — same class, and the pair is again the point

Also **compute differential-corpus vectors (§7c)**; same ownership split (arch supplies IR + expected
outcome, the cohort emits and cross-blesses, arch supplies no boundary hashes).

| Vector | Discriminates |
|---|---|
| **CV-9a** | `map` whose closure exceeds **depth** on one element of three → `[a, E, c]`. Fails any seat short-circuiting `depth_exceeded` — currently `entity-core-go` |
| **CV-9b** | `map` whose closure exhausts the **operations** budget mid-array → the single `budget_exhausted`, no array. Fails any seat containing it |
| **CV-9c** | **The provenance pair for §8.4** — the same `map` over a **value-form** `compute/error{code: "budget_exhausted"}` read from a bound value. MUST be byte-identical to a minted one at the boundary. **This is the arm `entity-core-go`'s minted-only carve-out fails**, and no existing vector reaches it |

**CV-9c is the discriminator this ruling turns on, and its absence is why the defect survived.** A
seat keyed on the minted arm alone passes CV-9b and fails only here — the identical structure as
CV-8a/CV-8b, one code family over.

### §6.2 D8's clause 1 (§10.3) — a DIFFERENT class, declared per L19

**`walk_tree_lookups` completeness is NOT a differential-corpus vector.** §7c's shape is
`(IR, root bindings, budget) → { boundary-hash | error{code} }`, and dependency registration
**produces no boundary** — it decides whether a *later* re-evaluation happens. A corpus vector cannot
see it, which is exactly how three walkers of three different strengths passed everything.

**Class: `validate-peer` behavioural check** (`GUIDE-CONFORMANCE` §7.0's first row — behavioral, over
the wire, oracle-authored). **Satisfaction mode (§5.2b.1):** constructible today by an existing
harness — install a subgraph whose only `compute/lookup/tree` sits inside a `compute/let` binding
(and a second whose lookup sits inside a `compute/apply` arg), write the watched path, and assert the
re-evaluation fires. Both required states are reachable with landed operations: `system/compute:install`,
an ordinary tree write, and a read of the `result_path`. **No new surface is owed** — which is stated
explicitly because the last two checks pinned in this corpus without that question being asked were
unconstructible, and one of them turned out to be a missing operation rather than a harness gap.

**Ownership:** oracle-authored, so it is `entity-core-go`'s to write as the seat holding
`validate-peer`, and it runs against all three. **Row (b) — the `apply.args` arm — is the one that
discriminates a hand-enumerated walker from a recursive one**; a walker covering `let.bindings` alone
passes row (a).

---

## §8 The eval-limit carve-out — RULED `[2026-08-21]`

Corner 1 settled what happens to an *ordinary* error in a contained position. It did not settle the
three **evaluation-limit** codes — `budget_exhausted`, `depth_exceeded`, `cascade_limit` — and the
cohort split measurably: `entity-core-go` short-circuits all three everywhere; `entity-core-rust` and
`entity-core-py` short-circuit `budget_exhausted` and `cascade_limit` and **contain** `depth_exceeded`.

**The derivation is one sentence of §5.1, and it is decisive.**

> *"Every call to `evaluate()` decrements `operations` by 1. Recursive calls decrement `depth` by 1
> **(restored on return)**."*

§4.1's pseudocode implements exactly that, on every path including the failing one — `budget.depth += 1`
before the `budget_exhausted` return and before the ordinary return; `budget.operations` is decremented
and **never restored anywhere.** So the two counters are not two limits of the same kind:

| Counter | Lifetime | Is element *i*'s outcome a function of elements 1…*i*−1? |
|---|---|---|
| `depth` | decremented on entry, **restored on unwind** | **No.** Element *i* begins at the same depth every time, so whether it exceeds is a property of that element's own sub-expression |
| `operations` | decremented once per `evaluate()`, **never restored** | **Yes.** Whether element *i* exhausts the budget depends entirely on what elements 1…*i*−1 cost |

**§8.1 `budget_exhausted` — SHORT-CIRCUITS.** Containing it makes the result array's contents a
function of **where the budget ran out**, and that split point is not pinned by anything. §10.4 makes
memoization table size and eviction policy implementation-defined; a memo hit skips an `evaluate()`
call and therefore an operations decrement, so **two conformant peers given identical IR, identical
inputs and an identical budget can exhaust at different elements.** Contained, that yields
`[v₁ … v_{k−1}, E, E, …]` with a different *k* per peer — **different boundary bytes for the same
program**, which §8.1's determinism `[MUST]` and AE-1's materialized-boundary equivalence both
forbid. Short-circuited, every peer that exhausts answers with the one code and the boundary is
identical. **The second reason is independent of the first and would carry it alone:** an array of
contained `budget_exhausted` elements is a *successful-looking result* — a well-formed array, a
readable boundary hash, no error at the top — reported for an evaluation the peer **aborted**. That
is the failure §3.5 already refused when it declined to clamp a negative `range(n)` to `[]`.

**§8.2 `cascade_limit` — SHORT-CIRCUITS**, and it is the easiest of the three. Per §7.3 the counter is
not merely shared across elements, it is **shared across the entire causal chain and tracked
cross-peer by `chain_id`** (`SYSTEM-COMPOSITION` §3.1–§3.4). §7.3 further says reaching it **freezes
the subgraph** — recovery is re-installation. A frozen subgraph that nonetheless emitted a
well-formed array carrying element-wise `cascade_limit` values would be reporting a value for a
computation that is structurally halted, and §7.3 draws exactly this line itself when it contrasts
the freeze against budget exhaustion (which *"does NOT freeze… may be transient"*).

**§8.3 `depth_exceeded` — CONTAINS, like every other error, and it needs no carve-out at all.**
`depth` is restored on unwind, so it is element-local by the spec's own parenthetical. `map(f, xs)`
where `f` recurses too deeply on element 2 and not on the others yields `[a, E, c]` — the element-wise
NaN model §1.5 mandates, evaluated identically by every peer, because *each element's depth budget is
the same one.* **rust and py are right, and their criterion — *"a resource shared across the
elements," not "limit-ness"* — is the correct criterion.** It is adopted, re-derived here from §5.1
rather than from their agreement (L18): had all three seats short-circuited `depth_exceeded`, §5.1
would still say `restored on return` and the ruling would be the same.

**`entity-core-go` is the 1-of-3 outlier and held its position rather than voting, which is the
correct posture and is recorded as such.** The spec arbitrates, and it arbitrates against them here.

### §8.4 The second clause — the disposition is keyed on the CODE, in BOTH arms (`SA-PY-25`)

`entity-core-py` asked the sharper question: **does an evaluation limit have a value-form
representation of its own at all?** It does, and the spec **mandates its creation**:

> §7.3: *"Budget exhaustion during reactive re-evaluation **writes a `compute/error` to the
> result_path**."*

A downstream expression reading that `result_path` through `compute/lookup/tree` receives a
**value-form `budget_exhausted`**, produced by a conformant peer doing exactly what §7.3 requires.
The value form is not hypothetical and cannot be legislated away.

**So there are only two provenance-symmetric dispositions available, and D1 forbids the third.**
D1 (§2) is that a `compute/error` behaves identically however it was produced; §2.4 states the
underlying fact more strongly still — *"Two errors with the same `code` **are** the same materialized
entity."* Minted and value-form are therefore not two things a rule may distinguish. Given §8.1
refutes *both-contain* for `budget_exhausted`, the only surviving disposition is **both
short-circuit, keyed on the `code`.**

**`entity-core-go`'s carve-out fires only on the minted arm, so it contains a value-form
`budget_exhausted` while short-circuiting a minted one. That is a defect** — the exact §2.4
provenance asymmetry Corner 1 was opened to kill, reinstated three codes wide inside the carve-out
Corner 1 created. **`entity-core-rust` names this trap precisely and avoided it:** *"keyed on the
CODE, never the variant."* Adopted verbatim as the rule.

**The consequence, stated so nobody reads it as an oversight:** a program may store a literal
`compute/error{code: "budget_exhausted"}` and thereby short-circuit a `map` that would otherwise
contain its errors. **That is correct and is not a hole.** The materialized boundary is
content-addressed over `code` alone (§2.4), so the stored value and the minted one **are the same
entity by the protocol's own identity rule** — making them behave differently would require the
evaluator to know something the boundary cannot express. An author can already halt an expression by
placing an error in a consumed position; this adds no authority they did not have.

---

## §9 D3's rollout — RULED `[2026-08-21]`

D3 is ruled and undisputed, and it still could not land, because the `type_system` conformance check
compares each seat's **published descriptor** against its siblings' local type tables: **whoever
narrows first goes RED against the other two.** `entity-core-py` landed it and carries a deliberate
`1F`; `entity-core-rust` implemented it and backed it out citing DRAFT; `entity-core-go` has not
landed it. That is three different tactics for one ruling, which is the signal that the missing
decision is arch's.

**Ruled: there is no lockstep, and there must not be a tolerance in the checker.**

A synchronized cut across seats that commit independently is not an instrument this cohort has — the
seats do not share a clock, and pretending otherwise just relocates the RED window without shrinking
it. The other tempting fix — teaching `type_system` to accept the old shape *and* the new one for the
duration — is **dual-kind acceptance, which this corpus forbids outright** (`AGENTS.md`: *no legacy
paths, migration windows, dual-kind acceptance, or mirror-writes*). A checker that accepts both shapes
cannot fail the divergence it exists to catch, and the window in which it is blind is precisely the
window in which the divergence is real.

**So the FAIL is real, it is the instrument working, and it is carried in the open:**

1. **`entity-core-py`'s tactic is the correct one and is blessed retroactively.** Land the edit,
   carry the `F`, label it. They should not have been alone in it.
2. **Landing order: `entity-core-go`, then `entity-core-rust`.** go is the tier lead and the seat py's
   `F` currently scores against; go landing collapses the divergence from 2-against-1 to 1-against-1.
3. **Each seat records the transient failure as `EXPECTED — D3 transition, closes when the third seat
   lands`, and NO seat baselines it away.** A baselined failure is indistinguishable from a fixed one,
   which is the whole reason the baseline in this repo only ever ratchets down.
4. **The window closes when the third seat lands, and the seat that closes it says so.**

**What makes tolerating the RED correct here rather than sloppy, and it is a fact about this
particular delta:** D3 is **declaration-only. Nothing on the wire moves.** No boundary hash, no
evaluation result, no interop surface changes — the two shapes describe the same evaluated operand.
So the transient FAIL costs a red row in a report and buys nothing bad; it is not standing in for an
interop break anyone could suffer. **A delta that did move the wire would not get this ruling**, and
would need a different one.

---

## §10 §7.1's `walk` — RULED, and it is a live defect at three seats' expense `[2026-08-21]`

Filed as D5 (*"a separable defect, flagged not fixed here"*) on the belief that the traversal was
incomplete only for **future** array-valued reference fields. **That was wrong in both directions**,
and chasing D3's retraction (§3.2) is what surfaced it.

**§10.1 The spec contradicts itself, and the prose is the half that is right.** §7.1's pseudocode
recurses on one condition — `for field_value in entity.data.values(): if field_value is system/hash` —
which enters no container. §7.1's own **"Conservative static collection"** paragraph, eleven lines
below, states the property normatively:

> *"`walk_tree_lookups` is conservative: **all `compute/lookup/tree` paths reachable in the expression
> graph are registered**, including paths behind untaken `compute/if` branches."*

**The pseudocode cannot achieve the prose rule on the grammar that exists today.** `compute/apply.args`
is `{map_of: system/hash}`; `compute/let.bindings` is an array of `{name, value: system/hash}`. A walk
that enters neither registers **no dependency inside any function argument or any `let` binding** —
which is most of every non-trivial expression, including §3.5's own *"Sequencing side effects"* worked
example.

**§10.2 All three implementations exceed the pseudocode, and they do not agree with each other.**
Measured by reading source, not reported:

| Seat | Commit | Walker | Container behaviour |
|---|---|---|---|
| `entity-core-go` | `c1b0708` | `walkDepValue` (`ext/compute/engine.go`) | Recurses over `[]interface{}` and both map kinds, **unbounded depth** |
| `entity-core-rust` | `2ee6bf7` | `walk_hash_fields` (`extensions/compute/src/walker.rs`) | Recurses over `Value::Array` and `Value::Map`, **unbounded depth** |
| `entity-core-py` | `f09ae70` | `_walk_deps` (`entity_handlers/compute.py`) | **Hand-rolled, not recursive** — a scalar hash; a list of hashes; a list of dicts, keyed on the literal field name `"value"`; a dict, one level of values. **Nothing deeper** |

**py is correct today by enumeration, not by rule.** Its cases happen to cover exactly the container
shapes the current grammar uses — including a special case matching `compute/let.bindings` on the
string `"value"`. **One new args type nesting a reference one level deeper, or a binding shape whose
field is not called `value`, and py silently registers nothing** — and the failure mode is a reactive
expression that evaluates correctly once and is never woken again, which no boundary-hash vector can
see. *This is not a py defect being singled out: py implemented what could be enumerated because the
spec gave a pseudocode that enumerates. The spec is the defect.*

**§10.3 Ruled.**

1. **The prose rule is the normative property, and it is promoted to a `[MUST]`:** every
   `compute/lookup/tree` reachable in the expression graph is registered. An implementation is
   measured against that sentence, never against the pseudocode's shape.
2. **The pseudocode is corrected to descend into any container value** — array or map, recursively —
   collecting hash references wherever they sit, so that it illustrates the rule instead of
   contradicting it.
3. **The grammar invariant is stated and becomes checkable:** every reference field in the expression
   grammar is a **scalar `system/hash`** (after D3, `map_of`/`array_of` hash appear only in
   `compute/apply.args` and `compute/let.bindings`, both of which are enumerated in §2.1). **A new
   args type declaring a reference inside a container is a defect** unless it is added to that
   enumeration in the same edit.

***Enforcement point:*** clause 3 is a grep — an `array_of`/`map_of` whose `type_ref` is
`system/hash`, in a type block outside §2.1's enumerated two, is the violation. Clause 1 is a
behavioural check and is declared as such below.

---

## §11 Route

**One packet to `entity-core-go`**, relayed to rust and py. Corner 1 is the only one with a behaviour
change at more than one seat; Corners 2 and 3 are a declaration edit and a confirmation. §§8–10 are
new and each names its seat.

**Sequencing note (operator, 2026-08-21):** the substrate tier converges first — go drives rust and
py to clean — and the results go to `entity-browser-rust` after that, by the operator.

**T5 is DEFERRED and this proposal does not lift it.** Everything ruled here is *closing* work on
behaviour three seats have already built, plus two arch defects (§3.2's retraction, §10's pseudocode)
found in arch's own text. **The v3.27 fold is not taken in this proposal** — the rulings land here,
where they unstick three stopped seats without a version bump broadcasting a work order into a paused
track (**L21**). The fold is the operator's call and the argument for taking it soon is in §12.

---

## §12 The cost of NOT folding, stated so the decision is informed

Recorded because a deferral is a build-state claim and expires like one (**L9**), and because the
argument changed today.

**`EXTENSION-COMPUTE` v3.26 carries two sentences that are now known to be wrong**, in live published
text, at §3.5 and at §2.3 N1's scoping note:

- *"**The contained set is exactly three positions** — `assoc`'s `value`, `concat`'s elements,
  `group-by`'s `members`."*
- *"…and **only** those three."*

It is five. An implementer who has never heard of this cohort — the reader every spec in this corpus
is written for — reads those sentences and implements `map`'s output element and `fold`'s accumulator
as short-circuits. **Both fork a boundary hash**, and `fold`'s forks the produced *value*. That is not
a latent risk: it is the defect `entity-core-go` shipped, from the same reading, this week.

**The usual L21 objection does not bite the same way here, and the difference is checkable.** L21's
question is *who will read this version bump as an instruction?* For v3.24 the answer was "the seat
that leads new features, and it will read four unimplemented primitives as a work order" — and it
did. For v3.27 the answer is: **D1, D2 and D4 are already built at all three core seats**, D3 is a
one-line declaration edit with a landing order attached, and D5–D8 are corrections. **Nothing in
v3.27 asks any seat to build a capability that does not exist.** The bump reads as *catch up* to
nobody, because nobody is behind.

**Arch is not taking that decision unilaterally**, having told `entity-core-go` in
`ROUTING-2026-08-21-g` — yesterday — that the v3.27 fold was not happening this cycle. Reversing that
inside a day on arch's own judgment is the churn this board has already measured and named. **The
question goes to the operator with both costs on the table.**
