# Capabilities: User Guide

**Status**: Active
**Audience:** Application developers and operators configuring grants on Entity Core peers. Assumes you know what an EXECUTE request looks like and that handlers are dispatched by URI prefix; does not assume you've read all of V7 §5.
**Spec reference:** ENTITY-CORE-PROTOCOL.md §5 (Capability System), §6.3 (System Tree Handler), §6.8 (Handler Authority Model). Builds on PROPOSAL-COHERENT-CAPABILITY-AUTHORITY (implemented).
**Related guides:** GUIDE-COMPUTE.md, GUIDE-COMPUTE-PROGRAMMING.md.

---

## 1. What capabilities are (and what they enforce)

A **capability** is a signed entity (`system/capability/token`) that authorizes an EXECUTE to run. Every authenticated EXECUTE carries one in its `data.capability` field. Before the handler runs, the peer verifies the token's chain, checks revocation, and matches its scope against the request. If any check fails, the request is rejected with status 403 — the handler never sees it.

Capabilities are not access-control lists, role assignments, or session tokens. They are unforgeable proof, carried with each request, that some authority granted some scope to some grantee. They live as entities in the content store and the tree, like everything else.

**What a capability authorizes — four scope dimensions.** Each grant entry inside a capability matches a request along four orthogonal axes (V7 §3.6, §5.4):

| Dimension | What it gates |
|---|---|
| `handlers` | Which handler patterns this entry authorizes (e.g. `system/tree`, `local/files`). |
| `operations` | Which operation names within those handlers (e.g. `get`, `put`, `subscribe`, `install`). |
| `resources` | Which resource paths (the `resource.targets` field on EXECUTE). |
| `peers` | Which peers are in scope for the network dimension. |

A request must match **all four** dimensions from a **single** entry. Splitting `handlers: [system/tree], operations: [get, put]` across two entries does not authorize `tree:put` if neither entry alone covers it.

**What it does NOT authorize.** Capabilities are silent on:

- *Domain semantics.* The capability says "you may call `system/continuation:install`." It does not say "and your installed continuation is well-formed." That's the handler's job (§3 below).
- *Resource quotas, rate limits, fairness.* Those are implementation/policy concerns layered on top.
- *Identity beyond the chain root.* The chain root is a peer; everything else is delegation. There is no separate "user" namespace.

**Three things the system tracks alongside the token.**

1. **Authority chain** — the linked list of `granter → parent` references terminating at a peer-identity entity. Validated by `verify_capability_chain` at every dispatch (V7 §5.5).
2. **Revocation state** — whether the chain root is still bound in the tree. Removing the binding (`put(path, null)`) revokes the chain (V7 §5.1). Cryptographic validity alone is not enough.
3. **Storage location** — where the capability lives in the tree, used for `capability_path_for` lookups during revocation checks (V7 §5.1).

---

## 2. The kernel-vs-handler principle

There are two tiers of access in the system, and they have different security guarantees.

**Kernel tier — `system/tree:get` and `system/tree:put`.** Direct read/write to the location index and content store. The tree handler enforces capability scope (`handlers`, `operations`, `resources` match) and a defense-in-depth path check (V7 §6.3). It does **nothing else.** No domain validation, no semantic invariants, no authority-chain verification of embedded fields. Whatever bytes you hand it land in the store.

**Handler tier — domain handlers and system extension operations.** Each handler owns its entity types and validates them on creation. The continuation handler validates the `dispatch_capability` authority chain (the in-chain check, §3.1) before installing a continuation. The subscription handler validates the `deliver_token` authority chain before persisting a subscription. The role handler verifies the assigner's authority covers the grants the role will derive.

The tree handler is **kernel-level by design** — it's the lowest data primitive in the system. It is also the wrong place to expose to most application-level grants for the system handler namespaces. From V7 §6.3:

> For load-bearing entity types — types whose data carries semantic content that affects later runtime behavior (capability references, dispatch targets, state-machine fields, scoped grants) — the proper creation path is the owning handler's operation. (...) Direct `tree:put` is permitted (the kernel exposes it by design) but bypasses the handler's semantic validation.

This is a **convention, not a reservation.** The protocol cannot prevent a sufficiently-broad grant from writing anywhere; it can only document where validation lives and tell you to grant accordingly.

### 2.1 What "load-bearing" means

An entity type is load-bearing when its **data fields carry meaning the system later acts on**:

- A capability hash that authorizes a future dispatch (`continuation.dispatch_capability`, `subscription.deliver_token`, `compute/apply.capability`).
- A target path or operation that something later resolves and calls.
- A grant set or assignment that triggers token derivation.
- A state-machine field that drives subsequent transitions.

Ordinary types — strings, numbers, application data, role *definitions* (not assignments) — are not load-bearing. Direct `tree:put` for them is fine; nothing semantic depends on a handler having seen them.

