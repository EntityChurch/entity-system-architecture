# ANALYSIS — types inside core-owned namespaces: what is actually wrong, and what changing it would cost

**Date:** 2026-07-31, **revised 2026-08-01** · **Status:** analysis complete; **one upstream proposal
recommended carrying a rename + a type-ref fix**. The 07-31 conclusion ("no migration") was reached by circular
reasoning and is **withdrawn** — see the banner below.
**Question asked:** the 07-31 handoff flagged `system/protocol/inbox/delivery`, `system/protocol/inbox/notification`
(INBOX) and `system/handler/composition` (SYSTEM-COMPOSITION) as **extensions claiming namespaces core owns**, with
no rule covering it. The operator asked: is it an oversight, do we need proposal amendments to fix it, does it
impact keystone peers, and what does changing it cost?

---

> ## ⛔ REVISED TWICE — read this banner and then go to `PROPOSAL-INBOX-TYPE-NAMESPACE-CORRECTION`
>
> **Third pass, 2026-08-01 (final).** Both earlier passes rested on `ENTITY-CORE-MACHINE-SPEC.md` being
> authoritative. **It is not** — it is a generated secondary artifact of this repo that nothing is sourced from
> and that is not kept in sync. With it excluded, the facts are simpler than either pass concluded:
>
> - **`system/protocol/inbox/delivery` and `/notification` are defined only in this repo's extension specs**
>   (`EXTENSION-INBOX.md:80` / `EXTENSION-SUBSCRIPTION.md:86` + `EXTENSION-INBOX.md:94`). **No core spec
>   mentions either type.**
> - **So the original flag was right after all:** an extension *is* defining types inside a namespace whose every
>   core member is a wire message. Pass 1 said "not a violation, core owns them" — that rested on the machine
>   spec and is **withdrawn**. Pass 2 said "the placement defence was circular" — that part stands, and the
>   conclusion it reached (the prefix is wrong) is correct for better reasons than it gave.
> - **No upstream change is needed.** The fix is entirely in this repo's own extension specs.
> - **The `result` "drift"** (`primitive/any` vs `core/entity`) **was never a drift** — just the stale generated
>   copy. **Withdrawn.**
>
> **What survives from the analysis below:** the §9.5 floor-membership test (sourced from
> `ENTITY-CORE-PROTOCOL.md`, unaffected), INBOX §137's "any domain-specific message type" slot argument, and the
> mirrors-`execute/response` trap. **What does not:** every claim about who defines these types, the §3.2/§3.9
> taxonomy argument, the §3.9-siblings argument, and the extension-families-catalogue argument — all four were
> machine-spec observations.
>
> **`system/handler/composition` is unaffected and needs no action** in any pass.
>
> ---
>
> ## ⚠ (superseded) REVISED 2026-08-01 — the first conclusion was reached by circular reasoning
>
> **v1 of this analysis concluded "core defines these types, therefore the placement is correct."** That is
> circular: core's authority to place a type somewhere is not evidence that the placement is right. It justifies
> *any* placement and therefore justifies none. The operator caught it and asked the question v1 never asked —
> **are these protocol messages or not?**
>
> **They are not, and the evidence is unambiguous.** §1 below is rewritten; the recommendation **inverts** from
> "no migration" to "route a rename upstream." The ownership findings in §2–§3 stand (they were about a different
> question), and the `result` drift in §1.1 stands.

## 0. The question, asked properly

The handoff flagged `system/protocol/inbox/delivery`, `system/protocol/inbox/notification` and
`system/handler/composition` as **extensions claiming core-owned namespaces**. Two findings, and they are
independent:

| | Verdict |
|---|---|
| **Is an extension claiming a namespace it doesn't own?** | **No** — core defines the inbox types itself (`ENTITY-CORE-MACHINE-SPEC.md` §3.9), and `SYSTEM-COMPOSITION.md` is a core-model spec, so `system/handler/composition` is core extending core. **The flag, as worded, is not a violation.** |
| **Are the inbox types in the right namespace at all?** | **No.** This is the question the flag should have asked, and the answer is the opposite of the first. See §1. |

`system/handler/composition` survives both questions — `system/handler` holds handler-describing types and
`composition` describes a handler. **No action.** The rest of this document is about the inbox pair.

## 1. `system/protocol/inbox/*` is misnamed — four independent lines of evidence

**The criterion, derived from the namespace rather than assumed.** Enumerate every member of
`system/protocol/**` and state what they share:

| Type | In V7 §9.5 Core Type Floor? |
|---|---|
| `system/protocol/envelope` | ✅ |
| `system/protocol/execute` | ✅ |
| `system/protocol/execute/response` | ✅ |
| `system/protocol/error` | ✅ |
| `system/protocol/resource-target` | ✅ |
| `system/protocol/connect/hello` | ✅ |
| `system/protocol/connect/authenticate` | ✅ |
| `system/protocol/inbox/delivery` | ❌ |
| `system/protocol/inbox/notification` | ❌ |

**`system/protocol/**` means: the wire messages a peer exchanges on the transport, and the structural components
of those messages.** Seven of nine members satisfy it and are in the 53-type Core Type Floor. **The only two that
are not in the floor are the two under question** — a perfect discriminator that nobody had to invent.

**Evidence 2 — the machine spec's own taxonomy contradicts the prefix.** §3.2 is titled **"Protocol Types"** and
declares exactly the five `system/protocol/*` types. The inbox pair is not in it. They are declared in **§3.9,
"Inbox & Subscription Types."** The upstream spec sorts them *away* from protocol types and then names them
*as* protocol types.

**Evidence 3 — their own siblings disagree with them.** §3.9 declares six types. Four use
`system/subscription/*` (`system/subscription`, `/request`, `/limits`, `/cancel`); two use
`system/protocol/inbox/*`. **One section, one subject area, two conventions.** And `notification`'s fields are
`subscription_id`, `event`, `uri`, `hash`, `previous_hash` — it *is* a subscription event, sitting beside
`system/subscription/request` under a different prefix.

**Evidence 4 — they travel as cargo, not as framing.** `EXTENSION-INBOX.md` §137 is explicit: the `receive`
operation *"accepts any typed entity. The entity's type carries the semantic information —
`system/protocol/inbox/delivery` for async operation results, `system/protocol/inbox/notification` for
subscription events, `system/protocol/execute` for raw deferred dispatch, **or any domain-specific message
type**."*

**That last clause is decisive.** `delivery` occupies **exactly the same structural slot as an application's own
message types** — a chat message goes in that slot. The slot does not make a chat message a protocol type, so it
does not make these ones protocol types either. The wire message here is the EXECUTE that carries the payload;
the payload is not a wire message.

**The trap worth naming, because it is why this looks wrong to fix.** `delivery` carries
`{original_request_id, status, result}` — it is shaped almost exactly like `system/protocol/execute/response`,
which *is* a wire message. **It mirrors a protocol message without being one.** `execute/response` is framed by
the transport; `delivery` is application payload inside an EXECUTE. Same shape, different layer — and the name
asserts the resemblance while hiding the difference.

**Evidence 5 (added on review — the decisive one).** The machine spec **already catalogues extension types**, and
every other family sits in **its owning extension's** namespace:

| Section | Namespace | Owner |
|---|---|---|
| §3.8 Tree Extension Types | `system/tree/*` | EXTENSION-TREE |
| §3.10 Compute Types | `compute/*` | EXTENSION-COMPUTE |
| §3.11 Content Types | `system/content/*` | EXTENSION-CONTENT |
| §3.9 subscription half | `system/subscription/*` | EXTENSION-SUBSCRIPTION |
| **§3.9 inbox half** | **`system/protocol/inbox/*`** | **INBOX** |

**The inbox pair is the sole exception in the whole document.** This also disposes of the follow-on question
*"should core define these at all?"* — **cataloguing is not owning a namespace.** Core catalogues `compute/*`
without claiming compute is protocol. A complete type catalogue is by design; the prefix is the only defect.

**Conclusion.** The prefix is wrong. It is not a namespace *claim* (core owned it and placed it), it is a
namespace **error** — and "core put it there" was never a reason it belongs there.

**How it likely happened:** `entity-core-py` carries a `system/callback/* → system/protocol/inbox/*` rename note
at v7.8. The types were renamed *into* this namespace, which suggests a rename that fixed one problem (the vague
`callback`) and introduced this one without the placement being re-examined.

**Threshold judgment, stated honestly:** the operator's instinct was *"not sure it crosses the threshold but it
may — core protocol being aware of them and providing guidance doesn't justify the location."* **That is exactly
right, and the floor test is what converts the instinct into a decision:** core is aware of many type families
(bootstrap, identity, handler, connection, tree, bounds, compute, content) and gives guidance on all of them
without putting any in `system/protocol/**`. Awareness is not membership.

