# APP-CONVENTION-SHARE — the share record, its audience binding, and the audience-less publication — v0.2.1 DRAFT

**Version**: 0.2.1
**Status**: Draft
**Kind**: normative-spec · **Authority**: binding · **Governed-by**: `guides/GUIDE-APPLICATION-DEVELOPMENT.md` — FORMAT-only (§2.1).
**Depends:** `ENTITY-CORE-PROTOCOL.md` §3.6 / §5.2 / §5.4 (grant structure, `check_permission`,
pattern matching) · `EXTENSION-TREE.md` §3.3a (`published-root`) · `APP-CONVENTION-EMBED.md` §3
(`content-hash` atom).

> **What this document is.** A share is **not a new mechanism.** It is a titled capability grant plus a
> small record that names it, so that two independent front-ends produce byte-compatible entities and can
> read each other's shares. The convention adds **no** kernel feature, **no** required SDK surface, and
> **no** new authorization path (GUIDE-APPLICATION-DEVELOPMENT §2.2; proposal-first). Everything normative about *whether access is allowed*
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
| *What is withdrawal?* | **for an `app/share/record`** — a real revocation, not an unlisting. **For an `app/share/publication` there is no grant to revoke, so withdrawal is UNLISTING and nothing else — §2.5** | §5.1 revocation markers; §2.5 |

> **This table is written in `app/share/record`'s terms**, because every slot in it is a slot on a
> **grant** and an `app/share/publication` has none. Read it as the answer for the audience-bearing
> type; §2.5 answers the same four questions for the audience-less one, and **the withdrawal row is
> the one where the two types give opposite answers.**

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
namespace and a convention adds no kernel surface (GUIDE-APPLICATION-DEVELOPMENT §2.2).

| Type | Role |
|---|---|
| `app/share/record` | the share itself — what is shared, under what title, **to whom** |
| `app/share/publication` | a share with **no audience** — offered to anyone who can reach it, pull-only |
| `app/share/audience-entry` | one audience member's binding within a record |
| `app/share/follow` | a consumer's subscription to another peer's share **record** |

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

**No fixed-width hash form appears anywhere in this document** (SPECIFICATION-FORMAT §8.4.5). A convention that bakes one
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

**`app/share/record` is direct-audience-only.** A share offered to no one in particular is
`app/share/publication` (§2.5), **not** a `record` with an empty `audience` — the empty array keeps
its meaning above, *an authored share with no members yet*, which is the self-only state and is a
different thing from public.

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

**`record` names an `app/share/record` and MUST NOT name an `app/share/publication` (§2.5).** A
publication has no audience and no grant, so there is nothing for a follow to be scoped by; a
consumer tracking a publisher's publications issues the §2 type-filtered query instead (see §2.5).
**This is stated because the mistake is silent**: pointing a follow at the wrong tag returns a
correct, complete, **empty** answer, which is the same failure the box below describes.

**The follow surface is NOT LOCKED.** `strategy` is optional and its vocabulary is undefined here on
purpose — see §5. Everything else in this document is settled; this part is not, and an implementation
should not read the rest of the spec's firmness as extending to it.

> ### ⚠ `app/share/follow` and `app/feed/follow` are DIFFERENT TYPES, and the discriminator is the subject
>
> **This one follows a GRANT. `APP-CONVENTION-FEED` §2.4's follows a NAMESPACE.** That is not a naming
> accident and the two are not interchangeable:
>
> | | `app/share/follow` (here) | `app/feed/follow` |
> |---|---|---|
> | Subject | one titled **share record** | a peer **namespace** |
> | Authorization | an audience the publisher authorized | **none required** — public, pull-only |
> | Does the publisher know? | **yes** — they minted the grantee's token | **no**, and cannot |
>
> **An implementer who finds this type first and never sees the other will use the wrong tag**, and the
> failure is silent: a type-filtered query on the wrong tag returns a correct, complete, **empty** answer.
> They are not unified because doing so would require making the `record` field optional, which changes
> what an absent field means in an already-landed schema. **If the implementing peers converge on
> unifying them, that is recorded rather than ruled** — the same stance §5 takes on this follow surface.

---

### 2.5 `app/share/publication`

**A share with no audience.** It is `share-record` **minus `audience`, and nothing else differs** —
the audience model is the only axis this type splits on, so it is the only field that moves.

```cddl
share-publication = {
  type: "app/share/publication",
  data: {
    title:      tstr,                ; human-facing label; NOT an identifier
    target:     share-target,        ; what is offered — §2.2's atom, tagged, blob OR prefix
    ? note:     tstr,                ; optional human-facing description
    created_at: uint                 ; ms since epoch, publisher's clock
  }
}
```

