# PROPOSAL — `system/capability/policy/{key}`: §6.2 and §6.9a.1 contradict, and the newer text removes a capability

**Status:** **FOLDED 2026-08-17** — landed as `0.8.1 CAP-7` in `entity-core-protocol` `30ca731`
(§6.2 baseline policy surface: three key forms, resolution order, narrowed disambiguation note).
Validating half — `GUIDE-CONFORMANCE` §9 check **(v)** — unbuilt.
**Target:** `entity-core-protocol` `specs/ENTITY-CORE-PROTOCOL.md` §6.2 + §6.9a.1.
**Filed by:** `entity-browser-rust` (F6, `REVIEW-THE-UNIVERSAL-NAMESPACE…` §7, `67057be`). Verified here.
**Read at:** core-protocol `f83c256` (spec **Version: 0.8.0**) · core-rust `f23fb3b` · browser-rust `67057be`

---

## §1 The contradiction, both sides quoted

Two normative passages govern the **same path**, `system/capability/policy/{key}`.

**§6.2 — "Capability handler — baseline policy surface":**

> *"The path's `{peer_pattern}` is **exactly one of two forms**:"* — (a) the §3.5 invariant-pointer **hex**
> of the peer identity content_hash; (b) the literal segment **`default`**.
> *"**No other pattern forms are defined**"*… and, in the **Namespace disambiguation (0.8.1, UN-a/F47)**
> note: *"closed to exactly the two forms above"* … *"**the Base58 scope form MUST NOT be accepted at this
> policy path.** (Adding the Base58 form at this policy path would reopen the §3018 typo-attack closure)"*

**§6.9a.1 — "Seed capability storage path (normative — applies v7.64 dual-form discipline)":**

> *"**Two key forms** (per v7.64 §2.1 / §2.2): 1. **Hex form (canonical)** … 2. **Base58 form
> (pre-contact affordance):** `system/capability/policy/{peer_id_base58}` — the grantee's Base58 PeerID
> per §1.5… **Use this form when the operator has only the grantee's peer-id and cannot derive hex
> locally** — hashed-key peer-ids (`hash_type = 0x01`) don't carry the public key in the peer-id."*
> Resolution: *"1. Try hex form first… 2. If absent, try **Base58 form**… 3. If both absent, fall through
> to `default`."* Plus a SHOULD first-contact canonicalization.

**These cannot both hold.** One says the segment is closed to two forms and Base58 MUST NOT be accepted;
the other defines a three-step resolution order whose second step is Base58 at that exact path.

**Not a reading artifact.** §6.9a.0 and §6.9a's seed-write pseudocode both depend on the dual form
("*stored at `system/capability/policy/{key}` per the v7.64 dual-form discipline (hex form or Base58 form
per §6.9a.1 below)*"), and `entity-core-rust` implements the dual form (`is_valid_peer_pattern`,
`extensions/capability/src/lib.rs`, citing v7.64).

## §2 Which text is newer — and the filing seat has it backwards

browser-rust filed this as *"arch ruled from the older side; we write the form the newer side
authorizes."* **The precedence runs the other way.**

The spec header reads **`**Version**: 0.8.0`**. The `v7.x` line **precedes** the `0.8.x` line. So:

| passage | tag | position |
|---|---|---|
| §6.2 disambiguation note | **0.8.1**, UN-a/F47 | **newer** |
| §6.9a.1 | applies **v7.64** dual-form, inside a **v7.74** Phase-2 section | older |

**So the two-form closure is the later text, and the Base58 form browser-rust writes (`policy_key`,
`src/share.rs`) is non-conformant under it — as is core-rust's three-form validator.** This inverts the
consequence they expected and is why it is stated here rather than left implicit: **on precedence alone,
the resolution would go against them.**

**This does not weaken the finding — it sharpens it.** The contradiction is real either way, and §3 is
why precedence alone is the wrong basis to resolve it on.

## §3 The newer text removes a capability, and its stated rationale does not support the removal

Two defects in §6.2's closure, independent of which text is newer:

**(a) The rationale conflates two different things.** §6.2 justifies excluding Base58 as *"would reopen
the §3018 typo-attack closure."* The §3018 closure is about **partial-prefix matchers** — `00abc*`,
`00abcdef*` — excluded because *"partial prefixes have no semantic meaning the operator can reason about
and open a typo attack surface."* **A complete Base58 peer-id is not a partial prefix.** It is a second
complete encoding of a whole identity, and it opens no typo surface that a complete hex string does not
already open. The closure that keeps `00abc*` out does not entail keeping `z6Mk…` out; the note treats
them as one rule and they are two.

