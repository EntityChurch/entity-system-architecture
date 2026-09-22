# PROPOSAL — the two specifications that were leaning on the `system/*` reservation's old scope

**Status:** SUPERSEDED BY EVENTS (2026-09-08) — **the rule this proposal was written around has been
withdrawn from the core protocol entirely.** Kept as the record of how it got there and what depended on
it. Do not implement from it; see §0b.
**Target:** `specs/extensions/EXTENSION-COMPUTE.md` §4 · `specs/extensions/EXTENSION-TREE.md` §8.7 — both
now restated on their own basis rather than on the withdrawn rule's.

---

## §0b WITHDRAWN — the whole premise, not just the scope

**This proposal asked the wrong question, and so did the proposal above it.** Both took for granted that
`ENTITY-CORE-PROTOCOL` §6.2's `system/*` reservation *should exist* and argued about its **scope**. The
reservation should not exist, and it has been removed (`0.8.2.13`).

**Three findings, each checkable, and any one of them is disqualifying:**

1. **It was never authorized.** It entered the specification as **one row in a design-revision migration
   table** — *"System paths | Not reserved | `system/*` reserved for system handlers"* — with no rationale
   then and none in any revision since. **It was never a proposal.** It has been carried, and enforced by
   every implementation, ever since.
2. **It names a party the specification does not define.** *"User-installed handlers."* This corpus does
   not define a **user**, and does not distinguish one from the party running a deployment, the party
   administering it later, an extension author, or an ordinary caller. Implementations returned that
   undefined word to callers in the refusal message itself. **Three different words appear across three
   sections for this same undefined party, and no section defines any of them.**
3. **It is not part of the authorization system it appears to belong to.** Every implementation enforces it
   as a **hardcoded prefix match that fires before authorization is consulted at all.** Meanwhile
   `register` already derives its pattern from `EXECUTE.resource.targets[0]`, and the standard dispatch
   capability check on `resource` already decides, per caller and per path, whether that caller may install
   there. **So the rule overrode a decision the deployment had already made deliberately, using a mechanism
   that was finer-grained and already normative.**

**And it constrains nothing cross-peer.** It changes no wire form. A caller observes only a refusal it must
already handle, because capability denial produces one anyway. **Whether to permit installation at a
`system/*` path is a deployment's risk decision** — which extensions it composes, when it stops accepting
more, what grants it issues — and none of that is the protocol's to force.

**What replaces it: nothing normative.** §6.2 now states that install authorization is the capability check
on the install path, plus an informative note that a handler bound over a bootstrapped one substitutes its
behaviour, which many deployments will reasonably refuse.

**The live question, deliberately NOT answered:** `SYSTEM-COMPOSITION` has a startup phase, and *"once a
peer is composed and running it stops accepting further system extensions"* may well be the right shape.
**That is not written anywhere and is not being invented here.** It also runs into an unresolved
prerequisite: **the system/user boundary is not clear** — several standard extensions could reasonably be
user-space, and the corpus has put everything under `system/` without a stated basis.

**§2 and §3 below still describe real dependencies and the edits still stand**, but both were rewritten to
rest on their own basis instead of on the withdrawn rule. **`EXTENSION-COMPUTE` v3.29** keeps its builtin
override prohibition as a **cross-peer determinism** requirement — which is a genuine interop MUST and
survives the withdrawal intact — and **`EXTENSION-TREE` v4.8** now makes its least-privilege point without
restating any prefix rule.

---

## §0 The one-sentence version

**When `ENTITY-CORE-PROTOCOL` §6.2's `system/*` reservation was narrowed from "any handler" to "any
handler installed through dispatch," two other specifications were relying on the part that was removed** —
one of them explicitly, in a sentence telling implementers they needed no guard of their own. This
proposal makes that reliance stop being implicit.

---

## §1 The change under it, stated once so this document stands alone

`ENTITY-CORE-PROTOCOL` §6.2 previously read:

> System handlers live under `system/*` paths. Implementations MUST NOT allow user-installed handlers to
> register at `system/*` paths.

It now scopes that prohibition to the **dispatch path** — a handler installed via the `system/handler`
handler's `register` operation. **A peer's own in-process composition is no longer covered by it**,
because every standard extension owns a `system/*` namespace and under the unscoped reading no standard
extension could be installed on any peer at all.

