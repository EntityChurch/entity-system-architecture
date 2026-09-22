# PROPOSAL — the dispatch filter is one function of the name, and the privacy MUST binds the configuration rather than a row

**Status:** IMPLEMENTED — folded at `EXTENSION-REGISTRY` **v1.17** + `GUIDE-RESOLUTION`. D1–D10 verified 2026-09-06. Two rows read as unlanded to a token grep and are not: D2's old sentence survives only inside the blockquote explaining why it could never execute, and the one remaining `did-key` in the guide is a **DID-method maturity row**, not the `backend_kind` token D8 retired.
**Tier:** extensions — `EXTENSION-REGISTRY` §4, §4.1 step 2, §4.1a, §11.1; `guides/GUIDE-RESOLUTION` §6, §6.1, §6.4a
**Answers:** `entity-core-rust`'s §4.1-step-2 contradiction · `entity-core-go` spec-issue `2026-08-19-a` · `entity-core-py` `SA-PY-17`
**Read at:** arch `cf6871e` · go `6ae71c4` · rust `b4b7456` · py `23e77ff`
**Supersedes in part:** `PROPOSAL-DEFAULT-NAME-FORMAT-DISPATCH` (implemented) — D1's surviving half, and Addendum 2's unpropagated grammar change.

---

## §0 Summary

