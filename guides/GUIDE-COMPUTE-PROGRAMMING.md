# Compute: Programming Guide

**Status**: Active
**Audience:** Application developers building with the compute extension from the SDK side. Assumes you have read GUIDE-COMPUTE.md (the language reference) and want to know how to actually construct, install, and run compute expressions from program code.
**Spec reference:** EXTENSION-COMPUTE.md v3.19c; ENTITY-CORE-PROTOCOL.md §3.2 (path-as-resource convention, empty-params wire shape); SDK-OPERATIONS.md §11.3 (Handler Execution Models), §11.6 (Dynamic Handler Registration), Addendum A (notation conventions); SDK-EXTENSION-OPERATIONS.md §8 (Compute).
**Related guides:** GUIDE-COMPUTE.md (language reference), GUIDE-CAPABILITIES.md (capability model the compute handler enforces).

**Notation.** Code blocks use the SDK spec's illustrative pseudocode (SDK-OPERATIONS Addendum A): `;` for comments, `entity(type, data)` for entity construction, `peer.execute(uri, op, params, resource?)` for dispatch, `peer.tree.put(path, entity)` and `peer.tree.get(path)` for tree access. Each language implementation translates these into its own idioms. The validate-peer command in `entity-core-go/cmd/internal/validate/{compute,entity_native}.go` is the working source — patterns are extracted from it but expressed here in language-neutral form.

---

## 1. Two ways to look at compute from your code

You write **expressions** (entities of type `compute/literal`, `compute/lookup/*`, `compute/apply`, etc.) and you ask the compute handler to do one of two things with them:

- **Eval once.** `EXECUTE system/compute operation: "eval" resource: {targets: [<expression path>]}` → the handler reads the expression at that path, evaluates, returns the result entity in the response. Synchronous, no persistent state.
- **Install reactive.** `EXECUTE system/compute operation: "install" resource: {targets: [<root expression path>]}` → the handler audits the expression graph, captures dependencies, persists a subgraph entity, and writes the result to `{root}/result`. Re-evaluates whenever a dependency changes. Persistent until you `uninstall`.

A third use shows up indirectly: an **entity-native handler** is a `system/handler` whose `expression_path` points to a compute expression the dispatcher evaluates per request, with the EXECUTE's `params` and `operation` injected into the expression's scope. Application dispatches against the handler's URI; the expression returns the response.

This guide is about how to put expression entities into the tree, hand them to the compute handler, and read results back.

---

## 2. Building expressions from code

### 2.1 The bottom-up pattern

Expression entities reference each other by content hash. `compute/arithmetic` has `left: hash, right: hash`; `compute/let` has `bindings: [{name, value: hash}], body: hash`; `compute/apply` has `fn: hash, args: {name: hash}`. To build any composite expression, you build the leaves first, store them, then assemble.

The canonical helper:

```
; put_expr — store an expression entity and return its content hash so
; the next layer can reference it.
put_expr(path, expr_entity) → Hash:
  peer.tree.put(path, expr_entity)
  return expr_entity.content_hash
```

`tree.put` writes to both the content store (keyed by content hash) and the location index (keyed by path). For composing expressions, what you need is the hash. The path matters when you want to reference the expression by tree lookup later (hot-swapping, recursive self-reference).

The validate-peer suite calls this helper `putCE` (compute.go:2287-2290). Each language implementation will spell it differently, but the shape is the same: store-and-return-hash.

### 2.2 Worked example: `(a + 4) * 2`

```
; Leaves
H_a    = put_expr(tp + "/a",    entity(compute/literal, {value: 7}))
H_4    = put_expr(tp + "/lit4", entity(compute/literal, {value: 4}))
H_2    = put_expr(tp + "/lit2", entity(compute/literal, {value: 2}))

; (a + 4)
H_sum  = put_expr(tp + "/sum",  entity(compute/arithmetic,
                                       {op: "add", left: H_a, right: H_4}))

; (a + 4) * 2
H_expr = put_expr(tp + "/expr", entity(compute/arithmetic,
                                       {op: "mul", left: H_sum, right: H_2}))
```

After this, `tp + "/expr"` contains a `compute/arithmetic` entity whose left hash points at the sum entity, whose left hash points at the literal 7, and so on. Any peer with the content store can resolve the entire graph.

