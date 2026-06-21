# Roles: User Guide

**Status**: Active
**Audience:** Application developers and operators provisioning role-based authority on Entity Core peers. Assumes familiarity with capabilities (GUIDE-CAPABILITIES.md). Familiarity with identity (GUIDE-IDENTITY.md) is helpful for the identity-coherence sections but not required.
**Spec reference:** `EXTENSION-ROLE.md` (current draft), `ENTITY-CORE-PROTOCOL.md` §5 (capability system), §5.6 (the `is_attenuated` helper).
**Related guides:** GUIDE-CAPABILITIES.md, GUIDE-IDENTITY.md, GUIDE-MULTISIG.md, GUIDE-GROUP.md.

---

## 1. What roles are (and what they aren't)

The core protocol gives you per-peer authorization: each peer is a root authority, each peer signs caps, every cap chain bottoms out at a peer-identity entity. That's enough to grant Bob `get` access to one specific path. It is *not* enough to express "Bob is a manager on team-alpha," "Carol is a moderator across all forums," or "every member of this team gets the same five permissions." You can hand-issue identical cap bundles to every member and then revoke them by hand when someone leaves — but that's a management problem the protocol shouldn't make you solve manually.

The role extension fills that gap. It defines named permission *templates* (role definitions), a way to bind a peer to a role within a *context* (assignment), and a way to deny a peer all role-derived access within a context (exclusion). The capability system stays the enforcement mechanism; roles are the management layer on top.

**Roles are templates, not authority.** A role definition by itself authorizes nothing. Assignment is what derives the actual cap tokens, and the assigner's authority is what those tokens ultimately chain to. If the assigner can't already do everything the role would mint, the assignment is rejected (the assigner-coverage check, §8).

**A role lives in a context.** Contexts scope roles independently — `admin` in `group/team-alpha` is different from `admin` in `service/media-server`. Cross-context interaction is convention, not protocol.

**Two parallel mechanisms (just like identity).** Caps are the chain-validated permission objects; role assignments and exclusions are entities that the role handler validates at the entity level. They don't mix — the role handler emits caps but never *checks* a cap with role logic.

What the role extension does:
- Defines named grant bundles (role definitions at `system/role/{context}/{role_name}`).
- Binds peers to roles within contexts (assignments at `system/role/{context}/assignment/{peer_id}/{role_name}`).
- Denies peers within contexts (exclusions at `system/role/{context}/excluded/{peer_id}`).
- Mints derived caps when a peer is assigned (and revokes them on unassign / exclude / re-derive).
- Re-issues caps when a definition changes (`re-derive`).
- Allows a member to delegate their role-authority to another peer (`delegate`).

What the role extension does NOT do:
- Modify the core protocol's chain verification.
- Change capability scope semantics.
- Enforce "an excluded peer can do nothing" universally — exclusion is a three-layer model (§7), not a magic denial.
- Coordinate roles across contexts. A peer's roles in different contexts are independent; cross-context queries are deferred to a future query extension.

If you're using the core protocol + identity and want "Bob can read Alice's photos," you don't need the role extension — issue a single cap. You need this extension when you have **multiple peers who should hold the same set of grants** OR when you want **structural eviction within a scope** (denial that revokes every role-derived cap and blocks future derivations; per the three-layer model in §7, this catches every role-mediated path but does not touch caps the peer holds via other routes unless those handlers opt in).

---

## 2. The model in one diagram

```
           ┌──────────────────────────────────────────────────────────┐
           │ Role definition (template)                                │
           │   system/role/{context}/{role_name}                       │
           │   { name, grants: [grant_entry, ...], metadata }          │
           │                                                           │
           │   - Tree data; not load-bearing on its own.               │
           │   - May contain template variables {context}, {peer_id}. │
           │   - Mutated via system/role:define (default).             │
           └──────────────────────┬────────────────────────────────────┘
                                  │
                                  │ assignment references the role by name
                                  ▼
           ┌──────────────────────────────────────────────────────────┐
           │ Role assignment                                           │
           │   system/role/{context}/assignment/{peer_id}/{role_name}  │
           │   { role, assigned_by, assigned_at }                      │
           │                                                           │
           │   - Created via system/role:assign.                       │
           │   - Triggers cap derivation at assign time.               │
           │   - Multi-role per (peer, context) — multiple entries     │
           │     under {peer_id}/ for the same peer.                   │
           └──────────────────────┬────────────────────────────────────┘
                                  │
                                  │ derived caps land at:
                                  ▼
           ┌──────────────────────────────────────────────────────────┐
           │ Role-derived caps                                         │
           │   system/capability/grants/role-derived/                  │
           │     {context}/{peer_id}/{token_hash}                      │
           │                                                           │
           │   - Standard caps. Granter = the runtime peer that signed │
           │     them. Parent = the role handler's own grant.          │
           │   - Pinned storage path so revocation can find them.      │
           └───────────────────────────────────────────────────────────┘

           ┌──────────────────────────────────────────────────────────┐
           │ Exclusion (orthogonal to assignment)                      │
           │   system/role/{context}/excluded/{peer_id}                │
           │   { peer_id, excluded_by, excluded_at, reason }           │
           │                                                           │
           │   - Created via system/role:exclude.                      │
           │   - Three-layer enforcement (§7).                         │
           └───────────────────────────────────────────────────────────┘
```

Three things to internalize:
1. **A role definition produces no authority.** Authority comes from assignment plus the assigner's caller cap.
2. **The derived cap is signed by the assigner's runtime peer.** The core protocol's rule that root capabilities must be granted by the local peer is unchanged — the role extension does not invent a new chain root.
3. **Exclusion is a parallel control surface.** It deletes derived caps (layer 1), blocks future derivations (layer 2), and optionally surfaces at handler-level checks (layer 3). It does NOT touch caps that weren't role-derived.

---

## 3. Defining a role

A role definition is a named grant bundle. The minimum ceremony is a single call to `system/role:define` carrying the proposed grant set.

### 3.1 The shape of a role

```
system/role/{context}/{role_name} := {
  name:     "manager",
  grants:   [
    { handlers:   {include: ["system/tree"]},
      resources:  {include: ["shared/{context}/*"]},
      operations: {include: ["get", "put", "list"]} },
    { handlers:   {include: ["system/group"]},
      resources:  {include: ["system/group/{context}"]},
      operations: {include: ["status"]} }
  ],
  metadata: { ttl: 86400000 }   ; optional; ms — handler default lifetime for derived tokens
}
```

`grants` is an array of grant entries — exactly the same five-dimensional shape used in capability tokens (handlers, operations, resources, peers, constraints). The role bundles them under a single name.

### 3.2 Template variables

Two template variables are recognized in path values within `grants`:

| Variable | Substituted with at issuance | Example |
|---|---|---|
| `{context}` | The role's context path (e.g., `group/team-alpha`) | `shared/{context}/*` → `shared/group/team-alpha/*` |
| `{peer_id}` | The assignee's peer-identity hash (canonical Base58) | `users/{peer_id}/*` → `users/Bob_Base58/*` |

Templates resolve at *issuance* time (when `assign` runs), not at definition time. The literal string lives in the role entity; the assigner's runtime peer applies the substitution.

### 3.3 Reserved names

The role-name segments `assignment` and `excluded` are reserved — they collide with the dedicated subtree paths used by the role handler. Any other name is permitted; conventional choices are `member`, `admin`, `manager`, `engineer`, `auditor`, `friend`, `follower`, `reader`. Use whatever makes the deployment legible.

### 3.4 Who can define a role

`system/role:define` is the default path for creating or mutating a role definition. The handler runs the assigner-coverage check at definition-write time — the caller's capability MUST cover the proposed grant set — and triggers `re-derive` cascade on mutation (§5).

In practice, role-definition write authority is **admin-tier**: a peer that can mutate `system/role/{context}/{role_name}` can silently change downstream grants for every assignee. Bind the `define` op to the deployment's admin set, not to the general assigner population.

Implementations MAY alternatively keep raw `tree:put` legal for role definitions and trigger `re-derive` via a tree-mutation observer when the deployment already gates the underlying tree path tightly. Both paths converge on the same property: only authorized actors mutate definitions, and mutation always cascades through `re-derive`.

---

## 4. Assigning a role

Assignment is what turns a template into actual caps. The mechanics are simple; the *authority* model is what makes it interesting.

### 4.1 The typical flow (identity-aware deployment)

In a deployment running the identity extension, role assignments are typically authorized by Alice's operator key (the day-to-day key the identity extension calls "Op") via the local peer→Op cap installed by `system/identity:configure`. Concretely:

```
alice_runtime_peer.execute(
  "system/role", "assign",
  resource: "system/role/group/team-alpha/assignment/{Bob_id}/manager",
  params: { role: "manager" },
  capability: <local peer→Op cap on alice_runtime_peer>
)
```

