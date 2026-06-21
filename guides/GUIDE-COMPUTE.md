# Compute Extension: User Guide

**Audience:** Application developers who want to add computation to entity trees.
**Spec reference:** EXTENSION-COMPUTE.md v3.19c

> **Updated for v3.18 (standard-IR floor).** Since v3.12 the spec gained the standard-IR floor: three new core inline types (`compute/index`, `compute/length`, `compute/numeric-cast`), the collection stdlib (`map`/`filter`/`fold`) promoted from optional to **MUST-given-COMPUTE**, and a pinned integer-arithmetic model (the WASM/LLVM/JVM model — sign-agnostic `add`/`sub`/`mul`, signed-default `div`/`mod`/`compare`, point-of-use casts). §2, §4, and §10.2 reflect these.
>
> **Updated for v3.19 (the value model + navigation + error surface).** Three tranches landed together: **v3.19a** — navigation composes (`field`/`index`/`length` to arbitrary depth, `field` over an entity *or* a record value; §2, §4.5), an evaluated `compute/error` returns at **status 200** as a value (§7.1), and the `filter` lambda arg is renamed `predicate` → **`fn`** (uniform with `map`/`fold`; §10.2). **v3.19b** — `compute/scope` bindings are **`kind`-tagged** and navigation is by kind, not key-shape (§8.3); a new `scope_unreachable` error code (§7.2). **v3.19c** — `compute/construct` **materializes to a normal bare entity** (the compute value model is compute-internal; what reaches the tree/handlers is a plain entity == hand-built), and navigating a materialized `system/hash` field returns the hash (follow it with `lookup/hash`; no auto-resolve) (§8.5). The value model is now pinned and three-way ratified across Go/Rust/Python.

---

## 1. What compute is (and isn't)

The compute extension embeds an expression language in the entity type system. Expressions are entities — content-addressed, storable, transferable. The compute handler evaluates them and produces result entities. Reactive mode re-evaluates automatically when dependencies change.

**What it IS:**

- A pure expression language: literals, arithmetic, comparison, logic, conditionals, let bindings, lambdas/closures
- Content-addressed computation — same inputs always produce the same result hash across all peers
- Spreadsheet semantics — referencing a cell with a formula gives you the formula's result, not the formula itself
- Reactive — installed expressions re-evaluate when their tree dependencies change
- Capability-gated — impure operations (tree reads, handler dispatch) require authorization

**What it is NOT:**

- Not a general-purpose programming language — no loops, no mutation, no side effects (except the `store` builtin)
- Not a query language — use the query extension for searching entities. Compute transforms values.
- Not automatically reactive — you must install expressions via the `install` operation. Uninstalled expressions are inert data in the tree.

**Relationship to other extensions.** Compute is independent of inbox, subscription, and continuation. All four share the internal tree notification pattern (tree change triggers behavior), but each defines its own use. Compute adds reactive re-evaluation on top of tree changes. Subscriptions deliver notifications to remote peers. They compose naturally: a compute result write triggers subscription notifications to subscribers watching that path.

---

## 2. Expression types at a glance

The language has 16 expression types organized in three families:

### Core expressions (the language primitives)

| Type | What it does | Example |
|------|-------------|---------|
| `compute/literal` | Introduce a value | `{value: 42}` |
| `compute/lookup/scope` | Read from evaluation scope | `{name: "x"}` — reads binding `x` |
| `compute/lookup/tree` | Read from entity tree (impure) | `{path: "app/data"}` |
| `compute/lookup/hash` | Read from content store (pure) | `{hash: H, path: "app/data"}` |
| `compute/apply` | Call a closure or handler | `{fn: H_closure, args: {a: H_lit}}` — optional `capability`, `resource` for handler dispatch (§4.3) |
| `compute/if` | Conditional (short-circuits) | `{condition: H, then: H, else: H}` |
| `compute/let` | Sequential bindings + body | `{bindings: [{name: "x", value: H}], body: H}` |
| `compute/lambda` | Create a function (captures scope) | `{params: ["a"], body: H}` |

### Inline expressions (sugar for common operations)