### 2.3 Other common shapes

**`compute/let` with sequential bindings:**

```
H_x_val  = put_expr(tp + "/x-val", entity(compute/literal, {value: 10}))
H_body   = put_expr(tp + "/body",  /* expression that uses lookup/scope("x") */)
H_let    = put_expr(tp + "/let",   entity(compute/let, {
                                             bindings: [{name: "x", value: H_x_val}],
                                             body:     H_body,
                                          }))
```

**`compute/lambda` capturing scope:**

```
; lambda(n, acc): if n <= 0 then acc else apply(self, {n: n-1, acc: acc+n})
H_body   = put_expr(tp + "/body", /* assembled bottom-up */)
H_lambda = put_expr(tp + "/sum-iter",
                    entity(compute/lambda, {params: ["n", "acc"], body: H_body}))
```

**`compute/apply` calling a closure:**

```
H_f   = put_expr(tp + "/f-ref", entity(compute/lookup/scope, {name: "f"}))
H_arg = put_expr(tp + "/arg",   entity(compute/literal, {value: 5}))
H_app = put_expr(tp + "/apply", entity(compute/apply, {
                                          fn:   H_f,
                                          args: {n: H_arg},
                                       }))
```

**`compute/lookup/tree` reading a tree path:**

```
; Always pass an absolute path: /{peer_id}/{tree path}
qual = "/" + peer_id + "/" + data_path
put_expr(tp + "/lookup", entity(compute/lookup/tree, {path: qual}))
```

`lookup/tree` is impure — it reads the tree at eval time. The handler grant must cover the read. (See §5.)

**`compute/index` and `compute/length` over an array (v3.14+):**

```
H_arr  = put_expr(tp + "/arr",  /* an expression evaluating to a CBOR array */)
H_idx  = put_expr(tp + "/idx",  entity(compute/literal, {value: 2}))
H_elem = put_expr(tp + "/elem", entity(compute/index,  {array: H_arr, index: H_idx}))
H_len  = put_expr(tp + "/len",  entity(compute/length, {array: H_arr}))
```

Out-of-range or negative index → `index_out_of_range`; `index`/`length` on a non-array → `type_mismatch`.

**`compute/numeric-cast` for unsigned division (v3.14+):**

```
; unsigned div of a value in [2^63, 2^64): the cast must be the DIRECT operand
H_y      = put_expr(tp + "/y",      /* value expression */)
H_y_uint = put_expr(tp + "/y-uint", entity(compute/numeric-cast, {value: H_y, to_type: "primitive/uint"}))
H_2      = put_expr(tp + "/two",    entity(compute/literal, {value: 2}))
put_expr(tp + "/udiv", entity(compute/arithmetic, {op: "div", left: H_y_uint, right: H_2}))
; NOTE: `let y_uint = cast(y) in div(y_uint, 2)` is signed-default — the unsigned
; intent is consumed at the operation, never carried through a binding (rule 11).
```

**`map` / `filter` / `fold` — standard builtins (MUST-given-COMPUTE, v3.14+):**

Invoke via `compute/apply` against the logical builtin path with the pinned args fields. Implementations typically evaluate these *internally* — the path is a canonical operation name, not necessarily a tree-resident handler.

```
; double every element: map(λx. x*2, collection)
H_x     = put_expr(tp + "/x",     entity(compute/lookup/scope, {name: "x"}))
H_2c    = put_expr(tp + "/2",     entity(compute/literal, {value: 2}))
H_dbl_b = put_expr(tp + "/dbl-b", entity(compute/arithmetic, {op: "mul", left: H_x, right: H_2c}))
H_dbl   = put_expr(tp + "/dbl",   entity(compute/lambda, {params: ["x"], body: H_dbl_b}))

put_expr(tp + "/mapped", entity(compute/apply, {
  path:      "system/compute/builtins/map",
  operation: "eval",
  args:      {collection: H_arr, fn: H_dbl},   ; system/compute/map-args fields
}))
```