What happens inside the role handler:

1. Reads `ctx.resource.targets[0]` and parses out `(context: "group/team-alpha", assignee: Bob_id)`.
2. Reads the role definition at `system/role/group/team-alpha/manager`. (404 `role_not_found` if absent.)
3. Resolves templates: `{context}` → `group/team-alpha`, `{peer_id}` → `Bob_id`.
4. **Exclusion check.** If `system/role/group/team-alpha/excluded/{Bob_id}` exists, returns 403 `assignee_excluded`. This fires *before* the authority check — excluded peers don't even get to the authority check.
5. **Assigner-coverage check (the key invariant).** The handler asks: "are the role's grants a (non-strict) attenuation of the caller's authority?" using the core protocol's `is_attenuated` helper. If the caller's authority does NOT cover everything the role would mint, the handler returns 403 `assigner_authority_insufficient`. **Fail-closed — partial tokens are NOT minted.** Either the assigner has full authority or the assignment doesn't happen.
6. The assignment entity is written at `system/role/group/team-alpha/assignment/{Bob_id}/manager`.
7. Derived caps are minted, signed by `alice_runtime_peer`, and persisted at `system/capability/grants/role-derived/group/team-alpha/{Bob_id}/{token_hash}`.
8. The caps are delivered to Bob (out-of-band; the role extension does not specify the delivery mechanism — see §5.3 for the parallel re-derive case where delivery IS pinned).

### 4.2 Who's the granter on the derived cap

This is where most confusion happens. Be explicit:

- **The assignment entity's `assigned_by`** is the EXECUTE author — typically Alice's operator key (in identity-aware deployments) or whoever drove the EXECUTE.
- **The derived cap's `granter`** is the runtime peer that ran the assign — `alice_runtime_peer` in the example above (specifically, the content_hash of that peer's `system/peer` entity).
- **The derived cap's `parent`** is `null`. Per v2.0 PR-1, runtime role-derived caps are **root caps** — structurally identical to the startup-time L0 derivation path. There is no `parent` to walk. Authority at issue time comes from RL2 (caller's cap covers the role's grants); validity at use time comes from the cap's own signature + tree binding. The previous "parent = role handler's grant" model conflated issue-time authority with use-time chain validation; v2.0 separates them cleanly.
- **The derived cap's `grantee`** is the assignee — `Bob_id` (specifically, the content_hash of Bob's `system/peer` entity).

The operator key never appears in the cap chain. The operator authorizes the `assign` (via the local peer→Op cap; that cap is the *caller capability* the assigner-coverage check looks at); the runtime peer signs the derived cap. Since the cap is a root cap, there is no chain link above it.

Read the chain bottom-up: Bob's derived cap → terminates (root). The operator key is nowhere in the chain. Side benefit: §10.2 op-key rotation invariance becomes structural — there is no link in the cap chain that could possibly reference the operator key.

### 4.3 Core-protocol-only deployments

In deployments that don't install the identity extension (one peer = one keypair = one identity), role assignments are caused by the peer-owner directly — either via the bootstrap path (§9) or via a dispatched EXECUTE under whatever cap the peer-owner has issued to the assigning identity. The role extension's mechanics don't change. The assigner-coverage check still runs against `ctx.caller_capability`.

### 4.4 What scope the assigner needs

The check says: "the assigner's authority MUST cover everything the role would mint." Practically this means:

- If the role's grants include `handlers: ["system/tree"], operations: ["put"], resources: ["shared/{context}/*"]`, the assigner's cap must cover that scope.
- "Cover" here is the core protocol's existing `is_attenuated` arithmetic — the role's grants must be a (non-strict) attenuation of the caller's. Equal scope passes; strict subset passes; broader scope fails.
- The assigner does NOT need to be able to issue caps to *Bob specifically* — the `peers` dimension on the assigner's cap is checked normally; if the assigner's cap is `peers: {include: ["*"]}`, any assignee works.

