# PROPOSAL — `ENTITY-CORE-PROTOCOL` §9.1 and `EXTENSION-HISTORY` §4.2: a 403 that reports a capability DENY takes core's closed set; a 403 that reports a domain refusal is the extension's own

**Status:** DRAFT — 2026-09-16.
**Target:** `ENTITY-CORE-PROTOCOL` §9.1's authorization-path code discipline (one sentence of scope) ·
`GUIDE-EXTENSION-DEVELOPMENT` (the rule an extension author needs before minting a 403) ·
`EXTENSION-HISTORY` §4.2 as the proven instance.
**Answers:** a finding filed by the generation seat on 2026-09-08, from re-pinning a history spec for
generation. **It sat unread for eight days**; the corpus-wide census below was taken when it was read.
**Tier:** `extensions/` — it governs what an extension may mint.

---

## §1 The finding as filed, and it is correct

`EXTENSION-HISTORY`'s code set names a **403 that core's authorization registry does not define.**
Verified this session against the landed text of both documents:

- `EXTENSION-HISTORY` v1.10 emits **`403 access_denied`** at two sites, both immediately after
  `check_history_access(ctx.capability, …)` — §4.2's dual capability check — and tabulates it in its
  own Appendix A.
- **`access_denied` occurs ZERO times in `ENTITY-CORE-PROTOCOL`.**
- Core's §9.1 authorization-path discipline reads: a request-time authorization `DENY` MUST carry
  *"`capability_denied` by default, or a more-specific defined authorization code
  (`scope_exceeds_authority`, `unresolvable_grantee`, `capability_revoked`) where one applies"*, and
  **"implementations MUST NOT surface a generic catch-all default."** That is a **closed set of four.**

⇒ **On the narrow question the seat asked, they are right and the code is wrong.** Both sites refuse
on a **capability check**, which is exactly what core's paragraph governs.

## §2 The census, because the instance is not the population

**Reading the finding produced a bigger number than the finding.** Every `403` code paired with a
status across `specs/` and `guides/`, counted this session:

**~20 distinct 403 codes** — `permission_denied` (5) · `capability_denied` (5) ·
`embedded_cap_unauthorized` (3) · `read_only_root` (2) · `peer_excluded_in_context` (2) ·
`payload_unauthorized` (2) · `assigner_authority_insufficient` (2) · `access_denied` (2) ·
`type_not_authorized` · `path_traversal_rejected` · `not_subscription_owner` ·
`deliver_token_insufficient` · `controller_invalid` · `content_store_requires_type_scope` ·
`assignee_excluded` · `view_tree_invalid` · `capability_revoked` — against core's **four**.

⛔ **So the naive reading of core's paragraph — *these are all non-conformant* — would collapse
twenty codes into one and destroy real diagnostic information.** That reading is wrong, and this
proposal exists to say why in text, because nothing currently does.

## §3 The rule — the discriminator is WHAT WAS CHECKED, not what status was returned

**`CD-1` [MUST]** — A `403` that reports the outcome of a **capability check** (a §5.2 request-time
authorization `DENY`) **MUST** carry a code from core's enumerated authorization set:
`capability_denied`, or one of the defined more-specific codes where it applies. **An extension MUST
NOT mint a new code for this case.**

**`CD-2`** — A `403` that reports a **domain refusal** — a condition the extension itself defines,
evaluated after or independently of the capability check — **is the extension's own code** and is
conformant. `read_only_root`, `not_subscription_owner` and `path_traversal_rejected` are this kind:
the caller's capability was sufficient and the extension refused for a reason of its own.

**`CD-3`** — An extension declaring a 403 code **states which of the two it is**, beside the code.

> ⭐ **Why this is the right cut, stated so it can be argued with.** A capability DENY is a statement
> about **the authorization layer**, and the authorization layer is core's — a caller receiving it
> must be able to reason about it identically whatever handler it dialled, which is precisely what an
> extension-local synonym destroys. A domain refusal is a statement about **the extension's own
> rules**, which core knows nothing about and could not enumerate. ⇒ **the boundary between the two
> code spaces is the boundary between the two layers**, and that is why it is not a style question.
>
> ⚠ **The tell that the current text is under-specified rather than violated:** core's paragraph is
> scoped *"a request-time authorization DENY (§5.2)"* in its first clause and then reads as though it
> governs every 403 in the system. **Twenty codes were minted in the gap.** Nobody misread it
> carelessly; it does not say which side it is on for a code an extension defines.

## §4 What changes, and what deliberately does not

| | |
|---|---|
| `EXTENSION-HISTORY` §4.2, two emit sites + Appendix A | **`access_denied` → `capability_denied`.** Both sites refuse on `check_history_access(ctx.capability, …)`; `CD-1` applies. **Proven, and folded with this proposal** |
| `ENTITY-CORE-PROTOCOL` §9.1 | one scope sentence, so the paragraph says which 403s it governs |
| `GUIDE-EXTENSION-DEVELOPMENT` | the `CD-1`/`CD-2`/`CD-3` rule, where an extension author meets it |
| ⛔ **The other ~18 codes** | **NOT swept here, and not asserted non-conformant.** Each needs its emit site opened to decide which side of `CD-1`/`CD-2` it falls on — `permission_denied` (5 occurrences) is the one most likely to be a synonym for `capability_denied` and is the place to start. **A census is not a verdict**, and calling twenty codes wrong from a grep is the error this proposal's own §2 exists to avoid |

## §5 Conformance

| id | level | requirement |
|---|---|---|
| `CD-R1` | `[MUST]` | A 403 reporting a capability-check DENY carries a code from core's authorization set |
| `CD-R2` | `[MUST NOT]` | An extension mints a new code for a capability-check DENY |
| `CD-R3` | `[MUST]` | An extension declaring a 403 code states whether it is a capability DENY or a domain refusal |

**Driven by:** `CD-R1`/`CD-R2` are checkable by a corpus analyzer once `CD-R3`'s declaration exists —
the same shape as the conformance-inventory ratchet, and the same reason it needs the declaration
first. **Until then this is an authoring rule with a manual sweep behind it, and the sweep is owed.**

## §6 Open

1. **`permission_denied` ×5 is the first thing to open** — five occurrences of a word that is either a
   domain refusal or a synonym for the core code, and the two readings have opposite fixes.
2. **Whether `CD-3`'s declaration gets a machine-readable form** or stays prose. Prose first; a gate
   when the population is known.