---

## §2 `EXTENSION-COMPUTE` — the dependence was explicit, and it inverts

### §2.1 What the text said

The builtin override prohibition — the rule that the handlers at `system/compute/builtins/*` cannot be
replaced — ended with this parenthetical:

> *(This is reinforced by — and **a subset of** — V7's general rule that user-installed handlers MUST NOT
> register at `system/*` paths; **an implementation enforcing that general rule needs no separate
> compute-specific guard.** The statement is kept here for its determinism rationale and as
> defense-in-depth.)*

### §2.2 Why the narrowing breaks it

**The subset relation is what fails.** Once the general rule covers only the dispatch path, the compute
prohibition is no longer inside it — it is the **wider** rule, and it is the only one covering the
in-process path. Concretely, after the narrowing and before this edit:

> Peer-owner code composing a peer in-process could bind its own handler body at
> `system/compute/builtins/arithmetic`. Nothing refuses it. Two peers dispatching to that path then
> disagree about what `"add"` means, which is precisely the cross-peer determinism property the
> prohibition exists to hold.

**And the sentence would have actively caused it**, because it tells an implementer that enforcing the
core rule is sufficient. An implementer who did exactly what the specification said would ship the hole.

### §2.3 The edit

The parenthetical is replaced with a statement that the prohibition **binds both installation paths** and
that the core reservation alone does **not** satisfy it. The determinism rationale is unchanged and is the
reason the wider scope is the right one. No operation, type, or error code changes.

**Landed as v3.28.**

---

## §3 `EXTENSION-TREE` — the ordinary case

§8.7's least-privilege paragraph restated the reservation in the retired wording — *"prevents
user-installed handlers from registering there"* — as an unmarked restatement, so a reader arriving at it
would get the old scope from a document with no indication it was a copy.

The edit restates it at the current scope and adds that the in-process composition path is not constrained
by it. **The least-privilege point the paragraph exists to make is unchanged:** being under `system/*` is
organizational and confers no special tree access.

**Landed as v4.7.**

---

## §4 What to revert, and what comes back if you do

**Both edits are self-contained and independently reversible.**

| Revert | Consequence |
|---|---|
| `EXTENSION-COMPUTE` v3.28 only | §2.2's hole is open: nothing in the corpus stops an in-process installer rebinding a compute builtin, and the specification actively tells implementers no guard is needed |
| `EXTENSION-TREE` v4.7 only | a published paragraph states the retired scope. No hole; a wrong sentence |
| Both, **and** the core narrowing | the pre-2026-09-08 position, in which no standard extension can be installed on any peer through any mechanism the corpus describes |

**The third row is the one that matters for judging the first:** the compute edit is not a new
restriction, it is the *preservation* of one that existed before the narrowing. If the narrowing stands,
§2.3 is what keeps the property it had.

---

## §5 The open question this proposal deliberately does not answer

**There is input on the `system/*` namespace from the peer-generation work that has not reached the
specification side.** Namespace issues have been raised there which are not routed into, or reflected in,
any document in this corpus.

**This proposal, and the narrowing it follows from, were both written without that input.** Neither
consulted it, because neither knew it existed.

**So the honest status of the whole `system/*` question is: ruled, folded, routed — and not yet checked
against material that is known to exist and known not to have arrived.** That material should be brought
in and the narrowing re-examined against it. **If it contradicts the narrowing, the narrowing is what
moves**, and §4 is the revert path.

**This is named here rather than discovered later.** A ruling made without input that was known to be
outstanding is a ruling with a stated expiry.

---

## §6 Delta table

| # | File | Section | Change | State |
|---|---|---|---|---|
| C1 | `specs/extensions/EXTENSION-COMPUTE.md` | §4, override prohibition | the prohibition binds both installation paths; the core reservation alone does not satisfy it | **LANDED v3.28, ahead of this proposal** |
| C2 | `specs/extensions/EXTENSION-TREE.md` | §8.7 | the reservation is restated at its current scope | **LANDED v4.7, ahead of this proposal** |

**No other file.** The homes were enumerated by searching for the reservation's **mechanism token** —
the refusal code — rather than for its subject noun, which is how both were found and why the original
proposal's subject-based sweep missed them.