- **No `audience` and no minted token.** Authorization at fetch is **none required, pull-only**. A
  publication carrying an `audience` field is **invalid**.

> **"No grant" is a statement about the AUDIENCE MODEL, and it does NOT mean the publisher authorizes
> nothing.** There is no member to enumerate and no per-member token minted for anybody — that is the
> single axis this type splits on. **It is not an instruction to leave the target unreachable**, and
> reading it that way makes the next sentence unkeepable and `SHARE-9` unpassable: a consumer
> presenting no token would be refused, not for lacking one, but for there being no authority at all.
>
> **[MUST]** A publisher of an `app/share/publication` **MUST** ensure the `target` is retrievable by a
> consumer that presents no token and performs no binding lookup. **Where the peer's connection-time
> floor does not already cover the target, something must authorize that read, and authoring it is part
> of publishing.**
>
> **The mechanism is the core's and this convention does not restate it** (`SPECIFICATION-FORMAT`
> §10.3): `ENTITY-CORE-PROTOCOL` §4.4 makes an inbound peer's initial scope the **union** of the SHOULD
> floor and the matched policy entry — *"implementations without a policy table populated for peer A
> deliver only the SHOULD floor"* — and §6.2 resolves that entry **exact-match-or-`default`-fallback**.
> A floor that covers only the type and handler namespaces reaches no published target, so on such a
> peer the `default` entry is what keeps the promise above.
>
> ⚠ **AUTHORING THAT ENTRY HAS A SECOND EFFECT, IN THE OPPOSITE DIRECTION, AND IT IS EASY TO SHIP
> BLIND.** By `ENTITY-CORE-PROTOCOL` §6.2 the **same** entry is a **union term** at §4.4
> authenticate-response and a **per-peer ceiling** at `system/capability:request` — both intended, and
> that section is the authority for both. §6.2 also states that with no entry at all, request-time
> *"pure-attenuation flow … works … by skipping the policy ceiling — step 3 only enforces bounds that
> exist."* **So creating a `default` entry where a deployment had none converts *no request-time
> ceiling* into *this request-time ceiling*, for every peer holding no entry of its own.**
>
> **[SHOULD]** A `default` entry authored to satisfy the `[MUST]` above **SHOULD** be written as the
> **union** of the publication's read grants with whatever that deployment already intends to allow at
> `request` — never as the publication's grants alone. **The failure is silent in the direction that
> matters:** the publication becomes reachable, so the change looks correct, while unrelated requests
> from unlisted peers begin failing subset-validation. `SHARE-10` is the vector.
- **A consumer needs no token and performs no binding lookup.** Reaching the bytes is the whole
  protocol.
- **The publisher cannot know who fetched it**, and an implementation MUST NOT present it as though
  it can.
- **`target` is `share-target` unchanged**, so a publication of a subtree is as ordinary as a
  publication of one blob. §2.2's note on why `share-target` is not an `entity-ref` applies here for
  the same reason: the publisher is the sharer, so the authority term is already known.

> **Why this is a separate type rather than a value inside `audience`.** `audience` is an enumeration
> of authorized wielders, each holding a token minted for them; a publication has neither, so there
> is nothing to enumerate. Every in-field encoding breaks something already load-bearing: the empty
> array is taken (*authored, no members yet* — §2.2), a sentinel `grantee` makes that field accept a
> value that is not a `peer-id`, and a sibling flag leaves two fields able to disagree with no way
> for the schema to forbid it. **§2.4 already refused the same move on the same grounds**, declining
> to unify `app/share/follow` with `app/feed/follow` because it *"would change what an absent field
> means in an already-landed schema."*

**Retrieval takes no new mechanism.** §2 already gives it: cross-peer aggregation is a `type_filter`
query over the universal tree **with no peer filter**, and the type tag is the index key. A reader
wanting a publisher's publications issues that query. **No `publication`-side follow type is defined
here on purpose** — what one would add is a persisted intention plus a cursor, which is a general
subscription concern and does not get minted piecemeal inside this convention.

> **Withdrawal UNLISTS; it does not retract — and this follows from the definition, so it is not an
> implementation gap.** A publication requires no authorization to retrieve, so there is no grant to
> revoke and a withdrawal has nothing to act on but the entity itself. Deleting or unpublishing an
> `app/share/publication` removes it from type-filtered discovery. **Any party already holding the
> content hash may still retrieve the bytes**, and no mechanism in this convention or in the content
> layer changes that.
>
> **The asymmetry against §2.2 is the point.** For an `app/share/record` a withdrawal has a real lever
> at the capability layer — emptying the member's `audience-entry` stops that member's token
> validating — so lingering bytes are a hygiene problem and the user-visible promise stays keepable.
> **For a publication, discovery IS the whole access path, and discovery is the half that binds
> nobody.**
>
> **[SHOULD]** An interface offering this action names it for what it does — *stop listing*, *stop
> offering* — and **SHOULD NOT** present it as deletion, retraction or recall. **The failure mode is
> silent**: nothing errors, no check fails, and the party left holding the false belief is the person
> who published. This is §2's *correct, complete, empty answer* pointed at the publisher instead of
> the consumer.

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

