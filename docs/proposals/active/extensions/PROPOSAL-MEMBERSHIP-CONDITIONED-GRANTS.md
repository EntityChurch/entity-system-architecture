# PROPOSAL — membership-conditioned grants: let a capability name a condition instead of a person

**Status:** DRAFT — 2026-09-06. First pass. **Not ratified, not folded.**
**Tier:** extensions — `ENTITY-CORE-PROTOCOL` §5.2 `constraints` (convention, no schema change) +
`EXTENSION-GROUP` §6 (the discharge object) + a conformance vector.
**Answers:** the moderation/authorization cost in
`EXPLORATION-THE-SECOND-FALSIFICATION-…` §3.2's group-moderated forum.

---

## §1 The problem, in the forum

A forum topic is a union over a topic hash; a **group** publishes the curated, moderated view
(`EXPLORATION-THE-SECOND-FALSIFICATION` §3.2, and it is the ruling we already have). To moderate, the
group must authorize its members to act — post to the curated stream, redact, pin.

**Today a capability names a person.** `ENTITY-CORE-PROTOCOL` §1029: `grantee` is the *"hash of
grantee's peer entity"*, and it MUST resolve to a present `system/peer` or the chain is rejected `401
unresolvable_grantee`. `EXTENSION-GROUP` §7.4 Option A states the resulting pattern outright: *"parent
issues caps to subgroup (subgroup's published handle as grantee). **Subgroup distributes among
members**."* And §6.3's removal path must *"revoke any caps issued to them."*

**So membership churn is capability traffic** — or half of it is, and the corpus already fixed the
other half.

> **`EXTENSION-ROLE` §2.3 exclusion already does the LEAVE side, and it establishes the pattern this
> proposal extends.** An exclusion *"denies all access within that context **regardless of held
> tokens**."* **That is a live, tree-consulted decision that overrides a validly-held capability** — so
> the corpus already accepts that a verifier consults tree state and not only the token, and a removal
> already costs one write rather than N revocations.
>
> **What ROLE does not do is the JOIN side.** §1: *"Assigning a peer to a role means **issuing those
> grants**."* Assignment is still per-peer token derivation. So the cost this proposal targets is
> narrower than the first draft claimed: **not membership churn in general, but admission** — every new
> member of a 200-member forum requires a capability issued to them by the group before they can act.

**This proposal is the positive dual of an exclusion**: a live, tree-consulted *allow* condition, where
ROLE has a live, tree-consulted *deny*. **The group already maintains the membership list as entities**
— `system/group/{id}/members/{member}` — and the admission path cannot see it.

**This is the one place the research arc found a real cost.** Everything else it examined came back
confirming (§6).

## §2 What is proposed — and it is a convention, not a mechanism

**Nothing in the substrate is missing, and that is the point.** `ENTITY-CORE-PROTOCOL` §983 already
defines the field:

> `constraints: {map_of: {type_ref: "primitive/any"}}` — *"Domain-specific narrowing fields,
> **handler-interpreted**. Each key is a named restriction. Absent = unconstrained. Adding a key
> narrows access."*

**A membership-conditioned grant is one reserved `constraints` key.** Sketch, to be argued in review,
not adopted here:

```
constraints: {
  requires_member_of: <hash of the system/group entity>
}
```

**The obligation on the verifier:** the request is authorized only if the grantee holds a current
`system/group/{G}/members/{grantee}` entry — supplied by the requester in `envelope.included`, signed
by the group, and checked exactly like any other entity. **The group and the verifier never
communicate.**

**Why it must be pinned rather than left to each handler.** `constraints` is handler-interpreted by
design, which is right for a *local* narrowing. **This one is cross-peer observable**: the grant
travels, and a verifier that ignores the key **widens** access while a verifier that does not
recognize it **denies**. Two conformant peers diverge on the same capability — which is exactly the
`MAY`/`SHOULD` case `AGENTS.md` says to lean MUST on. **The proposal is the pin, not the plumbing.**

**Cost of a leave becomes zero capability operations.** Remove the member entry; every verifier that
next checks the grant fails the condition. That is the property being bought.

## §3 Prior art, and what it is called

This is a **third-party caveat** — the one primitive macaroons have that we do not. A *first-party*
caveat is one the verifier evaluates itself (expiry, path scope): that is `constraints` as used today.
A *third-party* caveat says **valid only if you also present a discharge from party P attesting
condition C**, and the verifier checks the discharge without contacting P.

**Macaroons make this expensive for a reason that does not apply to us.** Their construction is
symmetric — HMAC chaining from a shared root key — so a third-party caveat must smuggle a per-caveat
key to two parties at once, hence the encrypted VID/CID pair. **All of that machinery exists to move a
symmetric secret.** We have public keys, content addressing, and signed third-party claims, so the
discharge is just an entity the requester carries.

**We have two candidate discharge objects already** and choosing between them is §5's open question:

| Candidate | Fit |
|---|---|
| `system/group/{G}/members/{peer}` | Exact for the forum. Already written on every join |
| `system/attestation` | General — `attesting` / `attested` / `properties` / `not_before` / `expires_at` / `supersedes`, signed and content-addressed. Covers conditions that are not group membership |

## §4 What this is NOT

- **Not a core protocol change.** No new type, no schema change, no wire change, no new crypto.
- **Not a claim that the system lacked a capability.** It is extensible and this is expressible today;
  the deliverable is agreement so two implementations read the same grant the same way.
- **Not attribute-based access control in general.** One reserved key with one discharge shape.
  Generalizing it is a later question and is explicitly out of scope.

## §5 Open questions — the reason this is DRAFT

1. **Which discharge object** — the group member entry (narrow, exact, already written) or a
   `system/attestation` (general, one more object to mint). **Leaning member entry for v1**, since the
   forum is the driving case and the general form can be added without moving the first.
2. **Freshness.** A member entry the requester supplies is as stale as the requester likes. Bound by
   the grant's own expiry? By `not_before`/`expires_at` on the entry? **This is the same question
   SPKI §5.4 frames as *"how long are you willing to let the world believe something false?"*** and it
   is the real design work in this proposal.
3. **Revocation interaction.** How does this compose with §5.1 `is_revoked`? A failed condition is not
   a revocation and should not be reported as one.
4. ~~**Does `EXTENSION-ROLE` already cover this?**~~ **ANSWERED — it covers half, and the half it
   covers is the precedent.** §2.3 exclusion is already a live tree-consulted deny that beats a held
   token, so the *leave* side is solved and the pattern is already ratified in this corpus. §1's
   *"assigning a peer to a role means issuing those grants"* is the *join* side, and it is per-peer
   issuance. **The proposal narrows to admission only** (§1), and it should very likely land **in
   `EXTENSION-ROLE` as the dual of exclusion** rather than as a bare `constraints` convention —
   an assignment that names a condition instead of a peer. **That relocation is the first thing to
   settle in review**, and it may make §5.1's discharge question moot, since ROLE already has
   `context` as the scoping noun.

## §6 Why this is the only proposal from the arc

Recorded so the rest of the research is not re-opened looking for deliverables. **The arc's other
findings came back confirming, and confirmations do not become proposals:**

- **The tree substrate** — the alternatives (JMT, Verkle, prolly) were compared over five revisions
  with four-seat review. Settled; no change.
- **`max_delegation_depth`** — an optional DoS bound, **set in no product code in any of the five
  trees** (py unit tests only). Nothing owed.
- **Radicle's collaborative objects** — a DAG of signed ops, topologically sorted, tiebroken by content
  hash. **That is `EXTENSION-REVISION`.** And their `Op.identity` (each op commits to the identity-doc
  head it was authored under) is **`EXTENSION-QUORUM` §4.2's `as_of` historical-state resolution**,
  which is already a MUST with cross-impl vectors TV-Q-V16a–c, plus prior-signer-snapshot pinning.
  **We are ahead here, not behind.**
- **Certificate Transparency's witness** — real and designable on our substrate, but it detects
  publisher equivocation on a signed root. **It is not on the path to a forum** and belongs in the
  verification-ladder work, not here.
- **The forum's actual open problem is navigation** — finding anyone at all who is in a topic —
  which `EXPLORATION-WHAT-CONVERGES` §4.4 already names and this arc did not move.
