# PROPOSAL — the share-as-grant cluster: what a share is, where it lives, and which slot carries the audience

**Status:** RULED 2026-08-17 — Q2 / Q6 / Q7 answered. Fold = author `APP-CONVENTION-SHARE`.
**Tier:** applications (L5), with one correction that is core-protocol reading, not new normative text.
**Answers:** `entity-browser-rust` `ROUTING-2026-08-16-g` §Q2/§Q6/§Q7, restated with urgency in
`ROUTING-2026-08-17-comprehensive` §5. Their ranked blocker #2.
**Read at:** browser-rust `ca3c760` · workbench-go `4b34418` · core-protocol `f83c256` · arch `6aab719`

---

## §0 Summary of the ruling

| Ask | Ruling |
|---|---|
| **Q6** — is a Share soundly *a titled `GrantEntry`*? | **Yes on the shape, NO on the audience carrier.** `resources` = what is shared is right and withdrawal-as-revocation is right. **`peers` is not the audience** — it is the network dimension, matched against the *target* peer. See §1 |
| **Q2** — extension-tier share catalog, or application convention? | **Application convention.** `published-root` stays singular and is the *verification anchor*, not a catalog. The convention is authored as `APP-CONVENTION-SHARE` in `specs/applications/`, type tags under **`app/share/*`** — not under any one app's prefix, not under `system/`. See §2 |
| **Q7** — does a group in a `peers` scope resolve at authoring or check time? | **Neither — the fork is posed on the wrong dimension.** Check-time resolution is foreclosed by landed text (F40 + §3067). Group-as-audience today is **per-member minted tokens + `revoke` on leave**; the join-side gap is real, named, and filed. **The frozen §5.2 vector set is NOT reopened.** See §3 |

---

## §1 Q6 — the shape is sound; the audience carrier is a category error

**What is right, and should be built:** a share **is** a titled grant. `resources` says what is shared,
`handlers` + `operations` say how it may be touched, and withdrawal is a real revocation rather than an
unlisting. `entity-browser-rust` reached this from `DESIGN-SHARE-FOLLOW-AND-THE-GRANT-AS-INTERFACE` and it
is the correct instinct: it collapses a second, softer permission system into the real one.

**What is wrong:** the claim that *"`peers` is the audience."* Landed core text says `peers` is the
**network dimension** — which peer a grant may be *used against* — and never the wielder.

`ENTITY-CORE-PROTOCOL.md` §5.2 `check_permission`:

```
target_peer = extract_peer(execute.data.uri, local_peer_id)
...
peers_scope = grant.peers or {include: [local_peer_id]}
if not matches_scope(target_peer, peers_scope, local_peer_id):
    continue
```

and §3.6's dimension table: *"`peers` — Peer scope — which peers the grant applies to. **When absent,
defaults to local peer only** (`{include: [local_peer_id]}` constructed at evaluation time)."* §5.4's
one-purpose-per-field statement is explicit: *"`peers` answers **which peers are in scope**."*

**The audience is the `grantee` slot.** §5.2 step 3 hard-DENYs unless
`hash_equals(capability.data.grantee, execute.data.author)`, and the three-slot note names it:
*"**Grantee (of the leaf)** — the *wielder*: the identity that authors the EXECUTE."*

### §1.1 What the mistake does in practice

Alice shares an album that lives on Alice's peer, with Bob. Under the mistaken reading she mints a grant
carrying `peers: {include: [bob_id]}`. Bob presents it in an EXECUTE to Alice. On **Alice's** peer,
`local_peer_id = alice` and `target_peer = alice` (extracted from the request URI) — so the scope
`[bob_id]` does not match and the request **DENYs**, with `403 capability_denied` and no indication that
the wrong dimension was populated.

**The correct shape is smaller than the wrong one:**

- `grantee` = Bob. That is the audience, one minted token per audience member.
- `resources` = the album prefix. `handlers`/`operations` = how it may be read.
- **`peers` is omitted entirely.** Absent, it defaults to `{include: [local_peer_id]}` = Alice — which is
  exactly right for "my content, on my peer." Populating it is not merely unnecessary; populating it with
  the audience is the defect above.

