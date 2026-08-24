# PROPOSAL — an empty `grants` array: one glyph, three objects, and only one of them is defined

**Status:** **FOLDED 2026-08-17** — landed as `0.8.1 CAP-1 / CAP-2 / CAP-3` in `entity-core-protocol` `30ca731`
(§6.1 dispatch step 1 · §6.2 writing-policy-entries · the `policy-entry` CDDL). Conformance is **not** yet
proven: `GUIDE-CONFORMANCE` §9 checks **(r)** and **(s)** are the validating half and are unbuilt.
**Target:** `entity-core-protocol` `specs/ENTITY-CORE-PROTOCOL.md` §6.1 · §6.8 · §6.2 · the
`system/capability/policy-entry` CDDL comment.
**Filed by:** `entity-core-go` (B, restated after their own retraction) and `entity-browser-rust`
(`ROUTING-2026-08-17-c` §1 / §3, A1 + A3, co-signed from `entity-workbench-go`'s N1). Verified here
against all three cohort implementations.
**Read at:** core-protocol `f83c256` (spec **Version: 0.8.0**) · core-go `cc537ea` · core-rust `f23fb3b` ·
core-py `40d1df2` · browser-rust `588eb9c`. All four impl reads are **source reads taken here**, not
peer-reported.

---

## §0 Summary

An empty `grants: []` array appears on **three different entities**, and the spec defines its meaning on
exactly one of them.

| # | object | where | spec says | cohort (measured here) |
|---|---|---|---|---|
| **D1** | **handler grant** — `system/capability/grants/{pattern}` | §6.1 entity-native dispatch step 1 vs §6.8 | **both, oppositely** — §6.1 *"MUST be present and non-empty"*, §6.8 *"Empty grants are valid"* | go + rust **dispatch**; **py rejects 403** |
| **D2** | **`policy-entry.grants`** — `system/capability/policy/{peer_pattern}` | §6.2 | **nothing** | rust + py **accept**; **go rejects 400** |
| **D3** | *(not a defect — the settled case)* `resources: {include: []}` inside a grant-entry | §3.6 / §4.4 | defined and correct | uniform |

**D1 is a contradiction inside one document about one object. D2 is silence. D3 is included only so
the fix does not disturb it.** They are filed together because the reason all three exist is one
drafting habit — writing "empty" about a *container* when the rule is about the *contents*, or vice
versa — and because a reader who resolves D1 by analogy will get D2 wrong.

**The cohort shape is the argument.** Both live defects split **2–1, and the dissenter is a different
implementation each time** — py alone on D1, go alone on D2. **No implementation is wrong twice, and
none is refutable from the text.** Each seat read one sentence of one document and implemented it
faithfully; the sentences disagree. That is what makes this a spec defect cluster rather than three
impl bugs, and it is why the fix belongs upstream before any seat is asked to move.

**Ruling asked for:** empty is **valid and meaningful** on both D1 and D2, with distinct semantics
stated at each site, and **entry removal is a third, non-equivalent operation** (§3).

---

## §1 D1 — §6.1 and §6.8 state opposite rules about the same grant

**§6.1, "Native vs entity-native execution", entity-native dispatch step 1:**

> **Verify handler grant exists.** The grant at `system/capability/grants/{pattern}` MUST be present
> **and non-empty**. Missing grant → `permission_denied`. Without this check, the expression would run
> with no capability ceiling.

**§6.8, "Handler Authority Model":**

> **Empty grants are valid.** A grant with an empty `grants` array (no scope entries) is a valid, signed
> grant for a pure-functional handler. The handler is registered, the grant is present and verified, but
> the handler has no impure authority. Any impure operation in the handler's expression or code fails its
> per-op capability check because nothing in the grant covers the target. **This is correct — pure-functional
> handlers need no impure authority.**

**Same path, same entity, opposite rules.** `entity-core-go` originally filed this and `:3533` as two
different objects, then re-read and retracted that framing themselves; the retraction is right and the
sharpened finding is the one that stands.

**The consequence is not hypothetical, and it lands on the case §6.8 was written for.** §6.1 step 1 is
the **entity-native** path — the branch taken when the handler manifest carries `expression_path`. A
compute-expression handler that is pure-functional is the single most natural instance of §6.8's
"pure-functional handler", and §6.1 step 1 refuses to dispatch it. **The two sections disagree hardest
precisely where they overlap.**

**Which text is wrong.** §6.1's own stated rationale — *"without this check, the expression would run with
no capability ceiling"* — **argues for presence, not for non-emptiness.** An empty grant *is* a ceiling:
it is the ceiling of nothing, and §6.8's next sentence spells out that every impure operation then fails
its per-op check. The ceiling model is satisfied by the presence-and-validity check; `non-empty` adds no
safety and deletes a legitimate handler class. **§6.1's `and non-empty` is the defect.**

### §1.1 It is already a live behavioral divergence, not a latent one

This was filed as a documentation contradiction. **It is not — the cohort has already split on it**, and
each side traced to a different sentence.

**`entity-core-rust` dispatches, and resolved the contradiction explicitly in source.**
`core/peer/src/connection.rs:2465` @ `f23fb3b` — the comment above the guard is a verbatim statement of
the resolution this proposal asks for:

```rust
// §S3: empty `grants` is valid — a pure-functional handler (no impure
// authority) is a registered handler. The expression runs; per-op
// capability checks (lookup/tree, apply, store) fail naturally because
// an empty scope covers nothing. Distinct from a missing/invalid grant,
// which is fail-closed here.
let grant = match ctx.handler_grant.as_ref() { Some(g) => g, None => { /* 403 */ } };
```

Presence is fail-closed; emptiness is not checked. **A seat hit the contradiction, chose §6.8, and wrote
down why.** *(Nit for rust: `dispatch_entity_native`'s own doc comment four functions away still says
"Caller already verified handler grant exists **and is non-empty** (§7.1)" — stale against the guard it
describes.)*

**`entity-core-go` also dispatches.** `loadValidatedGrant`, `core/protocol/execute.go:627` @ `cc537ea`
checks presence, then `VerifyHandlerGrant` (authority · signature · temporal — the §6.8 checks). **No
grant-count check anywhere on the path.**

**`entity-core-py` rejects.** `make_entity_native_handler`,
`packages/entity-handlers/src/entity_handlers/compute.py:3702` @ `40d1df2`:

```python
# V7 §7.1 fail-closed: handler grant MUST be present and non-empty.
if not ctx.handler_grant or not ctx.handler_grant.get("grants"):
    return _entity_native_error(403, ERR_PERMISSION_DENIED,
        f"Entity-native handler grant is empty at "
        f"system/capability/grants/{ctx.handler_pattern}")
```

**So a pure-functional entity-native handler — the exact case §6.8 blesses — dispatches on go and rust
and returns `403 permission_denied` on py.** That is a cross-peer observable divergence on a `MUST`,
reachable by any deployment that registers a compute-expression handler needing no impure authority.
**py is conformant to §6.1 and non-conformant to §6.8, and cannot be both.**

**This raises D1 from a drafting nit to the priority item in this proposal.** It also means the fix is
not free for py: deleting the guard is a behavior change that wants a vector.

### Proposed edit (D1)

In §6.1 entity-native dispatch step 1, replace:

> The grant at `system/capability/grants/{pattern}` MUST be present and non-empty.

with:

> The grant at `system/capability/grants/{pattern}` MUST be present and MUST pass the §6.8 handler-grant
> validation (presence · authority · signature · temporal validity). A grant with an empty `grants` array
> is valid per §6.8 — it is the ceiling of nothing, and every impure operation in the expression then
> fails its own per-op capability check. Missing or invalid grant → `permission_denied`.

§6.8 is unchanged; it is the correct text.

---

## §2 D2 — `policy-entry.grants` emptiness is undefined, and the cohort is 1–2

### §2.1 The measurement

**`entity-core-go` rejects** — `handleConfigure`, `ext/capability/handler.go:464` @ `cc537ea`:

```go
if len(pe.Grants) == 0 {
    return handler.NewErrorResponse(400, "invalid_params",
        "policy-entry MUST specify at least one grant entry")
}
```

**`entity-core-rust` accepts** — `handle_configure`, `extensions/capability/src/lib.rs` @ `f23fb3b`
validates `entity_type` and `peer_pattern`, then writes. No grant-count guard. Its two `grants.is_empty()`
guards sit on `handle_request` and `handle_delegate` — the two operations go also guards.

**`entity-core-py` accepts** — `_handle_configure`,
`packages/entity-handlers/src/entity_handlers/capability.py:706` @ `40d1df2`:

```python
grants = data.get("grants")
if not isinstance(grants, list):
    return _error(400, "invalid_request", "configure requires `grants` (array of grant entries)")
```

A type check, not a count check. **`[]` is a list.**

**This closes browser-rust's *"py remains unmeasured"*: the split is go alone against rust and py.**
The spec is silent — §6.2's writing-policy-entries paragraph states no minimum and the `policy-entry`
CDDL carries none — so **no implementation is refutable from the text.** That is the finding, and it is
why this is a spec defect rather than three impl bugs.

### §2.2 What an empty policy entry *means* — and it is not the same thing on both consultation paths

The policy table is consulted at two trigger points, and **the entry plays opposite roles at each.**
`entity-core-rust` states this in its own source comment and it is the sharpest framing anyone has put
on it:

> *"this entry is a CEILING here and a FLOOR on the connection path"*

- **At `system/capability:request` (§6.2 step 3) the entry is a CEILING.** The request's `grants` must be
  a subset of the matched entry's `grants`. With an empty entry, **every non-empty request fails subset
  and returns `403 scope_exceeds_authority`.** That is a hard, immediate, total refusal of new mints for
  that peer.
- **At authenticate-response (§4.4) the entry is a FLOOR term in a union.** *"The initial grant scope
  delivered to A is the **union** of the SHOULD floor above and the matched policy entry's grants."*
  An empty entry contributes nothing to the union, so **the peer still receives the §4.4 SHOULD floor.**

**Both readings are coherent, both are useful, and together they are exactly the withdrawal semantics the
application tier needs:** *stop minting anything new for this peer, without severing its ability to
connect and read type definitions.* **An empty policy entry is a withdrawal, not a ban.** It is not a
deny-list and it does not revoke the §4.4 floor — a distinction the app tier must state out loud, because
a UI that says "removed" when the peer can still handshake is lying by omission.

### §2.3 Why go's `400` is the defect rather than the safe default

Because of the **exact-match-then-`default`** resolution rule, an empty entry is **not** expressible any
other way. §6.2:

> An exact match on `{caller_peer_hex}` takes precedence; otherwise the `default` entry is the fallback;
> if no `default` entry exists, the caller has no policy entry.

An exact-match entry **suppresses `default` by existing.** So:

| operator intent | expressed as | result at `request` | result at §4.4 |
|---|---|---|---|
| "this peer gets nothing beyond the floor" | **empty exact entry** | every request 403 | SHOULD floor only |
| "this peer reverts to whatever everyone gets" | **remove the entry** | `default`'s grants are the ceiling | floor ∪ `default`'s grants |

**On go the first row is unreachable.** The operator's only remaining move is removal — which, on any
deployment whose `default` entry carries a public share, hands the withdrawn peer *the public scope* at
the moment of withdrawal. **The guard that looks conservative produces the more permissive outcome.**
That is the concrete harm, and it is the one `entity-browser-rust` is standing on: their withdrawal path
writes the empty entry, and on go it 400s while the grant survives, with a status code no UI reads.

### Proposed edit (D2)

Add to §6.2, in the **"Writing policy entries"** paragraph:

> **An empty `grants` array is valid and meaningful (normative).** `configure` MUST accept a
> `policy-entry` whose `grants` array is empty and MUST write it. An empty entry is the *withdrawal*
> form: because an exact-match entry suppresses the `default` fallback, an empty entry at
> `system/capability/policy/{peer}` means *"this peer matches, and is granted nothing."* Its effect
> differs by consultation path and both are intended — at `system/capability:request` the entry is the
> per-peer **ceiling**, so an empty entry causes every non-empty request to fail subset-validation with
> `403 scope_exceeds_authority`; at §4.4 authenticate-response the entry is a **union term**, so an empty
> entry contributes nothing and the peer still receives the §4.4 SHOULD floor. **An empty entry is a
> withdrawal of policy grants, not a ban** — it does not revoke the SHOULD floor and is not a deny-list.
> Implementations MUST NOT reject `grants: []` at `configure`.

And to the `system/capability/policy-entry` CDDL comment, on the `grants` field:

```
    grants:       {array_of: {type_ref: "system/capability/grant-entry"}}
                                                          ; MAY be empty. An empty array is the
                                                          ; withdrawal form — the entry matches and
                                                          ; grants nothing, suppressing the `default`
                                                          ; fallback. See §6.2 writing-policy-entries.
```

---

## §3 D3 / A3 — entry removal is a third operation, and it is not equivalent to either

`entity-browser-rust` raised this as A3 and it is correct: `tree:remove` on
`/{me}/system/capability/policy/{key}` reaches the *"no policy entry"* state on all three
implementations today and sidesteps D2 entirely. **It must not be presented as the portable spelling of
the same thing, because it is a different thing** — see the table in §2.3. Removal drops the peer through
to `default`; the empty write does not.

The two operations are already distinguishable from §6.2's resolution rule, but **only by a reader who
holds the resolution rule and the withdrawal intent in mind at once** — which is exactly the reader the
spec should not require. One sentence closes it.

### Proposed edit (D3)

Add to §6.2, immediately after the D2 paragraph:

> **Removal is a distinct operation.** Deleting the entity at `system/capability/policy/{peer}` is not
> equivalent to writing it with empty `grants`. Removal restores fallback: the peer resolves to the
> `default` entry (or to no policy entry if none exists). The empty write suppresses fallback. An
> operator withdrawing a specific grant while leaving the peer its baseline uses **removal**; an operator
> withdrawing everything policy-granted, *including* what `default` would give, uses the **empty write**.

---

## §4 What this proposal does NOT claim

- **Not** that any of the three implementations is buggy on D2 *as of today's text*. The spec is silent;
  that is the defect. Go becomes non-conformant only once this lands, and the fix is a five-line deletion.
- **Not** that empty `resources: {include: []}` inside a grant-entry is affected. §3.6 and §4.4 define it
  ("*the capability handler … processes `request`, `delegate`, and `revoke` without reading or writing
  tree data*"), the cohort is uniform, and this proposal deliberately leaves it untouched. It is listed
  in §0 only so the D1 edit is not read as sweeping it in.
- **Not** that empty grants on `request` or `delegate` are valid. All three impls reject there and should
  — a request for nothing is a malformed ask, not a withdrawal. **This proposal does not disturb those
  guards**, and the asymmetry is deliberate: `configure` writes *policy*, `request`/`delegate` mint
  *tokens*.
- **Not** a claim about how withdrawal *revokes tokens already minted*. It does not — see
  `PROPOSAL-CAPABILITY-MINT-TEMPORAL-CEILING`, which is the other half of browser-rust's packet and the
  half that carries the product consequence.

## §5 Blast radius

| | |
|---|---|
| **Wire** | **None.** Policy entries are local storage; §6.2: *"each peer has its own policy table at this path"* |
| **Frozen §5.2 vector set** | **Untouched.** This is §6.1/§6.2/§6.8 handler-and-policy surface, not `check_permission` |
| **`entity-core-go`** | **D2: one deletion** — the `len(pe.Grants) == 0` guard at `ext/capability/handler.go:464`. **D1: already conformant** (`loadValidatedGrant` has no count check) |
| **`entity-core-rust`** | **Both already conformant.** One stale doc comment to correct (`dispatch_entity_native`, §1.1) |
| **`entity-core-py`** | **D2: already conformant. D1: one deletion** — the second guard in `make_entity_native_handler` (`compute.py:3702`). The `handler_grant_hash is None` presence check above it **stays** — that half is correct and is what §6.8 requires |
| **`entity-browser-rust`** | `share::policy_writes_for`'s doc comment becomes true rather than rust-specific. **Their `AGENTS.md` marks the withdrawal path BLOCKED — this unblocks the structural half only**; the temporal half is the sibling proposal |
| **`entity-core-go` — as oracle author** | Two `validate-peer` checks, **neither of which exists today**: `configure` with `grants: []` (accept + write + suppresses `default`) in the `capability` category, and dispatch of a pure-functional entity-native handler whose grant is empty (**expect 200, not 403**) in the `entity_native` category. **The second is the one that would have caught D1** |
| **`entity-core-keystone`** | **Authors nothing.** Re-scores the peer matrix when the oracle is re-pinned |

## §6 Ask

1. **Route to `entity-core-protocol` as a core-spec revision.** **D1 first and separately** — it is a
   same-document contradiction with a live 1–2 cohort split, and it should not wait on D2/D3.
2. **No cohort read is owed on this proposal.** All three trees were read here for both defects, at the
   commits pinned in the header, by opening the source. The per-seat work in §5 is the complete list as
   measured at those commits — **re-verify before acting if HEAD has moved**, per the standing
   build-state rule.
3. **Two conformance vectors** per §5. The D1 vector is the load-bearing one: a cohort that had it would
   have caught this before the spec did.