The line to ask yourself: *if an attacker can write any byte sequence as the data of this entity type, what can they cause to happen later?* If the answer is "nothing — it's just data the application reads," it's ordinary. If the answer is "they can pick which capability gets used, where the dispatch goes, or what permissions get derived," it's load-bearing.

### 2.2 What this means for your grant configuration

For each load-bearing extension, grant the handler's create operation, not the underlying tree path:

| Want to allow | Grant this | NOT this |
|---|---|---|
| Installing a continuation | `system/continuation:install` | `system/tree:put` on `system/continuation/suspended/*` |
| Subscribing to a pattern | `system/subscription:subscribe` | `system/tree:put` on `system/subscription/*` |
| Assigning a role | `system/role:assign` | `system/tree:put` on `system/role/{ctx}/assignment/*` |
| Excluding a peer from a context | `system/role:exclude` | `system/tree:put` on `system/role/{ctx}/excluded/*` |
| Installing a compute subgraph | `system/compute:install` | `system/tree:put` on compute expression paths |

The handler operations validate. The raw `tree:put` does not.

---

## 3. Coherent capability — the principle

A capability configuration is **coherent** when the grants you hand out cannot be used to bypass the validation the system handlers would otherwise apply. R0 and R1 are the two structural rules that together produce coherence (PROPOSAL-COHERENT-CAPABILITY-AUTHORITY).

**R0 — Create-Operation Coverage.** Every load-bearing entity type has a handler operation responsible for its creation. Application-level grants cover that operation, not raw `tree:put`. R0 is about *where* validation happens.

**R1 — Embedded Capability Authority.** When a create operation accepts an embedded capability reference (a hash to a `system/capability/token` that will authorize a *future* dispatch), the handler verifies at creation time that the writer's identity is in that capability's authority chain. R1 is about *what* validation actually checks.

R0 says "use the handler." R1 says "and here's what the handler must enforce." Either alone is insufficient — R0 without R1 just exposes a handler that doesn't validate; R1 without R0 has nowhere to live.

**The three slots (read this first — it prevents the recurring confusion).** A capability presented in an EXECUTE has three independent identity slots; conflating them is *the* recurring source of cross-peer capability bugs. The general statement and the reasoning are in **ENTITY-CORE-PROTOCOL.md §5.2 "Cross-peer capability provenance — the three slots"** — read it once and the rest of this section is mechanical:

- **Root** — the peer that *owns the resource* being acted on. Authority over X's resource can originate only from X. (`verify_capability_chain`, V7 §5.5.)
- **Grantee** (of the leaf) — the *wielder*: whoever authors the EXECUTE that presents the cap. V7 §5.2 hard-denies `grantee != author`.
- **In-chain granters** — every party that attenuated, *including any installer/minter that pre-mints a cap for later use*. The R1 install check below asks only that the writer appears as a granter **anywhere in the chain** — **not** that the chain roots at the writer.

**Locally these three collapse onto one identity** (a peer mints from its own authority and immediately wields it), which is why local-only reasoning silently omits the other two slots — the documented root cause of the EXTENSION-CONTINUATION §3.1a (in-chain) and §4.2 case 3 (grantee) corrections. Whenever a flow crosses peers, fill all three explicitly.

### 3.1 The in-chain check, concretely

Every embedded capability has an authority chain — a linked list of `granter → parent` references that bottoms out at a root capability granted by a peer identity. The R1 check asks: *does the EXECUTE author appear anywhere in that chain?* It is an **in-chain** check, **not** a chain-*root* check — the writer must be a granter *somewhere* in the chain, not its root (EXTENSION-CONTINUATION §3.1a; reading it as a root check is exactly what broke cross-peer continuations). Distinct from the *grantee* slot (the wielder, checked at dispatch by V7 §5.2) and the *root* slot (the resource owner): this check is the **in-chain granter** slot only.

V7 §5.5 exposes the helper:

```
check_creator_authority(cap, writer_identity, ctx) → (found, chain, error)
```

Three outcomes:

- `error(ChainUnreachable)` — some link in the chain is missing from both the EXECUTE envelope's `included` map and the local content store. Reject with **404 `chain_unreachable`**.
- `(false, chain)` — chain is reachable, but the writer's identity is not in it. Reject with **403 `embedded_cap_unauthorized`**.
- `(true, chain)` — chain is reachable and the writer is in it. Accept. The handler MUST persist the capability and its full chain to the content store so future operations can resolve them without re-walking the envelope.

### 3.2 Why in-chain, not "your scope covers what you embed"

A natural-sounding alternative would be: "the writer's grant scope must cover the embedded capability's scope." This is the wrong rule.