**(b) It makes a real case unwritable, and §6.9a.1 names that case.** *"Hashed-key peer-ids
(`hash_type = 0x01`) don't carry the public key in the peer-id."* For such a peer, the identity
content_hash **cannot be derived** from the peer-id alone — so before first contact there is **no hex to
write**. Under §6.2's closure the operator's only remaining option is `default`, i.e. granting an unknown
peer the fallback scope instead of the intended one. **The pre-contact policy entry is the affordance;
the closure deletes it and offers nothing in its place.**

This matters beyond bootstrap. It is the shape a **named-audience share** needs: the sharer frequently
holds the recipient's peer-id and has never connected to them.

## §4 Proposed resolution

**Keep the dual form; fix the disambiguation note.** Concretely:

1. **§6.2** — replace *"exactly one of two forms"* with the **three** resolvable forms (hex · Base58 ·
   `default`) and the §6.9a.1 resolution order, so the two sections state one rule in one voice. §6.2
   remains the definitional home; §6.9a.1 keeps applying it to seed writes.
2. **§6.2 disambiguation note** — retain the load-bearing half (**the policy path and the `peers:` scope
   are distinct namespaces that "collide only in the informal name"** — this is correct, valuable, and is
   what caught the share-audience bug) and **narrow the exclusion to what §3018 actually closes:
   partial-prefix matchers, in either encoding.** Drop *"the Base58 scope form MUST NOT be accepted at
   this policy path"* as written; the thing that must not be accepted is the **path-shaped `peers:` scope
   form** (`/{peer_id}/…`, with slashes), not a bare Base58 peer-id segment.
3. **Carry §6.9a.1's SHOULD-canonicalize into §6.2** so the Base58 form is explicitly transitional —
   which is what makes (a) safe: the Base58 entry is a pre-contact placeholder that converges to hex on
   first contact.

**Why this direction rather than enforcing the closure:** the closure's benefit is a narrower key
namespace; its cost is an unwritable pre-contact entry for hashed-key peer-ids and a cohort-wide
non-conformance flag on a form two impls already write. The benefit is available at lower cost by
excluding partial prefixes only, which is what §3018's own reasoning supports.

## §5 Blast radius

| | |
|---|---|
| **Wire** | **None.** A policy path is local storage; §6.2's own text says *"each peer has its own policy table at this path"* |
| **`entity-core-rust`** | Already implements the dual form. **Becomes conformant** under this proposal; non-conformant under the alternative |
| **`entity-browser-rust`** | `policy_key` writes Base58. Same |
| **`entity-core-{go,py}`** | **Measured 2026-08-17 — both already ship the three-form reading; see §5a.** Both become conformant under this proposal and non-conformant under the alternative |
| **Frozen vector set** | Untouched. `authz_peers_target_from_uri` and the P-1/P-2/P-3 trio are §5.2 `peers`-scope vectors; this is §6.2 policy-path keying, the **other** side of the namespace the disambiguation note separates |

## §5a The cohort read this proposal asked for — taken 2026-08-17, and it is 3/3

**§7 asked for a go/py read and §5 declined to assert their state. Both are now measured, by opening the
source.** `entity-browser-rust` supplied go (`ROUTING-2026-08-17-c` §4); it was re-derived here rather
than adopted, and py — unmeasured by them — was read here.

**`entity-core-go`** — `validatePeerPattern`, `ext/capability/handler.go:640` @ `cc537ea`:

- accepts the literal `default`;
- accepts hex at 66 / 98 chars (SHA-256 / SHA-384, format-relative — not a hardcoded 66);
- **accepts a Base58 PeerID** — `if pid := crypto.PeerID(p); pid.Validate() == nil { return nil }`;
- **rejects globs explicitly, before the length checks** — *"policy-entry.peer_pattern partial-prefix
  wildcards are rejected per V7 §4."*

go also implements the v7.65 §3.6 rule-3 lazy canonicalization (Base58 entry minted pre-contact,
rewritten to hex on first cap-match), with operator-facing debug logs on both branches — i.e. it ships
**§4 item 3**, the transitional-form discipline, not merely the acceptance.