## 1.1 A second and independent defect — a duplicated definition that has already drifted

*(This one was found by v1 and stands unchanged. It is orthogonal to the naming question above: the definition
would be duplicated and drifted wherever the type lived.)*

`system/protocol/inbox/delivery` is defined **twice**, in two repos, and the two copies **disagree**:

| Source | `result` field |
|---|---|
| `entity-core-protocol` — `ENTITY-CORE-MACHINE-SPEC.md` §3.9 (**owner**) | `{type_ref: "primitive/any"}` |
| this repo — `EXTENSION-INBOX.md` §2.1 (**reproduction, unmarked**) | `{type_ref: "core/entity"}` |

`EXTENSION-INBOX.md` re-declares the full type — not a behavioral note, a competing definition — and carries **no
"canonical there" pointer**. `system/protocol/inbox/notification` is likewise re-declared (§2.2), though its
fields still agree.

**This is exactly the failure shape the 07-31 handoff §7 named** — *"a spec reproducing an upstream type without
naming who owns it"* — the same one that had just been fixed for `system/protocol/error` in `SDK-OPERATIONS`. It
was found again here within a day, in a spec nobody was auditing, which is evidence the discipline needs to be
**normative rather than remembered**. It now is: `SPECIFICATION-FORMAT.md` **§8.4.2**.

**Which reading is right: `core/entity`.** `EXTENSION-INBOX.md` §4.1 and §322 argue it explicitly — `result`
carries a full inline entity `{type, data, content_hash}` so entity identity survives the delivery chain,
consistent with `EXECUTE_RESPONSE` (`ENTITY-CORE-PROTOCOL.md` §3.4). Upstream's `primitive/any` is the weaker,
older statement. **The fix belongs upstream**, not here — see §4.

**Why it has not bitten yet** (and why it would have, later): `primitive/any` **admits** `core/entity`, so a
permissive implementation interoperates with a strict one **as long as every emitter happens to send a full
entity**. The drift only surfaces when some peer takes `primitive/any` literally and puts a raw value in
`result` — at which point a consumer expecting `{type, data, content_hash}` breaks, across a peer boundary, with
both peers conformant to *a* committed spec. That is the classic latent-interop shape.

## 2. What the implementations actually did (read from source, not inferred)

| Impl | Where the type lives | `result` shape |
|---|---|---|
| **entity-core-go** | `core/types/delivery.go` — **core layer** | `cbor.RawMessage` — structurally permissive, accepts either reading |
| **entity-core-rust** | `core/types/src/core_types.rs` — **core layer** | `t("core/entity")` — **followed the extension spec, not upstream** |
| **entity-core-py** | `entity_core/protocol/delivery.py` — **core layer** | carries a `system/callback/* → system/protocol/inbox/*` rename note (v7.8) |

**Two things this settles.** First, **all three impls place these in their core layer**, which independently
confirms core ownership — nobody treated them as extension types. Second, **Rust implemented the extension spec's
`core/entity` rather than upstream's `primitive/any`**, so the stricter (and correct) reading is already the built
one. Fixing upstream to say `core/entity` **ratifies what exists** rather than asking anyone to change code.

## 3. Keystone impact: negligible — the operator's instinct was right

The concern was that keystone peers might be affected. **They are not.** All 22 generated-language conformance
reports carry these types in the *absent* list, with an explicit disposition:

> `system/protocol/inbox/delivery absent — outside V7 v7.72 §9.5 Core Type Floor — matched-if-present under
> --profile core, not-a-FAIL-if-absent`

**Keystone peers do not implement these types**, they sit outside the §9.5 Core Type Floor, and their absence is
not a failure. 44 report files mention them (92 occurrences each of the two types) — but every mention is a
**generated artifact listing a type as absent**, regenerated by a tool run, not hand-maintained.

**Corrected cost estimate:** a keystone-facing rename would be a regeneration, not a migration. *(Flagged as a
cost claim per the standing rule — arch's cost estimates have been wrong five times this arc. This one at least
rests on reading the reports rather than assuming.)*

## 4. Recommendation — one upstream proposal carrying two corrections

Both defects live in the same upstream section, are found by the same read, and touch the same lines in the same
three implementations. **Route them as one change, not two** — splitting them doubles the cohort's cost for no
benefit.

**To `entity-core-protocol` (the owner). This repo cannot make either edit** — the types are upstream's, and
implementations implement the spec rather than define it.