The point of embedding a capability is that **someone else dispatches with it later** — the continuation handler when `advance` fires, the subscription engine when a notification ships, the compute handler when a subgraph re-evaluates. A holder of a capability they legitimately received via delegation should not be able to install it as a deferred dispatch by some other component, because that other component would then be wielding their authority outside their lifetime, possibly outside their attenuation chain.

The in-chain rule is tighter: **only embed capabilities your own identity is in the authority chain of.** That means caps your peer issued, or caps you hold by being a granter at some level in the chain. Sub-identities (delegated actors) naturally fail this check — they aren't granters; they're grantees. They can't directly install deferred dispatches, but they can invoke handler operations that internally create such entities under the *handler's* authority. That's by design.

### 3.3 Where R1 applies (current extensions)

| Extension | Embedded cap field | Create op | R1 check |
|---|---|---|---|
| Continuation | `dispatch_capability` | `system/continuation:install` | CT2 — in-chain: writer appears as granter anywhere in chain (§3.1a) |
| Subscription | `deliver_token` | `system/subscription:subscribe` | SB1 — in-chain against EXECUTE author |
| Compute (static literal) | `compute/apply.capability` literal hash in expression | `system/compute:install` | CP1 — in-chain checked at install audit |
| Compute (runtime value) | `compute/apply.capability` from scope/lookup | (eval time, dual-check) | covered by F1-F5 in COMPUTE-APPLY-RESOURCE-CEILING |
| Role | role-derived grants (no single chain) | `system/role:assign` (and `system/role:re-derive`) | RL2 — assigner's caller cap MUST cover the derived grants, checked via V7 §5.6's `is_attenuated` primitive |

Role is the odd one out: its derivation produces grants synthesized from a template, so there's no single chain to walk. The equivalent rule is "the assigner's own authority must cover everything the assignment will mint." Implementation reuses V7's existing `is_attenuated` (which already does scope-vs-scope coverage with attenuation rules) — no new core primitive needed. This prevents inflation — an assigner cannot manufacture broader grants than they themselves hold.

### 3.4 Generalized rule: GR1

Any handler that **stores, persists, or forwards** an accepted `deliver_token` (or equivalent embedded delivery cap) for later use MUST run the in-chain check at receipt time. Subscription is the canonical case. Future async-delivery handlers fall under the same rule.

The inbox `receive` operation does *not* fall under GR1 — the deliver_token there is being used as the dispatch capability on the delivery EXECUTE itself, validated by V7 §5.2 standard dispatch chain. GR1 is for handlers that *keep* the token; receive just processes the delivered entity.

---

## 4. The three capabilities in a subscribe flow

Subscription has the most capability machinery of any extension. There are three different capabilities involved, each rooted at a different peer in cross-peer scenarios, each validated at a different moment. They get conflated regularly. (EXTENSION-SUBSCRIPTION §1.2 has the canonical table; this section walks through it.) Each is itself a three-slot capability (§3 "The three slots" / V7 §5.2): for the **caller capability**, root = B (resource owner), grantee = A (the EXECUTE author wielding it), in-chain granters = whoever attenuated; for the **deliver_token**, root = A (owns the inbox), grantee = B's engine (the future wielder). Don't conflate "rooted at" with "granted to" — that conflation is the recurring bug.

| Capability | Carried in | Authorizes | Issued by | Validated by |
|---|---|---|---|---|
| **Caller capability** | EXECUTE `data.capability` | The subscribe operation itself, with the resource-target scope | The subscribee's namespace authority | Standard dispatch chain (`verify_request`, `check_permission`) |
| **Subscription deliver_token** | `params.deliver_token` | Future async dispatch from the subscription engine to the subscriber's inbox | The subscriber's namespace authority | Subscribe handler: `grants_access` + in-chain check (SB1) |
| **Inbox EXECUTE-level deliver_token** | EXECUTE `data.deliver_token` (per EXTENSION-INBOX §2.3) | A specific async delivery EXECUTE | The deliverer's namespace authority | Standard dispatch chain at the receiver |

Walked through for **Alice subscribing to Bob's data** (Alice = peer A, Bob = peer B):

- The **caller capability** is **B-rooted** — B granted A access to subscribe to that pattern in B's namespace. Validated when A's request arrives at B.
- The **subscription deliver_token** is **A-rooted** — A is authorizing B's subscription engine to later deliver to A's inbox. Validated by B's subscribe handler before persisting the subscription, with two checks: (1) does the token's scope cover the inbox URI A nominated? (2) is A in the token's authority chain (SB1)?
- Later, when a matching write fires on B, B's engine constructs a notification EXECUTE whose `capability` field is the deliver_token. **That's the inbox-EXECUTE-level deliver_token** — it authorizes this specific delivery. A's peer validates it via the standard dispatch chain when it arrives.

Two roots that meet only in the subscribe envelope. Don't try to use one for the other:

