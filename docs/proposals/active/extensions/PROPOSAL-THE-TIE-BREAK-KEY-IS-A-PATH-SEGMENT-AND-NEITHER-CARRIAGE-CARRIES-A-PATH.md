# PROPOSAL — D1's tie-break key is a path segment, neither carriage carries a path, and §6.5.1c's own justification says otherwise

**Proposes:** `EXTENSION-NETWORK` §6.5.1a / §6.5.1c · `EXTENSION-REGISTRY` §4.1.1
**Status:** DRAFT (2026-09-13) — **not folded. A schema change to a landed type that three core peers
implement; it goes to the implementations before it goes into the spec.**
**Tier:** extensions — `EXTENSION-NETWORK` §6.5.1a / §6.5.1c, and one clause in `EXTENSION-REGISTRY` §4.1.1.
**Answers:** an application-tier implementation finding filed 2026-09-12, on the D1 tie-break key.
**Filed with:** a measurement in two trees and a shipped, disclosed substitute. Nothing is blocked on this.

> **In one sentence.** §6.5.1a D1 orders transport profiles by `(priority asc, profile-id lex)`;
> `profile-id` is **a path segment and not a field**; and **neither carriage that moves profiles between
> parties carries a path** — so the rule is not under-specified, it is **unsatisfiable**, in exactly the
> case the sections exist for.

---

## §1 The measurement

The desktop implementation hit this implementing transport ranking, having first read the landed spec.
Their finding, verified here against the corpus text:

- **`profile_id` is not a field on a profile entity.** §6.5.1's schema block carries ten fields and this
  is not among them. Amendment 8 Q2 defines it positionally: *"the final path segment of
  `system/peer/transport/{peer_id}/{profile-id}`."*
- **A reference implementation derives it from the tree path, which is the only place it exists.**
  `Peer.collectProfileCandidates` computes `profileID := path.Base(e.Path)` and sorts
  `(effectivePriority asc, profileID lex)` — a faithful D1 including the reserved-`primary`-unset-is-0
  rule. **This is not a defect report against that implementation**; it is correct for the input it takes, which is
  profiles read at their own paths.

| carriage | what it carries | path available? |
|---|---|---|
| `system/peer/transport-set.profiles` (§6.5.1c) | the member entities **inline** | **no** |
| registry binding `transports` (`REGISTRY` §3) | a bare `system/hash` per element | **no** |

⇒ A consumer holding two equal-`priority` profiles through either carriage **has no tie-break
available**, and the `primary`-unset-is-0 half of the priority default is likewise unreachable.

---

## §2 ⛔ The sentence that makes this worse than an omission

**§6.5.1c justifies inlining its members with a claim about where D1's key lives:**

> *"And selection under §6.5.1a D1 orders on `priority` and `profile-id`, **both of which live inside the
> members**, so a by-hash encoding could not defer a single fetch."*

**`profile-id` does not live inside the member.** Half of that sentence is false, it is **load-bearing**
(it is the argument for inline-vs-hash), and it is the reason nobody noticed: a section that has already
told you the key is in the member gives you no reason to go looking for it.

⇒ ***The defect is not that the rule was left vague. It is that a section asserted the rule was
satisfiable and nothing checked.*** `grep -n "ProfileID\|profile_id" core/types/network.go` → **0
results**, which is one command and was never run against that sentence.

**Why it stayed invisible:** it only bites when a peer publishes **two profiles of one family at equal
priority** — the mirror/redundancy case — and there is one live federation with one profile per
peer. *Every fixture models one instance of a plural relationship, so the plural case is untestable and
green.* **That is this corpus's own equivalence-collapse shape**, and it is the same reason §8.4.5's
width lock survived three reviews.

---

## §3 The two carriages are different and one rule was never going to fit both

This is why none of the four shapes the filing implementation listed is right as a single answer: **they
asked one question and there are two.** The discriminator is already in the spec — **§6.5.1a D7,
positional authority.**

> **D7.** A transport profile carries the subject peer's authority when it is obtained in one of exactly
> three ways: **read at its own path in that peer's namespace**, **walked from that peer's signed root**,
> or **covered by a verified `system/peer/transport-set`**. **Obtained any other way it carries none.**

| carriage | D7 | so what does D1 order? |
|---|---|---|
| tree read / signed-root walk | ✅ authoritative, **and has a path** | works today, unchanged |
| **transport-set** (§6.5.1c) | ✅ authoritative, **and has no path** | ⛔ **§4 — the set must carry the id** |
| **registry binding** | ❌ **not one of D7's three ways** | ⛔ **§5 — D1 does not apply at all** |

⭐ **The registry binding is not in D7's list**, which no one has noticed and which settles its half by
itself.

---

## §4 D1 — the transport-set carries its members' ids

**A set that inlines positionally-identified members without their positions is a LOSSY CARRIAGE.** It
drops publisher-expressed preference — both the tie-break key and the reserved `primary` name — and it
drops it silently.