Three implementations read `EXTENSION-REGISTRY` §4.1 step 2 three ways, because the paragraph
contains two sentences that answer the same question differently. **Nobody misread it.** One
sentence is set-valued (*"eligible at the **union** of their `backend_kinds`"*), the next is
per-backend (*"backends **without** an entry default to match all; backends **with** one are
consulted ONLY when the pattern matches"*), and the two disagree on exactly the cases that decide
whether the section's own privacy MUST can be evaded.

**The per-backend sentence is the defect, and it is a category error rather than a wording
problem.** `name_format_dispatch` rows name **`backend_kinds`**, not backends. "A backend without a
`name_format_dispatch` entry" therefore has no referent: a backend does not *have* an entry, its
*kind* is *named by* zero or more rows. The sentence was written as if dispatch were a per-backend
attribute — a route-table row hanging off each chain entry — which is precisely the reading D1 was
withdrawn for. **D1 was withdrawn; the sentence that encodes the same mistake one level down was
not.**

This proposal states the filter once, as a function, and moves the privacy MUST from a **row** to
the **configuration** — because a MUST that binds one row is evadable by not writing that row, which
is the hole all three seats found from different directions.

| Ruling | |
|---|---|
| **Eligibility is a pure function of the name** | `eligible(name)` = union of the `backend_kinds` of every rule whose pattern matches. A chain entry is consulted iff its `backend_kind` ∈ `eligible(name)`. No per-backend default, no "match all" escape. §1 |
| **No rule matches ⇒ empty set ⇒ `chain_exhausted`** | Fail-closed. The current text says *"treated as matching the catch-all"*, which **can never execute** — see §2. |
| **An absent or empty `name_format_dispatch` disables the filter** | The one place "no filtering" is correct, stated explicitly rather than reached by fallthrough, and bound by the MUST below. §3 |
| **The privacy MUST binds the configuration, not the catch-all row** | A shipped configuration MUST NOT make a name-transmitting backend eligible for an unscoped name — by naming it, or by omitting the filter. §4 |
| **`REG-DISPATCH-CATCHALL-LOCAL-1` is retargeted from *remote* to *name-transmitting*** | As written the vector **fails the spec's own recommended default list.** §5 |
| **The schema still declares the grammar this corpus closed** | `<POSIX shell-glob>` at §4, and two more POSIX references. Addendum 2 closed the grammar in prose and never touched them. §6 |

**No wire change, no new entity type, no new error code, no renumber.** Every delta is peer-local
config semantics, one conformance vector, and prose.

---

## §1 The filter, stated once

```
eligible_kinds(config, name):
    rules := config.name_format_dispatch
    if rules is absent or empty:
        return ALL                                  ; filter disabled — §3
    matched := [ r for r in rules if dispatch_match(r.pattern, name) ]
    return  union( r.backend_kinds for r in matched )   ; empty if nothing matched

consult(entry) :=  entry.backend_kind ∈ eligible_kinds(config, name)
```

That is the whole mechanism. It has no per-backend branch, no fallback, and no ordering — `matched`
is a set union, so row order is irrelevant, which is what the D1 withdrawal already established and
what this restates in executable form.

**What each seat had, and why each was reasonable:**

| Case | go / py | rust | This ruling |
|---|---|---|---|
| kind named by **no** rule; some other rule matches | excluded | **consulted** (*"without an entry → match all"*) | **excluded** — a kind reaches eligibility only by being named |
| kind named by a rule; **no** rule matches the name | **consulted** (chain unfiltered) | excluded | **excluded**, and the chain reports `chain_exhausted` |

**All three implementations change, exactly one branch each, and none of them "wins."** That is the
correct outcome when the text was contradictory: converging onto either seat's reading would ratify
half of a sentence that should not have been written.

- **`entity-core-rust`** — `extensions/registry/src/resolver.rs`: the `restricted` set and the
  `allowed` closure collapse to membership in the union. Row 1 flips.
- **`entity-core-go`** — `ext/registry/registry.go` `Resolve` step 2: the `any` guard is deleted, so
  an empty match set narrows to the empty chain instead of leaving the chain unfiltered. Row 2 flips.
- **`entity-core-py`** — `packages/entity-handlers/.../registry.py` `_dispatch_allows`:
  `if not matched_any: return True` becomes `return False`. Row 2 flips.

**py had already flipped row 1 to go's reading on 2026-08-18 (`SA-PY-17`) on an independent privacy
argument.** That matters for how this is read: the split was **2–1, not 1–1**, and the two seats that
converged did so by reasoning about the MUST rather than by copying each other. Their argument is
adopted here. Their remaining row-2 behaviour is the half neither of them re-examined.

---

## §2 *"Treated as matching the catch-all"* can never execute

§4 currently ends its filter paragraph with:

> A name matching no entry is treated as matching the catch-all (§4.1a).

**This sentence is unreachable, and its two branches are each impossible.** The catch-all is the
pattern `*`, and `*` matches **every** name — any run of characters including none, anchored at both
ends (§4's closed grammar). So:

- **If a catch-all row is configured**, every name matches it. "A name matching no entry" is
  therefore false for every name, and the sentence never fires.
- **If no catch-all row is configured**, the sentence directs the resolver to treat the name as
  matching a row that does not exist. There is nothing to take `backend_kinds` from.

It is a no-op with no referent, and it is the residue of D1 — the withdrawal note calls it *"the one
surviving half."* It should not have survived. **go and py both implemented it as *"no filtering"***,
which is the one reading the words do not support: the catch-all is the most restrictive row in the
recommended list, so "treated as matching the catch-all" cannot mean "consult everything."

**The replacement is what §4.1a already says for the mirror case.** A dispatch entry naming a backend
absent from the chain *"yields the empty set and the chain reports `chain_exhausted` (§4.1 step 4,
fail-closed)."* A name matching no rule is the same situation reached from the other side, and it
gets the same answer.

**This is the direction that costs the least when it is wrong.** A name that resolves nowhere is a
visible, recoverable misconfiguration. A name that resolves *everywhere* is a silent disclosure, and
§1 of the parent proposal already established that the disclosure direction is unrecoverable.

---

## §3 The empty dispatch list, and the discontinuity that must be stated

`eligible_kinds` returns ALL for an absent or empty list, and the empty set for a non-empty list
that matches nothing. **That is a real discontinuity — zero rules admits everything, one
non-matching rule admits nothing — and it is deliberate.** It is the ordinary filter-absent /
filter-present-and-excluding distinction, and all three implementations already behave this way on
the empty list (`peer.py:303`, `registry.py:320`, rust `tests.rs:412`, go's `hasCfg &&
len(...) > 0` guard). Removing it would brick every minimal and test configuration in the cohort for
no privacy gain: a peer whose chain holds only `local-name` transmits nothing regardless.

**It is stated in the spec rather than left to be inferred**, because an inferred discontinuity is
how the per-backend sentence got written in the first place.

**What makes it safe is §4, not the discontinuity itself.** An empty filter is a door to the same
disclosure, so the MUST has to reach it.

---

## §4 The privacy MUST moves from the row to the configuration

**Current form** (§4.1 step 2): *"the catch-all MUST NOT name a backend whose consultation transmits
the queried name."*

**The rule binds one row, so it is evadable by not writing that row** — and both remaining doors are
real:

1. **Omit the kind from every rule.** Under rust's reading that made a `dns-txt` backend "match all"
   and it was consulted for **every** name — strictly worse than the catch-all case the MUST
   forbids. §1 closes this **by construction**: an unnamed kind is never eligible. It needs no MUST
   clause, which is the argument for fixing the mechanism rather than adding a rule.
2. **Ship no `name_format_dispatch` at all.** The filter is disabled (§3), every kind is eligible,
   and the catch-all row the MUST binds does not exist. **This door is still open after §1** and is
   the reason the MUST has to move.

**Proposed normative form:**

> **A distribution's shipped `system/registry/resolver-config` MUST NOT make a name-transmitting
> backend eligible for an unscoped name.** The name-transmitting kinds are `dns-txt`,
> `well-known-url`, `did-web` and `consensus-anchored` (the step-2 table). This binds the
> configuration as a whole: naming such a kind in any rule whose pattern matches unscoped names —
> the catch-all `*` being the usual one — violates it, and so does shipping an absent or empty
> `name_format_dispatch` while such a kind sits in the `resolver_chain`. An operator MAY override
> this on their own peer; a distribution MUST NOT ship it.

**This is the corpus's own "state the invariant once, generally, with the instances called out"
rule** (`AGENTS.md`, the equivalence-collapse meta-rule), applied to a MUST that was written at the
width of the instance that motivated it. The catch-all row is an instance of the invariant, not the
invariant.

**Reachability of the check (L12).** The actor is the resolver at config load. Its inputs are the
`name_format_dispatch` list, the `resolver_chain`, and the name-transmitting kind set — the first
two are fields of the config entity the resolver is loading, the third is the fixed §2.4.1
vocabulary partitioned by the step-2 table. All three are in hand at the moment the check runs; no
network read, no cross-peer state. The load-time refusal `REG-DISPATCH-CATCHALL-LOCAL-1` already
mandates is therefore still constructible in its widened form.

---

## §5 `REG-DISPATCH-CATCHALL-LOCAL-1` contradicts the recommended default list

§11.1 reads:

> A resolver-config whose catch-all entry names a **remote** backend MUST be refused or normalized
> at load; and resolving a bare (unscoped) name MUST produce **no read against any remote
> registry**.

**§4.1a row 6 — the recommended catch-all — names `peer-issued`,** and a `peer-issued` resolution is
a read against a remote registry's signed tree. **A peer shipping the list this spec recommends
fails the vector this spec recommends.**

The vector was written at v1.7, when the catch-all was `["local-name", "pinned"]` and *remote* and
*name-transmitting* coincided. `8bfc9b6` re-keyed the catch-all to **name transmission, not
remoteness** — §4.1 step 2 now carries a three-row table saying so in as many words, and admits
`peer-issued` because *"every request is content-addressed; the queried name is matched inside a
node already fetched and never appears in a request."* **The re-key never reached §11.1**, and it
never reached `GUIDE-RESOLUTION` §6.1/§6.4a either, both of which still say *local-only*.

**This is the fourth seat's-eye view of the same drift and no seat reported it**, because no seat has
built a web-native backend yet — the vector passes vacuously on a chain with nothing remote in it
except `peer-issued`, which every impl's fixtures configure explicitly. It fails the moment someone
ships the default list against a live registry.

Retargeted: the discriminator is name transmission, the vector's observable is unchanged (the
absence of a request), and `peer-issued`-through-the-signed-root is explicitly admitted.

---

## §6 The schema still declares the grammar Addendum 2 closed

`EXTENSION-REGISTRY` §4, the `resolver-config` schema, line 3 of the `name_format_dispatch` block:

```
pattern:        <POSIX shell-glob>,
```

**That one word is the entire provenance of the `?` question.** `git log -S 'POSIX shell-glob'`
places it in `70c52b4` — the initial public release — and it is the reason `entity-core-go` reached
for `path.Match`, which was the correct reading of what was written. Addendum 2 closed the grammar to
`*`-only in prose twenty lines below, added *"MUST NOT delegate this to a path-glob or shell-glob
library"*, and **left the declaration that says to do exactly that.** The parent proposal's own
scope line — *"Not in scope: the glob grammar itself (§4's POSIX choice stands)"* — is now false and
was not revisited when the grammar was closed.

Two further POSIX references survive the same way: §4.1a's *"A POSIX glob cannot express
'undotted'"*, and `GUIDE-RESOLUTION` §6's *"a list of **POSIX-glob → backend-kinds** rules."*

**A closed grammar with an open declaration is worse than an open grammar**, because the
declaration is what an implementer reads first and the MUST is what a reviewer reads. That is how
this cost three impls a divergence and one pulled conformance check.

---

## §7 Spec deltas (executable form)

| # | File | Section | Edit |
|---|---|---|---|
| **D1** | `EXTENSION-REGISTRY.md` | §4 schema | `pattern: <POSIX shell-glob>` → the closed wildcard pattern, pointing at the grammar block. §6 |
| **D2** | `EXTENSION-REGISTRY.md` | §4 filter paragraph | Replace *"A name matching no entry is treated as matching the catch-all"* with the empty-set / `chain_exhausted` rule, and state why it cannot execute. §2 |
| **D3** | `EXTENSION-REGISTRY.md` | §4.1 step 2 | Replace the per-backend sentence with `eligible_kinds` in executable form; state the absent/empty-list case explicitly. §1, §3 |
| **D4** | `EXTENSION-REGISTRY.md` | §4.1 step 2 | Widen the privacy MUST from the catch-all row to the shipped configuration, naming both doors. §4 |
| **D5** | `EXTENSION-REGISTRY.md` | §4.1a | *"A POSIX glob cannot express 'undotted'"* → the dispatch matcher. §6 |
| **D6** | `EXTENSION-REGISTRY.md` | §11.1 | `REG-DISPATCH-CATCHALL-LOCAL-1` retargeted from *remote* to *name-transmitting*; `peer-issued`-via-signed-root explicitly admitted. §5 |
| **D7** | `guides/GUIDE-RESOLUTION.md` | §6 | *"POSIX-glob → backend-kinds"* → wildcard-pattern. §6 |
| **D8** | `guides/GUIDE-RESOLUTION.md` | §6.1 | Catch-all row: `local-name, pinned — local only` → the v1.13 row-6 kinds. `did:key:*` → `self-certifying`, not the undeclared `did-key`. (Both tokens were removed from the spec at v1.12 and the guide kept them.) |
| **D9** | `guides/GUIDE-RESOLUTION.md` | §6.4a | *"`*` routes to local-only backends"* → the name-transmitting formulation, matching D4. |
| **D10** | `EXTENSION-REGISTRY.md` | §4.2 | **An unknown `backend_kind` is not a name-transmitting kind and MUST NOT be treated as one.** The refusal is scoped to the four declared kinds; the forward risk is discharged by the check running **at load** (§11.1), not by refusing early. §10 |

**Not in scope:** the matcher grammar itself (settled at v1.13, uncontested, three impls agree) ·
row order (settled by the D1 withdrawal) · the four name shapes · any backend implementation ·
per-backend query privacy beyond dispatch filtering (§11.4 keeps that deferred).

---

## §8 Cohort impact

| Seat | Owed |
|---|---|
| **`entity-core-rust`** | Row 1: `restricted`/`allowed` in `resolver.rs` collapse to union membership. Their §4.1-step-2 contradiction report is **confirmed** — the text did support their reading, and it is the text that changes. |
| **`entity-core-go`** | Row 2: delete the `any` guard in `Resolve` step 2 so an empty match set narrows to the empty chain. Their spec-issue `2026-08-19-a` is **confirmed on the security argument and corrected on the fallback** — *"leave the chain unfiltered"* was never what the text said, which their own filing anticipated. |
| **`entity-core-py`** | Row 2: `_dispatch_allows` returns `False` on `not matched_any`. `SA-PY-17`'s row-1 analysis is **adopted as the ruling's rationale**. |
| **all three** | `REG-DISPATCH-CATCHALL-LOCAL-1` is retargeted (D6) — re-check any fixture asserting *remote*. A wire vector for the grammar's negative-match row becomes constructible once row 2 is uniform, which is what `registry.v15` was pulled for. |
| **`entity-browser-rust`** | **Already conformant on §1 and §4 — confirmed by reading `name_dispatch.rs` at `2940a8f`, not by report (§10).** Two items: **(a)** D10 — `validate_rules` refuses an unknown kind in a broad pattern, which §4.2's MUST does not permit; scope the refusal to the four declared kinds and rely on the load-time re-check. **(b)** the module doc at `name_dispatch.rs:136` still calls the pattern *"a POSIX shell-glob"* — the same stale declaration D1 removes from the spec, and the one your own `dispatch_match` was written to contradict. |
| **`entity-workbench-go`** | **You re-keyed to name transmission before the spec did, and D6 ratifies it.** One real delta: `ValidateResolverConfig` skips every rule whose pattern is not exactly `CatchAllPattern`, so it enforces the **old, row-scoped** MUST. D4 widens it — a broad non-catch-all pattern, and an absent/empty `NameFormatDispatch` alongside a name-transmitting chain entry, are both violations your validator cannot currently see. `browser-rust`'s `is_broad` is a working reference for the first half. Your unknown-kind reading is **upheld** (D10). |
| **both app-tier seats** | The *guide's* §6.1 table (D8) still carried `pinned` and `did-key` after the spec dropped them at v1.12 — a default copied from the guide rather than from §4.1a carries two kinds §4.2 requires be discarded. Check which of the two you copied. |
| **oracle / keystone** | No re-pin. No encoding changes; D6 adds a discriminator to an existing vector. |

---

## §10 The app tier already built this, and the unknown-kind divergence it exposed

**`entity-browser-rust` (`2940a8f`, `src/content_site/name_dispatch.rs`) is the only seat in the
cohort already conformant with §1 and §4 — and it got there before the ruling.**

- `eligible_backends(rules, name)` is the **pure union**, order-preserving, deduplicated, with **no
  per-backend default and no fallback**: nothing matches ⇒ empty vector. That is §1 exactly.
- `validate_rules` + `is_broad` is **§4's widened MUST**, and they widened it the same way — the
  check binds *"a pattern that can match a name carrying no explicit authority marker,"* not the
  catch-all row. They call it D-B, and `is_broad`'s doc states the failure direction argument
  verbatim: *"a pattern we cannot confidently classify is treated as broad, because calling a broad
  pattern narrow is what leaks."*
- `disclosure_of` keys on **transmission, not remoteness** — §5's retarget, already done.

**They reached it by reading step 2's stated intent rather than its contradictory sentence**, and no
core seat was told, because the question looked like an engine question. It is worth recording that
the seat holding the working answer was the one nobody asked — the inverse of the tier error L15
documents, and the second time in two days that an app-tier seat's dispatch work has been ahead of
the core seats'.

**`entity-workbench-go` (`0ba80c6`, `entitysdk/resolver_config.go`) had already re-keyed to name
transmission too** — their comment names *"the remoteness keying this commit replaces."* **Their
validator is the narrow form D4 widens:** it skips every rule whose pattern is not exactly
`CatchAllPattern`, so it sees only the catch-all row and cannot see an absent dispatch list at all.

**Where the two app-tier seats diverge, and it is live today:** an **unknown** `backend_kind` in a
broad pattern. browser-rust refuses it (*"an unknown backend kind is name-transmitting"*);
workbench-go admits it (*"an unrecognized kind is inert by §4.2 … we have been bitten twice by a
guard that was too strict and never once by one that was too loose"*). Both wrote out their
reasoning; neither knew the other had chosen the opposite.

**§4.2 settles it on landed text, in workbench-go's favour, and browser-rust's concern is
discharged rather than dismissed.** §4.2's MUST is explicit — an unknown kind *"MUST cause the entry
to be skipped with a warning, NOT cause the whole config to be rejected"* — and a skipped entry
consults nothing. Refusing the config also rejects a deployment authored against a newer vocabulary,
which is the case §4.2 exists to permit. **browser-rust's forward risk is real** (a kind unknown
today may be declared name-transmitting tomorrow, leaving a validated config carrying a violation
nobody re-examined) **and it is answered by *when* the check runs**: §11.1 mandates the check **at
load**, so the peer that upgrades its vocabulary re-runs it against the stored config and the entry
that was inert becomes a refusal at the moment it stops being inert. **A write-time-only check is the
variant that fails here.** D10.

---

## §9 What this proposal does NOT claim

- **Not** that any seat misread the spec. All three readings are supported by some sentence in it;
  that is the defect being fixed.
- **Not** that the matcher grammar was wrong or that closing it was wasted — it was correct, it is
  uncontested, and the `?`/`[…]` rows are the cheap half. The expensive half was that the schema
  declaration was never updated to match, which is D1 here.
- **Not** a claim that any shipped distribution is leaking. §5's contradiction is currently vacuous
  in every cohort fixture; it fails on first contact with a real web-native backend.
- **Not** a build-state claim about the unbuilt backends (`dns-txt`, `well-known-url`, `did-web`,
  `consensus-anchored` are `paper` per `GUIDE-RESOLUTION` §6.4, read at that commit).