- The caller cap does **not** authorize delivery. Even if A's caller cap somehow covered A's inbox, B's engine doesn't get to wield it — it's B-rooted.
- The deliver_token does **not** authorize the subscribe op. B's subscribe handler will reject a deliver_token used as the caller cap (wrong root, wrong scope).

### Common confusion

> "If I'm subscribing to my own data, do I still need a deliver_token?"

Yes. The two-cap structure is uniform across local and cross-peer subscriptions. In the local case both caps happen to be rooted at the same peer, but they still play distinct roles — the caller cap authorizes subscribe; the deliver_token authorizes notifications.

> "Why doesn't the engine just use the caller cap for delivery?"

Because the engine and the subscriber may be different peers, and the caller cap might not cover the subscriber's inbox, or might not even be wieldable by the engine. The deliver_token is the subscriber's explicit "yes, deliver here" — split out so that the subscriber controls the delivery scope independently of how broad their subscribe authorization happened to be.

---

## 5. Per-extension coherent capability checklist

The system extension specs each carry a "Coherent Capability" sub-section near the top. This is the operator's view of those.

### 5.1 Continuation

**Load-bearing type:** `system/continuation`, `system/continuation/join`. The `dispatch_capability` field authorizes the future dispatch when `advance` fires.

**Create op:** `system/continuation:install` (CT1). Validates that the EXECUTE author is **in** the `dispatch_capability`'s authority chain (CT2 — the in-chain check, §3.1; *not* a chain-root check, EXTENSION-CONTINUATION §3.1a), persists the cap and chain, then writes the continuation under the handler's grant. (Cross-peer, the dispatch_capability is additionally B-rooted with grantee = the dispatching host peer — EXTENSION-CONTINUATION §4.2 case 3; the three slots, §3.)

**Grant pattern:** application-level callers get `system/continuation:install`. They do not get `system/tree:put` on `system/continuation/suspended/*`.

**System case:** system handlers (e.g., the inbox handler creating continuations for delivery routing) call `install` themselves; the EXECUTE author is the local peer identity and is in-chain as granter, so the in-chain check passes by construction (the local case where root == in-chain granter == grantee collapse — §3 "The three slots").

**Sub-identity case:** delegated sub-identities calling `install` directly are naturally rejected — their identity isn't a granter in the dispatch_capability chain. Sub-identities do not set up deferred dispatches; they invoke handlers that may internally create continuations under the handler's own authority.

### 5.2 Subscription

**Load-bearing type:** `system/subscription`. The `deliver_token` field authorizes future async deliveries.

**Create op:** `system/subscription:subscribe`. Runs `grants_access` (existing scope check) AND the in-chain check on the deliver_token (SB1).

**Grant pattern:** application-level callers get `system/subscription:subscribe`. They do not get `system/tree:put` on `system/subscription/*`.

**Read access:** if applications need to enumerate their own subscriptions, grant read on `system/subscription/*` separately — that's not load-bearing.

### 5.3 Role

**Load-bearing types:** `system/role/{context}/assignment/{peer_id}/{role_name}` (triggers token derivation; multi-role per (peer, context) supported per role v1.2 R6) and `system/role/{context}/excluded/{peer_id}` (adds denials). Role *definitions* themselves are not load-bearing — they're templates.

**Create ops:** `system/role:assign`, `system/role:unassign`, `system/role:exclude`, `system/role:unexclude`, `system/role:re-derive`. The assign and re-derive handlers run RL2 — the assigner's caller capability covers the derived grants, via V7 §5.6's `is_attenuated`.

**Grant pattern:** application-level callers (or admin tooling) get the role handler ops. They do not get `system/tree:put` on assignment or exclusion namespaces.

**Storage path for derived tokens:** `system/capability/grants/role-derived/{context}/{peer_id}/{token_hash}`. `unassign` deletes the assignment AND the persisted tokens at this path; `is_revoked` finds the missing entry and rejects on the next dispatch. (Role v1.2 R4.)

**Three-layer exclusion:** (1) Token revocation is the primary mechanism — `unassign` and `exclude` delete derived tokens from the storage path, and V7 standard `is_revoked` rejects them at next dispatch. (2) Block new derivation — `assign`, `re-derive`, and the SDK L0 bootstrap path consult the exclusion subtree before minting tokens. (3) Optional handler-level checks — context-aware handlers MAY consult the exclusion subtree at request time. Exclusion is bounded by these three layers, not a magic universal denial. (Role v1.2 R7.)

**Why RL2 is stricter than the in-chain check:** role-derived grants don't have a single chain — they're synthesized from the template. The right invariant is "the assigner can already do everything the assignment will let the assignee do." Without it, an assigner could mint tokens broader than their own.

### 5.4 Compute

**Load-bearing types:** `system/compute/subgraph` (carries `installation_grant`, `authorized_data_hashes`) and any compute expression containing literal `compute/apply.capability` references.

