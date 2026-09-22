# The operation surface — the write path is the cost, and the coordination is bilateral

**Status:** EXPLORATION (2026-09-13). **Three questions worked against landed text:** *what handlers, operations and entities have to be written to get this off the ground* · *what the coordination surface is for adding and removing people* · *how fracturing is avoided.* **Plus the adjudication of a seven-finding implementation packet that changes one of the assembly's headline claims.**

---

## §0 Why this document exists

**The assembly has been described four times in grants, namespaces and trees, and never once in
operations.** That is a real omission and it hid three things:

1. ⭐⭐⭐ **The write path is where the cost is, and nobody had priced it.** Publishing a claim mutates
   the tree, which moves the root, which obliges a republish inside 30 seconds, which re-signs. **Every
   claim publication costs a root republish** — and that turns paging from a reader-side nicety into the
   thing that makes *expiry* affordable at all.
2. ⭐⭐ **The genuinely new surface is one handler and two entity types** — because the *adoption* side
   reuses an entity that already exists.
3. ⛔⭐⭐⭐ **The chain this composes onto has never executed in any implementation**, and the one
   artifact that would change that is named in the spec as *"owed and unscheduled."* **That reframes what
   this proposal is.**

---

## §1 The handlers and operations — what exists, and what has to be written

### 1.1 What already exists and is reused unchanged

| handler | operations used | role here |
|---|---|---|
| `system/tree` | `get` · `put` | read adopted tables (local, cached remote data); write own claims |
| `system/content` | `get` · `ingest` | the outbound fetch; landing the bytes |
| `system/registry` | `resolve` · `invalidate-cache` · `set-resolver-config` | the substrate this is a backend of |
| `system/substitute/sources` | `consult` (in-process) · `system/substitute/<type>:try` | the miss-path chain and its convention dispatch |
| `system/capability` | `request` · `delegate` · `revoke` · `configure` | issuing and withdrawing the grants of §3 |
| `system/group` | `form` · `add_member` · `remove_member` · `add_subgroup` · `attest_acting_on_behalf` | the coordination surface (§3) |
| `system/subscription` | cross-peer `subscribe` (§6.1, §6.3 mirror) | keeping an adopted table current |

### 1.2 ⭐ What has to be written — and it is small

```
NEW HANDLER  system/locator
  resolve(subject, hints?)          → LocatorResult      ; the read
  publish(claim)                    → ()                 ; assert one
  retract(claim_ref)                → ()                 ; withdraw before expiry
  invalidate-cache(subject | null)  → ()                 ; mirrors the registry's own second op

NEW ENTITY TYPES
  system/locator/claim              ; one assertion, individually signed
  system/locator/page               ; the paging carrier — `page` equal to its key

NEW CONVENTION HANDLER
  system/substitute/locator:try     ; plugs into the §6 dispatch pattern, no substrate change

NOT NEW — reuse
  system/substitute/source with substitute_type: "locator"
```

⭐⭐ **The adoption decision needs no new entity and no new operation.** `system/substitute/source`
already carries `source_peer_id`, an **opaque** `endpoint`, `priority` (ascending), `enabled`,
`expires_at` and `supersedes` — which is precisely *"a table I have adopted, at this precedence, until
this date."* ⇒ ***adopting a table is a `tree:put` of a source entry; dropping one is setting
`enabled: false` or unbinding it.*** **No `adopt` / `drop` operations should be minted.**

⚠ **And one operation is owed that is not ours and not new:** *an operation to ask a peer for a
**prefix fingerprint***. The computation is normative in the tree tier; only the request shape is
missing. **It is already on the docket as an open question and it is what makes reconciliation between
two tables cheap.** Named here because the locator backend is a caller for it.

### 1.3 ⛔ And the artifact that actually blocks everything: the driver

**`EXTENSION-SUBSTITUTE` §9.1, verbatim:**

> *"**no conformance client can enter the §3 chain over the wire in any implementation**, by design…
> All three implementations independently deferred the driver and left a comment saying so at the call
> site."*
>
> *"The **driver** — the SDK closure-fetch / dispatcher tree-walk that would make these wire-reachable —
> is named here as **owed and unscheduled**, and it is *not* this extension's to define. **It lands with
> the consumer-side work**; when it does, these vectors convert to wire-driven and this subsection
> retires."*

⇒ ⭐⭐⭐ ***The locator backend IS that driver.*** **That is a larger and more accurate claim than *"one
backend and three clauses"*:** it is the consumer-side work that makes an entire landed extension's
conformance surface reachable and **retires §9.1.**

