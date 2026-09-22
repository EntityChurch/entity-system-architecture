# APP-CONVENTION-SHARE — the share record and its audience binding — v0.1 DRAFT

**Version**: 0.1
**Status**: Draft
**Domain:** `applications/` (third member — see `CHARTER.md`). **Charter class:** FORMAT-only.
**Depends:** `ENTITY-CORE-PROTOCOL.md` §3.6 / §5.2 / §5.4 (grant structure, `check_permission`,
pattern matching) · `EXTENSION-TREE.md` §3.3a (`published-root`) · `APP-CONVENTION-EMBED.md` §3
(`content-hash` atom).

> **What this document is.** A share is **not a new mechanism.** It is a titled capability grant plus a
> small record that names it, so that two independent front-ends produce byte-compatible entities and can
> read each other's shares. The convention adds **no** kernel feature, **no** required SDK surface, and
> **no** new authorization path (charter #2, #3). Everything normative about *whether access is allowed*
> already lives in `ENTITY-CORE-PROTOCOL` §5.2 and is untouched here.

---

## 1. The model in one page

A **share** is a titled grant over the sharer's own content, plus a record that makes it addressable and
enumerable.

| Question | Slot | Why |
|---|---|---|
| *What is shared?* | the grant's `resources` scope | §3.6 — the data-path dimension |
| *How may it be touched?* | the grant's `handlers` + `operations` scopes | §3.6 |
| **Who is the audience?** | **the minted token's `grantee`** — or equivalently the `{peer_pattern}` key of a `system/capability/policy` entry | §5.2 step 3 hard-DENYs unless `hash_equals(capability.data.grantee, execute.data.author)`. The grantee is the *wielder* |
| *Which peers may this be used against?* | the grant's `peers` scope — **OMITTED for a share** | §3.6: absent → `{include: [local_peer_id]}`, which is already correct for "my content, on my peer" |
| *What is withdrawal?* | a real revocation, not an unlisting | §5.1 revocation markers |

### 1.1 `peers` is not the audience (normative, and the most-repeated error in this area)

**A share record's grant MUST NOT carry the audience in its `peers` scope.** `peers` is the **network
dimension** — which peer the grant may be used *against* — and `check_permission` matches it against the
peer extracted from the request URI:

```
target_peer = extract_peer(execute.data.uri, local_peer_id)
peers_scope = grant.peers or {include: [local_peer_id]}
if not matches_scope(target_peer, peers_scope, local_peer_id):
    continue
```

**The failure is silent and total.** Alice shares an album on Alice's peer with Bob, and mints
`peers: {include: [bob_id]}`. Bob presents the token to Alice. On Alice's peer `target_peer = alice`, the
scope `[bob_id]` does not match, and the request DENYs with `403 capability_denied` — carrying no
indication that the wrong dimension was populated.

**Implementations MUST omit `peers` entirely on a share of the sharer's own content.** Populating it is
never merely redundant.

> **Why this survives review, and why it is stated here rather than left to the reader.** Locally, root,
> grantee and sole in-chain granter collapse onto one identity, so a share tested against one's own peer
> passes with `peers` populated *any* way. It springs apart only at the cross-peer seam. Two distinct
> things are also both informally called a "peer pattern" — the `system/capability/policy/{peer_pattern}`
> path segment (a legitimate audience carrier) and the `peers:` IdScope inside a grant entry (not one).
> `ENTITY-CORE-PROTOCOL` §6.2's namespace-disambiguation note says they *"collide only in the informal
> name."*

---

## 2. Type vocabulary — `app/share/*`

The cross-impl contract is the **type tag**, not the path. Type tags live under **`app/share/*`**.

**This is load-bearing, not cosmetic.** Cross-peer aggregation is a `type_filter` query over the universal
tree **with no peer filter** — the type tag *is* the index key. A tag under one application's prefix
(`app/entity-browser/share`) makes browser↔go aggregation impossible **even with a perfect mirror**,
because the query does not match. A tag under `system/` is equally wrong: `system/` is the substrate
namespace and a convention adds no kernel surface (charter #2).

| Type | Role |
|---|---|
| `app/share/record` | the share itself — what is shared, under what title, to whom |
| `app/share/audience-entry` | one audience member's binding within a record |
| `app/share/follow` | a consumer's subscription to another peer's share |

### 2.1 Shared atoms

**IMPORTED from `APP-CONVENTION-REFERENCE` §2.1, which is the single home.** Restated here so this
document reads standalone; on any disagreement `APP-CONVENTION-REFERENCE` is the authority.

```cddl
; Self-describing (format_code, digest) per V7 §1.2/§1.4 — the leading varint is the
; content_hash_format and THE DIGEST LENGTH FOLLOWS THE CODE. NOT fixed-width.
; SHA-256 → 33 B is one instance; SHA-384, BLAKE3 and future codes are equally valid.
; Unknown code → unsupported_content_hash_format (V7 §4.7).
content-hash = bstr

peer-id      = tstr                  ; V7 §1.5 Base58 peer-id
tree-path    = tstr                  ; absolute or peer-relative per V7 §1.4
```

**No fixed-width hash form appears anywhere in this document** (charter #6). A convention that bakes one
re-locks the cage the hash-agility arc removed.

### 2.2 `app/share/record`

```cddl
share-record = {
  type: "app/share/record",
  data: {
    title:       tstr,               ; human-facing label; NOT an identifier
    target:      share-target,       ; what is shared
    audience:    [* audience-entry],  ; may be empty — an authored share with no members yet
    ? note:      tstr,               ; optional human-facing description
    created_at:  uint                 ; ms since epoch, sharer's clock (EXTENSION-CLOCK is local state)
  }
}

share-target = blob-target / prefix-target        ; TAGGED — no untagged ambiguity
blob-target   = { tag: "blob",   hash: content-hash }
prefix-target = { tag: "prefix", path: tree-path }
```

**`target` is what the grant's `resources` scope covers.** A `blob-target` shares one content-addressed
object; a `prefix-target` shares a subtree. The record does not restate the scope — the grant is the
authority and the record is the label. **A consumer MUST NOT infer authorization from the record.**

> **`share-target` is deliberately NOT an `entity-ref`, and the reason is §1.1's reason.** The two
> shapes are already the atom's pinned/live split — a hash and a path, tagged — **minus the authority
> term, because a share is over the sharer's own content on the sharer's own peer**, so the publisher
> is the sharer and naming it again would be restating a term that is already known. **This is the
> implied-authority form described in `APP-CONVENTION-REFERENCE` §3.4**, the same rung `site:`
> occupies. A `peer` slot here would be the third place in this document where someone could put the
> wrong peer, and §1.1 exists because of the first two.

### 2.3 `app/share/audience-entry`

```cddl
audience-entry = {
  type: "app/share/audience-entry",
  data: {
    grantee: peer-id,                ; THE AUDIENCE. Matches the minted token's grantee.
    ? via:   audience-origin,        ; how this member was derived — informative, for UI
    added_at: uint
  }
}

audience-origin = "direct" / "group"  ; "group" records that a group membership produced this entry
                                      ; at authoring time. It is NOT resolved at check time — see §3.
```

### 2.4 `app/share/follow`

```cddl
share-follow = {
  type: "app/share/follow",
  data: {
    publisher: peer-id,              ; whose share this follows
    record:    content-hash,         ; the followed app/share/record
    ? strategy: tstr,                ; NOT LOCKED — see §5. Peers converge; arch records.
    ? since:   content-hash          ; last state this consumer applied, if it tracks one
  }
}
```

**The follow surface is NOT LOCKED.** `strategy` is optional and its vocabulary is undefined here on
purpose — see §5. Everything else in this document is settled; this part is not, and an implementation
should not read the rest of the spec's firmness as extending to it.

---

## 3. Group audience — per-member tokens, and the join-side gap

**A group audience is materialized at authoring time as one `audience-entry` per member, each with its own
minted token whose `grantee` is that member.** There is no group identifier anywhere in the check path.

Three carriers were considered and two are foreclosed by landed core text:

- **Check-time resolution inside the scope match — FORECLOSED.** `peers` is an `id-scope`, compared as
  **literal identifiers**; the pattern grammar is closed to a literal id or `*`. A membership-resolution
  step is strictly more than the canonicalization already forbidden, and it would require the checking peer
  to hold the group's `members/` subtree — which `EXTENSION-GROUP` §4.2 makes a per-deployment privacy
  decision. **Two peers would reach different ALLOW verdicts on the same token.**
- **A group-shaped policy key — FORECLOSED.** `system/capability/policy/{peer_pattern}` is closed to
  exactly two forms (invariant-pointer hex, or the literal `default`), with no partial-prefix matchers. A
  group identifier is neither.
- **Group as grantee, delegating to members — NOT AVAILABLE.** This is the shape the model wants, and
  `system/capability:delegate` is self-attenuation-only in v1. Filed as `[ASK-CORE]` in §7.

### 3.1 Withdrawal — what a UI may claim

**Removing a member takes two operations, and the two mint paths behave differently.**

| Path | Recallable? | Bound |
|---|---|---|
| `system/capability:request`-minted | **No.** Returned inline with no tree write, so the granter never holds the token hash and `revoke` — keyed by hash — cannot name it | the token's own `expires_at` |
| The §4.4 authenticate-response capability | **Yes.** Recorded per-peer at mint time, so the granter can name and `revoke` it immediately | immediate on revoke |

**A complete withdrawal is the policy write (removing or emptying the member's policy entry, which makes
every subsequent `request` fail subset-validation with `403 scope_exceeds_authority`) PLUS a `revoke` of
that peer's delivered capability.** No step is added to the check path.

**A conformant implementation MUST NOT present withdrawal as ending existing access on the `request` path.**
A UI MAY say *"access ended"* for a connected peer once the revoke is issued; for tokens already requested
it MAY say only *"no new access."*

> **This document MUST NOT quote a withdrawal latency.** Nothing in `ENTITY-CORE-PROTOCOL` §6.2 currently
> bounds a `request`-minted token's lifetime by the policy entry's `ttl_ms` or by the caller's own
> expiry — a `request`-minted token has `parent: null`, so §5.6 does not reach it. *"Bounded by TTL"* is a
> promise the substrate does not yet keep, and a convention that quotes one states a guarantee no
> implementation can honor.

### 3.2 The join-side gap is real and MUST be surfaced

**A member who joins a group *after* a share is authored receives nothing until someone re-authors the
share.** This is the genuine cost of third-party delegation being deferred. **An implementation MUST
surface it in the UI rather than implying away** — a group audience is a snapshot at authoring time, and
presenting it as a live membership binding is a misrepresentation of what the tokens do.

---

## 4. Mirroring — the publisher's paths, verbatim

**A consumer mirroring another peer's shares MUST write them at the publisher's own prefix under
`/{publisher}/…`, verbatim, and MUST NOT rewrite them to a prefix of its own choosing.** A cached remote
namespace is *"structurally identical to that peer's own authoritative namespace — the same paths, the same
types, the same semantics"* (`ENTITY-CORE-PROTOCOL` §1.4).

**How a consumer learns the prefix:** `system/peer/published-root.prefix` (`EXTENSION-TREE` §3.3a), which
is REQUIRED precisely because *"two conformant publishers may legitimately publish different extents"* and
*"a consumer MUST read the extent from `prefix` and MUST NOT infer it from the publisher's identity or from
what it happens to find."*

**`published-root` stays singular and is the verification anchor, not a share catalog.** It is the signed
commitment to serve an extent — the first step of the walk-from-signed-root threat model. Making it plural
or audience-scoped breaks the property that a consumer reads one signed artifact and knows exactly what it
covers. **Enumeration of shares is a `type_filter` query over `app/share/record` (§2), not a field on the
published root.**

**Path convergence across implementations is therefore neither required nor wanted** — *paths are
convention; the entity graph is coherence.* What must converge is the **type tag**.

---

## 5. NOT LOCKED — the follow surface awaits peer convergence

**This is not arch's call and it is not ruled here.** Whether `app/share/follow` carries a delivery
`strategy` at all, and what its values are if it does, is for the **implementing peers to converge on**. Arch
records what they land on; it does not pick for them.

**What is known.** Two delivery forms are in the field and both work: a **revision-free closure transfer**,
and a **diff from a last-seen version**. There is no evidence yet that one should be the convention's, and
the reliable-delivery characteristics of the closure form have not been assessed by anyone.

**The genuine open question, for the peers.** Is retrieval a term of the *cross-impl contract* — something
two impls must agree on to interpret each other's follow records — or is it application-local mechanics that
does not belong in a shared entity type? **That question has a real answer and this document does not know
it.** It bears directly on whether the field exists.

**Until they converge:** `strategy` is OPTIONAL, uninterpreted by this convention, and **no implementation
should treat its own form as universal or read another peer's value as meaningful.** `since` is separate and
is not in question — progress is meaningful to any reader.

**`[OPEN-CONVENTION-1]`** — resolved by peer convergence, then recorded here. **Not by an arch ruling.**

### 5.1 Also deliberately out of scope

**Rendering, share UX, front-end wiring** — charter #2: format is the contract; presentation is per
front-end and lives in the application. **When enforcement tightens** — nothing here says when a
development-mode open-grants shortcut comes off; that is the consumer's call.

## 6. Floor (charter #4)

A peer with no group extension, no revision extension and no subscription engine is a **valid participant**:

- It authors `app/share/record` with a `direct` audience and mints one token per member.
- It serves the shared bytes over the ordinary read path; the grant is the only authorization.
- It follows another peer's share by re-reading the record and target on demand.

Capability **adds** — group audiences, diff-based follow, subscription-driven invalidation. None is assumed.

---

## 7. `[ASK-CORE]` — third-party delegation

Group-as-audience without a re-authoring step needs `system/capability:delegate` to mint for a grantee other
than the caller. **This convention does not propose it.** It records that the L5 tier has now produced a
**second** consumer for a deferral the core made on its own schedule — which is the evidence a future
amendment would need.

**Deferral pinned so it expires visibly:** `(ENTITY-CORE-PROTOCOL.md, §6.2 delegate — "cross-peer delegate
is deferred alongside third-party grantee", entity-core-protocol@f83c256)`. **Re-check when that section
moves.**

---

## 8. Conformance vectors — OWED, NOT SHIPPED

**Charter #5: a convention is a byte-level cross-impl contract and is not validated until vectors exercise
it.** v0.1 ships **none**, and is therefore **not ratifiable**.

The vector set this document owes:

| # | Vector | Asserts |
|---|---|---|
| SHARE-1 | `app/share/record` with a `blob-target`, canonical ECF + expected hash | the record encoding is byte-stable cross-impl |
| SHARE-2 | the same with a `prefix-target` | tagged-union discrimination |
| SHARE-3 | a record whose grant omits `peers`, presented cross-peer → **200** | §1.1, the correct shape |
| SHARE-4 | the same grant with `peers: {include: [grantee_id]}` → **403 `capability_denied`** | §1.1, the defect made observable — **the vector that makes the silent failure loud** |
| SHARE-5 | `content-hash` under a non-SHA-256 format code round-trips | charter #6, no fixed width |
| SHARE-6 | policy-entry removal → subsequent `request` yields `403 scope_exceeds_authority`; a previously `request`-minted token still verifies | §3.1, both halves of the withdrawal claim |

**SHARE-4 and SHARE-6 are the two that matter** — they are the assertions that fail loudly if an
implementation adopts the intuitive-but-wrong reading.