**Create op:** `system/compute:install`. The install audit walks the expression graph and runs the in-chain check on every static-literal `compute/apply.capability` it finds (CP1). Dynamic capability values (computed at eval time from `lookup/scope` etc.) are covered separately by the runtime dual-check (COMPUTE-APPLY-RESOURCE-CEILING F1-F5).

**Grant pattern:** application-level callers get `system/compute:install` and `system/compute:eval`. They do not get `system/tree:put` on compute process paths.

**Check ordering:** the in-chain check runs before resource-coverage (cheaper, more fundamental). The error codes differ (`embedded_cap_unauthorized` vs `permission_denied`), so callers can tell why a request was rejected.

### 5.5 Other extensions

Extensions without load-bearing capability fields (revision, transaction, history, query, clock, content, network) don't need a Coherent Capability section because there's nothing for R1 to check. They still expose handler operations and benefit from R0 — grant the operations, not raw `tree:put` to the handler's data namespace, for the usual reasons (validation, audit, future evolution).

---

## 6. Admin grants: coherent vs unconstrained

A common question: "the admin or peer-owner has all the authority anyway — do R0 and R1 apply to them?"

Both modes are valid. They're for different situations.

**Mode A — Admin with coherent constraints.** Admin still routes through handler create operations even though they could bypass them. Concretely: admin gets a broad grant on `system/continuation:install`, `system/subscription:subscribe`, `system/role:assign`, `system/compute:install`, and so on, but does NOT get a wildcard `system/tree:put` on the system handlers' namespaces.

Why you'd want this:

- *Configuration clarity.* When you read the grant set, you can see which capabilities the admin component is using and which validation paths fire.
- *Compromise containment.* If an admin token leaks, the attacker still can't embed arbitrary capabilities — the in-chain check rejects them.
- *Composability.* When admin code delegates to sub-components, the sub-components inherit the same R0/R1 discipline.
- *Audit and history.* Operations leave handler-specific traces (assign-request, install-request) instead of opaque tree puts.

This is the recommended default for production deployments.

**Mode B — Full admin, no constraints.** A single grant entry with wildcards on all four scope dimensions, including `system/tree:put` on every namespace. The admin can do anything, by any path.

When this is appropriate:

- *Bootstrap.* Before the capability and handler systems are fully wired, you need raw tree access. The peer-owner is operating as the peer at this point; there is no separate validating authority.
- *Maintenance and recovery.* Repairing inconsistent state, manually editing entities the system can't reach via its own ops, debugging.
- *Trusted single-developer environments.* A local development peer where one person is everything; ceremony has no benefit.
- *Single-machine scripting.* Quick automation against your own peer where the peer-owner identity *is* you.

What you give up: every protection in §3. Operations that violate handler invariants succeed silently. The system can't tell whether an entity was created via `install` (validated) or via raw `tree:put` (not). History records the write either way; semantic correctness becomes a thing you preserve by hand.

**The key distinction.** Mode A is "I have the authority but I still go through the front door, because the front door has guards I want firing." Mode B is "I am the building owner, and right now I'm walking through the wall." Both are sometimes correct. Don't confuse them: a Mode B grant is not a more powerful Mode A grant — it's a categorically different posture. SDK access levels (§7 below) make the boundary between L0 (Mode B for store access) and L1 dispatched ops (Mode A or narrower) explicit so the choice is visible in code.

**SDK guidance.** The SDK's Level 0 direct-store API (SDK-OPERATIONS §2.7) is the in-process equivalent of Mode B for the peer owner. It's well-defined, every L0 mutation still fires the emit pathway (history, subscription, query all see the write), but no dispatch and no capability validation happen. L0 is **for the peer owner's code only.** It MUST NOT be exposed to delegated actors, handler bodies running under attenuated grants, or compute expressions evaluating under installation grants. Those contexts use the dispatched API and inherit R0/R1 for free.

---

## 7. Grant progression — from open to delegated

There's a natural progression from a fresh peer to a production deployment with delegated identities. Each level is independently useful (SDK-OPERATIONS §11.2A).

**Grant on the resource the handler will act on.** When you write grant entries, the `resources` dimension matches against the EXECUTE's `resource` field at dispatch time — that's where path-acting handler operations carry the path the caller targets (V7 §3.2 path-as-resource convention). For continuation install, role assign/exclude, compute eval/install/uninstall, and `system/handler:register/unregister`, the path the caller authorizes against is the operation's resource target, not a field nested inside params. So a grant entry of `{handlers: ["system/continuation"], operations: ["install"], resources: ["system/continuation/suspended/*"]}` covers any install whose resource target falls under that prefix. Same shape across all the path-acting operations.

**Level 0 — Open access.** One grant entry, wildcards everywhere. Single-developer dev mode.