1. **Rename out of the wire namespace.** Recommended targets, following §3.9's own existing convention:
   - `system/protocol/inbox/notification` → **`system/subscription/notification`**. Highest-confidence half:
     its fields are subscription fields, and it lands beside `system/subscription/request` / `/limits` /
     `/cancel`, which are already declared in the same section.
   - `system/protocol/inbox/delivery` → **`system/inbox/delivery`**.
     *(An earlier draft flagged a conflict with the mailbox **tree paths** `system/inbox/network` /
     `system/inbox/local`. **That objection does not survive the precedent:** `system/tree` is simultaneously a
     handler path (`"system/tree"`) and a type prefix (`system/tree/path`, `system/tree/snapshot`), and the
     corpus shares these spaces throughout. Withdrawn.)*
2. **Align the `result` type-ref to `core/entity`** (§1.1). One line, and it ratifies what Rust already built.

**Cost — bounded, and this is the part that makes the rename affordable:**

- **Not a locked-wire-core change.** Neither type is in the V7 §9.5 Core Type Floor, no opcode is renumbered, and
  the framing is untouched. `AGENTS-STANDARD`'s "the locked wire core is never renumbered" does not bite.
- **Keystone: regeneration, not migration.** All 22 generated peers list these as absent / `not-a-FAIL-if-absent`;
  the 44 report mentions are generated artifacts.
- **Three impls: a constant plus its references**, all in the core layer (`core/types/delivery.go`,
  `core/types/src/core_types.rs`, `entity_core/protocol/delivery.py`) plus inbox/subscription handler and test
  call sites. Mechanical, but real, and **it is the cohort's cost to size, not arch's** — the four-times-wrong
  record on arch cost estimates applies here.
- **Timing:** cheaper now than at any later point, and the reason is specific — **no keystone peer implements
  these types, and py has not built the punch path that would multiply the call sites.** The population that
  would have to migrate only grows.

**Honest counter-argument, recorded so the decision is made and not assumed:** the rename buys **no runtime
behavior**. It buys a namespace whose meaning holds, which is worth real cost only because these prefixes are
what every future author reasons from — and this arc has now produced two separate errors (a wrong flag, then a
circular defense of it) that a correct prefix would have prevented outright. If upstream judges the churn not
worth it, **the fallback is to state the exception explicitly in §3.9** — "these two are not wire messages
despite the prefix; the prefix is historical" — so the next reader is not misled. **Silence is the one option
that is clearly wrong.**

**Fix here (this repo's lane), gated on the above:**

- **`EXTENSION-INBOX.md` §2.1/§2.2 MUST stop restating the definitions** and reference
  `ENTITY-CORE-MACHINE-SPEC.md` §3.9 as canonical, keeping only the behavioral notes that are genuinely INBOX's.
  Held until upstream lands so the two do not cross in flight.
- **Done:** `SPECIFICATION-FORMAT.md` **§8.4.2** now makes ownership, reference-not-restate, **and the placement
  test** normative.

## 5. The generalizable lesson — two of them, and the second is the expensive one

**1. A namespace audit that asks "who owns this path" cannot find a misplaced type.** Both flagged paths are
owned by their definers, so an ownership audit returns clean — while a duplicated, drifted definition and a
mis-prefixed type both sit there untouched. **The question that finds this class is: what do this namespace's
existing members have in common, and does the new type have it?** That test is now folded (§8.4.2) and it is
mechanical — enumerate, state the shared property, check membership.

**2. "The owner put it there" is not evidence, and it is a seductive non-argument.** v1 of this analysis
concluded the placement was correct because core had made it. That reasoning validates every placement ever made
and therefore validates none — and it read as *rigorous* because it was grounded in a real finding about
ownership. **The tell: a conclusion that could not have come out differently no matter what the evidence said.**
Ownership answers *may they*; it never answers *should they*. §8.4.2 now says so in normative text, because this
failure is going to look reasonable again.

---

**Routed upstream (`entity-core-protocol`):** one proposal — rename the inbox pair out of `system/protocol/**`
+ align `result` to `core/entity`.
**Folded:** `SPECIFICATION-FORMAT.md` §8.4.2 — ownership, no-restatement, **and the namespace placement test**.
**Open here:** `EXTENSION-INBOX.md` §2.1/§2.2 reference-not-restate rewrite, gated on upstream.
**No action:** `system/handler/composition` — passes both questions.