⛔ **And it demolishes a claim the assembly made four revisions running.** *"The entire rest of the chain
applies unchanged"* is an **inheritance claim over code that has never run**: §9.1 records that this
extension had *"zero behavioural checks in any of the three implementations for its entire life — and it
read as covered."* **Reported by implementation measurement: one core hardcodes the claimed source to
`None` and does not install the orchestrator in its peer binary; another installs it but its only caller
is a test; the third says so in-source.** *(Peer-reported against their own trees, adopted; not
independently re-measured here.)*

---

## §2 The write path — which is where the cost is

**"Is this just tree gets?" No, and the write side is the expensive one.**

```
READ    locator:resolve → tree:get (local, adopted tables)
                        → substitute consult (in-process)
                        → content:get at /{candidate}/…      ← the only outbound call
                        → content:ingest
WRITE   locator:publish → validate
                        → tree:put  the claim entity
                        → tree:put  the signature at the invariant pointer
                        → tree:put  the page binding
                        → ROOT MOVES → republish → re-sign → seq++
```

⭐⭐⭐ **Every published claim costs a root republish**, because the serving convergence rule binds the
publisher's republish cadence at 30 seconds and the root is a signed, sequenced object.

⇒ **Three consequences, and the third is new:**

1. **Batch.** A hundred claims published together cost one republish; published singly they cost a
   hundred. **`publish` should take a set, not a claim** — a schema decision falling directly out of the
   write path and invisible from the read path.
2. **Sign per claim, bind per page.** The authorship instrument signs **entries**, so each claim carries
   its own detached signature; the *page* is what the tree binds.
3. ⭐⭐ **Paging is what makes EXPIRY affordable, not just what makes reading affordable.** A claim
   expires, so it must be re-asserted. **In a monolithic set every re-assertion rewrites the whole
   object — for every reader, on every cycle.** With key-addressed pages, re-assertion rewrites **one
   page**. ⇒ ***expiry and monolithic membership are actively incompatible, and that is a sharper reason
   for paging than reader cost.***

> ⭐ **Measured by an implementation on the sibling case and it lands here unchanged:** at roughly
> **119 bytes per entry, 50,000 holders is ~5.9 MB fetched whole.** Under expiry, that whole object is
> rewritten every refresh cycle. **This answers the assembly's standing *"pruning cost at aggregator
> scale"* question by shape rather than by measurement: the monolith is the wrong object and the
> question partly dissolves.**

---

## §3 The coordination surface — adding and removing people

### 3.1 Membership and authorization are two separate acts

**This is the thing to get right, and the group extension already separates them.**

```
ADD     system/group:add_member          ; K-of-N signed per the group's governance
        system/capability:request        ; …and SEPARATELY, a grant is issued
             or :delegate                ;    (attenuated from an admin's own cap)

REMOVE  system/group:remove_member       ; the membership record
        system/capability:revoke         ; …and SEPARATELY, the authority
```

⚠ **Adding a member entry grants nothing.** The member record is a *statement about membership*; the cap
is *the authority*. **Two operations, and a deployment that conflates them will produce members who
cannot read and ex-members who still can.**

⭐ **And the group extension states the discipline for the write path too:** direct `tree:put` into
`system/group/{group_id}/members/*` is *permitted* — the kernel exposes `tree:put` by design — but
*"bypasses the group handler's validation"*, so **application grants SHOULD cover
`system/group:add_member` / `:remove_member` / `:form` rather than raw `tree:put`.** *That is the general
rule for every managed namespace, not a group quirk.*

### 3.2 ⛔ Removal is the hard half, and the honest answer is short TTLs

**Revocation is a tree operation:** `put(path, null)` unbinds the cap's root, plus a marker at
`system/capability/revocations/{root_hash_hex}`; a chain whose root is unbound is revoked *"regardless of
whether the chain is cryptographically valid."* ✅ **Both path-bound and wire-only caps converge on one
`is_revoked` check.**

⚠ **But a cap already issued and cached outlives the decision until it is checked or expires**, and the
group extension names the residue precisely — *"outstanding multi-sig caps signed before a constituent's
compromise inherit the compromise"* — with the mitigation stated as a **SHOULD: short cap TTLs, to bound
the window.**

⇒ ***Removal is eventual, and its latency is a TTL you chose.*** **That is the honest statement, it is
already normative, and a design that implies otherwise is overclaiming.**