`filter` takes `{collection, fn}` (unary closure → bool); `fold` takes `{collection, fn, initial}` (binary closure `(acc, element) → acc`). The lambda arg is **`fn`** in all three (v3.19a, F11 — `filter`'s field was `predicate` before v3.19; update any pre-v3.19 builder code). Semantics are pinned: `map`/`filter` preserve index order, `fold` threads `initial` left-to-right.

**Navigation composes; constructed entities are bare (v3.19).** Two value-model facts a builder must encode:
- `compute/field`/`index`/`length` compose to arbitrary depth, and `field` reads an entity *or* a record value — so `field("name", index(items, 0))` lowers directly. Disambiguation is by the value's `kind`, never by sniffing keys (EXTENSION-COMPUTE §2.2/§2.3, N3).
- `compute/construct`'s result, once it leaves compute eval (stored / returned / passed as an `apply` arg / transferred), is a **plain bare entity == hand-built**: entity-valued fields are bare `system/hash` refs, no kind-tags. When you navigate back into one and hit a `system/hash` field, you get the **hash** — follow it explicitly with `compute/lookup/hash`; do not auto-resolve, and never size/slice a `system/hash` by a fixed byte length (it is variable-length, V7 §1.2). Identify refs by *type*.

### 2.4 SDK status of an expression builder

SDK-EXTENSION-OPERATIONS §8 currently flags an expression builder as planned-not-yet-specified. For now, idiomatic SDK code constructs entities of each type directly and uses a `put_expr` helper as above. A higher-level builder API (`literal()`, `lookup_tree()`, `apply()`, `let_in()`, `lambda()`, `index()`, `length()`, `numeric_cast()`, `map()`/`filter()`/`fold()`, etc.) is on the roadmap — when it lands, the bottom-up pattern is what it will compile to. Each language impl will surface this differently (typed builder structs, fluent APIs, S-expression DSLs); the underlying entity graph is the same.

---

## 3. Eval: synchronous one-shot

The handler is `system/compute`. You tell it which expression to evaluate by giving it a tree path in `resource`. Compute expressions can live at any tree path — they don't have to live under `system/compute/`.

```
; Expression previously stored at app/sheet/B1 (any tree path works).
expr_path = "app/sheet/B1"

uri      = "entity://" + peer_id + "/system/compute"
resource = {targets: ["/" + peer_id + "/" + expr_path]}
params   = empty()                                      ; or {budget: ...}

response = peer.execute(uri, "eval", params, resource)
```

What's where:

- **URI** = `system/compute`. Dispatch routes by longest-prefix match on the URI; nothing past `system/compute` is needed.
- **`resource.targets[0]`** = the tree path of the expression. The handler reads it from `ctx.resource.targets[0]` and does `entity_tree.get(...)` to fetch the expression.
- **`params`** = empty (or `{budget: ...}` to override the default budget). Operation-specific options only — the path is in resource per V7 §3.2's path-as-resource convention.

The path lives in one place: `resource`. The dispatch capability check authorizes against it; the handler reads from it. One source of truth.

The empty-params shape is `entity(primitive/any, {})` — a `primitive/any` entity whose data is the canonical CBOR encoding of an empty map (single byte `a0`). Each SDK should expose a single helper for constructing this. See V7 §3.2 for the normative wire shape.

### 3.1 Decoding the response

`response.result` is a CBOR-encoded entity:

```
if response.status != 200:
  ; REQUEST-level failure — malformed/unauthorized EXECUTE, transport, handler
  ; not found, or install pre-audit rejection. See §7.
  ...

result_entity = decode_entity(response.result)

; A successful eval (status 200) may STILL carry a propagated error in the body:
; an evaluated compute/error is a VALUE returned at 200 (v3.19a, F10).
if result_entity.type == "compute/error":
  ; eval-time error (incl. an impure-op permission_denied during eval) — see §7.1
  return handle_error(result_entity.data.code)

if result_entity.type == "compute/result":
  return result_entity.data.value
return result_entity
```

The handler wraps the value in `compute/result` for the explicit eval path. Entity-native handler dispatch (§6) skips the wrapper — the response is the raw value entity.

**The status/body split (v3.19a, F10) — internalize this.** A `4xx` means the **request** failed (unauthorized EXECUTE, transport, handler-not-found, install pre-audit). A successful eval that *produces* a `compute/error` — including a `permission_denied` raised while *evaluating* an impure op — comes back at **status 200** with the error in the body, because errors are values that compose. **So you cannot tell success from a propagated error by status alone; you must decode the 200 body and check for `compute/error`.** (The request/dispatch-level denial check in §7.5 — `4xx` + `compute/error` body — is the other half: that one *is* a transport-level denial and stays `4xx`.)

### 3.2 Useful helpers

```
; Eval an expression at a tree path; return the raw response.
compute_eval_at_path(peer_id, expr_path) → Response

; Decode the value out of a successful eval response.
decode_compute_value(response) → any

; Build it, store it, eval it, give me the answer.
compute_eval_and_extract_value(peer_id, path, expr_data) → any:
  ent = expr_data.to_entity()
  peer.tree.put(path, ent)
  return extract_result_value(peer_id, path)

; Decode the error code from an error response.
decode_compute_error_code(response) → string
```

`compute_eval_and_extract_value` is the convenience for "build it, store it, eval it, give me the answer." Used heavily in validate-peer's compute test suite for ~100 deterministic expression checks.

---

## 4. Install: reactive subgraphs

### 4.1 Install

```
; 1. Build and store the expression at some path.
b1_path = "app/sheet/B1"
peer.tree.put(b1_path, /* assembled expression — typically uses lookup/tree of inputs */)

; 2. Send install with the qualified path in resource.
qual_b1 = "/" + peer_id + "/" + b1_path
uri      = "entity://" + peer_id + "/system/compute"
resource = {targets: [qual_b1]}
params   = empty()                                      ; or {result_path: "...", budget: ...}
response = peer.execute(uri, "install", params, resource)

if response.status != 200:
  ; failed audit — see §6
  ...

; 3. Result will appear at b1_path + "/result" once initial eval completes.
result_path = qual_b1 + "/result"
```

What the handler does (EXTENSION-COMPUTE §3.3 and §7):

- **Audit the graph.** Walks the expression starting at the resource path, follows hash refs to children, collects the closure environment, identifies impure operations (`lookup/tree`, `apply` to handlers), and runs install-time capability checks (R0/R1; see GUIDE-CAPABILITIES §5.4).
- **Persist a subgraph entity** at `system/compute/processes/{id}` recording the root path, dependencies, installation grant, and result path.
- **Evaluate once** to populate `{root_path}/result`.
- **Subscribe to dependencies.** Subsequent writes to any tree path the expression depends on trigger re-evaluation.

### 4.2 Read the result

The result is just an entity at `{root}/result` (where `root` is the resource path you installed against). Read it like any other tree binding:

```
ent = peer.tree.get(b1_path + "/result")
if ent.type == "compute/result":
  return ent.data.value
```

For tighter feedback loops, subscribe to that path (subscription extension) and react to notifications.

### 4.3 Triggering re-evaluation

Anything that mutates a path the expression depends on. The simplest case: the spreadsheet pattern from validate-peer:

```
; Setup: A1 = literal 10, B1 = A1 * 2 installed reactively.
peer.tree.put("app/sheet/A1", entity(compute/literal, {value: 10}))
; ... build B1 = lookup/tree(A1) * 2, install ...

; Mutation triggers re-eval:
peer.tree.put("app/sheet/A1", entity(compute/literal, {value: 25}))
; After cascade settles, /app/sheet/B1/result holds 50.

peer.tree.put("app/sheet/A1", entity(compute/literal, {value: 0}))
; /app/sheet/B1/result holds 0.
```

The compute handler doesn't poll — it observes tree mutations through the emit pathway (SYSTEM-COMPOSITION §1.1) and fires re-eval when one of its tracked dependencies changes. See GUIDE-COMPUTE §5 for the cascade rules.

### 4.4 Uninstall

```
uri      = "entity://" + peer_id + "/system/compute"
resource = {targets: [subgraph_path]}
params   = empty()
peer.execute(uri, "uninstall", params, resource)
```

This removes the subgraph entity and the dependency subscriptions. The result entity at `{root_path}/result` is left behind; the application can clean it up if desired. (Uninstall semantics — particularly whether/how to tombstone the result — are evolving; check EXTENSION-COMPUTE §3.1 for the current contract.)

### 4.5 Frozen subgraphs

If a re-evaluation hits the cascade depth limit, exceeds budget, or fails install-grant verification (the installer's grant was revoked), the subgraph's status is set to `frozen` with an error code (e.g., `cascade_limit`, `installation_grant_invalid`). Frozen subgraphs do not re-evaluate until manually unfrozen. Read the subgraph entity at `system/compute/processes/{id}` to inspect status and the last error.

---

## 5. The capability model for compute

Compute is the most security-sensitive extension because expressions can dispatch to handlers (via `compute/apply`) and read tree state (via `compute/lookup/tree`). The model has three layers; understanding which one fires when is essential for debugging "why was my expression rejected?"

### 5.1 The handler grant ceiling

Every entity-native handler is registered with an `internal_scope` — the grant the dispatcher uses when evaluating its expression. The expression's impure operations (tree reads, sub-dispatches) are checked against this grant. **The handler's grant is a ceiling.** No matter what capability the caller passes in, the expression cannot exceed the handler's grant.

In validate-peer's `lookup_tree_outside_scope` test, the handler's grant covers `/{peer_id}/app/validate/entity-native/lto/*` only. The expression tries to read `/{peer_id}/app/validate/entity-native-forbidden/value`. The dispatcher denies the read with a 4xx + `compute/error` body — the dispatch never reaches the data even though the caller's capability was wildcard.

### 5.2 The dual-check (caller cap + handler grant)

`compute/apply` to a handler accepts an optional `capability` field — a hash to a capability the apply should dispatch with, *overriding* the handler's grant. This is how an entity-native handler runs a sub-dispatch under the caller's authority (e.g., proxy-style handlers).

The dispatcher enforces a **dual-check**: both the handler's grant AND the provided capability must cover the target operation+resource. If either fails, the dispatch is denied. This is the EXTENSION-COMPUTE F1-F5 model from PROPOSAL-COMPUTE-APPLY-RESOURCE-CEILING.

```
; validate-peer dual_check_both_pass — both handler grant and provided cap
; must cover system/tree:get on the target resource.
H_apply_resource = put_expr(slot + "/apply-res-lit", entity(compute/literal, {
  value: {targets: ["/" + peer_id + "/" + data_path]},  ; system/protocol/resource-target
}))

H_caller_cap    = put_expr(slot + "/cap-lookup",
                           entity(compute/lookup/scope, {name: "caller_capability"}))

H_resource_arg  = put_expr(slot + "/res-lit",
                           entity(compute/literal, {value: "/" + peer_id + "/" + data_path}))

put_expr(expr_path, entity(compute/apply, {
  path:       "system/tree",
  operation:  "get",
  resource:   H_apply_resource,                    ; F1+F4: thread resource through
                                                   ;        to dispatched EXECUTE
  args:       {resource: H_resource_arg},
  capability: H_caller_cap,                        ; F5: override capability
}))
```

**`resource` is required when `capability` is set** (F5 structural rule). Without it, the dual-check would resolve only to the operation+handler dimension and miss the resource ceiling — exactly the gap that proposal closed.

The `caller_capability` scope binding is provided by the dispatcher when invoking an entity-native handler. The expression references it via `lookup/scope("caller_capability")` and embeds the resulting hash in `compute/apply.capability`. This is the canonical "handler running under the caller's authority" idiom.

### 5.3 R0/R1 at install time

`system/compute:install` audits the expression graph for embedded capability literals (`compute/apply.capability` set to a `compute/literal` hash). For each one, it runs the chain-root check (R1, CP1) against the installer's identity. If the literal capability isn't in the installer's authority chain, install rejects with **403 `embedded_cap_unauthorized`**.

Dynamic capability values — `compute/apply.capability` set to a `lookup/scope("caller_capability")` or some other runtime expression — can't be checked at install time. They go through the runtime dual-check at eval time instead. Both paths converge on the same security guarantee; see GUIDE-CAPABILITIES §3.3.

**Check ordering.** Chain-root runs before resource-coverage at install time (cheaper, more fundamental). The error codes differ (`embedded_cap_unauthorized` vs `permission_denied`) so you can tell from the response which check failed.

### 5.4 What this means for your code

- For a basic entity-native handler that only reads/writes within its own subtree, give it a tight grant covering just that subtree. The handler will be unable to escape no matter how creatively the expression is written.
- For a handler that needs to dispatch on the caller's behalf, use `compute/apply` with `capability: lookup/scope("caller_capability")` and `resource: literal(target)`. The dispatcher dual-checks; the handler's grant is the upper bound.
- For a handler installing capabilities into long-lived expressions (literal `compute/apply.capability` references), make sure the cap chain you embed includes your own identity. R1 will reject anything else.

---

## 6. Entity-native handlers — write, register, hot-swap

### 6.1 The model

A `system/handler` entity has a `pattern` (the URI prefix it serves) and an `expression_path` (a tree path holding a compute expression). At dispatch time, the runtime:

1. Resolves the handler by longest-prefix match on the URI (V7 §6.6).
2. Reads the expression at `expression_path`.
3. Builds an evaluation scope: `params` (the EXECUTE's params entity), `operation` (the EXECUTE's operation string), `caller_capability` (the EXECUTE's capability), and any others the runtime defines.
4. Evaluates the expression. The result becomes the EXECUTE response.

The expression sees no compiled handler code. The handler IS the expression entity. Replace the entity, replace the handler — instantly, no restart.

### 6.2 Registration

```
manifest = entity(system/handler/manifest, {
  pattern: "app/myhandler",
  name:    "my-handler",
  operations: {
    ; Operations are advisory — entity-native handlers branch on
    ; lookup/scope("operation"); the runtime doesn't bind operation
    ; logic to declarations.
    compute: {input_type: "primitive/any", output_type: "primitive/any"},
    ping:    {input_type: "primitive/any", output_type: "primitive/any"},
  },
  expression_path: "app/myhandler/expr",
  internal_scope:  scope,                  ; grant the handler runs under
})

req = entity(system/handler/register-request, {
  manifest:        manifest,
  requested_scope: scope,
})

uri      = "entity://" + peer_id + "/system/handler"
response = peer.execute(uri, "register", req, null)
```

`system/handler:register` atomically writes the manifest, the interface entity, and the grant entry at `system/capability/grants/{pattern}`. Use the register operation rather than `tree:put` — this is the R0 case for handlers.

### 6.3 Calling it

```
; EXECUTE against the handler's URI; the handler's expression evaluates
; with operation="compute" and params={x:7} in scope.
uri      = "entity://" + peer_id + "/app/myhandler"
params   = entity(primitive/any, {x: 7})
response = peer.execute(uri, "compute", params, null)
```

The result entity in the response is the **raw expression value**, not a `compute/result` wrapper (PROPOSAL-ENTITY-NATIVE-HANDLER-DISPATCH E3 — entity-native handlers return the evaluated entity directly, so callers don't need to unwrap).

### 6.4 Hot-swap

The handler entity points at `expression_path`. Replace the entity at `expression_path`; the next dispatch picks up the new expression. No re-registration, no restart.

```
; Initial expression: literal(1)
peer.tree.put("app/myhandler/expr", entity(compute/literal, {value: 1}))
; dispatch returns 1

; Hot-swap to literal(99)
peer.tree.put("app/myhandler/expr", entity(compute/literal, {value: 99}))
; next dispatch returns 99
```

This is the property that makes entity-native handlers different from compiled handlers: their behavior is data, not code. The validate-peer `hot_swap_expression` test exercises this directly.

### 6.5 Branching on operation

Multiple operations on one handler is the same expression that branches on `lookup/scope("operation")`:

```
; Expression: if compare(operation, "double") then literal(20) else literal(30)
H_op     = put_expr(slot + "/op",         entity(compute/lookup/scope, {name: "operation"}))
H_double = put_expr(slot + "/lit-double", entity(compute/literal,      {value: "double"}))
H_cond   = put_expr(slot + "/cond",       entity(compute/compare,
                                                 {op: "eq", left: H_op, right: H_double}))
H_then   = put_expr(slot + "/then",       entity(compute/literal, {value: 20}))
H_else   = put_expr(slot + "/else",       entity(compute/literal, {value: 30}))
peer.tree.put(expr_path, entity(compute/if, {
  condition: H_cond, then: H_then, else: H_else,
}))
```

Same handler URI, different operations, different branches.

### 6.6 The fail-closed contract

If an entity-native handler is registered but its grant entity is missing (revoked, partial restore, storage corruption), the dispatcher MUST refuse to invoke it — status 403, no expression evaluation. This is PROPOSAL-ENTITY-NATIVE-HANDLER-DISPATCH §7.1, validated by `missing_grant_fail_closed`. The grant is what gives the handler authority to do anything; without it, the dispatcher has no basis to run the expression.

In practice this means: don't `tree:put` a handler entity directly to bypass `register`. Use the register operation, which atomically writes manifest + grant. Direct `tree:put` of the handler alone leaves the handler in the dispatch index but with no grant — every call returns 403. (validate-peer uses direct `tree:put` *intentionally* for the negative test, demonstrating the failure mode.)

---

## 7. Debugging

### 7.1 "My expression returned an error"

An eval that completes and yields an error returns at **status 200** with a `compute/error` entity in `response.result` (v3.19a, F10 — errors are values; see §3.1). Decode the error code:

```
decode_compute_error_code(response) → string:
  result = decode_entity(response.result)
  if result.type != "compute/error":
    return error("expected compute/error, got " + result.type)
  return result.data.code
```

Common error codes:

| Code | What happened |
|---|---|
| `invalid_expression` | The expression entity is malformed (missing fields, wrong types, structural violation like `capability` without `resource`). |
| `permission_denied` | An impure operation (tree read, sub-dispatch) hit a grant the handler or caller cap didn't cover. |
| `embedded_cap_unauthorized` | Install audit found a literal `compute/apply.capability` whose chain-root doesn't include the installer (CP1). |
| `installation_grant_invalid` | Reactive re-eval — installer's grant was revoked between install and re-eval. Subgraph freezes. |
| `cascade_limit` | Re-evaluation cascade exceeded the depth threshold. Subgraph freezes. |
| `budget_exhausted` | Eval consumed more budget (steps, allocations) than the configured limit. |
| `depth_exceeded` | Recursive evaluation exceeded the configured depth. |
| `index_out_of_range` | `compute/index` index is negative or ≥ array length. |
| `cast_out_of_range` | `compute/numeric-cast` float→integer is out of range, `NaN`, or ±`Inf`. |
| `scope_unreachable` | A resumed/transferred closure needs a scope binding absent from both the local content store and the envelope `included` map (v3.19b/c). A clean attributable error — never a silent substitution or partial result. |

> **`permission_denied` at 200, not 4xx (v3.19c).** When `permission_denied` arises from *evaluating* an impure op (a `lookup/tree` outside scope, a blocked `store`/dispatch), it is a propagated `compute/error` at **status 200** — decode the body (§3.1). A `4xx` `permission_denied` is a *request/dispatch-level* denial (an unauthorized EXECUTE, or an entity-native handler dispatch the grant doesn't cover — §7.4), a different thing.

### 7.2 "Install rejected my expression"

Run install with verbose tracing if your SDK supports it. The most common causes:

- **Literal capability whose chain-root isn't yours.** R1 (CP1) rejects. Switch to a capability you issued, or use a runtime lookup (`lookup/scope("caller_capability")`) and let the dual-check fire at eval time.
- **`compute/apply` with `capability` but no `resource`.** Structural — F5 rejects. Add a literal `system/protocol/resource-target` for the apply.
- **Cycle in the expression graph.** Audit walker detects and rejects.
- **Closure environment unreachable.** A `lookup/scope("foo")` references a binding the audit can't trace from the install root.

### 7.3 "Reactive subgraph isn't updating"

Things to check:

1. **Subgraph status.** Read the subgraph entity (typically at `system/compute/processes/{id}`); if `frozen`, look at the error field.
2. **Result path.** Is `{root}/result` actually populated (where `root` is the resource path you installed against)? If not, the initial eval failed silently (look at the install response, not the result path).
3. **Dependencies.** What paths does the expression read via `lookup/tree`? Are mutations actually landing at those paths? (A common mistake: reading `app/A1` while writing `app/sheet/A1`.)
4. **Cascade limits.** Long chains of dependent subgraphs can exceed the cascade depth. Restructure or raise the limit.

### 7.4 "Entity-native handler returns 403"

Either the grant doesn't cover the dispatch (caller cap missing/insufficient), or the grant entity is missing entirely. Check `system/capability/grants/{pattern}` — should be a grant matching the handler. If it's missing, re-register via `system/handler:register` (don't `tree:put` the handler entity directly).

### 7.5 Tracing tips

The validate-peer test suite uses two tools that are worth replicating in your application code:

- **Verbose mode** that prints wire-level request/response traces. Even a basic version (log the URI, operation, response status) speeds debugging dramatically.
- **`is_denial_response` predicate** — distinguish "the dispatcher reached the handler and the handler/runtime denied" (4xx + `compute/error` body) from "the dispatcher couldn't find the handler" (404 + generic error). The first is a contract violation; the second is misconfiguration. Treat them differently.

```
is_denial_response(response) → bool:
  if response.status < 400 or response.status >= 500:
    return false
  result = decode_entity(response.result)
  return result.type == "compute/error"
```

---

## 8. Performance notes

### 8.1 Memoization

Compute is content-addressed: same expression hash + same scope = same result. Implementations may memoize evaluation by `(expression_hash, scope_hash)`. Building expressions bottom-up (so identical sub-expressions get the same hash) lets memoization catch shared structure across calls.

For reactive subgraphs the handler may also cache per-dependency results so that a re-evaluation doesn't recompute branches whose inputs didn't change. See EXTENSION-COMPUTE §7 for what the spec says on caching; specifics are implementation-defined.

### 8.2 Tail-call optimization

The compute evaluator MUST treat tail-position calls as iteration (no stack growth). This is what makes recursion via `lookup/tree` self-reference (§2.3) practical. validate-peer's tail-recursive sum test validates this with `n` of order 10^4+ without stack issues.

### 8.3 When to bridge to language-native

Pure expressions are great for portability and reactivity. They're the wrong tool for:

- Long-running computations with large internal state
- Calls into external libraries (image processing, ML models, heavy crypto)
- Performance-critical inner loops where every entity hash adds overhead

The escape hatch: write a language-native handler for the heavy work and call into it from your entity-native expression via `compute/apply` (handler-mode). The expression remains content-addressed and reactive; the heavy lifting runs in compiled code. SDK-OPERATIONS §11.3 covers the three execution models — precompiled, language-native, entity-native — and when each fits.

---

## 9. Reference

### 9.1 Operations

| Operation | URI | Path source | What it does |
|---|---|---|---|
| `eval` | `system/compute` | `resource.targets[0]` | One-shot evaluation of expression at the given path |
| `install` | `system/compute` | `resource.targets[0]` | Install reactive subgraph; audit, persist, initial eval, subscribe |
| `uninstall` | `system/compute` | `resource.targets[0]` | Remove reactive subgraph and its dependency subscriptions |
| `register` (handler) | `system/handler` | `params.manifest.expression_path` (for entity-native) | Atomically register a handler manifest + grant (entity-native or language-native) |

### 9.2 Validate-peer files (Go reference impl)

The validate-peer command in `entity-core-go` exercises these patterns directly. The file mapping below points you at the canonical exercise of each shape — useful as a working reference even if your implementation language is different.

| File | What's in it |
|---|---|
| `cmd/internal/validate/compute.go` | ~100 deterministic eval tests; reactive spreadsheet (§4); install audit (CP1, runtime dual-check) |
| `cmd/internal/validate/entity_native.go` | Full entity-native handler contract: registration, scope params/operation, dual-check pass/block, hot-swap, fail-closed |
| `cmd/internal/validate/client.go` | `SendExecute`, `TreePut`, `TreeGet`, async EXECUTE plumbing |
| `cmd/internal/validate/continuations.go` | Continuation install/advance using compute results |
| `cmd/internal/validate/subscriptions.go` | Subscribing to compute result paths for change-driven flows |

### 9.3 Spec pointers

- EXTENSION-COMPUTE §3 — Operations and audit walk
- EXTENSION-COMPUTE §7 — Reactive evaluation, dependency tracking, cascade
- EXTENSION-COMPUTE §1.1 — Coherent capability (R0/R1, CP1)
- ENTITY-CORE-PROTOCOL §6.6 — Handler dispatch (longest prefix), §3.7 — Handler types
- SDK-OPERATIONS §11.3 — Three handler execution models
- SDK-OPERATIONS §11.6 — Dynamic handler registration
- SDK-OPERATIONS Addendum A — Notation conventions used in this guide
- SDK-EXTENSION-OPERATIONS §8 — Compute SDK shape (eval, install, uninstall)
- GUIDE-COMPUTE.md — Expression language reference (types, semantics, patterns)
- GUIDE-CAPABILITIES.md — Capability model, kernel-vs-handler, R0/R1
