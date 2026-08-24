# PROPOSAL — the v3.24 collection primitives' four corners: a lost return shape, a re-litigated error code, and a rule that was already written

**Status:** FOLDED 2026-08-20 — all six deltas verified against the tree before this marker (L3)
**Spec:** `specs/extensions/EXTENSION-COMPUTE.md` §3.5, §9.1 — v3.24 → v3.25
**Origin:** `entity-core-go` spec-issue `2026-08-20-e`, filed while implementing v3.24 (go leads new
features). Three corners raised; a fourth found while deriving them.
**Class:** completion of the folding session's own text. Two of the three raised corners resolve
**against** the reading the filing seat shipped, and both times on landed text they had no reason to
suspect — one on this repo's own design record, one on a cross-impl ruling this corpus already paid to
reach.

---

## §0 Shortest form

| # | Corner | Ruling | Whose reading it is |
|---|---|---|---|
| **D1** | `group-by`'s return shape is not pinned | **`[system/compute/group{key, members}]` — keys are IN the result** | **Against go.** The shape was ruled in the design record and the fold lost it |
| **D2** | `assoc`'s out-of-range index → `type_mismatch` | **`index_out_of_range`** | **Against go — and against the v3.24 text they correctly implemented.** Arch's defect |
| **D3** | `range(n)` for negative `n` is unstated | **`count_out_of_range`, a new §9.1 code** | Against go's `type_mismatch`; their rejection of clamp-to-empty is upheld |
| **D4** | error-as-value flow-through is not pinned | **Confirmed — and it is not a new rule.** §7.2's consumed-operand rule already decides it | **With go.** Their split is the spec's, generalized correctly |

**The one-line summary of the session:** three of four corners were already answered somewhere in this
corpus, and none of the three answers was in §3.5.

---

## §1 D1 — the return shape was ruled, and the fold dropped it

### The finding as filed

> *"§3.5 pins the ordering … but never states the **container** the groups come back in. Array-of-arrays?
> Array of `{key, elements}` pairs? go's reading: an **array of arrays** … **keys dropped**."*

Correct that it is unpinned in §3.5. **Not an open design question.**

### Where the answer was

`PROPOSAL-COMPUTE-COLLECTION-PRIMITIVES` §3b, `docs/proposals/implemented/extensions/`:

> **The primitive:** `group_by(collection, key_fn) → [(key, members)]` — one pass, **O(N log N)**,
> producing groups the existing combinators consume natively (`map`/`filter`/`fold` over
> `[(key, members)]`).

§7's ruling table adopts §3b by reference — *"**`group_by`** (§3b) | **ADOPT** | MUST-given-COMPUTE."*
**The keys are in the ruled shape, and the fold's §3.5 prose — "returns the elements grouped by that
key" — silently narrowed it to the elements.**

The rationale §3b gives is not decorative about this. It justifies the primitive by *"the shape of a
spatial index, a **histogram**, a bucketed aggregation, a **router**"* — and a histogram whose bin
labels have been dropped is a list of counts nobody can read. **The generalization argument that earned
the primitive its ADOPT is the argument that requires the key.**

### Why the recover-by-reapplication answer does not rescue key-dropping

The tempting defense of go's reading is that a key is recoverable — `map` `fn` over the first element of
each group. It is true and it is not sufficient:

1. It re-runs caller code the primitive already ran. `fn` is arbitrary and may be expensive; the whole
   claim of `group-by` is *one pass*.
2. It is only sound for a **total, pure** `fn` over a **non-empty** group. Purity holds here, but the
   reconstruction is a second contract the spec would have to state, and it would have to state it in
   the place it declined to state the shape.
3. **It makes the caller re-derive a value the primitive computed and threw away**, which is the exact
   O(B·N)-flavoured waste §3b exists to retire, in miniature.

### The shape: an entity, not a positional pair — and this is the one genuinely-chosen call here

`(key, members)` is shape-agnostic prose. Two readings satisfy it, and this proposal picks one:

| | `[key, members]` — a 2-element array | **`system/compute/group{key, members}`** |
|---|---|---|
| new names | none | **one pinned type name** |
| consumed by | `index(g,0)` / `index(g,1)` — positional | `field(g,"key")` / `field(g,"members")` — named |
| precedent in this extension | none | **`compute/let`'s `bindings`**, typed `array_of primitive/any` and documented as `[{name, value}, ...]` — the extension's existing array-of-pairs is field-named |
| tension with v3.24's own text | **yes** — a `[key, members]` pair is heterogeneous **by construction**, and v3.24 is the revision that introduced *"element types MUST match"* for arrays (`concat`) | none |

**The tension row decides it.** Adopting a result shape that is inherently a heterogeneous array, in the
same revision that first constrains array element types, puts one new primitive's output against another
new primitive's precondition. The entity shape has no such tension.

**The cost is smaller than the filing seat estimated, and that is worth stating to them.** Their ask says
a key-carrying shape *"would be a new registered type in all three impls."* Under §2.3 N1 and §4.1's
materialization rule — *"encoding is by the runtime kind of the evaluated value, **never** the constructed
type's declared schema (keeping it type-extension-independent)"* — a constructed entity's bytes do not
depend on the type being registered anywhere. **`system/compute/group` is a pinned string, not a
type-extension registration.** A peer with no type extension produces identical bytes.

*This is the one delta here that is a choice rather than a recovery. It is flagged as such: a seat that
wants the positional pair should say so and the argument above is the thing to attack.*

---

## §2 D2 — `assoc`'s error code re-litigated a settled cross-impl ruling, in the losing direction

**This is arch's defect, introduced at v3.24, and it is the most expensive item here.**

§3.5 as folded says: *"An out-of-range `index` is a `type_mismatch` error-as-value."*

`EXTENSION-COMPUTE` §2.2, landed, on `compute/index`:

> **Out-of-range index → `index_out_of_range` error.** Negative indices are out of range … A `uint`-annotated
> index (e.g. `cast(-4, uint)`) is **not** a `type_mismatch`: `int`/`uint` are annotations, not distinct
> value types (§2.2), so any integer bit-pattern is a valid index *argument*; an out-of-bounds
> **magnitude** — whether read signed (negative) or unsigned (≥ length) — is `index_out_of_range`.
> *(Ruled from the compute corpus's first cross-impl run — `ARCH-RESPONSE-COMPUTE-CORPUS-FIRST-RUN`,
> R3/F-2.)*

And the run itself, `docs/research/reviews/ABSORPTION-compute-corpus-first-crossimpl-run.md` §F-2:

> `index(arr, cast(-4, uint))` → Go `type_mismatch`, Rust `index_out_of_range`.
> **Verified: Rust is correct, Go is wrong.** … there is no basis for `type_mismatch` — an integer
> bit-pattern is a valid index argument regardless of its annotation. **Go fixes it.**

**So: `entity-core-go` was ruled wrong on precisely this question, on a cross-impl run, and fixed it.
Then v3.24 handed them a clause telling them to do it again one operation over, and they implemented it
faithfully** — `ext/compute/builtins_v324.go` carries a test named
`out-of-range-is-type-mismatch-NOT-index-out-of-range` and a comment noting the contrast *"for the same
condition."* **The implementer saw the contradiction, read it as intent, and pinned it.** That is the
correct behaviour from an implementer against landed text, and it is the reason a defect of this shape
does not surface as a question.

`assoc`'s `index` is an index into an array, out of range when negative or ≥ length. It is §2.2's
question with no distinguishing feature. **`index_out_of_range`.**

### Why the rule did not transfer on its own, which is the durable part

F-2's fix landed as a sentence **about `compute/index`**, in `compute/index`'s bullet. When a second
operation taking an array index arrived eighteen days later, nothing carried the ruling to it — not the
error-code table (whose row reads *"`compute/index` index is negative or ≥ array length"*, naming the one
operation), not a vector, not a grep that would fire.

**A ruling written at the width of the operation that produced it does not bind the operation it was not
written for.** That is `AGENTS.md` L14's lesson — *a rule written at the width of the incident* — arriving
on an error code instead of a version header, and it is why §9.1's row is widened here rather than a second
sentence being added to `assoc`.

### The cost had it shipped