```
grants: [{
  handlers:   {include: ["*"]},
  operations: {include: ["*"]},
  resources:  {include: ["*"]},
  peers:      {include: ["*"]}
}]
```

This is Mode B from §6. Fine for getting started. Don't ship this.

**Level 1 — Per-handler entries.** Multiple entries, each scoped to specific handlers/operations/resources. First production step.

```
grants: [
  ; Read tree
  {handlers: {include: ["system/tree"]},
   operations: {include: ["get"]},
   resources: {include: ["public/*", "knowledge/*"]}},

  ; Subscribe (the handler op, not the namespace tree:put)
  {handlers: {include: ["system/subscription"]},
   operations: {include: ["subscribe", "unsubscribe"]},
   resources: {include: ["knowledge/*"]}},

  ; Install continuations
  {handlers: {include: ["system/continuation"]},
   operations: {include: ["install"]},
   resources: {include: ["*"]}}
]
```

Configuration-only — change `grants.toml` (or your builder config). No new SDK code.

**Level 2 — Per-identity policy.** Different connecting identities get different grants. Add a grant policy:

```
GrantPolicy := {
  default: [GrantEntry]            ; fallback for unknown identities
  rules: [{
    match:  IdentityPattern        ; peer ID pattern, group membership, etc.
    grants: [GrantEntry]
  }]
}
```

One new SDK call (`set_grant_policy`); same grant entry shape. The role extension (§5.3) is the structured way to bundle grants for L2.

**Level 3 — Runtime delegation.** Application receives a grant and attenuates it for sub-components, downstream peers, or short-lived workers. Uses `delegate_grant`, `revoke_grant`, `inspect_grants`. Each child grant entry MUST be covered by a parent entry (V7 §5.6 attenuation rule — no escalation). Children inherit all parent excludes and may add more.

The progression is independent of R0/R1. At every level, the grants you hand out should target handler create operations, not raw `tree:put` to their namespaces.

---

## 8. Pitfalls and antipatterns

### 8.1 Broad `tree:put` on a system handler namespace

```
; ANTIPATTERN
{handlers: {include: ["system/tree"]},
 operations: {include: ["put"]},
 resources: {include: ["system/continuation/suspended/*"]}}
```

Allows an actor to fabricate continuation entities with arbitrary `dispatch_capability` references. Even if they don't currently hold a useful capability, they can scan the content store, find one (an admin's, the system handler's), embed its hash, and trigger advance later. R1 doesn't fire because the handler isn't involved.

```
; COHERENT
{handlers: {include: ["system/continuation"]},
 operations: {include: ["install"]},
 resources: {include: ["*"]}}
```

Routes through the install operation. Chain-root rejects any capability the actor isn't in the authority chain of.

### 8.2 Confusing the three subscription capabilities

- Sending the caller cap as the deliver_token. Wrong: deliver_token must be the subscriber-rooted token authorizing notifications, not the subscribee-rooted access token.
- Expecting deliveries to use the caller cap's scope. Wrong: the engine wields the deliver_token, not the caller cap. They have separate scope.
- Granting `system/subscription:subscribe` and assuming notifications "just work." Subscriber still has to issue and supply a valid deliver_token whose scope covers their inbox URI.

### 8.3 Embedding a capability you only hold by delegation

Holding `K_admin` because admin delegated it to you for some narrow purpose does not mean you can install `K_admin` as a continuation's `dispatch_capability`. R1 rejects this unless your own identity appears as a granter in `K_admin`'s chain. The right pattern is to issue your own attenuated capability (which then chains through your identity) and embed that.

**Application-level recurrence in SDK language-native handlers.** This same antipattern recurs one layer up when a language-native handler accepts a capability hash from its EXECUTE input and embeds it in an entity the handler creates (a continuation it installs, a subscription it registers, a role it assigns, a compute subgraph it installs). The system-level R1 check on the underlying handler operation runs against the handler's identity — not against the *original* caller. If the handler embeds caller-provided caps without an independent chain-root check, a caller can launder a cap they don't own through the handler's authority. The handler **MUST** validate chain-root itself before embedding caller-supplied caps; the SDK exposes the primitive at the handler context (see SDK-OPERATIONS §11.3 "Handler-context chain-root primitive"). Entity-native handlers are not subject to this — the compute install audit validates static-literal cap references at install time.

### 8.4 Mixing role definition with role assignment

Role definitions (`system/role/{ctx}/{name}`) are templates. Direct `tree:put` to define roles is fine — you might write definition entities at admin tooling time, in config, or via admin scripts.

Role assignments (`system/role/{ctx}/assignment/*`) and exclusions (`/excluded/*`) are load-bearing — they trigger token derivation or denials. These go through `system/role:assign`/`:exclude`. RL2 enforces that the assigner's authority covers what gets minted.