> **Proposed.** `system/peer/transport-set.profiles` becomes a map from `profile-id` to the member
> entity, or an array of `{profile_id, profile}` pairs — the implementations pick the encoding; **the normative
> content is that the id travels with the member.**
>
> **[MUST]** A consumer orders a verified set's members by `(priority asc, profile-id lex)` using the
> **carried** id, which is D1 unchanged.
>
> **[MUST]** A set's carried id for a member **MUST** equal the final segment of that member's
> self-published path where both are available; on disagreement a consumer **MUST** fail closed for that
> member. *(D5's `transport_type` precedent — the fail-closed rule the two-sources case needs.)*

**Why not put `profile_id` inside the profile entity** — option 1 as filed, and the objection to
it is right: it creates **a second source of truth against the path** for the tree-read case, which is
the majority case and currently has exactly one. **Carrying the id on the *set* keeps one source per
carriage**: the tree path in one, the map key in the other. That is the same discipline
`APP-CONVENTION-FEED` §4.2 uses for `page` — *"this page's own number — MUST equal its key."*

**Why not make array order significant** — option 2 — it reverses §6.5.1c's *"array order is NOT
significant"* and costs the set a property for a reason that is not the real problem. **The problem is
not that we lack an ordering; it is that we lost an identifier.** Restore the identifier.

**Why not a content-derived tie-break** — option 3 — it is deterministic and carriage-independent and it
**makes the publisher's preference inexpressible**: two equal-priority mirrors could never be ordered by
their owner. Arguably that is what *equal priority* means, which is why this is the runner-up rather
than wrong; it loses the reserved-`primary` rule outright, and that rule is landed and 3-way green.

---

## §5 D2 — a registry binding's transports are not D1's subject, and `REGISTRY` §4.1.1 should stop saying so

**`REGISTRY` already calls them a hint** (§246: *"`transports` is a **cached hint, not the binding's
substance**"*), and **D7 excludes them from positional authority.** So:

> **Proposed, `REGISTRY` §4.1.1.** A binding's `transports` elements are **hints and carry no positional
> authority** (NETWORK §6.5.1a D7). **D1 orders a peer's own claims and does not apply to them.**
>
> **[MUST]** A consumer **MAY** attempt a binding's transports in any deterministic order and **MUST
> NOT** present that order as the target peer's preference. To obtain the peer's own ordering it
> **MUST** obtain and verify a `system/peer/transport-set`, or read the profiles at their own paths.
>
> **[MUST NOT]** A consumer **MUST NOT** synthesize a `profile-id` for a profile that arrived without
> one — from the hash, the array index, an endpoint string or anything else.

**That last clause is the filing implementation's ask and is adopted whatever else is decided.** Their
words: *a consumer inventing its own tie-break silently is deterministic per implementation and different
per implementation, which is precisely the failure D1 exists to prevent, while presenting as having
followed the rule.*

> **`REGISTRY` §4.1.1 currently sends a binding consumer to D1 by name** — *"per NETWORK §6.5
> priority-selection"* — so the cross-reference is the thing to fix, not just the silence. `priority`
> ordering still applies and is still useful; it is the **tie-break** that has no subject here.

---

## §6 What the filing implementation shipped, and why it does not need to change much

Disclosed with the filing and correct under §5 as proposed: D1's `priority` ordering implemented with the
OPTIONAL default of 100 and an anti-vacuity arm listing transports in reverse priority order;
`advertised_at` read by nothing (D3); **the `primary`-unset-is-0 rule deliberately NOT approximated**
because they have no profile-id and *"guessing at one from another field would invent a preference the
publisher did not express, in the one direction that outranks everything they did"*; equal priorities
keeping the binding's listed order with `TransportOptions.TieBreak` carrying a sentence naming D1's
missing key; and a **stable** sort doing normative work, because an unstable one would make one
consumer's two runs over one binding disagree.

⇒ **Under §5 that is conformant as shipped**, minus the reporting obligation, which is one sentence they
have already written into the surface.

---

## §7 What this proposal does NOT establish

- **Not checked against the other two reference implementations.** The measurement is the desktop implementation's tree
  and core-go's source. **Both other core peers implement D1 and neither was read** — the rust/py
  columns are **unknown, not clean.**
- **Nothing is non-conformant today.** The rule is satisfiable where profiles are read at their paths,
  which is what core-go does, and no live deployment publishes two equal-priority profiles of one family.
- **`system/peer/transport-set` has no implementation anywhere.** The filing implementation says so explicitly. **§4
  is therefore a schema change to a type with zero implementations and one consumer-side rule** — the
  cheapest moment this will ever be fixable, which is the argument for doing it now rather than the
  argument for it being urgent.
- **The encoding is not chosen here** (map vs pair-array). That is an implementation-side call and the `ECF`
  determinism properties of each want checking by someone who encodes.
- **Weight-based balancing among equal-priority mirrors stays out of v1**, per §6.5.1a's existing note.
