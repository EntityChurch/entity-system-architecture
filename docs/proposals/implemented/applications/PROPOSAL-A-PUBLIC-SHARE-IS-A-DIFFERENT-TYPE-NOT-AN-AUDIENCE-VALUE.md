# PROPOSAL — a public share is a different type, not an audience value

**Status:** IMPLEMENTED 2026-09-09 — folded into `APP-CONVENTION-SHARE` v0.2 (D1–D4, and SHARE-7/8/9 added to §8)
**Target:** `specs/applications/APP-CONVENTION-SHARE.md` — a new §2.5, the §2 type table, and one
sentence each in §2.2 and §2.4
**Raised by:** the browser implementation, which ships a public share as its default and has no
carrier for it

---

## §1 The question, as asked

> *Is a public share out of scope for `APP-CONVENTION-SHARE` v0.1 by design, or is `audience` missing
> a non-enumerated form?*

**Answer: neither.** It is in scope, `audience` is not missing a form, and **a public share is a
different entity type.**

---

## §2 Why `audience` must not carry it

`audience-entry.grantee` is a `peer-id`, and `audience` is *an enumeration of authorized wielders,
each with its own minted token* — §3 materializes a group audience as one entry per member precisely
so that each has a token of its own.

**A public share has no wielders and no tokens.** There is nothing to enumerate. Every proposed way
of expressing it inside `audience` breaks something that is already load-bearing:

| Proposed shape | What it breaks |
|---|---|
| **empty `audience`** | §2.2 already assigns the empty array a meaning — *"an authored share with no members yet"*. That is the self-only case, and it is a **different state** from public |
| **a sentinel entry whose `grantee` is the policy table's `default`** | makes `grantee` accept a value that is not a `peer-id`, so the field's declared type stops being true, and every consumer must special-case a magic value before using it |
| **a sibling flag (`open: bool`, `audience-kind`)** | leaves two fields able to disagree — an `open` share with a populated `audience` has no defined meaning, and the schema cannot forbid it |

---

## §3 Why a separate type is the right answer, and the document already argued it

**The test for minting a type is behavioural: a new type is warranted when a conformant consumer must
*behave* differently.** A public share clears it on every axis that matters here:

| | direct share | public share |
|---|---|---|
| Consumer must find its own binding and present a token | **yes** | **no — there is no token** |
| Publisher knows who the audience is | **yes** — they minted it | **no, and cannot** |
| Authorization at fetch | grant-checked | **none required, pull-only** |

**And this convention has already made exactly this call once, deliberately.** §2.4's warning box
separates `app/share/follow` from `app/feed/follow` on the same two axes — *authorization* and *does
the publisher know* — and its stated reason for refusing to unify them is the one that governs here:

> *"They are not unified because doing so would require making the `record` field optional, which
> changes what an absent field means in an already-landed schema."*

**Making `audience` express "public" is the same move on the same document**, and it changes what an
absent or empty `audience` means in an already-landed schema. The convention's own reasoning
forbids it.

---

## §4 The delta

> **§2.5 `app/share/publication` `[new]`** — a titled offer of content with **no audience**. It is
> `share-record` **minus `audience`, and nothing else differs**, which is the ruling stated as a
> schema: the audience model is the only axis this split is on, so it must be the only field that
> moves.
>
> ```cddl
> share-publication = {
>   type: "app/share/publication",
>   data: {
>     title:      tstr,                ; human-facing label; NOT an identifier
>     target:     share-target,        ; what is offered — the SAME atom as §2.2, tagged
>     ? note:     tstr,                ; optional human-facing description
>     created_at: uint                 ; ms since epoch, publisher's clock
>   }
> }
> ```
>
> - **Carries no `audience` and no grant.** Authorization at fetch is **none required, pull-only**.
> - **A consumer needs no token and performs no binding lookup.** Reaching the bytes is the whole
>   protocol.
> - **The publisher cannot know who fetched it**, and MUST NOT be written as though it can.
> - **`target` is `share-target` verbatim — `blob-target` OR `prefix-target`.** A publication of a
>   subtree is as ordinary as a publication of one blob, and narrowing this field to a bare
>   `content-hash` would make the public path the only one that cannot offer a subtree. §2.2's
>   note on why `share-target` is not an `entity-ref` applies here unchanged and for the same
>   reason: the publisher is the sharer, so the authority term is already known.

**Two sentences added to landed sections, and the second is not optional.**

> **§2.2** — *`app/share/record` is direct-audience-only. An offer with no audience is
> `app/share/publication` (§2.5), not a `record` with an empty `audience` — the empty array remains
> "authored, no members yet".*

> **§2.4** — *`app/share/follow`'s `record` field names an `app/share/record` and MUST NOT name an
> `app/share/publication`.* Without this the split recreates precisely the failure §2.4's own
> warning box exists to prevent: a type-filtered query on the wrong tag returns a correct,
> complete, **empty** answer, and nothing in either schema forbids the mistake.

---

## §5 What this resolves downstream, and it is the reason to do it now

**It unblocks the live cross-implementation divergence, and it resolves it differently from the
obvious fix.** One implementation currently emits a share type that this convention does not define;
the apparent remedy was to rename it to `app/share/record`. **That would have been wrong** — that
implementation's own default is a public share, and `record` cannot express one, so the rename would have
produced entities claiming to be direct shares while carrying no audience.

