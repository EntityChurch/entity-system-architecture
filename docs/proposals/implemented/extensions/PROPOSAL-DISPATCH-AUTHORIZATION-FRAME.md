# PROPOSAL — the resource check follows the field, not the door; and a stored capability is still its granter's

**Status:** **IMPLEMENTED 2026-08-18** — all four deltas folded, plus two items the cohort found
while implementing. D1 `ENTITY-CORE-PROTOCOL` §5.2 · D2 `EXTENSION-CONTINUATION` §3.5 ·
D3 §5.5a helper list · D4 vector. **Added at fold: the no-resource ruling** (a sub-dispatch naming
no resource has none — it MUST NOT inherit the parent's; go aligns to py+rust, §7) and the
**grantless/self third state** raised by `entity-core-rust` (§8). `ENTITY-CORE-PROTOCOL` 0.8.0 → 0.8.2.
**Tier:** extensions + core — `ENTITY-CORE-PROTOCOL` §5.2, §5.5a; `EXTENSION-CONTINUATION` §3.5.
**Answers:** `entity-core-py` `ROUTING-2026-08-18-b` SA-PY-9 and SA-PY-10 (`dev` @ `06162e5`).
**Read at:** arch `1782d6d` · core-protocol `d382d2c` · core-go `94df9e6` · core-py `961f5fe` ·
core-rust `1a6955b`

---

## §0 Summary of the ruling

| Ask | Ruling |
|---|---|
| **SA-PY-9 — does §5.2's dispatch resource check bind an *internal* sub-dispatch?** | **Yes — and the spec already says so; the question is mis-framed and the better framing convicts.** §5.2 conditions the resource dimension on **the presence of a resource on the EXECUTE**, never on the dispatch's provenance. `entity-core-go` does not take the "resource absent" branch — it **computes a child resource, holds it, and constructs a synthetic `ExecuteData` that omits it** before the check. That is dropping a field, not exercising a carve-out. **Non-conformant, not ambiguous.** §1 |
| **SA-PY-10 — which peer frames a *stored* `dispatch_capability`'s resources?** | **Granter-framed. `entity-core-py` is right, and §5.5a already names the losing side non-conformant by class.** There is no "acceptance re-frames it" reading available: §5.5a's *"cross-peer cap minting helpers … MUST use explicit cross-peer form"* covers exactly `installJoinFromData`. **py's WARN against core-go's fixture is correct behavior and MUST NOT be tuned away.** §2 |
| *(found while ruling)* | **`EXTENSION-CONTINUATION` §3.5's worked example is a spec defect and it is the proximate cause.** It spells a cross-peer dispatch capability in peer-relative form — `["peer_b/data/shared/*"]` — which under §5.5a authorizes `/{granter}/peer_b/data/shared/*`, i.e. nothing. **The example teaches the exact error the fixture makes.** §3 |

**No wire change, no new entity type, no new error code, no renumber.** D1 is a clarifying sentence;
D2 is a one-character fix to an example plus its explanation; D3–D4 are conformance surface.

---

## §1 SA-PY-9 — the check is conditioned on the field, not on the door

### What §5.2 actually says

`check_permission`'s own pseudocode (`ENTITY-CORE-PROTOCOL` §5.2, `d382d2c`):

```
check_permission(execute, capability, handler_pattern, local_peer_id):
  ; Called after handler resolution (§6.5). Checks all four grant dimensions.
  ; When resource is present, the same grant must match all four dimensions.
  ; When resource is absent, resource dimension is unchecked at dispatch —
  ; handler may still check internally (§6.3, §6.7).
  resource_target = execute.data.resource                          ; may be null
  ...
    if resource_target is not null:
      if not check_resource_scope(resource_target, grant.resources, local_peer_id):
```

**The conditional is `resource_target is not null`.** There is no wire-entry predicate, no
`is_sub_dispatch` flag, and no sentence anywhere in §5.2 that distinguishes the two. The spec never
had the ambiguity the ask assumes, because it never had the axis.

### Why the "it's ambiguous" reading does not survive contact with core-go's source

`entity-core-go` `core/protocol/local.go`, `(*Dispatcher).makeLocalExecute` @ `94df9e6`:

```go
if !capability.CheckPermission(types.ExecuteData{Operation: operation}, capData, pattern, …) {
```

…and **thirty lines later**, in the same closure:

```go
childResource := callerCtx.Resource
if execOpts.Resource != nil { childResource = execOpts.Resource }
childResource = normalizeResourceTargets(childResource, d.LocalPeerID)
…
childCtx := &handler.HandlerContext{ … Resource: childResource, … }
```

**The dispatch has a resource. The check is handed a literal that does not.** Under §5.2 that is not
"resource absent at dispatch" — the resource exists, is computed from caller-influenced input, and is
propagated into the child context as the authorization target. `capability.CheckPermission` reaches
`Dimension 3: Resources (when specified on execute)` (`core/capability/check.go`) and skips it,
because the caller manufactured a struct in which it was never specified.

**Guards accounted for on the path** (per this repo's L8 twelfth-form rule — a "no check" claim must
rule out the guards, not just cite the site). Between closure entry and the `CheckPermission` call
`makeLocalExecute` performs: a `chain_depth` ceiling (429), handler resolution (404), the
`grantToCheck` selection with a 403 `missing_handler_grant` when no grant exists, granter resolution
(403), the L1 operation+handler-pattern check, and `loadValidatedGrant(pattern)` for the **child
handler's** grant. **None of the six is the resource dimension**, and `loadValidatedGrant` validates
the callee's install grant rather than the caller's scope over the target path. The resource
dimension is unreached on this path, full stop.

### The corroboration the ask already found, restated as the decisive one

`ENTITY-CORE-PROTOCOL` §6.2:

> **Resource carries the install path (§3.2 path-as-resource).** Both `register` and `unregister`
> derive the pattern from `EXECUTE.resource.targets[0]` … The handler path is the caller's
> authorization target — **the standard dispatch capability check on `resource`** validates that the
> caller may install/remove a handler at that path.

`register` **always** carries a resource — it is where the pattern lives. So on any dispatch path,
`resource_target is not null` holds and §5.2's conditional fires. A path that skips it leaves
`system/handler:register` authorized by nothing, which is what core-py measured in its own tree
before `00b2c05`: a sub-dispatch under a grant scoped to `app/*` installed at pattern `pwn`.

**This is not a new requirement and does not need one.** It needs a sentence that forecloses the
mis-reading, because two of three implementations reached it.

### Disposition

- **`entity-core-py`** — conformant as of `00b2c05`. The reading was right and the fix is right.
- **`entity-core-go`** — **non-conformant.** Not a spec gap to wait on; the fix is to pass the
  resource it already computes. Ordering note in §4.
- **`entity-core-rust`** — **unverified, and we did not verify it.** `core/handler/src/lib.rs`
  carries both `resource_target` on the context and a `resource: Option<ResourceTarget>` documented
  as *"Override resource target for the child dispatch"*, so the field is plumbed; whether the check
  consults it on the in-process path is **an open question routed to that seat**, not a finding. We
  did not run an exhaustive search and make no absence claim.

---

## §2 SA-PY-10 — granter-framed, and the text is not close

### The ruling

**A capability's resource patterns canonicalize against its granter's `peer_id`, and nothing about
being stored, accepted, or later wielded by the verifier changes that.** `entity-core-py`'s first
reading is correct.

### Why there is no second reading

`ENTITY-CORE-PROTOCOL` §5.5a, in landed normative text:

> A capability's resource patterns canonicalize against its granter's `peer_id`. Consequently,
> peer-relative resource patterns (bare `*`, `system/foo`, etc.) on a capability are
> **granter-local**: they authorize action only within the granter's own namespace.
>
> A capability minted to authorize action against **another peer's** namespace (cross-peer dispatch)
> MUST express its resources in explicit cross-peer form … A bare peer-relative resource on a
> foreign-granted cap does NOT authorize the target peer's namespace; presenting such a cap
> cross-peer MUST fail at dispatch with **403 `capability_denied`**.

And then, naming the class the failing fixture belongs to:

> **Operator-facing consequence**: cross-peer cap minting helpers (test fixtures, integration test
> scaffolds, role-derived token helpers, multi-sig helpers) MUST use explicit cross-peer form for cap
> resources. **Helpers that use peer-relative form on foreign-granted caps are non-conformant.**

A continuation's `dispatch_capability` is minted by A, persisted at B, wielded by B against B's own
namespace. Granter is A. It is a foreign-granted cap presented against another peer's namespace —
the case §5.5a was written for. **`entity-core-go`'s `installJoinFromData` → `CreateDispatchCapability`
(granter = client identity, resources = peer-relative join targets) is a test-fixture minting helper
using peer-relative form on a foreign-granted cap.** That sentence disposes of it by name.

**The "acceptance re-frames it" alternative is foreclosed structurally, not just textually.** §5.5a's
stated reason for the whole rule is that the granter frame and the verifier frame are *byte-identical
for same-peer capabilities*, so only the foreign-granter case exposes the bug. A rule that re-frames
on acceptance would restore exactly the latency §5.5a exists to remove, and — as core-py argued and
we adopt — it is reachable by an attacker who arranges for something to be stored. That is V2(a)
under-enforcement through a second door.

### The consequence we are explicitly ratifying

**`continuations.join_target_received` reading WARN against `entity-core-py` is the correct result.**
The checker is right and the fixture is wrong. core-py declined to soften its checker to make the
symptom disappear and reported the trace instead; **that was the correct call and this ruling
vindicates it.** The fix is core-go's, in its validator fixture, and it is one form change:
`peer_b/data/shared/*` → `/peer_b/data/shared/*`.

### Where the blame actually sits

Not with the fixture author. See §3 — the spec's own worked example told them to write it that way.

---

## §3 The example that taught the error

`EXTENSION-CONTINUATION` §3.5, **Capability interaction** (arch `1782d6d`):

> A capability scoped to `{resources: {include: ["peer_b/data/shared/*"]}}` authorizes any
> dynamically-extracted path matching that pattern.

**Under §5.5a this authorizes `/{granter_peer_id}/peer_b/data/shared/*` — a path in the granter's own
namespace with a literal segment named `peer_b`. It authorizes nothing at peer B.** The example is
evidently *intending* peer B (it says so in the name it chose), which is precisely the mis-reading
§5.5a's cross-peer-form MUST exists to prevent.

**This is the highest-leverage item in the proposal.** A normative rule and a worked example that
contradict it do not split the difference — implementers copy the example. One seat's fixture
already did, a second seat's engine then had to decide whether to honor it, and the disagreement
surfaced as a WARN in a conformance run rather than as a spec-review finding. **A worked example is
normative surface in every way that matters to an implementer.**

---

## §4 Spec deltas (executable form)

| # | File | Section | Edit |
|---|---|---|---|
| **D1** | `ENTITY-CORE-PROTOCOL.md` | §5.2, at `check_permission`'s existing comment block | Add, beside *"When resource is absent, resource dimension is unchecked at dispatch"*: **the condition is the presence of `execute.data.resource`, not the provenance of the dispatch.** An implementation MUST apply the resource dimension on every dispatch that carries a resource target, including handler-to-handler (in-process) sub-dispatch. An implementation MUST NOT construct a synthetic `execute` that omits a resource target the dispatch holds; where a child dispatch derives a resource (inherited from the parent context or supplied by an execute-option), **that** is the value the check receives. Cross-reference §6.2, whose `register`/`unregister` install-path authorization is assigned to this check and has no other. |
| **D2** | `EXTENSION-CONTINUATION.md` | §3.5, *Capability interaction* | **Fix the worked example**: `["peer_b/data/shared/*"]` → `["/peer_b/data/shared/*"]`, and append one sentence: a `dispatch_capability` is minted by one peer and wielded at another, so its resources are **granter-framed** per `ENTITY-CORE-PROTOCOL` §5.5a and MUST be written in explicit cross-peer form. A peer-relative pattern here names the *granter's* namespace and authorizes nothing at the dispatching peer. |
| **D3** | `ENTITY-CORE-PROTOCOL.md` | §5.5a, the *Operator-facing consequence* helper list | Add **continuation install helpers** to the enumerated helper classes (`test fixtures, integration test scaffolds, role-derived token helpers, multi-sig helpers`). It is the class that produced the live divergence and is the only one on the list reachable in production rather than in test scaffolding. |
| **D4** | `ENTITY-CORE-PROTOCOL.md` | §5.2 conformance surface | One vector: **`AUTHZ-SUBDISPATCH-RESOURCE-1`** — EXECUTE a handler holding a grant scoped to `app/*` which sub-dispatches `system/handler:register` with a caller-influenced resource target outside that scope; the sub-dispatch MUST fail **403 `capability_denied`**. This is the confused-deputy shape and **the only shape that reaches the path**, which is why the surface has no probe today. |

**Not in scope:** any change to §5.2's four-dimension model · any change to `check_resource_scope` ·
any relaxation of §5.5a · any change to what crosses a peer boundary · the question of *which*
handlers forward a caller-influenced resource (that is a per-impl audit, correctly scoped by core-py
as not-claimed).

---

## §5 Cohort impact

| Seat | Owed |
|---|---|
| **`entity-core-go`** | **D1** — pass the computed `childResource` into `CheckPermission` in `makeLocalExecute` (`core/protocol/local.go`). Note the ordering trap: `childResource` is currently computed *after* the check, so the fix is a move plus a pass, not just a pass. **D2** — fix `installJoinFromData` / `CreateDispatchCapability` in the validator to mint explicit cross-peer form; this is what clears core-py's `join_target_received` WARN, and **the WARN is correct until it lands.** |
| **`entity-core-py`** | **Nothing.** Conformant on both at `00b2c05` / `06162e5`. `TestAForeignGrantedCapFramesAgainstItsGranter` is adopted as the reference shape for D4's sibling case. |
| **`entity-core-rust`** | **Verify and report** whether the in-process path applies the resource dimension. Not a finding against them — an open question. If absent, D1. |
| **`entity-workbench-go` / `entity-browser-rust`** | Nothing directly. Relevant only if an SDK helper mints a cross-peer dispatch capability; D3's helper-class addition is the thing to check against. |
| **oracle / keystone** | D4's vector. **No re-pin** — adds a case, changes no encoding. |

---

## §6 What this proposal does NOT claim

- **Not** that SA-PY-9 is exploitable in any tree today. That needs the per-handler audit core-py
  explicitly did not do. `register` is a defect regardless of reachability because §6.2 assigns it an
  authorization that was not running.
- **Not** that `entity-core-rust` has either gap. Read, not proven; routed as a question.
- **Not** that core-go's fixture author was careless. §3 is the finding: the spec's example taught it.
- **Not** that D2 changes the capability model. §5.5a is unchanged; the example is being made
  consistent with it.