**`entity-core-py`** — `_valid_peer_pattern`,
`packages/entity-handlers/src/entity_handlers/capability.py:387` @ `40d1df2`. Same three forms, and its
docstring names this proposal's target directly: *"Base58 PeerID (**pre-configuration affordance** per
`PROPOSAL-V7-POLICY-DUAL-FORM-PRE-CONFIGURATION`)"*. Its hex branch is width-derived the same way
(*"a hardcoded 66 would reject"* a SHA-384 home peer).

**So all three ground-up implementations independently ship hex · Base58 · `default`, and all three
reject partial prefixes** — go by an explicit named guard, rust and py structurally, since a glob is
neither valid hex nor a decodable PeerID. That is the strongest available evidence that **§6.2's two-form
closure is the defective text**: it is the one reading nobody arrived at from the document, and the
form it forbids is the one every seat found it needed. §4's resolution is already built, three times.

**One qualifier, because the evidence cuts only so far.** These are three implementations of one
specification by one cohort — **cohort-consistent, not independent convergence** (ADR-0012). It is
evidence about what the text makes buildable, not a vote that overrides normative precedence. §3 remains
the argument; §5a is corroboration.

### §5a.1 A defect found while taking the read — go accepts uppercase hex

**Routed here rather than to core-go, because it is adjacent to the text this proposal amends.**
`validatePeerPattern` validates its hex branch with `isHexChar` (`handler.go:667`), which accepts
**uppercase `A-F`**:

```go
func isHexChar(c rune) bool {
    return (c >= '0' && c <= '9') || (c >= 'a' && c <= 'f') || (c >= 'A' && c <= 'F')
}
```

§6.2 (0.8.1, RT-14) is normative the other way: *"Lowercase is normative… Content-hash hex in **any** tree
path segment MUST be lowercase… an uppercase-hex path segment is non-conformant."* So `configure` would
accept a pattern that produces a non-conformant path segment.

**Two things make it worth stating rather than filing as a nit.** First, **go disagrees with itself inside
one file**: `isHexString` (`handler.go:510`), used in the canonicalization branch, is lowercase-only.
Second, **the other two seats are lowercase-only** — py's hex branch requires `pattern == pattern.lower()`
(`capability.py:405`), and rust's `is_invariant_pointer_hex` matches `b'0'..=b'9' | b'a'..=b'f'`
(`extensions/capability/src/lib.rs:1041`), with a doc comment that says *"lowercase hex"* outright.
**go is alone, and it is alone against its own neighbouring function.** Low severity, uncontested,
one-character fix.

## §6 What this does NOT change

- **The Q7 ruling stands, on either resolution.** `PROPOSAL-SHARE-AS-GRANT` §3 foreclosed a group-shaped
  policy key. A group id is **none** of the three forms — not an identity-entity hex, not a peer-id
  Base58, not `default` — so the group-as-audience conclusion is independent of this defect. browser-rust
  reached the same conclusion and it is correct.
- **The Q6 ruling stands.** The policy table is a legitimate audience carrier because it is keyed **per
  identity** and mints with `grantee: author`. Which *encoding* keys it is what is in dispute; that the
  key is an identity is not.
- **`peers:` scope semantics are untouched.** It remains the network dimension, matched literally, per
  F40.

## §7 Ask

Route to `entity-core-protocol` as a core-spec revision (proposal-first per ADR / AGENTS-STANDARD, even
though the repo is this team's).

**The cohort read this asked for is done — §5a, all three seats, source-read here.** Two items remain,
and neither blocks the revision:

1. **A `validate-peer` check** for the resolution order (hex → Base58 → `default`) at `configure` and at
   `request` lookup — `entity-core-go`'s `capability` category, per the §9 register's authoring
   convention. None exists. *(Not a keystone item and not a `specs/test-vectors/` corpus item — see
   `GUIDE-CONFORMANCE` §9a for why those are three different things.)*
2. **`entity-core-go`: the uppercase-hex fix** (§5a.1) — independent of how this resolves, since RT-14
   binds either way.

*(Re-tiered `process/` → `core/` on 2026-08-17. This proposal's declared target is
`ENTITY-CORE-PROTOCOL.md` §6.2 / §6.9a.1, and `INDEX.md` §0 derives the tier from the target — so
`core/` is where its routing seats, go · rust · py · keystone, are the ones the tier names. In
`process/` it would have routed to arch-tools, which does not implement this surface.)*