| Type | Operations |
|------|-----------|
| `compute/arithmetic` | add, sub, mul, div, mod |
| `compute/compare` | eq, neq, lt, gt, lte, gte |
| `compute/logic` | and, or, not |
| `compute/field` | Extract a field from an entity *or* a record value (composes with `index`/`length`; §4.5) |
| `compute/construct` | Build an entity from fields (materializes to a bare entity; §8.5) |
| `compute/index` | Element of an array at a position — `{array: H, index: H}` (§4.4) |
| `compute/length` | Element count of an array — `{array: H}` (§4.4) |
| `compute/numeric-cast` | Convert a numeric value between `int`/`uint`/`float` — `{value: H, to_type: "primitive/uint"}` (§4.4) |

### Lookups — three flavors

| Lookup | Resolves by | Purity | Use case |
|--------|-----------|--------|----------|
| `lookup/scope` | Binding name | Pure | Variables from let/lambda |
| `lookup/tree` | Tree path | **Impure** | Live data, reactive dependencies |
| `lookup/hash` | Content hash | **Pure** | Staged/immutable data, batch computation |

The impure/pure distinction matters: impure operations register reactive dependencies and prevent memoization. Use `lookup/tree` when you want the expression to react to changes. Use `lookup/hash` when the data is fixed.

---

## 3. Building expressions

Expressions reference sub-expressions by content hash. You build the graph bottom-up: create leaf entities first, then build parent entities that reference the leaves by hash.

### 3.1 A simple calculation: `3 + 4`

```
1. Create literal(3)  → hash H1
2. Create literal(4)  → hash H2
3. Create arithmetic{op: "add", left: H1, right: H2}  → hash H3
4. Store H3 at a tree path
5. EXECUTE system/compute eval targeting that path
6. Result: compute/result{value: 7}
```

Every sub-expression is a content-addressed entity. The arithmetic entity doesn't contain `3` and `4` directly — it contains their hashes. During evaluation, the compute handler resolves those hashes, evaluates the referenced entities, and computes the result.

### 3.2 Variables with let bindings

```
let x = 5
    y = x + 1
in y
```

Becomes:

```
literal(5)              → H_5
literal(1)              → H_1
lookup/scope("x")       → H_x
lookup/scope("y")       → H_y
arithmetic(add, H_x, H_1) → H_add    (x + 1)
let{
  bindings: [{name: "x", value: H_5}, {name: "y", value: H_add}],
  body: H_y
}
```

**Bindings are sequential** (`let*` semantics). Each binding's value is evaluated in the scope accumulated from all prior bindings. `y` can reference `x` because `x` was bound first.

### 3.3 Functions with lambda and apply

```
let f = lambda(a): a + 1
in f(10)
```

Becomes:

```
lookup/scope("a")         → H_a
literal(1)                → H_1
arithmetic(add, H_a, H_1) → H_body
lambda{params: ["a"], body: H_body}  → H_lambda

literal(10)               → H_10
lookup/scope("f")         → H_f
apply{fn: H_f, args: {a: H_10}}     → H_apply

let{bindings: [{name: "f", value: H_lambda}], body: H_apply}
```

Evaluating the lambda produces a `compute/closure` — a value type that captures the current scope. When applied, the closure's body is evaluated with the captured scope extended by the argument bindings.

### 3.4 Recursion via tree self-reference