### 3.3 What whoever manages a deployment actually touches

| to… | operations |
|---|---|
| stand up a group | `system/group:form` (L0 bootstrap on the founder's agent) |
| add a person | `:add_member` + `capability:request`/`:delegate` |
| remove a person | `:remove_member` + `capability:revoke` |
| let someone act for the group | `:attest_acting_on_behalf` / `:revoke_acting_on_behalf` |
| nest an org | `:add_subgroup` — each child is *"a separate `system/group:form` call with its own quorum, identity stack, and initial members"* |
| change who may read what | re-issue caps with different `resources` scopes — **no membership change at all** |
| publish where things are | `locator:publish` (batched) |
| decide whose tables to believe | `tree:put` a `substitute/source`, `priority`, `enabled` |

⭐ **Note the last three rows: audience, location and adoption are each changed WITHOUT touching
membership.** *Four independent axes, four independent operations — which is what makes the model
manageable rather than a single tangled ACL.*

---

## §4 ⭐⭐ Managed groups are organizational, not consensus — and the corpus already says so

**The design instinct is right and it is written into landed text.**

- **A group is an identity, not an agreement.** Its quorum is **governance *within* the group** —
  K-of-N over constituents *"drawn from members/admins"* — **not consensus across the network.** Nothing
  global agrees to anything.
- ⭐⭐⭐ **Federation is bilateral by MUST, with hub sync as an opt-in convenience.** The group
  extension's federation section states it outright: ***"per-member individual sharing as the
  foundation (always supported, MUST); bulk sync as opt-in convenience."*** ⇒ **many hubs, each
  optional, over a substrate of pairwise relationships that always work without them.**
- **Nesting is structural, not authoritative.** `add_subgroup` links identities that each keep their own
  quorum, controller and agents. **A parent does not gain authority over a child's keys.**

⇒ ***The topology is: many centres, each an identity, each with its own governance, joined pairwise —
and a hub is a peer that many parties happen to have adopted, never a party anybody must consult.***
**This is the same shape the locator aggregator has, which is not a coincidence: an aggregator is a hub,
and its authority is exactly the sum of the adoption decisions pointed at it.**

---

## §5 How fracturing is avoided — four mechanisms, three landed

**The worry is right: an identity-per-audience model multiplies identities, and multiplied identities
are where divergence lives.**

| fracture | what stops it | state |
|---|---|---|
| **two parties mean different things by one type tag** | one shared vocabulary at the application tier, with a gate that reports two seats emitting under one prefix with **zero tags in common** | ✅ landed + gated |
| **two tables disagree about where something is** | conflicts are **surfaced, never silently picked** — fail-closed by default, explicit pin to override | ✅ landed |
| **N identities become N islands** | names resolve to identities through the registry; `add_subgroup` links them structurally; **adoption is what joins them, and it is a decision rather than a topology** | ✅ landed |
| ⛔ **N identities, each publishing its own set object, with no shared shape for a GATHERED set** | ***nothing — and this is the real gap, see §6 finding 4*** | ⛔ **open** |

⇒ **Three of the four are already mechanical. The fourth is the one the assembly mis-scored as free.**

---

## §6 The implementation packet, adjudicated — six adopted, one corrected

**Seven findings against the assembly from a shipped implementation. They are better than the
document's own audit, and one of them changes a headline claim.**

| | finding | verdict |
|---|---|---|
| **1** | the chain has never executed; the locator backend IS §9.1's *"owed and unscheduled"* driver | ⛔⭐⭐⭐ **ADOPTED** — verified verbatim in §9.1. **Reframes the proposal** (§1.3) |
| **2** | §10 already groups three of the six seams and names the common dependency | ✅ **ADOPTED** — §10 scopes out *wildcard/bare-hash substitution*, *transitive substitute following* and the Mode A claimed-source field, all on *"when a driver emerges."* **The assembly re-derived §10's own grouping** |
| **3** | the cardinality seam is answered by `REGISTRY` §4.1.1's own second sentence, and is the same item as the aggregator question | ✅ **ADOPTED** — verbatim: *"the alternative (surface all hits, let the caller choose) **is the aggregator/federation case, deferred per §8.2**. Caller-side multi-hit awareness lives at the aggregator layer when Mode A ships."* **Two seams collapse into one** |
| **4** | the layer scored *"nothing new"* is the one with the unwritten artifact behind it | ⛔⭐⭐⭐ **ADOPTED, and it is the sharpest structural finding** — see below |
| **5** | the referral's termination dependency *"returns zero hits in the section it cites"* | ⚠ **PARTLY WRONG, and corrected rather than adopted** — see below |
| **6** | the locator table still has no paging, nine hours after the same rule became a MUST NOT for the same object, with numbers | ⛔⭐⭐ **ADOPTED** — the requirement was *stated* in the audit and never *applied* to the schema. §2 now carries it, and the numbers answer the standing cost question by shape |
| **7** | the referral is corroborated by a shipped product, with a production incident showing the alternative | ✅ **recorded as peer-reported corroboration**, not independently verified here |

### 6.1 ⛔⭐⭐⭐ Finding 4: closure does not close the layer the aggregate lives in

**`SYSTEM-DATA-EXCHANGE` §1.2 — *"Closure binds the entry layer. It does not bind the walk."***

| layer | closed? |
|---|---|
| **entry** — decode, obtain the signature, verify, attribute, render | **YES**, one code path |
| **set** — ***how a reader finds the entries*** | ⛔ **NO.** *"An author's own set and a gatherer's gathered set are different shapes, **and they are allowed to be**"* |

⛔ ***A locator table IS a set-layer object.*** The assembly scored the aggregation row *"nothing new"* on
the strength of closure — **and closure explicitly declines to close the set layer.** ⇒ **the aggregate's
shape is undesigned on purpose, and that is the unwritten artifact behind the row.**

⚠⚠ **And §1.2 carries a MUST NOT that a naive aggregator violates by construction:**

> **[MUST NOT]** *An implementation **MUST NOT** publish, under its own namespace, an author's own
> set-layer object over content that author did not place there.*
>
> *…because the authorship instrument signs **entries** and not sets, nothing would authenticate it.*
> ***"A reader following the unqualified sentence builds a forgery and every byte of it verifies."***

⇒ ⭐⭐ **The rule this yields is sharper than the audit's *"carry, do not merge"*:** *an aggregator
publishes **its own gathered set**, in a shape distinguishable from an authored set, carrying each
entry's original signature. It may never publish a reconstruction of somebody else's set under its own
name.* **The audit got the mechanism right and the object type wrong.**

### 6.2 ⚠ Finding 5, corrected — the citation resolves, and the narrower worry is real

**The claim was that the referral's termination dependency returns zero hits in the section cited.**
⛔ **It resolves: `EXTENSION-QUERY` §1.2's own *not specified* list contains *"Cross-peer query handler
(Level 3, future — types defined here are reusable)."*** The citation is correct.

✅ **But the narrower point stands and is worth keeping:** that line is a **one-line deferral of a
handler**, not a specification of the bounded walk. **The hop budget, the visited-set and the
progressive partial results exist only in the frozen pre-split archive** — confirmed: no
fan-out / visited-set / expand vocabulary appears anywhere in the extension. ⇒ **the dependency is real
and its only specification is in an archive that does not publish**, which is a materially different
problem from a dangling citation and a worse one to leave implicit.

---

## §7 What this changes, and the build order

**Revised claim about what the assembly is:**

> ⛔ *Was:* one registry backend plus three clauses in a landed extension.
> ⭐ **Is:** ***the consumer-side driver that a landed extension names as owed and unscheduled*** — which
> makes that extension's conformance surface wire-reachable for the first time, retires its §9.1
> carve-out, and carries a registry backend and a set-layer format as its parts.

**Build order, with the blocking item first:**

1. ⛔ **The gathered-set format** (§6.1) — paged, distinguishable from an authored set, entry-signed.
   **Nothing else can be specified over an undesigned object.**
2. **`system/locator/claim` + `page`**, with `publish` taking a **set** (§2).
3. **`system/locator:resolve`** and the `substitute/locator:try` convention handler.
4. **The consult-cap's locator form** — the grant, per the dimension analysis already done.
5. **The prefix-fingerprint operation** — not ours, already on the docket, and this is a caller for it.
6. ⚠ **Conformance checks that can actually fail** — a declared exclusion plus *an executed mutation*,
   because the cost of skipping that is measured: **zero behavioural checks for an extension's entire
   life, reading as covered.**

**What is no longer owed:** the cardinality seam (answered), and the *"is a third-party claim
admissible"* question (deployment policy). **Two seams down, one promoted to first.**

---

## Document history

- **2026-09-13** — created. Works the operation and coordination surfaces, prices the write path, and adjudicates a seven-finding implementation packet — six adopted, one corrected.