Common configuration: the operator holds a near-wildcard cap (the operator is the deployment's admin); roles are narrower; the check passes by construction.

### 4.5 Multi-role per (peer, context)

A peer MAY hold multiple role assignments under the same context. The path is `assignment/{peer_id}/{role_name}` — the trailing `{role_name}` segment makes each (peer, role) pair its own assignment entity. Bob can be both `manager` and `auditor` in `group/team-alpha`; the two assignments coexist; tokens accumulate.

**Token presentation.** When Bob makes an EXECUTE that the verifier checks, Bob presents *one* token covering the requested operation. The verifier validates that one token; there is no protocol-level token-union step. If a token is later revoked (via `re-derive` or `unassign`), Bob's other live tokens covering the operation continue to authorize.

---

## 5. Mutating a role definition — the re-derive lifecycle

A deployment evolves. The `manager` role gets a new permission. The `engineer` role drops one. Re-issuance to existing assignees is what `re-derive` does.

### 5.1 The trigger

`system/role:define` (the default write path) automatically triggers `re-derive` cascade on definition mutation. Implementations using the tree-mutation-observer alternative fire the cascade on observation. Either way, mutation MUST cascade.

The cascade is also explicitly invocable: `system/role:re-derive` against the role-definition path re-issues tokens for all current assignees.

### 5.2 The lifecycle

For each assignee, the handler does **issue T_new before revoking T_old** (default). The ordering is load-bearing:

| Step outcome | Result | Safety |
|---|---|---|
| Issue T_new persists, revoke T_old fails | Assignee briefly holds both tokens | **Safe** — over-authorization bounded by grace window |
| Issue T_new fails (no revoke yet) | Assignee keeps T_old | **Safe** — no stranding |
| Revoke-first (rejected default): revoke T_old persists, issue T_new fails | Assignee stranded with no token | **Unsafe** — rejected as default |

Implementations MAY offer revoke-first as a deployment-configurable mode for security-critical contexts where stranding is preferable to overlap; the default MUST be issue-first. No tree-level transactional-bundle primitive is required — per-entity content-addressing makes each leg recoverable on retry.

### 5.3 Grace window and delivery

- **Grace window (SHOULD).** A configurable overlap window during which both T_old and T_new are valid. Default: 0 (just the time between two consecutive per-entity writes). Tight-security deployments keep grace=0; availability-prioritized deployments extend.
- **Delivery (MUST).** T_new MUST reach the assignee via either inbox push or pull-via-subscription on `system/role/{context}/assignment/{peer_id}/...`. The mechanism MUST be named in the implementation; "the assignee will figure it out" is not conformant.
- **In-flight EXECUTE semantics.** EXECUTEs in flight using T_old during re-derive return **401 `capability_revoked`** (fail-fast). Clients retry with T_new. Standard retry pattern.

### 5.4 Multi-Op concurrency

In identity-aware deployments where the identity has multiple operator keys live concurrently (the multi-Op pattern from the identity extension), each operator may invoke `re-derive` independently. Each runs its own ordered-write flow. Tree CAS resolves at the assignment-entity level. Resulting tokens are independent — both may be issued; the assignee receives whichever delivers first; both pass the assigner-coverage check if either operator's authority covers the role definition. **Implementations MUST NOT serialize concurrent re-derives across operators** — that would block multi-Op operational concurrency.

---

## 6. Member-to-member delegation

A member who holds a role wants to delegate their role-authority to another peer — vacation coverage, role handoff, ad-hoc temporary authorization. The `delegate` op composes this through role's lifecycle rather than punching out to raw caps.

### 6.1 The flow

```
bob_runtime_peer.execute(
  "system/role", "delegate",
  resource: "system/capability/grants/role-derived/group/team-alpha/{Carol_id}/{token_hash}",
  params: {
    delegator: Bob_id,
    delegate:  Carol_id,
    context:   "group/team-alpha",
    role:      "manager",
    scope:     [<grant-entry subset>],     ; MAY attenuate from manager's grants
    expires_at: 1741800000000              ; optional
  },
  capability: <Bob's role-derived manager cap on team-alpha>
)
```

What happens:

1. The handler verifies Bob holds the named role (`system/role/group/team-alpha/assignment/{Bob_id}/manager` exists).
2. **The assigner-coverage check applies to Bob's authority, not the operator's.** `scope` MUST be an attenuation of the role's grants as held by Bob. A member can only delegate authorities they hold; they MAY narrow them.
3. Templates resolve (`{peer_id}` → `Carol_id`).
4. The delegation cap is issued: **granter = Bob's runtime peer**, **grantee = Carol**, **parent = Bob's role-derived cap**. Signed by Bob's runtime peer.
5. Stored at `system/capability/grants/role-derived/group/team-alpha/{Carol_id}/{token_hash}` — the role-derived path so the role extension's machinery (the layer-1 sweep, `unassign` revocation, `re-derive`) reaches it.

### 6.2 What revocation does to delegation caps

Delegation caps chain from the delegator's role-derived cap. The interactions:

- **Carol excluded in `group/team-alpha`.** The layer-1 sweep deletes Carol's delegation cap from `role-derived/{context}/{Carol_id}/...`. Layer 2 prevents Bob from re-delegating (the `delegate` op consults the exclusion subtree).
- **Bob excluded in `group/team-alpha`.** Bob's own role-derived cap is revoked (layer 1). Carol's delegation cap chains from Bob's now-missing parent — the standard `is_revoked` cascade rejects it on next use.
- **`unassign(Bob, manager, group/team-alpha)`.** Same cascade as exclusion-of-Bob.
- **`re-derive` of `manager`.** Bob's manager cap is re-issued; his outstanding delegation caps to Carol chain from the now-revoked old parent and become unverifiable. Bob MUST re-issue delegations after re-derive if continued coverage is required.
- **`expires_at`.** The chain verifier rejects the delegation cap on next use after expiry. No handler-side cleanup.

### 6.3 Why this composes through role rather than raw caps

Doing it via raw caps would bypass role's lifecycle. With `delegate`, exclusion / re-derive / unassign all flow naturally — the delegation cap is in the role-derived storage path, so the layer-1 sweep finds it, the cascade reaches it, the deployment's audit trail records it.

---

## 7. Exclusion — the three-layer model

Exclusion is the strongest control surface in the role extension. It is also the most over-claimed: earlier prose said "exclusion denies access universally," but that's only true where handlers opt in to checking. The honest model is three-layer.

### 7.1 The three layers

**Layer 1 (primary): Token revocation.** When `system/role:exclude` runs, the handler deletes the excluded peer's role-derived tokens from `system/capability/grants/role-derived/{context}/{peer_id}/...`. The standard `is_revoked` check sees the missing tokens and rejects them on next use. This is the foundational mechanism — no new role-extension machinery; just the protocol's existing primitives.

**Layer 2 (secondary): Block new derivation.** Before any path mints a new token for a peer, the exclusion subtree is consulted. If `system/role/{context}/excluded/{peer_id}` exists, derivation is rejected. This applies to:
- Runtime `system/role:assign`.
- The bootstrap derivation path (§9).
- `re-derive` (excluded peers don't get re-derived tokens even if the role-definition changes).

The invariant: **no derivation path mints a token for an excluded peer in that context, regardless of which entry point is used.**

**Layer 3 (tertiary, opt-in): Handler-level checks.** Handlers operating in a context-scoped namespace MAY check exclusion before processing requests:

```
handle_op(ctx, params):
  if is_excluded(context, ctx.execute.data.author, ctx):
    return error(403, "peer_excluded_in_context")
  ...
```

Group handlers SHOULD check; service-role handlers MAY; the generic `system/tree` handler does NOT. Layer 3 is the strongest semantics — "excluded peer cannot do anything in this context, even with a non-role cap" — but only where handlers opt in.

### 7.2 Fleet-wide reactive sweep

When Alice excludes Bob from `group/team-alpha` on `alice_desktop`, the exclusion entity syncs to `alice_phone`, `alice_server`, etc. **Each runtime peer that receives the exclusion entity MUST sweep its own `system/capability/grants/role-derived/{context}/{peer_id}/...` subtree and delete any role-derived tokens it issued to the excluded peer.**

Without this reactive bridge, `alice_phone` would keep issuing/holding role-derived tokens for Bob even after `alice_desktop` excluded him, defeating fleet-wide exclusion. The sweep is local to each runtime peer's own subtree (peers don't sweep each other's subtrees); the synced exclusion entity is the trigger.

This is what makes layer 1 actually fleet-wide rather than just "the issuing peer's tokens are revoked."

### 7.3 Exclusion vs. unassign

`unassign` removes a specific (peer, role) assignment; `exclude` adds a context-level denial regardless of assignment.

| | `unassign` | `exclude` |
|---|---|---|
| What's removed | One assignment entity at `assignment/{peer}/{role}` | Adds an entity at `excluded/{peer}` |
| What's revoked | Tokens derived from that specific assignment | All role-derived tokens for the peer in the context |
| What's blocked | Future tokens from that role | All future role-derived tokens for the peer |
| Reversible | Yes — re-assign | Yes — `unexclude` (but tokens were deleted; re-assignment needed for fresh tokens) |
| Use case | "Bob is no longer the manager (but is still an engineer)" | "Bob has been evicted; deny everything within this context" |

Both compose with the standard `is_revoked` check for downstream rejection. Both are parallel revocation flows with different triggers.

### 7.4 What exclusion does NOT touch

A peer with **non-role-derived** tokens AND an exclusion entry: layer 1 doesn't touch those tokens (they're outside the role's storage path). Layer 3 catches them at handler-level if the handler opts in. Without layer 3, the tokens remain usable at handlers that don't check exclusion.

If you want exclusion to bite for a specific handler, that handler MUST check `is_excluded(context, author, ctx)` at request time.

### 7.5 Removing an exclusion does not auto-restore tokens

Layer 1 deleted the role-derived tokens. `unexclude` removes the exclusion entity but does NOT re-mint tokens. Re-assignment (or `re-derive`) is required to issue fresh tokens.

---

## 8. The assigner-coverage check: the security invariant

The role extension's central guarantee is: **an assigner cannot mint tokens broader than their own authority.** Without it, anyone with `tree:put` access to `system/role/{context}/assignment/*` could create assignments that derive arbitrary caps. With it, the assigner's caller capability is the upper bound on what the assignment mints.

(The check is named "RL2" in the spec. We refer to it descriptively in this guide.)

### 8.1 The check

```
hypothetical_grants = derived_grants    ; grants the role would mint
if not is_attenuated(hypothetical_grants, ctx.caller_capability):
  return error(403, "assigner_authority_insufficient", ...)
```

It uses the core protocol's existing `is_attenuated` helper. The check is "the role's grants are a (non-strict) attenuation of the caller's authority." Equal passes; strict subset passes; broader fails.

### 8.2 Fail-closed

If the assigner's cap doesn't cover *any* grant entry in the role, the entire `assign` MUST fail with **403 `assigner_authority_insufficient`**. (An earlier draft suggested "omit the uncovered entry from the derived token" — that was a security regression letting low-authority callers receive partial tokens they didn't request. The current rule is fail-closed: surface the authority gap rather than silently mint a narrower token.)

### 8.3 Why this is stricter than chain-root

Other coherent-capability checks (continuation install, subscription subscribe, compute install) use chain-root: "the writer's identity must appear somewhere in the embedded cap's authority chain." That works for embedded caps because there *is* a single chain.

Role-derived grants are synthesized from a template; there's no single chain to walk. The right invariant is "the assigner can already do everything the assignment will let the assignee do" — coverage. The check uses scope arithmetic, not chain-walking. It's stricter (covers all five grant dimensions structurally) and tighter (no chain reachability concerns).

### 8.4 Determinism

`resolve_templates` and `derive_grants` MUST be deterministic across implementations. Non-deterministic derivation could pass the coverage check on one implementation and fail on another for the same inputs — breaking cross-implementation convergence. Cross-implementation test vectors for derivation determinism are recommended.

---

## 9. Bootstrap and anonymous fallback

### 9.1 Bootstrap exemption

The first role assignments on a fresh peer happen *before* the role handler is registered in the dispatch table. They are written via the SDK's L0 direct-store API under the peer-owner's authority. **Bootstrap derivation MUST be callable only via L0; runtime code paths MUST NOT invoke L0 derivation.** Leaving an L0 entry point reachable post-bootstrap is a soft privilege escalation surface — the assigner-coverage check is bypassed by construction on the L0 path.

### 9.2 Both bootstrap-derived and runtime-derived tokens are root caps

Per v2.0 PR-1, all role-derived capability tokens — bootstrap AND runtime — are root caps:

- **`parent: null`** (no chain to walk)
- **`granter: local_peer.content_hash`** (the runtime peer that signed)
- **`grantee: assignee.content_hash`** (per V7 §3.6)

The two paths differ in **how authority is established at issue time**, not in the cap shape:

- **Bootstrap-derived (startup-time L0 path).** No caller capability exists yet — peer-owner has authority by being the peer-owner; no RL2 check.
- **Runtime-derived (dispatched `:assign`).** Caller's capability runs through RL2 (the assigner-coverage check); RL2 must pass before the cap mints.

Both paths produce structurally identical caps. There is no "parent points to handler's grant" link for runtime caps — that was the v1.x model, which conflated issue-time authority (caller cap) with use-time chain validation (parent cap) and broke on production-shaped peers (peers with attenuated self-grants — the natural production shape). v2.0 separates the two cleanly: issue-time authority lives in RL2; use-time validity lives in the cap signature + tree binding.

Implementations MUST distinguish bootstrap from runtime derivation **at the entry-point level** (the bootstrap path runs only via L0; runtime via dispatched EXECUTE). The bootstrap path closes after handler registration — implementations MUST ensure the L0 derivation entry point is not reachable from any dispatched EXECUTE path or any code that runs after handler registration in the normal startup sequence.

### 9.3 Layer-2 exclusion applies to bootstrap, too

The `is_excluded(context, peer_id)` helper MUST fire on the L0 bootstrap path before issuing root caps, not only on the dispatched `assign` path. Implementations expose a single helper that both the handler and the L0 bootstrap path call. This prevents excluded peers from receiving bootstrap-derived caps via the only path that doesn't run through the role handler.

The helper is a pure tree:get — it returns a defined answer regardless of bootstrap order. Identity bootstrap and role bootstrap may run in any order; the exclusion check works correctly either way.

### 9.4 Initial-grant policy (anonymous fallback)

When an unknown peer initiates contact, the deployment chooses what role (if any) they receive. Three modes:

- **anonymous-allow.** Unknown peer receives a configured default role (e.g., `anonymous-reader` for a public blog, `proposer` for an open-source project). Standard bootstrap derivation.
- **anonymous-deny** (default). Unknown peers are rejected. No bootstrap derivation; an explicit role assignment must be in place before any cap is issued.
- **recognize-on-attestation.** Unknown keypair, known identity. Useful for identity-aware (Stage-2) deployments: the peer presents an agent cert signed by an identity the local peer recognizes (or has TOFU-accepted), and the policy derives a role based on what's known about that identity. Falls back to allow or deny if no recognized attestation. This mode is unavailable in Stage-1 deployments — no identity attestations exist without the identity extension.

Configuration is stored at the normative path `system/role/initial-grant-policy` (renamed from `system/role/bootstrap-policy` in v1.6 alongside the broader "bootstrap" → "startup" / "initial-grant" terminology cleanup). Suggested shape:

```
system/role/initial-grant-policy: {
  unknown_peer:       "anonymous-deny",   ; or "anonymous-allow", "recognize-on-attestation"
  default_role:       "anonymous-reader",
  identity_required:  true                ; for recognize-on-attestation mode
}
```

**Layer 2 (block-new-derivation) fires BEFORE the initial-grant policy.** An excluded peer cannot bypass via anonymous-allow.

Defaults: `unknown_peer: "anonymous-deny"`, `default_role: null`. Privacy-preserving and conservative; deployments opt-in to anonymous-allow when they have a public-facing role to grant.

---

## 10. Composing with identity

Identity tells you *who* a peer is. Roles tell you *what* a peer is allowed to do. The composition is at the seam where the operator key makes role assignments via the local peer→Op cap.

### 10.1 The local peer→Op cap is the caller capability

When `system/identity:configure` runs (the bootstrap exemption — see GUIDE-IDENTITY.md §3.2), it issues exactly one cap per runtime peer: a cap from the runtime peer to the operator key. The operator uses this cap as `ctx.caller_capability` when invoking `system/role:assign`. The assigner-coverage check validates that the operator's authority covers the role's grants.

### 10.2 What changes when the operator key rotates

Operator-key rotation is invisible at the cap layer. The local peer→Op cap is rooted at the runtime peer, not at the operator's identity. When the quorum signs a new operator-delegation, the runtime peer revokes the old local cap and issues a new one — but **existing role-derived caps are unaffected** (they were signed by the runtime peer, not by the operator key).

Future role assignments are made by the new operator key via the new local cap. The old operator key can no longer drive `assign` at this runtime peer. Existing assignments and derived caps continue to work.

This is the structural reason the operator key stays out of cap chains: rotating it is local management, not a global cap reissuance.

### 10.3 What changes when the runtime peer is retired

When the runtime peer is retired (its local peer→Op cap revoked), every cap chain rooted at that runtime peer dies on the next `is_revoked` walk:
- Role-derived caps the runtime peer issued → die.
- Delegation caps chained from those → die.
- Subscriptions and continuations rooted there → die.

The rotating peer SHOULD invoke `rotation_reissue_outstanding_grants` before retiring. This is the SDK-side helper that walks `system/capability/grants/*` under the rotating peer, re-issues each cap under a new authority, and emits each new cap for delivery via the consuming extension's normal flow.

### 10.4 What changes when Public_alice rotates

Nothing at the role layer. Public_alice rotation is processed by contacts via the attestation channel; cap chains are rooted at runtime peers, not at Public_alice. Role assignments persist; derived caps persist.

### 10.5 Identity-coherence cross-references

| Cross-reference | What it says |
|---|---|
| **Typical caller** | Assignments are typically authorized by the operator key via the local peer→Op cap. Role mechanics are unchanged regardless of caller — the operator is just the typical caller in identity-extension deployments. |
| **Assignees can be any peer ID** | An assignee may be an identity-extension-managed peer or a single-keypair (core-protocol-only) peer. The role extension does not interpret the assignee's identity-extension state; the cap targets whatever peer ID was named. |
| **Bootstrap composes** | Role bootstrap and identity bootstrap share the L0 direct-store mechanism and the layer-2 exclusion check. The `is_excluded` helper is pure `tree:get` — it works regardless of which extension's bootstrap ran first. |

The role extension does not depend on the identity extension. Core-protocol-only peers can use roles fully — assignments, exclusions, multi-role, delegation all work without identity. Identity is what makes the *typical* caller story (operator key via local cap) clean.

---

## 11. Composing with group

A group is a natural role-context. `system/role/{group_id}/...` is where group-scoped roles live. See GUIDE-GROUP.md for the group extension's full surface; this section covers the role-side seam.

### 11.1 Group context = role context

When a peer is a member of `acme`, the group's `acme_id` is the role context for any roles the member holds within Acme:

```
system/role/{acme_id}/manager                                   ← role definition
system/role/{acme_id}/assignment/{Public_alice_id}/manager      ← role assignment
system/role/{acme_id}/excluded/{Public_dave_id}                 ← group-scoped exclusion
```

Multi-role per (peer, context) means Alice can be both `manager` AND `auditor` in Acme. Tokens accumulate.

### 11.2 Member entry vs role assignment

Two ways to record "Alice is an admin in Acme":

- **Member entry** (group extension): `system/group/{acme_id}/members/{Public_alice_id}/...` with `role: "admin"`.
- **Role assignment** (role extension): `system/role/{acme_id}/assignment/{Public_alice_id}/admin`.

These overlap. The convention pinned by the group extension:

- Member entry records the member's PRIMARY role and acts as the membership record. Revoking the member entry = removing from the group entirely.
- Role assignments record ADDITIONAL roles within the group.
- Removing a member (`system/group:remove_member`) removes both the member entry AND all role assignments scoped to that group's context.

For everyday "Alice is a member with role X," the member entry is sufficient. For complex multi-role situations, role assignments add finer granularity.

### 11.3 Governance and role assignments

The group extension's governance pattern (founder K-of-N, admin set, all-members, on-chain, hierarchical) determines who can sign role-context operations within the group. Role assignments are entity-creations; the governance signing rules apply.

For a `founder_k_of_n` group with `threshold: 2`, two founders must co-sign a `system/role:assign`. For `admin_set`, the configured admin threshold applies. The role handler's coverage check is unchanged — it still validates that the caller's authority (here, the K-of-N joint cap or the admin's cap) covers the role's grants.

---

## 12. What gets stored where (and who can write it)

| Path | Who writes | Mutation gates | Sync exposure |
|---|---|---|---|
| `system/role/{context}/{role_name}` (definition) | `system/role:define` op (default) OR direct `tree:put` (alternative; admin-gated) | Coverage check at definition-write time + `re-derive` cascade on mutation | Per the deployment's sync-scope caps; typically visible to all peers in the context |
| `system/role/{context}/assignment/{peer_id}/{role_name}` | `system/role:assign` ONLY | Direct `tree:put` rejected at content validation | Per sync-scope caps; visible to context members |
| `system/role/{context}/excluded/{peer_id}` | `system/role:exclude` ONLY | Direct `tree:put` rejected at content validation | Per sync-scope caps; SHOULD propagate to all runtime peers in fleet (the fleet-wide reactive sweep relies on it) |
| `system/capability/grants/role-derived/{context}/{peer_id}/{token_hash}` | Role handler (assign / re-derive / delegate) ONLY | Pinned path so revocation flows can find tokens | Storage-local; tokens are caps validated by the standard chain mechanics |

**Application-level capability grants should target handler operations, not raw tree paths.** Per GUIDE-CAPABILITIES.md §2.2 (kernel-vs-handler principle):

| Want to allow | Grant this | NOT this |
|---|---|---|
| Defining roles | `system/role:define` | `system/tree:put` on `system/role/{context}/{role_name}` |
| Assigning roles | `system/role:assign` | `system/tree:put` on `system/role/{context}/assignment/*` |
| Excluding peers | `system/role:exclude` | `system/tree:put` on `system/role/{context}/excluded/*` |
| Removing assignments | `system/role:unassign` | `system/tree:put` (delete) on assignment paths |
| Re-deriving on definition change | `system/role:re-derive` | (handled internally by `define`) |
| Member-to-member delegation | `system/role:delegate` | direct cap issuance bypassing role lifecycle |

---

## 13. Pitfalls and antipatterns

### 13.1 Granting raw `tree:put` to assignment or exclusion paths

```
; ANTIPATTERN
{handlers: {include: ["system/tree"]},
 operations: {include: ["put"]},
 resources: {include: ["system/role/group/team-alpha/assignment/*"]}}
```

Allows the holder to fabricate assignments — and therefore mint role-derived caps — without the coverage check firing. Even if the holder doesn't have broad authority, they can scan for a role definition that gives broad grants and assign themselves to it.

```
; COHERENT
{handlers: {include: ["system/role"]},
 operations: {include: ["assign", "unassign"]},
 resources: {include: ["system/role/group/team-alpha/assignment/*"]}}
```

Routes through the handler. The coverage check rejects any assignment whose role grants exceed the assigner's authority.

### 13.2 Treating exclusion as universal denial

Exclusion is three-layer (§7), not magic. Layer 1 deletes role-derived tokens (the standard `is_revoked` check rejects them). Layer 2 blocks future role derivations. Layer 3 is opt-in handler-level checks. **Non-role caps are not touched by exclusion.** A peer with a directly-issued cap (e.g., from a runtime peer to a friend, not derived from a role) keeps it across an exclusion event.

If you want exclusion to bite for a specific handler, that handler MUST opt in (`is_excluded(context, author, ctx)` at request time).

### 13.3 Revoke-first re-derive

Don't. Issue T_new before revoking T_old. Revoke-first strands the assignee with no token if the issue leg fails. Issue-first is the MUST default; revoke-first is a deployment-configurable mode for specific security trade-offs only.

### 13.4 Forgetting the bootstrap exemption

The first role assignments on a fresh peer happen via L0 direct-store. Implementations that try to dispatch through `system/role:assign` during initial provisioning will fail (the role handler isn't registered yet). The L0 path is for the SDK's bootstrap code only; it MUST close after handler registration.

If your application code reaches the L0 derivation path post-bootstrap, you have a security hole. The coverage check is bypassed by construction on L0.

### 13.5 Defining a role with broader grants than the assigner holds

The role definition can carry any grants — the role extension doesn't validate the definition against any particular authority. The coverage check fires only at `assign` time, against the assigner's cap. This means: it's perfectly valid to have a `super-admin` role definition with wildcards; the check is what ensures only `super-admin`-level callers can actually assign it.

The danger: if you make the role definition mutable by lower-tier admins, and they bind it to a wildcard-cap handler op (e.g., they get `system/role:define` access without realizing it includes "and the resulting role can be assigned to anyone with sufficient authority elsewhere in the system"), they can engineer privilege-escalation paths via re-derive cascades. **Bind `define` access to the deployment's admin set.** §3.4.

### 13.6 Mixing role-derived caps with chain-root invariants

Role-derived caps don't have a single chain in the way embedded caps do. They're synthesized from a template at issuance. If you're trying to apply a chain-root check to a role-derived cap, you're using the wrong mechanism — the right one is the assigner-coverage check. See GUIDE-CAPABILITIES.md §3.3 for which extension uses which check.

### 13.7 Forgetting fleet-wide reactive sweep

When Alice excludes Bob on `alice_desktop`, the exclusion entity syncs to `alice_phone`. **`alice_phone` MUST sweep its own role-derived subtree.** Implementations that only revoke at the issuing peer leave stale tokens on other fleet members; cross-fleet attackers can use those tokens until they're independently revoked.

This is a real cross-implementation test surface: an exclusion arrives on a non-issuing runtime peer; that peer sweeps its own role-derived subtree; the issuing-side tokens that have already synced to it are deleted.

### 13.8 Assuming multi-role tokens union at verification

The chain verifier doesn't union grants across tokens. If Bob holds both `manager` and `auditor` tokens for `group/team-alpha`, he must present *one* of them (whichever covers the requested operation) on the EXECUTE. The verifier validates that one token. There's no "Bob's effective scope" computation.

If you write code that requires "the union of all of Bob's role tokens," you're operating outside the protocol's contract. The single-token-presentation model is intentional and matches the per-EXECUTE verification model.

### 13.9 Conflating role definition mutability with role assignment lifecycle

A role definition is a template. Mutating it cascades through `re-derive` (per §5) to all current assignees. Assignment lifecycle (`assign`, `unassign`) is per-assignee. **Don't conflate the two:** redefining `manager` to drop a permission does NOT remove anyone from the manager role — they continue as managers, with re-derived tokens reflecting the new permission set. Removing someone from the role requires `unassign`.

---

## 14. Worked examples

These walk concrete deployment shapes. The walkthroughs ground the role mechanics in real validation surfaces from `EXPLORATION-DEPLOYMENT-SHAPES.md` so you can trace from the guide back to the use case.

The first three walkthroughs (§14.0, §14.0a, §14.1) follow the same Acme small business through three deployment shapes: Stage-1 (role-only, no identity), the transition into Stage-2, and Stage-2 fully composed. The progression illustrates that role works without identity, that identity is genuinely additive, and that role state survives the transition.

### 14.0 Stage-1: Acme small business, role-only (no identity layer)

Acme is a small consulting firm: founder Alice runs the network; employees Bob, Carol, Dave use it day to day. Acme has decided NOT to install the identity extension yet — they want network management working before they take on the identity ceremony. This is a Stage-1 deployment.

**What's installed.** V7 + EXTENSION-ROLE + EXTENSION-SUBSCRIPTION + capability machinery. No identity extension. No quorum, no controller cert, no agent certs.

**Identity model.** Each person has one device with one keypair. A keypair = a peer. The peer's `system/peer` content_hash is its stable identifier. There is no "the same Alice across devices" — Alice's laptop is one peer, period. If Alice loses her laptop, she gets a new keypair = a new peer = new role assignments.

**Setup — Alice creates the network**

Alice's peer A boots up; her local SDK provisions root caps via the L0 startup-time path (per §9.2 / §4.5 of the role spec). At this point A holds caps allowing it to define roles and assign peers. No other peer is connected yet.

Alice defines two roles for the office context:

```
EXECUTE system/role:define
  resource: system/role/acme/staff
  params:   { grants: [
    { handlers: ["system/tree"],
      resources: ["shared/acme/*"],
      operations: ["get", "put"]
    },
    { handlers: ["system/tree"],
      resources: ["proposals/acme/*"],
      operations: ["get", "put"]
    }
  ]}
```

```
EXECUTE system/role:define
  resource: system/role/acme/manager
  params:   { grants: [
    ; everything staff has, plus role-management
    { handlers: ["system/tree"],
      resources: ["shared/acme/*", "proposals/acme/*"],
      operations: ["get", "put"]
    },
    { handlers: ["system/role"],
      resources: ["system/role/acme/assignment/*"],
      operations: ["assign", "unassign", "delegate"]
    }
  ]}
```

The role definitions land at:

```
system/role/acme/staff
system/role/acme/manager
```

Note `manager` includes `system/role:delegate` per RR-1.14 / PR-8.2 — it's a delegatable role. `staff` is not.

**Bringing peers in — connection + initial assignments**

Bob's peer B comes online and connects to A via standard V7 HELLO/AUTHENTICATE. Per Acme's deployment, A's initial-grant policy at `system/role/initial-grant-policy` is set to `anonymous-deny` (closed network — strangers don't get caps). So B's connection authenticates but B doesn't yet hold any role-derived cap.

Alice (out of band, knowing Bob's peer hash from when she set up his laptop) assigns B the staff role:

```
EXECUTE system/role:assign
  resource: system/role/acme/assignment/{B_peer_hex}/staff
  params:   { role: "staff" }
```

Where `{B_peer_hex}` is the lowercase hex of B's `system/peer` content_hash — what role's path encoding uses (per §3.1 of the spec). The handler:

1. Validates the resource path resolves cleanly.
2. Reads the role definition at `system/role/acme/staff`.
3. Performs the RL2 check — Alice's caller cap MUST cover the role's grants. Alice's startup-time root caps cover everything; check passes.
4. Resolves template variables (none in this role's grants).
5. Issues a derived cap with `granter: A_peer_hash`, `grantee: B_peer_hash`, `parent: null` (root cap per v2.0 PR-1).
6. Persists the cap at `system/capability/grants/role-derived/acme/{B_peer_hex}/{token_hash}` and the granter's signature at `{cap_path}/signature`.
7. Writes the linkage entity at `system/role/acme/derived-tokens/{B_peer_hex}/staff`.

Tree state after Bob's assignment:

```
system/role/acme/staff                              ← role def (Alice's)
system/role/acme/manager                            ← role def (Alice's)
system/role/acme/assignment/{B_peer_hex}/staff      ← Bob's assignment
system/role/acme/derived-tokens/{B_peer_hex}/staff  ← linkage
system/capability/grants/role-derived/acme/{B_peer_hex}/{T_B_hash}        ← Bob's cap
system/capability/grants/role-derived/acme/{B_peer_hex}/{T_B_hash}/signature  ← cap signature
```

**Bob receives his assignment**

Bob's peer B has subscribed to `system/role/acme/assignment/{B_peer_hex}/...` at peer-config time (standard subscription extension flow). When Alice's assign executes and the assignment entity binds on A's tree, sync delivers the entity to B's tree. B's runtime peer picks up the assignment on the subscription's next tick.

B's local SDK now sees the derived cap (delivered via the same subscription surface or via `system/capability/grants/role-derived/acme/{B_peer_hex}/...` — implementation choice). B can now perform `get` and `put` on `shared/acme/*` and `proposals/acme/*` on Alice's peer (or on the household-server peer — same cap, scoped to acme paths). The cap is signed by Alice; V7 chain-walk validates against Alice's `system/peer` entity (which B already has via the connection handshake).

**Adding more peers**

Carol joins as another staff member. Same flow: Alice runs `system/role:assign` for Carol's `{C_peer_hex}/staff`. Carol gets her cap.

Dave joins as a manager (covers the office during Alice's vacations). Alice assigns Dave the `manager` role. Dave's derived cap covers staff resources AND `system/role:delegate` on `system/role/acme/assignment/*`. Dave can now run his own role assignments — he's a delegated administrator.

**Day-to-day operations**

Bob, Carol, Dave each perform operations using their derived caps. Each EXECUTE includes the cap in the envelope; V7's `verify_capability_chain` validates root → leaf (in this case, root IS leaf — root caps). RL1 enforces the cap's grants against the EXECUTE's resource/op.

If Bob tries `system/role:assign` for someone — denied. His `staff` cap doesn't cover `system/role` operations. RL1 rejects at dispatch.

If Dave tries `system/role:delegate` — succeeds for `manager` (his role includes `delegate`); fails for `staff` (the staff role grants in Dave's manager cap include `delegate` only on the manager role, not on staff). Per PR-8.2.

**Removing peers**

Carol leaves the company. Alice runs:

```
EXECUTE system/role:exclude
  resource: system/role/acme/excluded/{C_peer_hex}
```

The exclusion entity binds at the resource path. The handler's layer-1 sweep walks `system/capability/grants/role-derived/acme/{C_peer_hex}/...` and unbinds Carol's cap + signature sibling. Carol's linkage entity is removed. Bob and Dave's caps are unaffected — exclusion is per-peer, not per-context.

If Carol later attempts to use her cap (say she cached it locally), V7's chain-walk fails — the cap's tree binding is gone (V7 §5.5 `is_revoked` returns true when the cap path is not bound). Excluded.

**What this Stage-1 deployment doesn't have:**

- **Multi-device for Alice.** If Alice wants her laptop AND her phone to both act as "Alice the manager," she'd need two role assignments — one per peer. Each device is a separate peer.
- **Recovery from key loss.** If Alice loses her laptop, she has no path back to "her" identity. The peer is gone; new keypair = new peer = new role assignments (Bob/Dave would re-issue Alice's manager role to her new peer if they have a manager cap that includes `delegate`).
- **Cross-network identity recognition.** Acme is a closed network; peers connect to each other via direct V7 caps. A peer outside Acme's deployment doesn't recognize Acme members as anything other than raw peer-IDs.
- **Controller rotation without disrupting contacts.** Not a feature without identity.

For Acme as a closed-network small business, this Stage-1 deployment is a complete usable configuration. Many small deployments stay here.

### 14.0a Transitioning Acme to Stage-2 (installing identity)

Six months pass. Alice's laptop dies. She lost her keypair. The deployment recovers (Bob and Dave re-issue Alice's role to her new laptop's keypair), but the experience is rough — Alice had to manually re-provision, re-pair with everyone, etc. Alice decides to install identity.

**What's added.** EXTENSION-ATTESTATION + EXTENSION-QUORUM + EXTENSION-IDENTITY on Alice's peer. (Each peer in Acme decides independently whether to install identity. Bob and Dave can stay Stage-1; they interoperate fine.)

**The transition for Alice's peer**

Alice's existing peer A still has its keypair. The transition wraps it into an identity rather than replacing it.

Step 1 — Alice creates a quorum. She picks two friends (or trusted secondary devices) as backup-key-holders: a hardware token H1 and a paper-backup H2. K-of-N = 2-of-3 (Alice + H1 + H2). The quorum is created via L0 startup-time:

```
QUORUM.system/quorum:create
  signers:   [A_peer_hash, H1_peer_hash, H2_peer_hash]
  threshold: 2
```

The quorum entity binds at `system/quorum/{q_hex}`.

Step 2 — Alice mints a top-level controller cert binding her existing keypair as the controller. The cert is signed K-of-N by the quorum (Alice signs; one of H1/H2 signs):

```
ATTESTATION.system/attestation:create
  attesting: q_hex
  attested:  A_peer_hash    ← Alice's existing keypair, unchanged
  properties: { kind: "identity-cert", function: "controller", mode: "public" }
```

The cert binds at `system/identity/public/cert/{cert_hash}`.

Step 3 — Alice runs `system/identity:configure` to bind her peer-config to the trusted quorum:

```
EXECUTE system/identity:configure
  resource: system/identity/peer-config
  params:   { trusts_quorum: q_hex, bindings: [] }
```

The peer-config entity binds at `system/identity/peer-config`. Per PI-2 phase 4, a local-peer→controller cap mints at `system/capability/grants/identity/peer-to-controller/{controller_hex}` (where `{controller_hex}` is hex of the controller cert's content_hash — A's identity, in this case, since she's controller).

**What survives the transition**

Alice's role assignments (manager, etc.) bound at `system/role/acme/assignment/{A_peer_hex}/manager` — the `{A_peer_hex}` is hex of A's `system/peer` content_hash, which is **unchanged** because Alice's keypair is unchanged. Same hex; same path; entity still bound.

Same for the role-derived cap at `system/capability/grants/role-derived/acme/{A_peer_hex}/{T_A_hash}`. Same path; cap signed by `granter: A_peer_hash` (A's `system/peer` content_hash); A's `system/peer` entity is unchanged. Cap chain-validates.

Same for Bob, Carol, Dave's role assignments to A's manager-derived caps if they had any — those reference A's hash and survive.

**TV-RV-2.1 in the wild.** This is exactly what the validation profile's TV-RV-2.1 verifies: existing role state survives identity install. The Stage-1 deployment becomes a Stage-2 deployment by adding identity, with no disturbance to role.

**What Alice gains**

Alice can now add agents (per-device peers) under her identity:

- Alice adds her phone. The phone has a new keypair P_phone, hence a new `system/peer` content_hash. Alice (as controller) mints an agent cert: `kind=identity-cert, function=agent, attesting=A_peer_hash (controller), attested=P_phone_hash`.
- The phone's agent peer is now part of Alice's identity. Contacts that recognize Alice (by caching her controller cert in 3-key default) will recognize the phone's caps as belonging to Alice.

**What Alice's phone needs that she didn't already have**

The phone is a NEW peer with a NEW content_hash. **It needs its own role assignments.** Alice's existing manager assignment is to `{A_peer_hex}` (her laptop), not to `{P_phone_peer_hex}`.

Alice runs (from her laptop, since the laptop holds the manager cap):

```
EXECUTE system/role:assign
  resource: system/role/acme/assignment/{P_phone_peer_hex}/manager
  params:   { role: "manager" }
```

Now her phone has a manager-derived cap. She can act as Acme's manager from her phone. Both her laptop and phone act for the same logical Alice; both have their own caps; either can rotate without affecting the other.

**TV-RV-2.2 in the wild.** New agent peers under the identity need new role assignments. The validation profile's TV-RV-2.2 catches impls that incorrectly assume identity-cert membership transfers role assignments — it doesn't, by design.

**What about Bob and Dave**

Bob and Dave can stay Stage-1 indefinitely. They have raw peer keypairs; they connect to Alice's laptop and phone via V7 caps; the identity layer is invisible to them. If Bob's peer wants to learn about Alice as an identity (rather than as two separate peers), he installs identity-aware contact caching — but he doesn't have to. Acme runs fine with mixed-stage peers.

When Bob eventually loses his laptop (or his phone, or both) and decides he wants recovery too, he runs the same transition for his peer. Same mechanics; independent of Alice. No coordination required — identity is per-peer-deployment.

**What still doesn't work after the transition**

Some things remain out of scope (per PROPOSAL-IDENTITY-INFRASTRUCTURE.md §3b informative scope-mapping):

- **Federated identity** (multiple quorums per user). Alice has one quorum.
- **Cross-Acme identity recognition.** If Acme partners with another company, members of the other company recognize Acme members as identities only via group/registry mechanisms (not at the role layer).
- **Skipping the controller layer.** Alice's identity has the controller→agent chain; this is structural.

For Acme's needs (recovery + multi-device for the founder), this is sufficient. The richer cases compose at higher layers.

### 14.1 Small business: Acme

Acme is a `founder_k_of_n` group (Alice, Bob, Carol; threshold 2). The group has its own identity stack and a runtime peer (`partnership_server`). Acme uses the role extension for in-group authority management.

**Setup:**

```
; Role definitions in Acme's context
system/role/{acme_id}/employee → {
  name: "employee",
  grants: [
    {handlers:   {include: ["system/tree"]},
     resources:  {include: ["shared/{context}/*"]},
     operations: {include: ["get", "list"]}},
    {handlers:   {include: ["system/group"]},
     resources:  {include: ["system/group/{context}"]},
     operations: {include: ["status"]}}
  ]
}

system/role/{acme_id}/engineer → {
  name: "engineer",
  grants: [
    ; engineer can put under shared/{context}/code/
    {handlers:   {include: ["system/tree"]},
     resources:  {include: ["shared/{context}/code/*"]},
     operations: {include: ["get", "put", "list"]}},
    ; ... plus employee-level grants (or assigned via multi-role)
  ]
}

system/role/{acme_id}/manager → {
  name: "manager",
  grants: [
    {handlers:   {include: ["system/tree"]},
     resources:  {include: ["shared/{context}/*"]},
     operations: {include: ["get", "put", "list", "delete"]}},
    {handlers:   {include: ["system/role"]},
     resources:  {include: ["system/role/{context}/assignment/*"]},
     operations: {include: ["assign", "unassign", "delegate"]}}     ; manager can manage roles + delegate (per §14.3 vacation coverage)
  ]
}
```

The `delegate` operation in the `manager` role's grants is what makes §14.3's vacation-coverage flow possible — without it, Bob (a manager) cannot drive `:delegate` to Carol because the protocol-layer dispatch check would reject the EXECUTE with `403 capability_denied: insufficient capability for handler scope`. Delegate-ability is opt-in per role; see EXTENSION-ROLE.md §5.6 for the convention.

**Onboarding Dave as an engineer:**

1. Acme's quorum (2-of-3 founders) drives `system/role:assign` (in implementation, the founder quorum signs the controller-cert attestation that `:configure` consumes; the resulting local-peer→controller cap is the single-sig caller cap on `:assign`. The quorum drives `:assign` through the configure-ceremony bridge, not as a direct multi-sig caller cap — V7 §5.5's `verifyRootGranter` requires the local peer in the cap's signers, which a founder quorum that doesn't include partnership_server cannot satisfy directly):
   ```
   resource: system/role/{acme_id}/assignment/{Public_dave_id}/engineer
   params:   {role: "engineer"}
   ```
2. The coverage check validates: the founders' joint cap covers the engineer role's grants.
3. Layer-2 exclusion: no exclusion entry for Dave → proceeds.
4. Assignment entity persisted.
5. Derived caps minted, signed by `partnership_server`'s runtime peer, persisted at `role-derived/{acme_id}/{Public_dave_id}/...`.
6. Caps delivered to Dave (via Acme's onboarding flow — typically inbox push).

**Multi-role: Dave is also given on-call admin:**

```
resource: system/role/{acme_id}/assignment/{Public_dave_id}/admin
params:   {role: "admin"}
```

Dave now holds both `engineer` and `admin` tokens. When he EXECUTEs an operation, he presents whichever token covers it.

**Eviction:**

```
system/role:exclude
resource: system/role/{acme_id}/excluded/{Public_dave_id}
```

What happens:
- Layer 1: all of Dave's role-derived tokens at `role-derived/{acme_id}/{Public_dave_id}/...` are deleted.
- Layer 1 fleet-wide: the exclusion entity syncs to `alice_desktop`, `bob_laptop`, `carol_phone`, etc. Each of those peers sweeps its own `role-derived/{acme_id}/{Public_dave_id}/...` subtree.
- Layer 2: future `assign` and `re-derive` for Dave in `acme_id` context are rejected.
- Layer 3: any group-aware handler that opts in to the exclusion check rejects Dave's in-flight requests.

If Acme's group handler ALSO removes Dave's member entry (`system/group:remove_member`), the member entry is removed AND any role assignments scoped to Acme's context are removed — see GUIDE-GROUP.md §11.2.

### 14.2 Open-source collaborator network

A small open-source project. Three role tiers with graduated participation:

```
system/role/{project_id}/proposer → {
  name: "proposer",
  grants: [
    ; can read public docs, submit issues/PRs to a designated inbox
    {handlers:   {include: ["system/tree"]},
     resources:  {include: ["public/{context}/*"]},
     operations: {include: ["get", "list"]}},
    {handlers:   {include: ["system/inbox"]},
     resources:  {include: ["system/inbox/{context}/issues/*"]},
     operations: {include: ["receive"]}}
  ]
}

system/role/{project_id}/contributor → {
  ; commits to project branches; reviews PRs
  ...
}

system/role/{project_id}/maintainer → {
  ; admin authority over the project group + role assignments
  ...
}
```

**Anonymous fallback for unknown peers:**

```
system/role/bootstrap-policy: {
  unknown_peer:  "allow",
  default_role:  "proposer"
}
```

A first-time visitor who EXECUTEs against the project's public infrastructure is bootstrap-derived a `proposer` cap. They can read public content and submit through the issues inbox without any explicit role assignment. **Layer 2 exclusion still fires** — a previously-excluded peer attempting via anonymous-allow is rejected.

**Promotion to contributor:**

A maintainer (already assigned `maintainer`) drives `system/role:assign` for a peer they recognize. The coverage check validates the maintainer's authority covers `contributor`'s grants. The peer is now assigned; their proposer cap (if they had one) coexists with the new contributor cap.

**Quorum drift mitigation:**

Voluntary-participation groups face threshold fragility. Project maintainers may drift. Setting `threshold: 3-of-7` (rather than `2-of-3`) lets the project tolerate four maintainers going inactive before the quorum drops below threshold. This is deployment policy, not a role-extension mechanism.

### 14.3 Member-to-member delegation: vacation coverage

**Prerequisite:** the `manager` role's grants must include `system/role:delegate` on `system/role/{context}/assignment/*`. See §14.1's manager role definition; without that grant entry, Bob's `:delegate` EXECUTE fails at the protocol-layer dispatch check before the handler runs. Delegate-ability is opt-in per role (EXTENSION-ROLE.md §5.6).

Bob is a manager at Acme. He's going on vacation. He delegates his manager authority to Carol for two weeks:

```
bob_runtime_peer.execute(
  "system/role", "delegate",
  resource: "system/capability/grants/role-derived/{acme_id}/{Public_carol_id}/{token_hash}",
  params: {
    delegator:  Public_bob_id,
    delegate:   Public_carol_id,
    context:    "{acme_id}",
    role:       "manager",
    scope:      [<full manager grant set>],
    expires_at: 1741800000000   ; two weeks out
  },
  capability: <Bob's manager role-derived cap>
)
```

Carol now holds a delegation cap chained from Bob's manager cap. She can do everything a manager can do until expiry — except things that fail the coverage check against Bob's cap (which is impossible by construction; the scope is bounded to Bob's cap's coverage).

When Bob returns and the delegation expires, the standard `is_revoked` check rejects Carol's cap on next use. No handler-side cleanup needed.

If Bob is unassigned from `manager` mid-vacation, Carol's delegation cap chains from a now-revoked parent — she loses authority within sync-latency of Bob's unassign. Bob would need to re-issue under his new authority (if he was reassigned to manager), or Carol needs to be directly assigned.

### 14.4 Lateral trust escalation (adversarial)

Bob's runtime peer is compromised. The attacker holds Bob's runtime peer's keypair. What can they do with role-derived caps Bob holds?

**What the attacker can do:**

- EXECUTE under any role-derived cap Bob holds, against the resources those caps cover.
- Issue further delegation caps from Bob's role-derived caps to attacker-controlled peers — **but the coverage check attenuates** to the scope of Bob's caps. Attacker can't escalate beyond Bob.
- Redirect Bob's communications.

**What the attacker cannot do:**

- Mint role-derived caps for themselves (they're not in any assignment; layer-2 exclusion plus assignment-by-explicit-call blocks this).
- Modify role definitions (the attacker's caps cover what *Bob* could do, not necessarily `system/role:define`).
- Bypass exclusion (if Acme excludes Bob, the layer-1 sweep deletes the role-derived tokens on next sync; layer-2 prevents new derivations).

**Compromise window:**

Bounded by:
- Sync latency for the exclusion entity to reach all of Bob's runtime peers' attestation caches.
- Cap TTL (deployments using short TTLs reduce the exposure window).
- The attacker's ability to delegate further before Bob's caps are revoked — bounded by the coverage check (delegations attenuate).

The architecture keeps the blast radius bounded; it does not eliminate the compromise window. This is the structural property — see `EXPLORATION-DEPLOYMENT-SHAPES.md §14.10` for the full walkthrough.

---

## 15. Reference

### 15.1 Entity types

| Type | Path | Created via | Purpose |
|---|---|---|---|
| `system/role` (definition) | `system/role/{context}/{role_name}` | `system/role:define` (default) | Named grant template |
| `system/role/assignment` | `system/role/{context}/assignment/{peer_id}/{role_name}` | `system/role:assign` | Bind peer to role within context |
| `system/role/exclusion` | `system/role/{context}/excluded/{peer_id}` | `system/role:exclude` | Deny peer all role-derived access in context |
| `system/capability/token` (role-derived) | `system/capability/grants/role-derived/{context}/{peer_id}/{token_hash}` | Handler (assign / re-derive / delegate) | Standard cap; pinned storage path |

### 15.2 Handler operations (`system/role`)

| Operation | Authorized by | Async? | Purpose |
|---|---|---|---|
| `define` | Coverage check against the proposed grant set; admin-tier in practice | No | Create or mutate role definition; cascades through `re-derive` |
| `assign` | Coverage check against `caller_capability`; layer-2 exclusion check; typically the operator key via local cap | No | Bind peer to role; mint derived caps |
| `unassign` | Same authority as assign (SHOULD) | No | Remove specific assignment; revoke its derived tokens |
| `exclude` | Caller's cap covers the context | No (with internal sync); fleet-wide reactive sweep | Add denial entity; layer-1 deletes role-derived tokens |
| `unexclude` | Same as exclude | No | Remove denial entity; tokens are NOT auto-restored |
| `re-derive` | Caller authorizes against the role-definition path | Per-assignee independent flows | Re-issue tokens for all assignees on definition mutation; ordered writes |
| `delegate` | Coverage check against the delegator's authority (not the operator's) | No | Member-to-member delegation; cap rooted at delegator's runtime peer |

### 15.3 Status codes

| Code | When |
|---|---|
| 200 | Operation succeeded |
| 400 `malformed_resource` | Resource path doesn't decompose to expected (context, peer, role) shape |
| 400 `invalid_assign_request` | Required field missing (e.g., `role` selector on `assign`) |
| 401 `capability_revoked` | EXECUTE in flight using a token that was just revoked by `re-derive` or `unassign` |
| 403 `assigner_authority_insufficient` | Coverage check failed — assigner's cap doesn't cover role-derived grants |
| 403 `assignee_excluded` | Layer-2 exclusion check blocked the assignment |
| 403 `peer_excluded_in_context` | Layer-3 handler-level check rejected the request |
| 404 `role_not_found` | Assignment references a role definition that doesn't exist |

### 15.4 The role-identity-protocol seam

```
Core protocol capability system (chain verification)   ← unchanged
   ↑ used by
EXTENSION-IDENTITY (local peer→Op cap is the bridge)
   ↑ Op invokes
EXTENSION-ROLE (this guide)                             ← templates + assignments + exclusions
   ↑ produces
Role-derived caps (signed by runtime peer)              ← granter = runtime peer; chain root = local peer
   ↑ delivered to
Assignees (peers in the context)
```

### 15.5 Spec section pointers (for the reader who wants to dig deeper)

**EXTENSION-ROLE.md:**
- §1.3 — Coherent Capability (kernel-vs-handler principle applied to role)
- §1.5 — Identity coherence; shared `is_excluded` helper
- §2 — Type definitions
- §3 — Tree path conventions; reserved names
- §4 — Role handler; §4.3 the assigner-coverage check; §4.4 unassign/exclude/unexclude; §4.5 bootstrap exemption; §4.6 determinism; §4.7 anonymous fallback policy
- §5.1 — Grant derivation; §5.5 re-derive lifecycle; §5.6 member-to-member delegation
- §6 — Three-layer exclusion; §6.4.1 unassign token-revocation flow; §6.5 fleet-wide reactive sweep
- §7 — Bootstrap grant convention
- §9.1 — Attenuation invariant; §9.6 role-definition write access

**Other:**
- ENTITY-CORE-PROTOCOL.md §5 — capability system; §5.6 — `is_attenuated` primitive
- EXTENSION-IDENTITY.md §3.10 — `verify_k_of_n_signatures` (entity-level; distinct from cap-chain machinery)
- EXTENSION-IDENTITY.md §5.10 — bootstrap exemption (parallel to role's §4.5)
- EXTENSION-GROUP.md §7.2 — group composition with role
- GUIDE-CAPABILITIES.md §5.3 — role checklist (admin-with-coherent-constraints)
- GUIDE-IDENTITY.md §6 — composing identity with roles (the operator key as caller)

---

## 16. Document history

- **v0.3, second pass — staleness fixes per Go team flag:** Three sections aligned with v2.0 PR-1 (root-cap shape) and v1.6 (path rename). **§4.2** rewritten — "derived cap's parent is the role handler's own grant" → `parent: null` per PR-1; both bootstrap and runtime derivations produce root caps; structural difference is at the issue-time authority check, not the cap shape. **§9.2** rewritten — removed obsolete bootstrap-vs-runtime cap-shape distinction; both paths produce structurally identical root caps; difference is RL2 (runtime) vs no-RL2 (bootstrap, peer-owner authority). **§9.4** path renamed from `system/role/bootstrap-policy` → `system/role/initial-grant-policy` per v1.6 spec rename; `bootstrap-on-known-attestation` mode renamed to `recognize-on-attestation`; added note that Stage-1 deployments don't have access to the recognize-on-attestation mode. The §14.0 / §14.0a additions earlier in this revision were already aligned with PR-1 + v1.6 path; this pass brings the older sections (§4, §9) up to the same baseline.

- **v0.3:** Stage-1 / transition walkthroughs added for the role v2.0 cross-impl validation cycle. **§14.0** Stage-1 Acme deployment (role-only, no identity layer): Alice runs the network via raw peer-IDs; defines `staff` and `manager` roles; assigns Bob/Carol/Dave; day-to-day ops; exclusion when Carol leaves; what Stage-1 doesn't have (multi-device, recovery, cross-network identity). **§14.0a** Stage-1 → Stage-2 transition: Alice loses her laptop, decides to install identity; creates quorum + controller cert binding her existing keypair; runs `:configure`; her existing role assignments survive (TV-RV-2.1 from VALIDATION-PROFILE-ROLE.md); adds her phone as new agent peer; phone needs new role assignment (TV-RV-2.2); Bob and Dave can stay Stage-1; Acme runs mixed-stage. Use-case anchors for VALIDATION-PROFILE-ROLE.md Stage-1 (15 checks) and Stage-2 (7 additive checks).

- **v0.2:** Cleanup pass for new-reader clarity. Removed proposal-tracking acronyms (IA8/IA9/IA10/IA11/IA12/IA13/IA14/IA22, R6/R7/R10, RL2, RI1/RI2/RI3) from body prose in favor of descriptive language; "RL2" referenced once parenthetically in §8 as a spec-name pointer for readers cross-referencing the spec but referred to descriptively as "the assigner-coverage check" throughout. Replaced "V7" with "core protocol" or "the protocol" throughout (the spec file is `ENTITY-CORE-PROTOCOL.md`; "V7" was its version, not a name new readers would recognize). Dropped pedantic line-number references ("V7 §5.5 line 1974"). Clarified that the operator key's role in identity-aware deployments is to be the typical caller of `system/role:assign` via the local peer→Op cap; the operator never appears in the cap chain. No content changes; same sections, same examples, same mechanics — just legible to a reader who hasn't tracked every proposal.

- **v0.1:** Initial draft. Walks the model (definition → assignment → derived cap; exclusion as parallel control surface), defining a role, the typical operator-authorized assign flow, multi-role per (peer, context), the re-derive lifecycle (issue-first ordered writes; grace window; named delivery; multi-Op concurrency), member-to-member delegation, three-layer exclusion model with fleet-wide reactive sweep, the assigner-coverage check (`is_attenuated`-based; fail-closed; stricter than chain-root), bootstrap exemption (L0; closed after handler registration), anonymous fallback policy, identity composition, group composition (group_id as role context; member entry vs role assignment convention), per-path write authority, pitfalls, worked examples lifted from validation surfaces (Acme small business; open-source collaborator network; vacation-coverage delegation; lateral trust escalation adversarial walk). Companion to GUIDE-IDENTITY.md, GUIDE-CAPABILITIES.md, GUIDE-MULTISIG.md.