The language has no direct recursion (`fib` can't reference itself during its own definition). The pattern: store the lambda at a tree path and use `compute/lookup/tree` to self-reference.

```
fib = lambda(n):
  if(lte(n, 1), n, add(fib(n-1), fib(n-2)))
```

Where `fib` inside the body is `lookup/tree("path/to/fib-lambda")`. When lookup/tree reads a lambda entity from the tree, it evaluates it to a closure (spreadsheet semantic), which can then be applied.

This works because `lookup/tree` is evaluated at call time, not at definition time — by the time the recursive call happens, the lambda is already stored at the tree path.

---

## 4. Arithmetic and comparison rules

### 4.1 Arithmetic

- **Mixed int/float:** If either operand is float, both promote to IEEE 754 binary64; the result is a float. `add(1, 2.5) → 3.5`
- **Integer division:** Exact → integer. Non-exact → float. `div(10, 2) → 5`, `div(7, 2) → 3.5`
- **Truncated mod (integer-only):** The remainder's sign follows the dividend. `mod(-7, 3) → -1`, `mod(7, -3) → 1`. A **float operand to `mod` is a `type_mismatch`** — the float-promotion rule applies to `add`/`sub`/`mul`/`div`, not `mod`.
- **Division by zero:** Integer `div`/`mod` → `division_by_zero` error. Float → IEEE 754: `div(1.0, 0.0) → +Inf`, `div(0.0, 0.0) → NaN`.
- **Non-numeric operands:** `type_mismatch` error.

**The integer model (v3.16+).** Integers are 64-bit two's-complement bit patterns — the WASM/LLVM/JVM model. Three consequences worth internalizing:

- **`add`/`sub`/`mul` are sign-agnostic and wrap.** They produce the same bits whether operands are read signed or unsigned, so there is no "mixed int/uint" case and no overflow error — results wrap modulo 2⁶⁴. `add(3, -1) → 2`.
- **`div`/`mod`/`compare` are signed by default.** These are the only sign-sensitive integer ops; written plainly they treat operands as signed.
- **Unsigned `div`/`mod`/`compare` is reached by an explicit cast at the operation.** To divide a value in [2⁶³, 2⁶⁴) as unsigned, wrap the operand in `compute/numeric-cast → primitive/uint` *as the direct operand* — `div(numeric-cast(y, uint), 2)`. The unsigned intent is **syntactic and eager**: it does *not* survive a `let` binding or an `if` branch (`let y = numeric-cast(x, uint) in div(y, 2)` is signed-default). Signedness is a property of the operation, statically readable from the graph — there is no signed/unsigned *value* tag.
- **Integer results encode by their signed interpretation** (bit 63 set → negative). One wire form per result → one content hash → identical results across peers.

If you come from a language with distinct `int`/`uint` types: here they are type-system *annotations*, not value types. A `uint` doesn't "stay uint" through arithmetic — you cast where you need unsigned behavior, not where you define the value. See EXTENSION-COMPUTE.md §2.2 rules 8–11 for the normative statement.

### 4.2 Comparison

- **Equality across types:** `eq(1, 1.0) → true` (numeric). `eq(1, "1") → false` (incompatible types are never equal).
- **Ordering:** Numeric or string only. `lt("abc", "abd") → true` (lexicographic, UTF-8 byte order, no Unicode normalization). `lt(1, "abc") → type_mismatch` error.

### 4.3 Handler dispatch via `compute/apply` — capability and resource fields

`compute/apply` in handler mode dispatches to another handler. Two optional fields control authorization:

```
compute/apply := {
  path:       "system/tree"              ; handler to dispatch to
  operation:  "get"                      ; operation name
  resource:   H_resource_expr            ; → evaluates to system/protocol/resource-target
  args:       {path: H_lit_path}         ; operation params
  capability: H_cap_expr                 ; → evaluates to a capability token
}
```

**Without `capability`:** the dispatch uses the handler's own grant (`ctx.capability`). The handler grant is the ceiling — the expression can only reach what the handler was authorized for at registration.

**With `capability`:** the dispatch uses the provided capability instead. This is the "caller's cap" pattern — the expression narrows itself to what the caller can see (e.g., `lookup/scope("caller_capability")`). A **dual-check** applies: the handler grant AND the provided capability must both cover the target. Neither alone is sufficient.

**The `resource` field (v3.10).** When `capability` is present, `resource` MUST also be present. The resource evaluates to a `system/protocol/resource-target` (`{targets: ["path/pattern"]}`) and is checked against the handler grant at the resource level.

**Why this matters — the exploit without `resource`:**

```
; Handler grant: system/tree:get on app/data-service/*
; Admin caller (cap covers everything)
; Expression:
compute/apply(path: "system/tree", op: "get",
              args: {path: "system/secret/x"},
              capability: caller_cap)

; WITHOUT resource: dual-check passes at handler+operation only (resource = null)
; → Handler escapes its scope, reads system/secret/x via admin's cap

; WITH resource: dual-check checks handler grant against resource "system/secret/x"
; → Handler grant doesn't cover system/secret/* → DENY
```

The `resource` field is the resource-level ceiling. Without it, the dual-check only verifies handler+operation coverage, and the handler can escape its declared scope when an admin-level caller triggers it.

**At install time:** static-literal `capability` references are validated against the installer's authority chain (the embedded cap must chain to the installer's identity). Dynamic capabilities (computed at eval time) are checked at runtime via the dual-check. Additionally, static-literal `capability` without `resource` is rejected at install audit — the structural error is caught before any evaluation occurs.

See EXTENSION-COMPUTE.md §4.1 for full pseudocode; PROPOSAL-COMPUTE-APPLY-RESOURCE-CEILING for the security analysis.