`index(arr, -1)` and `assoc(arr, -1, v)` — the same malformed program, one document, two error codes, both
conformant. Every program that branches on an error code to distinguish "bad index" from "bad type" gets a
different answer depending on which array operation it reached, and the compute corpus's boundary-bytes
guarantee makes that a **divergent result**, not a cosmetic one.

---

## §3 D3 — `range(n)` for a negative `n`: not `type_mismatch`, and not clamp

**go's rejection of clamp-to-empty is upheld, and their positive answer is not.**

**Clamp is rejected**, on their reasoning and one more. Theirs: it hides a likely program bug. The
addition: `n` is a **loop bound**, so a silent `[]` propagates through every downstream `map`/`filter`/
`fold` and the program returns a plausible, well-formed, wrong answer with **no error anywhere in the
result**. Determinism is not at stake either way — both readings are deterministic — so the tiebreak is
diagnosability, and error-as-value wins it outright.

**`type_mismatch` is rejected on §2.2's own reasoning**, which go's implementation anchored to `assoc`
(`builtins_v324.go`: *"consistent with assoc's out-of-range index below"*) — so D2's defect propagated into
a second primitive. With `assoc` corrected, the anchor moves: §2.2 ruled that `int`/`uint` are annotations
rather than types, so **an out-of-domain magnitude is not a type error**. A negative `n` is a well-formed
integer argument whose magnitude is out of the declared domain. That is the `index_out_of_range` class, not
the `type_mismatch` class.

**But it is not an index**, and reusing `index_out_of_range` for an operation with no array and no position
overloads a code whose registry row names a specific condition.

**The corpus already shows how it resolves this.** §9.1 carries `cast_out_of_range` —
`compute/numeric-cast`'s own out-of-range code — minted rather than folded into `type_mismatch` or
`index_out_of_range`. **Two precedents, one pattern: an operation's out-of-domain magnitude gets its own
named code.** `range` is the third instance.

**Ruled: `count_out_of_range`** — *"`compute/builtins/range` `n` is negative, or exceeds the maximum
representable array length."* The naming follows `index_out_of_range` / `cast_out_of_range` exactly. The
second clause covers go's `uint64 > MaxInt64` case, which they raised and which is the same condition read
unsigned.

---

## §4 D4 — the flow-through rule is already written, and the filing seat under-claimed its own finding

go's ask: *"confirm the control-vs-data split, or pin a uniform rule,"* offered as *"a principled split,
mirroring the existing evaluator."*

**It does not mirror the evaluator. It restates §7.2**, which is landed normative text:

> **Error short-circuit normative.** Once a `compute/error` enters a value position, all expression types
> that **consume values** MUST short-circuit and propagate the error … `compute/apply` (both handler and
> closure modes — **over the operands the apply consumes**: `resource`, `capability`, and closure arguments
> bound to parameters; **a builtin's write payload is not a consumed operand — see the SA-9 `store` worked
> example**).

The `store` worked example is the data half already decided: a value **placed into** a destination is not
a consumed operand and does not short-circuit. `map`'s pinned semantics are the other half — a closure
returning an error puts that error in the output array as an element.

So this needs **no ruling**, and the disposition is different in kind from a confirmation: §3.5 gets a
sentence naming the four primitives' positions and **citing §7.2**, so the next implementer finds it
without re-deriving it. Applying §7.2 as written:

| Primitive | Position | §7.2 class | Behaviour |
|---|---|---|---|
| `range` | `n` | consumed — the magnitude is read to produce the array | **short-circuit** |
| `group-by` | derived key | consumed — compared to assign a group | **short-circuit** |
| `group-by` | element | copied into `members` | contain |
| `assoc` | `index` | consumed — read to position the write | **short-circuit** |
| `assoc` | `value` | placed into the output — the SA-9 `store` case | contain |
| `concat` | each `collection` | consumed — length read to copy | **short-circuit** |
| `concat` | element | copied into the output | contain |

**One reinforcement for the `group-by` key, because D1 makes it a live question rather than a formality.**
With keys now *in* the result, an error key would have an output position, so "contain" becomes arguable —
and it is still wrong. Key equality is **byte-identity over the canonical encoding**, so grouping by an
error would make the error's *message string* structurally load-bearing: two failures with different
messages become two groups, and one failure message reworded changes the result's shape. Short-circuit, and
§7.2 already says so.