**With this proposal the migration is a split, not a rename:** public shares become
`app/share/publication`; genuinely audience-scoped shares become `app/share/record`. **The
implementation that raised the question had already reached this conclusion from the other end** — that the rename alone
would be worse than the mismatch — which is why they asked instead of proceeding.

---

## §6 The noun — `offer` is unavailable, and the reason is the finding

**This draft first proposed `app/share/offer`. It was put to both implementing front-ends as an open
question, and the answer that came back is stronger than a preference: the word is already in both
trees, and it means opposite things in them.**

| | one front-end | the other |
|---|---|---|
| what it calls an "offer" | a share that **names specific peers**, from which a policy row is derived, **pending until the receiver accepts** | a manifest whose audience is **`Public`** — its own doc says *"today's offers are all of these"* |
| in this convention's terms | an `app/share/record` with a **non-empty** `audience` | exactly the type this proposal is minting |

**So `offer` is not merely taken — it is already ambiguous across the two implementations, on the single axis
this split exists to separate:** *is anyone named, and is anything granted.* Minting
`app/share/offer` would freeze one reading into the corpus and silently invalidate the other, in a
document whose §2.4 already warns that this exact class of mistake fails as a **complete, empty
answer** rather than an error.

**`listing` is worse and was almost taken on an unchecked claim.** It was offered as the leading
alternative on the grounds that it *"collides with nothing in either tree"*. It collides with the
substrate: `listing` is `EXTENSION-TREE`'s own return type — `get` yields *`entity? | listing`* — and
both front-ends carry it as a core-facing noun in the hundreds of occurrences. **A convention that
reuses a core return type as an application tag is the `app/share/*`-versus-`system/*` error running
the other direction.**

**Ruled: `app/share/publication`.** It is a noun, it is free as a tag in the corpus and in both
implementations, and it is accurate about the one thing that distinguishes the type — the content is
published, to no one in particular. Its cost is length, which is the cheapest of the costs on offer
here.

> **The transferable part is not the name.** Two independent front-ends had each shipped the word,
> neither knew the other had, and both were asked before the type was minted. **A vocabulary
> collision is only cheap to find while the proposal is open**, and the instrument that found it was
> asking the implementers rather than reasoning about English.

## §6a The remaining open questions

1. **Does a publication need a `follow` counterpart? No, and not for the reason this draft first
   gave.** The earlier recommendation pointed at the feed convention's namespace follow. **Opening
   that section rather than the warning box that summarizes it shows why that does not transfer:**
   `app/feed/follow` carries a `cursor` that is *"the last index page this reader applied"* — feed
   machinery, not a generic namespace subscription. **The correct answer is that no entity is needed
   at all.** A publication requires no authorization, and §2 already states the retrieval mechanism —
   cross-peer aggregation is *a `type_filter` query over the universal tree with no peer filter*, and
   the type tag **is** the index key. A reader wanting a publisher's publications issues that query.
   **What a follow entity would add is a persisted intention plus a cursor, and that is the general
   refresh-loop question**, which is in flight across several implementations and MUST NOT be minted
   piecemeal inside a share convention.
2. **Does the group-audience join gap (§3) touch this?** It should not — a publication has no
   members — but §3's join-side gap should be re-read once this lands to confirm it says nothing
   about the audience-less case.

---

## §7 Deltas

| # | Target | Change |
|---|---|---|
| **D1** | `APP-CONVENTION-SHARE` §2.5 **(new)** | the `app/share/publication` type, `share-record` minus `audience` |
| **D2** | `APP-CONVENTION-SHARE` §2.2 | one sentence: `record` is direct-audience-only; empty `audience` stays "no members yet" |
| **D3** | `APP-CONVENTION-SHARE` §2 type table | add the new tag *(the table is in §2, not §1 — an earlier draft of this row named the wrong section)* |
| **D4** | `APP-CONVENTION-SHARE` §2.4 | one sentence: `app/share/follow`'s `record` field is record-only and MUST NOT name a publication |

**Required checks — what an implementation must discriminate** *(the artefacts are the
implementations', per this tier's standard):* an `app/share/record` with an empty `audience` is
**self-only, not public** · an `app/share/publication` carrying an `audience` field is **invalid** · a
publication whose `target` is a `prefix-target` is **valid and ordinary** · a
consumer fetching a publication presents **no token** and is not refused for lacking one.

---

## §8 Fold record

**Folded complete into `APP-CONVENTION-SHARE` v0.2.** All four deltas landed and the conformance
section grew three cases, so nothing here is partial and this proposal is closed rather than held.

| Delta | Landed as |
|---|---|
| **D1** | §2.5 `app/share/publication` — `share-record` minus `audience`, `target` unchanged |
| **D2** | §2.2 — `record` is direct-audience-only; the empty array keeps *"no members yet"* |
| **D3** | §2 type table — the new tag, and `follow`'s role clarified to *record* |
| **D4** | §2.4 — `record` is record-only and MUST NOT name a publication |
| *(new)* | §8 SHARE-7 / SHARE-8 / SHARE-9 — the empty-audience state, the new type's shape in both directions, and the pull-only posture |

**The noun was settled by asking both implementing front-ends and then reading their source rather
than taking either account.** Both ship `offer` and mean opposite things by it; the leading
alternative `listing` is the substrate's own return type. §6 carries the evidence.

**One case the checks name and no implementation exercises yet:** a publication with a
`prefix-target`. It is the case the first draft could not express at all, and it is the one to watch
when the first implementation lands.
