# PROPOSAL — pin the default dispatch globs, before two app tiers ship two of them

**Status:** **IMPLEMENTED 2026-08-18, with D1 WITHDRAWN the same day** — five of six deltas stand.
~~D1 §4 (ordered, first-match-wins)~~ **withdrawn — operator ruling** · D2 §4.1 step 2 (the
catch-all MUST) · D3 new §4.1a (the default list) · D4 §4.1 step 1a (peer-id decode precedes glob
dispatch) · D5 `guides/GUIDE-RESOLUTION` §6.1 + §6.4a · D6 §11.1
(`REG-DISPATCH-CATCHALL-LOCAL-1`). `EXTENSION-REGISTRY` → 1.7.

> **D1 was wrong, and the spec already answered the question it asked.** D1 added a second
> precedence mechanism — *"evaluation stops at the first matching entry"* — to a document that has
> carried one since v1.0: `resolver_chain[].priority` (§4, *"ascending; lower = consulted first"*),
> §4.1 step 3 (*"filtered resolver-chain backends **in priority order**"*), and §4.1.1 (*"returns
> the **first hit** that passes validation… the **higher-priority** backend wins"*). Three landed
> statements, one mechanism. `name_format_dispatch` decides **which** backends are eligible;
> `priority` decides **which one answers**. Making the dispatch list itself precedence-bearing gave
> the resolver two orderings that disagree.
>
> **The tell was there before the fold and was read past.** This proposal's own §4 D1 row states the
> defect as *"the field is an array with no stated evaluation rule"* — but an array of **filters**
> needs no evaluation rule, which is why none was ever written. An absent rule was read as an
> omission rather than as the design. `entity_browser_rust` reached the opposite conclusion from the
> same text and pinned it in a test (`matching_several_rules_yields_the_union_not_the_first`); their
> reading was correct and this one was not.
>
> **Consequence for §4.1a:** the row order was justified by D1 (*"rule 4 precedes rule 5"*). With D1
> gone, rules 4 and 5 genuinely overlap on a dotted authority and `priority` resolves it — recorded
> in §4.1a rather than papered over, because a deployment that wants the narrower behavior now has
> to express it as priority or as a narrower pattern.
**Tier:** extensions — `EXTENSION-REGISTRY` §4, §4.1; `guides/GUIDE-RESOLUTION` §6.
**Answers:** `entity-browser-rust` B16b (their last arch blocker) and `GUIDE-RESOLUTION` §11 open
question 1.
**Read at:** arch `0b397c4` · browser-rust `664b36b` · workbench-go `128e7be`

---

## §0 Summary

**The grammar is already designed and already documented** — `GUIDE-RESOLUTION` §6.1's four shapes,
§6.3's dispatch-on-X unification. Nothing here is new design. What is missing is that the defaults
are **recommendations in a guide**, so each distribution invents its own, and two app tiers are days
from shipping two different ones.

**The one thing that is not a mere convention, and the reason this is normative rather than a guide
edit:** `EXTENSION-REGISTRY` §4.1 step 2 calls `name_format_dispatch` **the primary privacy
mechanism**. A default that routes the catch-all to a *public* registry sends every bare name a user
ever types — including private local handles — to a third party, and it does so silently, on the
happy path, in a configuration nobody chose. **The privacy property is not a property of the
mechanism; it is a property of the defaults.** Shipping the mechanism without pinning the defaults
ships the mechanism without the property.

| Ruling | |
|---|---|
| **The catch-all `*` MUST route to local-only backends** | Never to a remote registry. A public registry is reachable only through an **explicit** scoped form. §1 |
| **Dotted-vs-undotted is the authority discriminator** | `*@*.*` (domain) is matched **before** `*@*` (registry handle); the dot is what separates them. §2 |
| **Dispatch-on-X is structural, not a new glob** | `name@X` where X decodes as a V7 §1.5 peer-id is a **verification pin**, not a targeting instruction — resolved by the chain, then checked. §3 |
| **Order is first-match-wins and MUST be stated** | The list is ordered; the guide's table is not, and an unordered reading routes every domain to the handle backend. §2 |

**No wire change, no new entity type, no new error code, no renumber.** This pins the contents of a
peer-local config entity and the order in which its rules are read.

---

## §1 The catch-all, and the only rule here that is load-bearing

**`*` MUST resolve through local-only backends** — `local-name`, `pinned`, and self-certifying
decode. It **MUST NOT** be bound to `peer-issued`, `dns-txt`, `well-known-url`, `did-web`, or
`consensus-anchored` in a shipped default.

**Why this is a MUST and not a SHOULD.** The failure is silent, universal, and unrecoverable:

- **Silent** — a bare name that leaks still *resolves*, usually correctly. Nothing fails, no warning
  fires, and the user's local handle for a private contact has been sent to
  `entitychurchregistry.org` as a side effect of typing it.
- **Universal** — it is the catch-all. Every name that is not explicitly scoped takes this path, so
  the leak is not an edge case; it is the default case.
- **Unrecoverable** — a name is disclosed once. No later configuration change un-discloses it.

**And it is reachable by a well-intentioned author.** Binding `*` to the public registry makes bare
names "just work" against the registry a distribution ships with, which reads as good product
behavior. That is exactly how a privacy mechanism fails open: not by someone disabling it, but by
someone configuring it for convenience without knowing it was the mechanism.

**A public registry stays fully usable — through an explicit form.** `alice@entity-church` is one
`@` more typing and it is the user stating, in the name itself, which authority they are willing to
tell. That is the whole design of §6.1, made mandatory at the one position where the user is not
choosing.

**Deployments may still override.** §6.4a's *"a deployment MAY ship different globs"* is unchanged —
this pins **the default a distribution ships**, not what an operator may configure. An operator who
deliberately routes `*` to a registry they run is making an informed choice on their own peer; a
distribution making it for a million users is not.

---

## §2 The ordered default, and why order is the whole thing

**`name_format_dispatch` is an ordered list, first-match-wins.** §4's schema is a CBOR array and
§4.1 step 2 says "narrow to backends whose dispatch pattern matches" without saying what happens
when two patterns match. **Two do, always** — `*` matches everything, and `*@*.*` and `*@*` overlap
on every domain-shaped name. An unordered reading is not merely underspecified; it routes
`alice@example.org` to whichever entry the implementation happens to visit first.

**The default list, in order:**

| # | `pattern` | `backend_kinds` | Matches |
|---|---|---|---|
| 1 | `did:web:*` | `["did-web"]` | scheme-typed |
| 2 | `did:key:*` | `["did-key"]` | scheme-typed, self-certifying |
| 3 | `*.eth` | `["consensus-anchored"]` | scheme-typed by suffix |
| 4 | `*@*.*` | `["dns-txt", "well-known-url"]` | **domain-scoped — dotted authority** |
| 5 | `*@*` | `["peer-issued"]` | **registry-scoped — undotted handle** |
| 6 | `*` | `["local-name", "pinned"]` | **catch-all — local only (§1)** |

**Rule 4 before rule 5 is the discriminator `GUIDE-RESOLUTION` §6.2 describes in prose** — *"a known
registry handle (`@entity-church`) vs a DNS domain (`@example.org`, dotted)"* — expressed as
ordering rather than as a richer matcher. A POSIX glob cannot say "undotted", so `*@*` is written
broad and rule 4 takes the dotted names first. Reversing them sends every domain-scoped name to the
peer-issued backend, which will not resolve it, and the chain then falls through to a **catch-all
that must not see it either.**

**Entries for unbuilt backends are inert, not harmful.** Rules 1–4 name backends that are `paper`
today (`GUIDE-RESOLUTION` §6.4). A dispatch entry naming a backend absent from the resolver-chain
narrows to the empty set and the chain reports `chain_exhausted` — fail-closed, per §4.1 step 4.
**Shipping them now is deliberate:** it reserves the routing so that a name shape which *will* be
web-native does not fall through to the catch-all in the meantime, which is the same leak §1
closes, arriving by a different door.

---

## §3 Dispatch-on-X: `name@peer-id` is a pin, not a target

`GUIDE-RESOLUTION` §6.3 unifies targeting with the link-form's verification pin. **This is not a
sixth glob**, and stating that is the point of the rule:

- **X is a handle or a domain** → *targeting*: resolve `name` **using** authority X. Rules 4–5.
- **X decodes as a V7 §1.5 Base58 peer-id** → *verification pin*: resolve `name` however the chain
  normally would, and the result **MUST equal** X. Reach comes from the name; trust comes from the
  peer-id.

**The discriminator is structural and needs no glob**: peer-ids are Base58 multikey and registry
handles and domains are not, so the resolver tells them apart by decoding. A resolver MUST attempt
the peer-id decode on the authority part **before** glob dispatch, because rule 5 (`*@*`) would
otherwise capture `alice@z6Mk…` and send a private name to a registry to answer a question the
consumer could have answered locally — **§1's leak, one form up.**

**On a pin mismatch the result is refused and the chain advances** (§6a.4's fail-closed rule, same
disposition). A pin that resolves to a different peer is exactly the substitution case
`EXTENSION-REGISTRY` §6a.1a describes, caught one layer higher.

---

## §4 Spec deltas (executable form)

| # | File | Section | Edit |
|---|---|---|---|
| **D1** | `EXTENSION-REGISTRY.md` | §4, `name_format_dispatch` schema | ~~State that the list is **ordered, first-match-wins**.~~ **WITHDRAWN.** What landed instead: the list is a **filter** carrying no precedence — a name matching several entries is eligible at the **union** of their `backend_kinds`, and `resolver_chain[].priority` (§4.1 step 3, §4.1.1) is the sole precedence mechanism. The one surviving half is that a name matching no entry is treated as matching the catch-all. |
| **D2** | `EXTENSION-REGISTRY.md` | §4.1 step 2 | **The catch-all MUST route to local-only backends** (§1). Normative, with the reason stated in one clause: the step already calls itself the primary privacy mechanism, and a catch-all bound to a remote backend discloses every unscoped name a user types. |
| **D3** | `EXTENSION-REGISTRY.md` | new §4.1a | **The recommended default list** (§2's table), marked as the interoperable default a distribution SHOULD ship, with D2's catch-all rule as the one MUST inside it. Deployments MAY override (§6.4a unchanged). |
| **D4** | `EXTENSION-REGISTRY.md` | §4.1, before step 2 | **The authority-part peer-id decode precedes glob dispatch (MUST)** (§3), and a pin mismatch is refused fail-closed with chain advance. |
| **D5** | `guides/GUIDE-RESOLUTION.md` | §6.1, §6.4a | Point the four-shapes table at §4.1a as the normative home; replace §6.4a's *"promoting it to normative = a short proposal"* with the landed pointer. Keep the guide's teaching voice — the table stays, its authority moves. |
| **D6** | `EXTENSION-REGISTRY.md` | §11.1 | Vector **`REG-DISPATCH-CATCHALL-LOCAL-1`** — a resolver-config whose catch-all names a remote backend MUST be refused or normalized at load, and a bare name MUST NOT produce a read against a remote registry. The observable is **the absence of a request**, so the vector asserts on the registry peer receiving nothing. |

**Not in scope:** the glob grammar itself (§4's POSIX choice stands) · richer matching · any backend
implementation · the `reg:` alternative form (documented, not defaulted) · per-backend query privacy
beyond dispatch filtering (§11.4 keeps that deferred).

---

## §5 Cohort impact

| Seat | Owed |
|---|---|
| **`entity-browser-rust`** | **This is B16b, their last arch blocker.** Ship the §2 list as the default. Their instinct to hold rather than ship a second default chain entry was right and is now unnecessary. |
| **`entity-workbench-go`** | Same default list if they ship a resolver config. They have not yet, which is why this arrives before rather than after. |
| **`entity-core-{go,rust,py}`** | D1's ordering rule and D4's decode-before-dispatch are engine behavior. D2/D3 are config content — engines enforce D2 at load, they do not author the list. |
| **oracle / keystone** | D6's vector. **No re-pin** — adds a case, changes no encoding. |

---

## §6 What this proposal does NOT claim

- **Not** that any seat has shipped a leaking default. Both app tiers are pre-ship on this surface;
  that is the entire reason for the timing, and no build-state claim is made about either.
- **Not** that the four shapes are novel — they are `GUIDE-RESOLUTION` §6.1's, unchanged. This pins
  and orders them.
- **Not** a decision about the unbuilt web-native backends' record formats. §6.4's open invitation
  stands; only the routing is reserved.

---

## Addendum — two tokens in the shipped default were never declared `[2026-08-18, folded v1.12]`

**Source:** `entity-workbench-go`, `reviews/RESOLVER-CONFIG-FILTER-CORRECTION-2026-08-18.md` §2, filed
as *"two cohort observations, still inert."* They are inert only because nobody has built the
backends. **They are worse than inert: they are dead config in every conformant peer**, and this
proposal is the document that shipped them.

§2.4.1 is the canonical `backend_kind` vocabulary — declared in the spec as a *"cohort-convergence
pin."* It lists eight kinds. **Neither `did-key` nor `pinned` is one of them**, and §4.2's
forward-compat rule is explicit: *an unknown `backend_kind` MUST cause the entry to be skipped with a
warning.* So the recommended default list this proposal landed contains rows that a conformant
implementation is **required to discard**.

| Row | Was | Now | Why |
|---|---|---|---|
| 2 `did:key:*` | `["did-key"]` | `["self-certifying"]` | The row's own **Shape** column already said *"scheme-typed, self-certifying."* A `did:key:` name carries its key, so the self-certifying backend decodes it. **No new vocabulary is added** — the alternative, declaring `did-key` in §2.4.1, invents a kind for a mechanism that already has one |
| 6 `*` catch-all | `["local-name", "pinned", "peer-issued"]` | `["local-name", "self-certifying", "peer-issued"]` | `pinned` is **structurally unreachable from dispatch** |

**`pinned` is the more instructive of the two.** It is not merely undeclared — it **cannot be a
dispatch target at all.** §4.1 **step 1** returns a pinned match *immediately*, before the step-2
dispatch filter runs; and §4.1.2 uses `pinned` as a **`backend_id`** on the synthesized result. It is
a **result label**, and it was written into a **kind** field. workbench-go put it exactly right:
*"`pinned` has no constant anywhere in the cohort, because pinned bindings resolve at §4.1 step 1,
before dispatch runs."*

**Both survived because they are in a table.** A default-config table reads as data, so it is
reviewed as a list of plausible strings rather than as normative text with referents — and neither
token fails anything until someone builds the backend, at which point the failure is a silently
skipped row rather than an error. **This is the D10 class (*a normative construct may not name a
referent the corpus does not define*) in its quietest form:** a MUST cites a referent and fails
loudly; a table cites one and just does nothing.

**Also recorded, and it is the more valuable half of workbench-go's note.** Both app-tier seats
independently pinned a reading of §4.1a in tests, **and the pins disagreed** — visible in two trees
before the ruling, invisible to both seats. They explicitly did not ask for a mechanism. It is worth
keeping as evidence for what a *default-shipping* spec section costs: this table is the one artifact
here that implementations copy verbatim, so an undeclared token in it propagates by construction.

**Delta (verified):** `EXTENSION-REGISTRY` §4.1a rows 2 and 6; the §4.1 step-2 catch-all table's first
row now names `self-certifying` and notes that a pinned binding never reaches it; a paragraph
recording both corrections. **REGISTRY 1.11 → 1.12.**

**Cohort impact:** ship the corrected rows verbatim. `entity-workbench-go` emits the table verbatim by
design and is the seat that will notice first. No implementation loses a capability — both tokens were
being discarded already.

## Addendum 2 — the grammar is closed, and row 6 moved three times in one day `[2026-08-18, folded v1.13]`

**Source:** `entity-core-go`'s `-h` close-out. They found `path.Match` at their dispatch site, **did not
replace it**, and routed instead: *"a real defect but flagged-not-ruled on a cross-impl-observable
surface, so I routed my reading + the correct grammar rather than ship a fresh name-matcher three impls
would diverge on."* **That is the correct call and it is why this addendum exists rather than a
divergence.**

### The grammar

§4.1 said `*` matches any run of characters including none, and stopped. It never said what a `?`, a
`[`, or a `/` does — so an implementer reaching for the nearest stdlib gets a matcher that answers all
three differently from the spec's intent. Now closed: **every byte that is not `*` is a literal**, any
number of `*` is permitted (`*@*.*` is three and is in the spec's own table), `/` is not a separator,
and the match is anchored at both ends.

**Implementations MUST NOT delegate to a path-glob or shell-glob library.** `path.Match` and `fnmatch`
both grant `?` and `[…]` meaning this grammar does not, and most stop `*` at `/`. **A matcher that
omits those features and one that treats them as literals are indistinguishable until a pattern carries
one** — the `**` lesson, transplanted.

**No pattern is invalid, so there is no write-time rejection**, and that is stated rather than left to
infer: a registry MUST NOT reject a pattern containing `?`, `[`, or `\`. This is a real difference from
`EXTENSION-REVISION`'s four forms, which need a `400` because that grammar *can* be violated — and the
difference is worth naming, because "closed grammar" has meant "reject at write" every other time this
corpus has used the phrase.

`REG-DISPATCH-GRAMMAR-1` ships four rows; **`x*z` matching `x/y/z` is the one that fails against every
path-glob implementation.**

### Row 6 moved three times in one day, and a seat pinned the middle version

| | Row 6 `backend_kinds` |
|---|---|
| before | `["local-name", "pinned"]` |
| `8bfc9b6` | `["local-name", "pinned", "peer-issued"]` — the catch-all re-keyed to name transmission |
| Addendum 1 | `["local-name", "self-certifying", "peer-issued"]` — `pinned` removed as undeclared |
| **v1.13** | `["local-name", "self-certifying", "out-of-band", "peer-issued"]` |

**Addendum 1 removed `pinned` correctly and then failed to put the right token in its place.** `pinned`
is not a kind — but the kind a pin's synthesized binding actually carries **is** `out-of-band`
(§4.1.2's `kind: "out-of-band"`, `trust_anchor: "out_of_band"`), it is declared in §2.4.1, it is
name-blind, and §6a.4 makes it dispatchable in as many words: *"a pin matches only if explicitly
configured as its own chain entry."* So the row lost a legitimate capability for one revision. **The
diagnosis was right and the repair was incomplete**, which is a distinct failure from the one it was
fixing.

**`entity-browser-rust` ratified the middle version.** Their `name_dispatch.rs` pins all six rows as a
test against `8bfc9b6`, landed at `259e685` — hours before Addendum 1 moved row 6 under them. **They are
the second app-tier seat this has happened to in two days**; `entity-workbench-go` shipped
`ValidateResolverConfig` against §4 1.6 and had it withdrawn 78 minutes later.

**That is now a pattern and not an accident, and the cost lands where the spec is copied verbatim.**
§4.1a is the one artifact in this corpus that implementations are *told* to ship byte-for-byte, so every
edit to it invalidates a pin somewhere. workbench-go's observation — that two app-tier seats
independently pinning §4.1a is exactly the signal a default-shipping section wants — is the other half:
the pins are the reason the errors were found at all. **The conclusion is not to edit the table less; it
is that a table change is a cohort event and must be routed as one**, which the three edits above were
not.