### The corner go did not name, found while deriving D4

**`concat`'s element-type match, against an element that is a `compute/error`.** §3.5 says element types
MUST match and a mismatch is `type_mismatch`; §7.2 says an element is contained, not consumed. Read
naively these collide: concatenating `[1,2]` with `[<error>]` is either a `type_mismatch` or a pass-through.

**Ruled: a `compute/error` element is type-transparent** — it does not participate in the element-type
match and flows through untouched. Derivation is §1.5's own model, which the spec states outright:
*"the same model as NaN propagation in IEEE 754 — errors are values that flow through computation."* A NaN
in a float array does not change the array's element type; it is a poisoned value **of** that type, not a
value of a different one. The alternative would make `concat` the one operation that inspects elements for
errors, which is exactly the "consume" behaviour §7.2 scopes it out of.

---

## §5 What this does not rule

**§3.5's fairness clause stays not-ruled**, unchanged: adopting `concat` does not bless an in-compute
sharded step, and the tick contract's fairness posture is still owed. Nothing here touches it.

**No conformance vectors are authored here.** Per `GUIDE-CONFORMANCE` §7.0 the four deltas below are
**fixture-corpus vectors** (static `.diag` + canonical `.cbor`, arch-authored) — they are pure-function
checks over an evaluator, needing no harness capability and no live peer, and every seat can satisfy them
in-tree. **Satisfaction mode per §5.2b.1: constructible today by any conformance client, at any seat, with
no new surface.** They are owed and recorded, not built. *(L19.)*

---

## §6 Open items

**None.** Recorded explicitly per L9: this proposal folds with an empty open-items set, so nothing is
carried to `docs/COHORT-OPEN-ITEMS.md` on its account.

---

## §7 RULING — 2026-08-20

| # | Delta | File · section |
|---|---|---|
| **D1** | `group-by` returns `array_of system/compute/group`, each `{key, members}`; group order = first appearance, member order = input order. Type name pinned; **encoding is kind-driven per §2.3 N1, so no type-extension registration is implied** | `EXTENSION-COMPUTE` §3.5 |
| **D2** | `assoc`'s out-of-range `index` → **`index_out_of_range`**, not `type_mismatch` | `EXTENSION-COMPUTE` §3.5 |
| **D3** | `range(n)` with negative `n`, or `n` exceeding maximum array length → **`count_out_of_range`** (new code) | `EXTENSION-COMPUTE` §3.5 |
| **D4** | §3.5 gains a flow-through sentence **citing §7.2** with the seven-position table; `concat`'s element-type match is stated type-transparent for `compute/error` | `EXTENSION-COMPUTE` §3.5 |
| **D5** | §9.1: `index_out_of_range` row widened to name `assoc`; `count_out_of_range` row added | `EXTENSION-COMPUTE` §9.1 |
| **D6** | §3.5 heading `group_by` → `group-by` — the last snake-cased occurrence in the corpus | `EXTENSION-COMPUTE` §3.5 |

**Version: 3.24 → 3.25.** Not the cohort-finding carve-out. Two of these change the **bytes a conformant
peer produces** (D1) and the **code it returns** (D2, D3), so the cohort must be able to see that the text
moved; a fix-in-place with no rev bump is for restoring a meaning the text already carried, and D1 and D3
had no meaning to restore.

## §8 Cohort impact

| Seat | Owed |
|---|---|
| `entity-core-go` | Re-pin `group-by` to the key-carrying shape (**D1**); `assoc` → `index_out_of_range` (**D2**); `range` → `count_out_of_range` (**D3**). D4 is already what they built. **Their in-tree pins were the right posture** — a reference reading, published as one, routed rather than assumed |
| `entity-core-rust`, `entity-core-py` | Unbuilt. **Build against v3.25, not go's in-tree reading** — go's spec-issue says *"rust/py: match it or raise a divergence,"* and D1/D2/D3 supersede it |
| `entity-workbench-go` | Lever 1's consumer. `concat` is unchanged by all six deltas |
| arch | Four fixture-corpus vectors (§5), owed and recorded |

**Routing per `AGENTS.md`: one packet, to `entity-core-go`, with rust's and py's section written to be
relayed.** Not three packets.