### 4.4 Collections and numeric conversion — `index`, `length`, `numeric-cast`

Three core inline types round out the value layer. Like `field`/`construct`, they are pure, core, and cannot be overridden.

**`compute/index` — element at a position.** `{array: H_array, index: H_index}` returns the element of the (evaluated) array at the (evaluated) integer index.
- Negative or ≥-length index → `index_out_of_range` (no Python-style from-the-end indexing).
- `index` on `null` or a non-array → `type_mismatch`.

**`compute/length` — element count.** `{array: H_array}` returns the array's length. Empty array → `0`; `null`/non-array → `type_mismatch`.

**`compute/numeric-cast` — convert between numeric primitives.** `{value: H, to_type: "primitive/uint"}`. Intra-numeric only — the matrix is exactly {`int`, `uint`, `float`}².
- **int ↔ uint:** reinterpret the two's-complement bits at 64-bit width (`cast(-1, uint) → 2⁶⁴−1`).
- **int/uint → float:** to binary64; magnitudes above 2⁵³ lose precision — defined-lossy, not an error.
- **float → int/uint:** truncate toward zero; **out-of-range, `NaN`, `±Inf` → `cast_out_of_range`**.
- bad `to_type`, or non-numeric `value` (including `null`) → `type_mismatch`.

The cast is **eager and consumed at the point of use** (the integer model, §4.1) — its effect reaches the immediately-consuming operation and does not flow through `let` bindings.

### 4.5 Navigation composes — `field`, `index`, `length` (v3.19a, N.5)

The three navigation operations compose to **arbitrary depth**, and `compute/field` operates over an **entity *or* a record value** — so two-level access works directly:

```
field("addr", field("user", root))          ; user.addr — record-of-record
field("name", index(items, 0))               ; items[0].name — element then field
length(field("tags", entity))                ; count of a nested array
```

`field` reads `.data` when the value is a `kind:"entity"` value, and reads flat when it is a `kind:"value"` (record); the operation **never** distinguishes the two by inspecting keys (no "does it have `type`/`data`/`content_hash`?" heuristic — that misfires on legitimate records). This is the **navigate-by-kind** rule (§8.3); the kind is carried on the in-flight value, so composition resolves before anything is materialized.

---

## 5. Reactive mode — the spreadsheet

### 5.1 Install a reactive expression

Reactive mode requires three steps: write the expression, install it, and change a dependency.

```
1. Put A1 = literal(10) at app/sheet/A1
2. Put B1 = mul(lookup/tree("app/sheet/A1"), literal(2)) at app/sheet/B1
3. EXECUTE system/compute install resource: {targets: ["app/sheet/B1"]}
4. Change A1 to literal(25)
5. Read app/sheet/B1/result → compute/result{value: 50}
```

Install does four things:

1. **Audits** the expression graph (`audit_subgraph`) — walks the entire expression DAG from the root, with cycle detection and closure environment traversal. Collects four categories of impure operations:
   - `read_paths` — tree read targets from `compute/lookup/tree`
   - `handler_targets` — handler dispatch targets from `compute/apply` (path + operation + resource)
   - `write_paths` — literal write paths from `store` builtins
   - `data_hashes` — hash lookup targets from `compute/lookup/hash`

2. **Checks capabilities** — verifies the caller's grant covers all collected impure operations via `check_grant_covers`. Static-literal `compute/apply.capability` references are also chain-root validated: the installer's identity must appear in the embedded cap's authority chain (see §4.3 and PROPOSAL-COHERENT-CAPABILITY-AUTHORITY CP1). This prevents an installer from embedding a capability they don't legitimately hold.

3. **Creates metadata** — stores a `system/compute/subgraph` entity at `system/compute/processes/{id}` with the installation grant hash, authorized data hashes, and dependency index.

4. **Registers dependencies** — watches all `compute/lookup/tree` paths for changes.

**Installation grant.** The caller's capability at install time becomes the **installation grant** — the authorization under which all future re-evaluations run. The grant hash is stored on the subgraph metadata. If the installation grant is later revoked or expires, re-evaluation fails and the subgraph freezes with error code `installation_grant_invalid`. The subgraph must be re-installed to resume. This means the installer's authority is the ceiling for the subgraph's lifetime, not just at install time.

After install, any write to a watched path triggers re-evaluation. The result is written to `{expression_path}/result` (or a custom path). If the new result hash equals the old one, no write occurs (convergence check — prevents cascading).