**This section is the `app/share/record` half. §2.5 is the `app/share/publication` half, and the two
reach opposite conclusions** — here a withdrawal has a real lever at the capability layer, there it has
none by construction. **An implementation offering one *stop sharing* control over both types owes the
user two different sentences.**

**Both halves sit under the same tier-wide rule and neither restates it: removal from a tree is
UNPUBLICATION, never erasure** (`APP-CONVENTION-FEED` §7.5, and it is a `[MUST NOT]` on presenting
removal as deletion). What this section and §2.5 add is the *second* lever and whether it exists: FEED
§7.5 governs what a party already holding the bytes can do, and nothing here changes that answer for
either type.

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

**Rendering, share UX, front-end wiring** — GUIDE-APPLICATION-DEVELOPMENT §2.2: format is the contract; presentation is per
front-end and lives in the application. **When enforcement tightens** — nothing here says when a
development-mode open-grants shortcut comes off; that is the consumer's call.

## 6. Floor (SPECIFICATION-FORMAT §8.6)

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

## 8. Required checks — what an implementation must discriminate

**GUIDE-EXTENSION-DEVELOPMENT §7: a convention is a byte-level cross-impl contract and is not validated until conformance checks
exercise it — and the convention names the cases while the implementations and the conformance oracle produce
the fixtures and run them** (`GUIDE-CONFORMANCE` §5.1a). **This convention is authored; it is validated when
these have been exercised.**

The cases this document names:

| # | Vector | Asserts |
|---|---|---|
| SHARE-1 | `app/share/record` with a `blob-target`, canonical ECF + expected hash | the record encoding is byte-stable cross-impl |
| SHARE-2 | the same with a `prefix-target` | tagged-union discrimination |
| SHARE-3 | a record whose grant omits `peers`, presented cross-peer → **200** | §1.1, the correct shape |
| SHARE-4 | the same grant with `peers: {include: [grantee_id]}` → **403 `capability_denied`** | §1.1, the defect made observable — **the vector that makes the silent failure loud** |
| SHARE-5 | `content-hash` under a non-SHA-256 format code round-trips | SPECIFICATION-FORMAT §8.4.5, no fixed width |
| SHARE-6 | policy-entry removal → subsequent `request` yields `403 scope_exceeds_authority`; a previously `request`-minted token still verifies | §3.1, both halves of the withdrawal claim |
| SHARE-7 | `app/share/record` with an **empty** `audience` → read as **self-only**, never as public | §2.2, the state the split exists to keep distinct |
| SHARE-8 | `app/share/publication` carrying an `audience` field → **rejected as invalid**; one with a `prefix-target` → **accepted as ordinary** | §2.5, both directions of the new type's shape |
| SHARE-9 | a consumer fetching a publication presents **no token** and is **not refused for lacking one**; an `app/share/follow` naming a publication is **rejected** | §2.5 / §2.4, the pull-only posture and the silent-empty mistake made loud |
| SHARE-10 | on a peer with **no** policy entry, a peer holding no entry of its own issues a `request` the caller's own cap covers → **succeeds**; a publication is then published and its read authority authored; **the same `request` still succeeds**, and the publication is fetchable with no token | §2.5's `[SHOULD]` — that keeping the publication promise did not narrow the request path. **Both halves are required**: a run asserting only the fetch reports success while the regression is live |

**SHARE-4, SHARE-6 and SHARE-9 are the three that matter** — they are the assertions that fail loudly if an
implementation adopts the intuitive-but-wrong reading. **`SHARE-10` is the one that fails QUIETLY**, which is
why it is named separately: everything it guards keeps working from the publisher's side.

---

## Document History

**v0.2.1:** §2.5's *"no grant"* is disambiguated — it names the **audience model** (no member
enumerated, no per-member token), never an instruction to leave the target unauthorized, which is the
only reading under which `SHARE-9` can pass. Adds the reachability `[MUST]`, a `[SHOULD]` on how the
authorizing entry is written, and **`SHARE-10`** for the request-path narrowing that authoring one can
cause. Per the site-asset-child-arm-and-publication-grant proposal.
**Domain:** `applications/` (third member).