A grant that lets an actor write role definitions but not assign them is sensible (admins manage the menu; assignment authority is delegated narrowly). A grant that lets an actor write assignments via `tree:put` is the antipattern — they can mint tokens broader than they hold.

### 8.5 Forgetting chain reachability

R1 walks the embedded cap's authority chain. If any link is missing from both the EXECUTE envelope's `included` map and the local content store, the operation fails with **404 `chain_unreachable`** (V7 §5.5).

When testing, you'll see this if you serialize a request without including the parent caps. The fix is including the chain in the envelope. The install/subscribe handlers persist the chain on success, so subsequent operations work without re-shipping it.

### 8.6 Treating L0 store access as "the same as admin"

L0 is the peer-owner's prerogative within the local process. It's not a way to act as admin from a delegated context. The SDK MUST NOT expose L0 to handler bodies, compute expression evaluation, or any code running under a delegated grant. If your application code can reach L0, your application code is operating as the peer.

---

## 9. Reference

### 9.1 Status codes

| Code | When |
|---|---|
| 200 | Operation succeeded |
| 400 | Malformed request, missing required fields |
| 403 `permission_denied` | Capability scope does not cover the request |
| 403 `embedded_cap_unauthorized` | R1 in-chain check failed (writer not a granter anywhere in the cap's authority chain) |
| 403 `assigner_authority_insufficient` | RL2 — assigner's cap doesn't cover role-derived grants |
| 403 `deliver_token_insufficient` | Subscription deliver_token scope doesn't cover the inbox URI |
| 404 `chain_unreachable` | Embedded cap's authority chain has a missing link |
| 404 `dispatch_capability_not_found` | Continuation install — the referenced cap entity isn't in store or envelope |

### 9.2 Helpers (V7 §5.5)

| Helper | Purpose | Used by |
|---|---|---|
| `collect_authority_chain(cap, resolve_fn)` | Walk full chain from cap to root or error | The shared chain-walk primitive — every authority traversal uses this |
| `verify_capability_chain(cap, included, local_peer_id)` | Full per-level dispatch validation | Standard EXECUTE verification (V7 §5.2) |
| `is_revoked(cap, ctx)` | Walk chain to root, check tree binding | Dispatch-time revocation, reactive re-eval |
| `check_creator_authority(cap, writer_identity, ctx)` | Chain-root identity check; returns `(found, chain, error)` | R1 implementations: continuation install, subscription subscribe, compute install audit |

### 9.3 Spec pointers

- V7 §5 — Capability system (verification, revocation, attenuation, chain inclusion)
- V7 §5.5 — Delegation chain verification, chain-walk primitive, `check_creator_authority`
- V7 §6.3 — System tree handler, kernel-vs-handler principle (V1)
- V7 §6.8 — Handler authority model
- EXTENSION-CONTINUATION §1.1 — Coherent Capability; §3.2 — Install operation; §4 — Dispatch capability model
- EXTENSION-SUBSCRIPTION §1.1 — Coherent Capability; §1.2 — Three capabilities table; §3.1 — Subscribe handler and SB1
- EXTENSION-ROLE (post-v1.2) §1.3 — Coherent Capability; §4 — Role handler, RL1/RL2 (`is_attenuated`-based), R6 multi-role path, R4 storage path; §5 — Three-layer exclusion model
- EXTENSION-COMPUTE §1.1 — Coherent Capability; §3.3 — Audit walk and CP1
- SDK-OPERATIONS §2.7 — Access Levels (L0/L1/L2); §2.7.2 — Kernel-vs-handler principle; §11.2A — Grant Progression
- PROPOSAL-COHERENT-CAPABILITY-AUTHORITY (implemented) — full rationale, attack walkthroughs, test vectors

---

## 10. Privacy + cross-peer observability

Per `GUIDE-INSPECTABILITY.md` v1.2 §9 #4:

| Surface | Classification | Rationale |
|---|---|---|
| `system/capability/token` (content store entity) | **sensitive** | Bearer instrument. Token content + signature material let a holder authorize EXECUTEs. Cross-peer leak via inspect = capability escalation. |
| `system/capability/grants/{pattern}` (grant entity body) | **sensitive** | Hash-linked to signature material; leaking the body enables targeted lookup of the signature. |
| `system/capability/grants/{pattern}/signature` (raw ed25519 signature) | **sensitive** | Direct key-material exposure. The concrete L3-app-attack surface identified by the inspectability-baseline security audit (§5.1). |
| `system/capability/grants/role-derived/{context}/{peer_id}/{token_hash}` (per V7 §5 line 213) | **sensitive** | Role-derived token storage path; same reasoning as token body. |
| `system/capability/request`, `system/capability/grant`, `system/capability/revocation` (in-flight types) | **capability-controlled** | Scope-revealing operational state; visible to dispatcher chain but not stored. |

Per §9 #7: capability state has **two distinct cross-peer surfaces:**

- **Chain-bundled (normative, by design):** Authority chains travel cross-peer in EXECUTE envelopes' `included` map per V7 §3.1 + EXTENSION-CONTINUATION §4.3 (cross-peer dispatch capability). This is the only sanctioned cross-peer propagation of capability material. Receiver re-verifies via V7 §5.2.
- **Local-namespace:** Subscription-based propagation of `system/capability/**` MUST be refused. A `system/capability/*` subscription would let a downstream peer harvest signatures + tokens that were never explicitly delegated to it. Subscription handlers SHOULD reject patterns under `system/capability/` regardless of granted scope unless the scope explicitly enumerates a narrower path with operator-class authority.

**L3 application-UX implication.** Per `GUIDE-INSPECTABILITY.md` v1.2 §5.2 (promoted to required), inspect tooling and L3 surfaces MUST NOT render token bodies, grant entities, or signature material outside the operator role. Default renderer behavior:
- **sensitive** entries NEVER render to L3 surfaces; even operator-class surfaces require explicit "show key material" toggle with audit logging.
- **capability-controlled** entries render to operator-class L3 surfaces only; non-operator L3 surfaces show count + path family, not entity body.

Per the privacy and cross-peer observability audit (§2.1).

**Operator-class authority (v1.2.1 — definition).** Subscription refusal contracts and other "operator-class authority required" gates throughout the spec corpus (GUIDE-INSPECTABILITY v1.2 §3.4.1; EXTENSION-CONTINUATION §6.5; EXTENSION-SUBSCRIPTION §7.4; EXTENSION-INBOX §11; EXTENSION-REVISION §12; GUIDE-CAPABILITIES §10 above) reference a structural property of capability chains. The property is defined here so impls have a consistent test:

A capability chain is **operator-class** with respect to a target path family iff:

1. The chain roots at the peer's L0 bootstrap identity (or equivalent operator-controlled identity per deployment policy), AND
2. EITHER the root grant directly authorizes the operation on the target (single-hop operator grant), OR every intermediate sub-delegation's `resources` field **explicitly enumerates the target path family as a resource pattern** (not via `*` wildcard match alone).

**Wildcard tokens that happen to match a sensitive prefix via `*` are NOT operator-class.** Only explicit enumeration counts. This is the structural test that lets impls enforce the subscription refusal contracts without inventing new capability machinery — walk the cap chain, verify each link explicitly names the target path or its prefix as a literal resource pattern, refuse if any link is wildcard-only.

Implementation pattern (language-agnostic pseudocode):

```
fn caller_has_operator_class_for(ctx, target_pattern):
    chain = ctx.capability_chain()
    if not chain.roots_at_l0_bootstrap():
        return false
    for link in chain.links():
        if not any(explicitly_enumerates(r, target_pattern) for r in link.resources):
            return false
    return true

fn explicitly_enumerates(resource_pattern, target):
    # Matches iff pattern is literally the target, or a literal prefix
    # of the target. Wildcards do NOT count, regardless of match.
    if '*' in resource_pattern:
        return false
    return resource_pattern == target or
           target.startswith(resource_pattern + '/')
```

**Why this shape:** matches the threat model. Wildcard tokens proliferate via delegation chains; explicit grants don't. Sensitive prefixes (`system/capability/**`, `system/runtime/**`, `system/continuation/**`) should require operator-explicit grants, not accidental wildcard match from intermediate delegations. Uses existing `resources` field on V7 §5 grant scope — no new capability machinery; checkable in linear cap-chain walk. Symmetric to V7 §5.6 `is_attenuated`: "operator-class" is a property of the chain's *explicitness*, not a new authority tier.

**Implications across the spec corpus:**
- **GUIDE-CAPABILITIES §10** (above) — capability surface refusal contract uses this definition.
- **GUIDE-INSPECTABILITY v1.2 §3.4.1** — `system/runtime/**`, `system/continuation/**` subscription refusal uses this definition.
- **EXTENSION-CONTINUATION §6.5, EXTENSION-SUBSCRIPTION §7.4, EXTENSION-INBOX §11, EXTENSION-REVISION §12** — each invokes "operator-class authority" for prefix-refusal exceptions; this definition is what those subsections check against.
- **Impl-side enforcement** — both extension-level (subscription engine; e.g., entity-core-rust `extensions/subscription/src/lib.rs:196`) and SDK-tier (e.g., `entity-sdk` subscription wrappers per egui-entity-core-rust §4.4) check this. Defense-in-depth: SDK refuses at app-tier; extension refuses at substrate-tier. Either alone is sufficient; both is the audit-recommended posture.

Per the privacy and cross-peer observability audit (§2.1) and the synthesis of core-rust and egui feedback (§4.1).