### 5.2 The result path

By default, results are written to `{root_expression_path}/result` (the path the install authorized — i.e., the resource target). You can override this:

```
EXECUTE system/compute install
  resource: {targets: ["app/sheet/B1"]}
  params:   {result_path: "app/sheet/B1-result"}
```

### 5.3 What triggers re-evaluation

Only `compute/lookup/tree` paths register as dependencies. Changes to entities referenced by `compute/lookup/hash` do NOT trigger re-evaluation (hash references are immutable — there's nothing to react to).

Dependencies are exact-match, not prefix-match. A write to `app/sheet/A1` triggers expressions watching `app/sheet/A1` but NOT expressions watching `app/sheet`.

### 5.4 Cascade limits

Re-evaluation can produce tree writes, which trigger further re-evaluations. This cascade is bounded by the peer's cascade depth limit (recommended: 16). If the limit is hit, the subgraph freezes — the subgraph metadata's `status` field is set to `frozen` with error code `cascade_limit`, a `compute/error` is written to the result path, and no further re-evaluations occur until the subgraph is manually re-installed.

Two distinct freeze triggers:
- **`cascade_limit`** — cascade depth exceeded. Usually a dependency cycle between subgraphs (A writes to B's input, B writes to A's input). Fix the cycle, then re-install.
- **`installation_grant_invalid`** — the installation grant was revoked, expired, or no longer covers the subgraph's impure operations. Re-install with a valid grant.

Frozen subgraphs are inert — they don't consume resources, don't re-evaluate, and don't participate in cascades. The last valid result remains at the result path until explicitly overwritten.

---

## 6. Pure data access with `compute/lookup/hash`

### 6.1 Why two kinds of external lookup?

`lookup/tree` reads live data by path — it's impure and registers a reactive dependency. `lookup/hash` reads immutable data by content hash — it's pure and doesn't register any dependency.

Use `lookup/hash` when:
- You're building a batch computation over staged datasets
- You want memoization (pure expressions can be cached)
- You want deterministic cross-peer verification (same hash → same result, always)

Use `lookup/tree` when:
- You want the expression to react to changes
- You're building a live spreadsheet formula

### 6.2 Authorization model

`lookup/hash` carries a `path` field — an authorization hint. At install time, the audit verifies that the entity at `path` has the specified hash, and that the caller's grant covers tree GET at that path. The validated hash is sealed into the subgraph's authorized set.

At eval time, the sealed set is checked — O(1) membership test. No tree reads, no reverse index, no re-validation. The hash is immutable; the install-time authorization covers all future evaluations.

For explicit eval (no install), `lookup/hash` targeting compute-typed entities always resolves (Tier 1). Non-compute targets in explicit eval require `content_store_access` allowance (Tier 0). Without it, eval returns `not_found`. For ad-hoc external data access without admin privileges, use `lookup/tree`.

---

## 7. Error handling

### 7.1 Error propagation

Errors propagate through the expression graph like NaN in arithmetic. If a sub-expression produces an error, every parent expression short-circuits to that error. `add(1, lookup/tree("nonexistent"))` produces `not_found` — the arithmetic never runs.

**An evaluated `compute/error` is a *value*, returned at status 200 (v3.19a, F10).** Because errors are values (§1.5), an eval that *completes* and yields a `compute/error` returns **200** with the error in the body — not a `4xx`. A `4xx` is reserved for failures of the **request itself, before or around evaluation**: a malformed/unauthorized EXECUTE, transport faults, or the install pre-audit rejecting an under-capable subgraph. This is what lets error-handling compose (a parent expression can inspect a child's error as data). Two consequences worth internalizing:
- **Client code must decode the 200 body** to tell success from a propagated error — do not branch on HTTP-style status alone.
- An **impure-op `permission_denied` that arises *during* evaluation** (a `lookup/tree` outside scope, a `store`/dispatch the capability blocks) is a propagated `compute/error{permission_denied}` at **200** (v3.19c), exactly like any other propagated error — not re-raised as a transport `4xx`. (The pre-flight install audit MAY still fail-fast at `4xx`; the rule is the determinism-safe line — a runtime-vs-static split would make the status implementation-dependent.)

### 7.2 Error codes

| Code | Meaning |
|------|---------|
| `not_found` | Path or hash doesn't resolve |
| `type_mismatch` | Incompatible operand types |
| `division_by_zero` | div or mod by zero (integer) |
| `budget_exhausted` | Too many evaluation steps |
| `depth_exceeded` | Recursion too deep |
| `cascade_limit` | Reactive cascade too deep (subgraph frozen) |
| `invalid_expression` | Malformed expression or non-expression at eval target |
| `permission_denied` | Capability doesn't cover the operation |
| `installation_grant_invalid` | Installation grant revoked or expired (subgraph frozen) |
| `index_out_of_range` | `compute/index` index is negative or ≥ array length |
| `cast_out_of_range` | `compute/numeric-cast` float→integer is out of range, `NaN`, or ±`Inf` |
| `scope_unreachable` | A resumed/transferred closure needs a scope binding that resolves in neither the local content store nor the envelope `included` map (v3.19b/c — a clean attributable error, never a silent substitution) |

---

## 8. Patterns

### 8.1 Bottom-up construction

You can't build expressions top-down. Every hash reference must resolve to an entity that already exists. Build leaves first (literals, scope lookups), then operators, then control flow, then the root.

### 8.2 Recursion patterns

**Via tree (impure, reactive).** Store the lambda at a tree path, use `lookup/tree` in the body to self-reference. The expression is impure and registers a reactive dependency on the lambda's path — which means it re-evaluates if the lambda is updated.

**Via hash (pure, fixed).** Store the lambda, get its content hash, build the body with `lookup/hash{hash: H_lambda}`. The expression is pure — the hash is immutable, so the recursion target never changes. The challenge is bootstrapping: you need the lambda's hash to build the body, but you need the body to build the lambda. This is a fixpoint construction that an SDK resolves at graph-build time.

### 8.3 Scope capture is persistent — the kind-tagged value model (v3.19b)

When a lambda evaluates, it captures the current scope into a `compute/scope` entity stored in the content store. Closures transferred between peers carry their captured values: a closure created on Peer A with `x=100` in scope evaluates correctly on Peer B.

Since v3.19b, each scope binding is **`kind`-tagged** (`system/compute/scope-binding`) — either an inline `{kind:"value", value}` or a hash reference `{kind:"entity", entity_hash}` into the content store. This makes the captured scope content-addressed and self-describing, and it is the canonical rule for navigation, capture, and (when needed) cross-peer transfer. Three things follow:
- **Navigate by kind, not by shape (N3).** Navigation (`field`/`index`/`length`) reads a `kind:"entity"` binding via its `.data` and a `kind:"value"` (record) binding flat — and **never** sniffs keys to guess which. (§4.5.)
- **Bindings inherit the closure's authorization (N4).** A `kind:"entity"` binding rides the referencing closure's authorization; the binding entity need not itself be compute-typed or sealed. (This matters even same-peer, for app-typed captured values.)
- **Resolution is eager and content-store-direct (N4a/N6).** Bindings are resolved at apply time, by direct content-store access — scope entities are structurally part of the referencing closure and bypass the tree-scoped resolution tiers.

The value model is **compute-internal**: kind-tags live inside scope serialization and in-flight values; they never leak into a materialized entity (§8.5).

### 8.4 Content store is the resolution backbone

Every sub-expression reference is a content store lookup. If the expression sub-entities aren't in the content store, evaluation fails with `not_found`. Tree `put` operations write to both the location index and the content store — so entities put to tree paths are resolvable by hash. Scope bindings (§8.3) and a closure's captured `env` resolve **content-store-direct**; for a *transferred* closure, the scope subtree arrives via the envelope `included` map (§9.4), not by tree path.

### 8.5 Construct materializes to a bare entity; read-back returns the hash (v3.19c)

`compute/construct` builds an entity from fields. The result, once it crosses out of compute eval — stored to the tree, returned from an eval, passed as an `apply` arg, or transferred cross-peer — is a **normal bare entity, byte-identical to one a caller would hand-build** (`entity.NewEntity`). Concretely:
- **Entity-valued fields become bare `system/hash` content references** (per V7 §1.4 — entity references are `system/hash` byte strings in data), never nested inline envelopes; primitives and records inline. **No kind-tags appear in materialized data** — the value model stays inside compute (§8.3).
- This is what keeps a constructed entity interoperable with non-compute consumers (tree queries, other handlers) and restores dedup (compute output == hand-built → same hash). The materialized hash is three-way bit-identical across Go/Rust/Python.

**Read-back navigation (N3).** When you navigate *into* a materialized bare entity and land on a `system/hash` field, the value comes back **as the hash** (a `kind:"value"`); to follow the reference, do it explicitly with `compute/lookup/hash`. There is **no auto-resolve** — an implementation must not identify or dereference a ref by byte-length or shape (and must never size a `system/hash` by a fixed length: it is variable-length with an extensible varint format-code, V7 §1.2). Refs are identified by *type* and deconstructed structurally. Navigation auto-composes only where the kind is known in-flight (within scope / a single eval), not across a materialized boundary.

---

## 9. Compute in the Handler System

The entity system supports three handler execution models — precompiled system extensions, SDK-registered language-native handlers, and entity-native compute-backed handlers. See SDK-OPERATIONS §11.3 for the full comparison, trade-offs, and guidance on choosing a model.

This section covers what the compute extension specifically provides for the entity-native model: how expressions serve as handler bodies, reactive subgraphs, library-as-entities, and transferable compute.

### 9.1 Entity-native handler registration

```
system/handler:register {
  manifest: {
    pattern:         "app/myapp/transform"
    name:            "transform"
    operations:      {process: {input_type: "app/myapp/input", output_type: "app/myapp/output"}}
    internal_scope:  [{handlers: {include: ["system/tree"]}, operations: {include: ["get"]}, resources: {include: ["app/myapp/*"]}}]
    expression_path: "app/myapp/transform/logic"
  }
}
```

The expression at `app/myapp/transform/logic` is the handler body. When an EXECUTE targets `app/myapp/transform` with operation `process`, the dispatcher evaluates the compute expression (V7 §6.6). The expression receives the EXECUTE params as input and produces the response.

**Handler-mode params assembly (v3.3+).** When `compute/apply` dispatches to a handler, args are assembled into the target operation's declared `input_type`. Each arg is encoded per-field based on the type definition: small values inline, large values by hash reference. The `encode_arg_for_field` helper handles this — the expression author doesn't need to know the encoding rules, just the field names. See EXTENSION-COMPUTE §4.1 for the normative algorithm.

**Capability in handler-mode.** The `compute/apply.capability` and `compute/apply.resource` fields (§4.3) apply here: when an entity-native handler dispatches to another handler, the dual-check constrains what the dispatch can reach. The handler grant is always the ceiling, even if the caller has broader authority. See §4.3 for the full security model.

### 9.2 Reactive compute subgraphs

Entity-native handlers are dispatched on demand (an EXECUTE arrives). Reactive compute subgraphs are the event-driven complement: installed expressions re-evaluate automatically when tree dependencies change.

| | Entity-native handler | Reactive subgraph |
|---|---|---|
| Trigger | EXECUTE request | Tree dependency change |
| Registration | `system/handler:register` with `expression_path` | `system/compute:install` |
| Input | EXECUTE params entity | Tree entities at dependency paths |
| Output | EXECUTE response entity | Result entity at result path |
| Budget | Per-request, from caller's capability | Per-trigger, from installation grant |
| Hot-swap | Replace expression at expression_path | Replace expression at root path |
| Cascade | No (request/response) | Yes (result write triggers downstream) |

Both use the same evaluator, expression types, budget/depth, and TCO. The difference is what triggers evaluation and where the result goes.

**Cascade chains.** Subgraph A's result write triggers subgraph B's re-evaluation. The tree write notification (SYSTEM-COMPOSITION §1.1) fires the reactive engine, which re-evaluates B. Synchronous, bounded by cascade depth limit.

**Result wrapping.** Reactive results are written as `compute/result{value: V, expression: H}`. Downstream subgraphs use `compute/field("value", ...)` to extract the inner value.

### 9.3 Library as entities

**`map`/`filter`/`fold` are standard builtins (§10.2) — you don't hand-roll them.** The pattern below is for *your own* library functions; it also illustrates how the builtins themselves can be expressed (over `compute/index`/`compute/length` + recursion). With TCO, library functions are lambda entities in the tree:

```
system/compute/lib/fold   = lambda(fn, acc, items, idx):
                              if idx >= length(items) then acc
                              else fold(fn, fn(acc, items[idx]), items, idx+1)
system/compute/lib/map    = lambda(fn, items): fold(...)
system/compute/lib/filter = lambda(pred, items): fold(...)
```

Content-addressed, transferable, inspectable, versionable. Entity-native handlers and compute subgraphs reference library functions via `lookup/tree` (live, reactive) or `lookup/hash` (pinned version, pure).

**Performance.** Interpreted lambda chains: ~5 ops per function call. At 100K budget, ~20K elements per fold vs ~100K for compiled builtins. For hot-path processing, `compute/apply` handler mode dispatches to language-native handlers — the escape hatch from entity-native interpretation to compiled code.

### 9.4 Transferable compute

A compute subgraph with relative paths is a self-contained subtree. The `relative` field (`compute/lookup/tree` and `compute/lookup/hash`, v3.3) is what makes this work: when `relative: true`, the lookup's path is resolved against the subgraph root rather than the absolute peer namespace. An expression using only `relative: true` lookups for its internal references is portable — its references survive a move to a different prefix or peer.

```
app/compute/my-job/                   ← root (subgraph root)
app/compute/my-job/data/input         ← data (relative: "data/input")
app/compute/my-job/lib/normalize      ← helper lambda (relative: "lib/normalize")
app/compute/my-job/result             ← result
```

Transfer the subtree to another peer, install at a different prefix — `relative: true` references resolve against the new root.

**Transferring a closure (v3.19b/c, N7 — semantics pinned, wire deferred).** A `compute/closure` that captured scope is also transferable: its reachable scope subtree (the `kind:"entity"` bindings, transitively) travels in the envelope `included` map — the same materialized-subtree mechanism continuations and installed subgraphs already use. The *semantics* are pinned (what travels, how it resolves on arrival, and `scope_unreachable` if a needed binding is absent on both sides — §7.2); the cross-peer *wire build* is deferred until a consumer needs it, so today this is a documented contract rather than a shipped path. When it ships it must also honor cross-peer capability-provenance (V7 §5.8/§3.5), not just subtree bundling.

For entity-native handlers: the `expression_path` points within the handler's subtree. Transfer handler manifest + expression + data, register at the new peer. The expression path is part of the portable structure.

The three models form a progression: precompiled for system infrastructure, SDK language-native for application logic that needs compiled performance, entity-native for logic that needs to be transferable, inspectable, and auditable. `compute/apply` handler mode bridges entity-native expressions to language-native handlers when performance requires it.

---

## 10. Compute as a compilation target

The expression language is also a compilation target — an SDK can compile a higher-level language down to entity graphs. But the dynamic handler model means many use cases don't need compilation at all. A pipeline of named functions, wired by reactive dependencies, configured by tree writes — that's programming in the entity system.

### 10.1 What makes it a complete substrate

The core expression types form a lambda calculus with extensions: `let` bindings, `lambda`/`apply`, `if` conditionals, arithmetic/comparison/logic, and entity construction/field extraction. With TCO, this is Turing-complete up to the budget bound.

Content-addressed expression graphs are a natural intermediate representation:
- **Deduplication is free** — shared sub-expressions have the same hash, stored once
- **Determinism is structural** — same graph, same result, on any peer, at any time
- **Transfer is built-in** — expression entities are storable, transferable, inspectable like any other entity
- **Purity is explicit** — the type system distinguishes pure from impure at the expression level

### 10.2 What the compiler/SDK layer handles

- **Recursion construction** — The fixpoint bootstrap for pure hash-based recursion is a mechanical transformation.
- **Data structures** — Lists, maps, and iteration can be encoded as entity structures with `construct`/`field`/`index`/`length`. The collection operations (`map`, `filter`, `fold`) are **MUST-given-COMPUTE** builtins with spec-pinned argument types (`system/compute/{map,filter,fold}-args`): every COMPUTE-bearing peer provides them with identical semantics, so a transferred expression using them runs the same everywhere. The lambda argument is named **`fn`** uniformly across all three (v3.19a, F11 — `filter`'s old `predicate` field was renamed for uniformity). (They MAY be implemented as a canonical compute-expression over `index`/`length`/recursion, or natively with hash-equal results.)
- **Pattern matching** — Lowered to chains of `if`/`eq` checks.
- **Variable scoping** — The SDK emits the right `lookup/scope` vs `lookup/tree` vs `lookup/hash` references.
- **Graph construction** — Building entity DAGs by hand is verbose (15-30 entities for a simple function). A builder API that takes a higher-level representation and emits the entity graph is the SDK's primary value.