### §1.2 Why this survived review — and the corroboration

This is a **cross-peer seam equivalence-collapse** in the shape `AGENTS.md` already names. Locally, root,
grantee and sole in-chain granter collapse onto one identity, so a share tested against one's own peer
passes with `peers` populated *any* way. It springs apart only cross-peer. §5.2's own note says so:
*"That collapse is precisely why a model reasoned about only locally silently omits two of the three
slots."*

**The core spec already anticipated this exact confusion, by name.** §6.2's namespace-disambiguation note
(0.8.1, UN-a/F47) exists because two things are both informally called a "peer pattern":

> *"This policy-path `{peer_pattern}` … is a **distinct namespace** from the capability **`peers:` scope**
> IdScope patterns … **They collide only in the informal name 'peer pattern.'**"*

`entity-browser-rust` is building shares as **policy entries** (`policy_entries`, `ShareSync`, derived from
`configure`'s replace semantics) — i.e. at `system/capability/policy/{peer_pattern}`, the per-identity
policy table of `GUIDE-CAPABILITIES` Level 2. **That policy key is a legitimate audience carrier**; the
`peers:` scope inside the grant entry is not. The two were conflated because they share a name.

**Independent corroboration from the oracle:** the conformance vector frozen 2026-08-16 for this dimension
is named `authz_peers_target_from_uri` (`SIGNOFF-2026-08-16-vector-set-final.md`, the P-1/P-2/P-3 trio).
The vector set's own naming records that `peers` is matched against the **target extracted from the URI**.

### §1.3 Ruling

**A share is a titled grant. Its audience is carried by the `grantee` of the minted token, or
equivalently by the `{peer_pattern}` key of a `system/capability/policy` entry — never by the `peers:`
scope, which is omitted for a share of one's own content.**

No spec change is required for this; it is landed text read correctly. What *is* owed is a guide sentence
so the next seat does not re-derive it — see §5.

---

## §2 Q2 — `published-root` is the anchor, not the catalog; shares are an L5 convention

**`published-root` must stay singular, and the reason is in its own text.** `EXTENSION-TREE` §3.3a:

> *"**Two conformant publishers may legitimately publish different extents.** `prefix` is what
> distinguishes them… A consumer MUST read the extent from `prefix` and MUST NOT infer it from the
> publisher's identity or from what it happens to find."*

A published root is the **signed commitment to serve an extent** — the first step of the walk-from-signed-
root threat model. Audience-scoping it, or making it plural, breaks the property that a consumer can read
one signed artifact and know exactly what it covers. Q2 correctly identifies that a product needs N
addressable, audience-scoped publications; that is a **different object** from the extent commitment, and
it does not belong on the same entity.

**The extension tier already declined this surface, explicitly.** `EXTENSION-DISCOVERY` §10, *What this
extension does NOT cover*:

> *"**Inspect-before-grant UI** — fetching a candidate's public manifest / sites BEFORE the user decides
> is **L5 territory**… Substrate stays at find-and-prompt."*

**And the L5 domain exists for precisely the risk Q2 raises.** `specs/applications/CHARTER.md`: the domain
is *"a home for cross-impl, application-layer (L5) conventions — standards that let independent
applications and independent front-ends converge on **one shared format** instead of each reinventing
it."* Q2's stated risk — *"if `entity-workbench-go` grows the same feature it will invent a different
prefix and a different shape, and browser↔go sharing never works"* — is that sentence restated.

### §2.1 Ruling

**Shares are an application convention, authored as `APP-CONVENTION-SHARE` in `specs/applications/`.**
No extension-tier catalog. Three bindings follow from the charter and from landed precedent:

1. **Type tags live under `app/share/*`.** The precedent is `app/embed/{media_type}` (EMBED §3, *"the type
   tag is the dispatch key"*) and `app/site-*` (SEMANTIC-CONTENT-SITE §F-8, *"final"*). **Not**
   `app/entity-browser/share` — a cross-impl convention under one app's prefix is the reinvention the
   domain exists to prevent. **Not** `system/share` — charter discipline #2: *"a convention adds **no**
   kernel features and **no** required SDK surface,"* and `system/` is the substrate namespace.
2. **Format only.** Charter #1: the contract is the entity-type vocabulary — the `{type, data}` shapes and
   their schemas. Rendering, share UX and front-end wiring stay in the application.
3. **Encoding-agnostic.** Charter #6 forbids baking a fixed-width hash form: reference content by the
   self-describing `content-hash` `(format_code, digest)` per V7 §1.2/§1.4. *(The discipline exists
   because EMBED v0.1 shipped `hex33` and three independent reviews missed it.)*

   > **WITHDRAWN as a finding against `entity-browser-rust` — 2026-08-17.** This item originally read
   > *"this one bites today"* and named their `offers/{blob-hex}` and `target { blob | prefix }` as
   > violations. **They do not violate it.** Verified in code at `67057be` / core-rust `f23fb3b`:
   > `Hash::to_bytes` is `push_varint_u32(algorithm)` + digest (`core/hash/src/lib.rs`), `Share::id` is
   > `to_hex()` over those bytes — the §3.5 format-relative invariant-pointer hex, not a fixed width —
   > `share_entity` encodes `blob` as `ecf_bytes(h.to_bytes())`, which is exactly EMBED's
   > `content-hash = bstr`, and decode is width-agile in both directions (`Hash::from_bytes` reads the
   > varint; `digest_len_for_format` derives the length). **`{blob-hex}` is a placeholder in their prose,
   > not a form in their code.** The rule stands for `APP-CONVENTION-SHARE`; the finding against their
   > tree is withdrawn. *(L8, ninth form — a placeholder name in a peer's own routing document. Ours.)*

4. **Mirror paths are the publisher's, and MUST NOT be rewritten** `[added 2026-08-17]`.
   `ENTITY-CORE-PROTOCOL` §1.4: a cached remote namespace is *"structurally identical to that peer's own
   authoritative namespace — **the same paths**, the same types, the same semantics."* So a consumer
   mirroring Bob's shares writes them at **Bob's own prefix** under `/{bob}/…`, verbatim — never at a
   prefix of the mirrorer's choosing. **Path convergence across implementations is therefore neither
   required nor wanted**; the charter's own cited stance is *"paths are convention; the entity graph is
   coherence."* What must converge is the **type tag** (item 1), because that is the cross-peer index
   key — see the note below.

   **How a consumer learns the prefix:** `system/peer/published-root.prefix` (`EXTENSION-TREE` §3.3a),
   which is REQUIRED precisely so that *"two conformant publishers may legitimately publish different
   extents"* and *"a consumer MUST read the extent from `prefix` and MUST NOT infer it."*

> **Why item 1 is load-bearing, corrected upward.** This proposal originally justified `app/share/*` on
> naming precedent. **`entity-browser-rust`'s F3 supplies the real reason and it is stronger:** cross-peer
> aggregation is a `type_filter` query over the universal tree **with no peer filter** (their
> `list_all_sites` / `manifest_type_query`), so **the type tag is the index key.**
> `app/entity-browser/share` would make browser↔go aggregation impossible *even with a perfect mirror*,
> because the query would not match. The ruling was right; the justification given for it was the weaker
> half. Full model: `docs/research/explorations/EXPLORATION-THE-UNIVERSAL-NAMESPACE-AS-THE-MULTI-PEER-DATA-MODEL.md`.

**A convention improvises no protocol** (charter #3). If the share convention needs something the
substrate lacks — and §3 below finds exactly one such thing — it files the ask; it does not invent.

---

## §3 Q7 — the fork is on the wrong dimension, and one branch is foreclosed by landed text

Q7 asks whether a group in a `peers` scope resolves at **authoring** time (expansion) or **check** time
(reference). Per §1, `peers` is not where an audience lives, so the question is restated as it was meant:
**how does a group become the audience of a share?**

There are exactly three carriers, and landed text disposes of them:

**(a) Check-time resolution inside the scope match — FORECLOSED.** `peers` is an
`system/capability/id-scope`, and §3.6/§5.4 pin the matcher: *"`id-scope` values (operations, peers) are
compared as **literal identifiers**… The two MUST NOT be interchanged — a path dimension matched
literally, or an id dimension canonicalized, **is a conformance defect** (it produced a real ALLOW bug)."*
The id-scope pattern grammar (0.8.1, F40) is closed to two wildcard forms and states *"peer-ids are flat,
so peer patterns use a literal id or `*`."* A membership resolution step is strictly more than the
canonicalization F40 already forbids. It would also require the checking peer to hold the group's
`members/` subtree, which `EXTENSION-GROUP` §4.2 makes a per-deployment privacy decision — so two peers
would reach different ALLOW verdicts on the same token. That is the cross-peer ALLOW divergence F40 was
written against.

**(b) A group-shaped policy key — FORECLOSED.** §6.2's `system/capability/policy/{peer_pattern}` path is
*"a content-hash **hex** segment closed to exactly the two forms above (invariant-pointer hex or the
literal `default`)"*, with no partial-prefix matchers — deliberately, to keep the §3018 typo-attack closure
shut. A group identifier is not one of the two forms.

**(c) Group as grantee, delegating to members — NOT AVAILABLE IN V7 TODAY.** This is the shape the model
wants: Alice grants to the group's published handle; the group attenuates to each member; the member is the
leaf grantee and authors the EXECUTE, filling all three slots correctly. It is blocked on one landed
constraint: `system/capability:delegate` v1 is **self-attenuation only** — *"grantee = caller's
authenticated identity always. No grantee field… third-party delegation (grantee = some other peer) is
**deferred to a future amendment**"*, and the handler returns `501 unsupported_operation` when the caller
is not the local peer (v7.63 F1).

`EXTENSION-GROUP` §3.5's `acting-on-behalf-of-attestation` is **not** the missing piece and should not be
reached for: §3.7's two-mechanisms invariant is explicit that *"cap-chain machinery
(`verify_capability_chain`) MUST NOT process them."* It carries the group's voice outward; it does not
admit a member inward.

### §3.1 Ruling

**Group-as-audience is expressed today as per-member minted tokens, revoked on leave.** Concretely:

- The share's audience set is materialized as one token per member, `grantee` = that member.
- **Leaving takes two operations, and which ones depends on the path.**
  `[CORRECTED 2026-08-17 — see §3.1a. The original clause claimed immediate revocation and was wrong.
  SCOPED 2026-08-17 — the first correction then over-generalized the other way; `entity-core-rust` and
  `entity-core-go` caught it.]`
  Removing the member's policy entry — or writing it empty per
  `PROPOSAL-CAPABILITY-EMPTY-GRANTS-AND-POLICY-WITHDRAWAL` — makes every subsequent
  `system/capability:request` from that member fail subset-validation with `403 scope_exceeds_authority`.
  **The two mint paths then behave differently, and both matter to a share:**
  - **`request`-minted tokens are not recallable.** Returned inline with no tree write (§6.2), so the
    granter never holds their hashes and `revoke` — keyed by token hash — cannot name them. **Bounded by
    `expires_at`**, which makes the policy entry's `ttl_ms` the withdrawal latency on this path, and a
    share-design parameter rather than an afterthought.
  - **The §4.4 authenticate-response capability *is* recallable.** It is recorded per-peer at mint time,
    so the granter can name it and `revoke` it immediately.
  **So a complete withdrawal is the policy write plus a `revoke` of the peer's delivered capability.** A
  UI may say *"access ended"* for a connected peer once that revoke is issued; it may say only *"no new
  access"* for tokens already requested. **No step is added to the check path** — that part of the
  original claim holds throughout.
- **It is not the stale-snapshot failure Q7 feared.** Q7's objection to expansion was that *"a stale grant
  is indistinguishable from a current one."* A revoked token is **positively marked**, not merely stale, so
  the two are distinguishable by construction.

**`entity-browser-rust` may ship a `Group` audience on this basis.** It was deliberately unbuilt pending
this answer; the answer is that the mechanism exists and is additive.

**The residual gap is the join side, and it is real.** A member who joins *after* the share is authored
receives nothing until someone re-authors. That is not papered over: it is the genuine cost of (c) being
deferred, it MUST be surfaced in the UI rather than implied away, and it is filed as the ask below.

### §3.1a Correction — the leave side had a second gap, and `entity-browser-rust` found it

`[2026-08-17 — filed as A2, `ROUTING-2026-08-17-c` §2. Their finding, verified here from the landed text.]`

**§3.1's original "leaving revokes" clause named a mechanism the actor in the flow cannot reach.**
`revoke` takes a **token hash**; in the policy flow the granter authors a *policy*, the **member** calls
`request`, and the handler returns the minted token **to the requester**. The granter is never in
possession of the hash, and §6.2's *"`request` … return tokens inline — **no tree writes**"* means no
index can be built from the mint either. The clause was true about `revoke` and false about who could
call it.

**They accepted the ruling and built toward it before catching this** — their withdrawal path shipped on
the reading that withdrawal ends access. It does not; it ends *new* access. **The gap was mine, not
theirs**, and the corrected clause is folded into §3.1 above.

**The full ruling, the two rejected alternatives, and the core-protocol defect the correction exposed
live in `PROPOSAL-CAPABILITY-MINT-TEMPORAL-CEILING-AND-THE-WITHDRAWAL-BOUND`** — including the one that
matters most here: nothing in §6.2 currently bounds a minted token's lifetime by the policy entry's
`ttl_ms` or by the caller's own capability expiry, so *"bounded by TTL"* is a promise the spec does not
yet keep. **A share convention MUST NOT quote a withdrawal latency until that lands.**

### §3.2 The one ask this convention files upward (charter #3)

**`[ASK-CORE]` — third-party delegation.** Group-as-audience without a re-authoring step needs
`system/capability:delegate` to mint for a grantee other than the caller. V7 §6.2 defers it by name. This
proposal does **not** propose it — it records that the L5 convention has now produced a second consumer for
a deferral the core made on its own schedule, which is the evidence a future amendment would need.

**Per L9 (candidate), the deferral is pinned so it expires visibly:**
`(ENTITY-CORE-PROTOCOL.md, §6.2 delegate — "cross-peer delegate is deferred alongside third-party
grantee", entity-core-protocol@f83c256)`. Re-check when that section moves.

### §3.3 The oracle consequence, discharged

`TRIAGE-2026-08-17` A2 flagged that check-time group resolution *"puts a resolution step in the §5.2
`peers`-dimension check path whose vector set was frozen 08-16."* **This ruling excludes check-time
resolution, so the frozen set is untouched.** `authz_peers_target_from_uri` and the P-1/P-2/P-3 trio stand
as signed off; nothing in §1–§3 adds, removes or reinterprets a vector. **Keystone and the cohort census
are unaffected — no re-pin, no re-run.**

---

## §4 What is NOT ruled here

- **The follow verb and its `strategy` field.** browser-rust's revision-free closure form and
  workbench-go's `revision:fetch-diff` form are both legitimate; the discriminator is a convention-design
  question for `APP-CONVENTION-SHARE`, and Form 1's reliable-delivery caveat bears on it.
- **§2.1.3 verification through aggregation** — how a consumer verifies the original publisher's signature
  over an entity in a third party's tree. It is the same question as "what makes a Follow verifiable" and
  it belongs to the RELAY audit (`HANDOFF-2026-08-17-relay-audit-…`), not here.
- **Enforcement.** Nothing here says when `debug_open_grants` comes off. That is the consumer's call.

## §5 Fold — what lands, and where

1. **`specs/applications/APP-CONVENTION-SHARE.md`** — the convention: `app/share/*` type vocabulary, the
   share record, the audience-as-grantee binding, self-describing content-hash references, and conformance
   vectors (charter #5 — *"a convention is not validated until vectors exercise it"*).
2. **`guides/GUIDE-CAPABILITIES.md`** — one paragraph in §3 beside the three-slot note: *the audience of a
   grant is the `grantee`; the `peers` scope is the network dimension and is omitted for a grant over the
   local peer's own resources.* This is the cheap version of §1 and it is where the next seat will look.
   Cross-reference §6.2's existing "collide only in the informal name" note rather than restating it.

Both are additive. Neither touches the locked wire core, the frozen vector set, or any extension spec.
