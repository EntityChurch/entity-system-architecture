# PROPOSAL — `{window_id}` is a session-scoped slot address, and the set of windows needs a home

**Status:** DRAFT (2026-08-31)
**Target:** `guides/GUIDE-ENTITY-WORKBENCH-APP.md` §1, §3, §4.2, §4.2a (new), §8 ·
`guides/GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §5.1 · `guides/GUIDE-SDK-PATTERNS.md` §2 ·
`guides/GUIDE-SHELL-FRAMING.md` §7.1
**Provenance:** filed by `entity-browser-rust` as
`docs/architecture/reviews/PROPOSAL-WINDOW-ID-LIFETIME-AND-THE-WINDOW-INDEX.md`, against arch
`94855c2`, with a built implementation (`src/window_index.rs`, browser-rust `2a1725a`) and a
falsified two-arm gate. Their `docs/SPEC-AMBIGUITIES.md` §1 filed the same thing as a missing
warning and withdrew that framing.
**Seats measured for this proposal:** `entity-browser-rust` `2a1725a` (source read) ·
`entity-workbench-go` `bf48191` (source read — **not** the `b9feae5` the filing seat wrote
against; HEAD moved) · `entity-core-rust` `bindings/sdk/src/sdk.rs` (doc-comment path examples
only) · `entity-core-{go,py}`, `entity-core-keystone`, `entity-core-formalization` — searched,
**zero** occurrences of `workspace/windows` or `app/state/window`.

---

## §0 What is ruled, in one page

The filing seat's finding is **upheld and generalized**. Four of its five asks land; one lands
much larger than filed; two of its supporting claims are wrong and are corrected here without
moving the conclusion.

| | Ruling |
|---|---|
| **Q1** `{window_id}` lifetime | **Session-scoped slot address.** Derived from §8 and §3.2, not from what the impls do |
| **Q2** does the roster slot exist | **Yes — and it is not new.** `app/state/layout` has sat unclaimed in two canonical guides that §4.2 does not know about. That row is retired; `app/state/window-index` is added |
| **Q3** what it carries | `{id, content_type, peer_id}` — the corpus's own spellings, snake per `STYLE-NAMING-CONVENTIONS` |
| **Q4** the `app/state/window` fallback | **The load-bearing item, not "one small thing."** On the fallback the type discriminator does not exist, so the filing seat's own mitigation is structurally unavailable to anyone who takes the guide's easy on-ramp. `content_type` becomes required |
| **Q5** the no-roster fallback rule | **Accepted, as a checkable disjunction** rather than a bare SHOULD: index, or sweep, or do not take the persist arm |
| **Q6** §1 vs §6 | **A contradiction in our own text, which the filing seat's §1 argument rests on.** §1 tiers the action wire shape `MUST`; §6 says it is speculative and impls are `NOT REQUIRED`. §6 owns the surface and wins; §1 is corrected |
| **Q7** `GUIDE-SHELL-FRAMING` §7.1 | Its `Selection` example emits source attribution that §5.4 rule 1 forbids, spells `kind` where §5.4 pins `type`, and embeds a `window_id` that Q1 makes meaningless to a later reader. Rewritten |

**Nothing in the wire core. Nothing renumbered. No path re-keyed. No version bump** — these are
guides, which carry `Status:` and not a version header.

---

## §1 Q1 — `{window_id}` is a session-scoped slot address

**Derived from the corpus, first.** The filing seat notes that both impls made it a per-session
counter. That is **corroboration, cited last**, per L18 — it is our own cohort, and if both impls
had chosen the other reading the argument below would be unchanged.

1. **§8 puts per-window state on a conditional arm** — *"persist if the application offers session
   resumption; MAY be ephemeral otherwise."* A durable key is a key to a slot that is there. An
   identifier whose slot the guide explicitly permits to not exist cannot be the corpus's durable
   handle for a window; the guide has already said the slot is optional, and it did not except the
   key from that.
2. **§3.2 locates durable identity in the type, not the position.** Path-position-invariance says
   the state entity *"at `…/windows/3/state` … carries the same fields as"* the one at
   `…/screens/0/windows/3/state` **of the same type** — the invariant is carried by the type name,
   and the ordinal is explicitly the part that moves. A field the guide designs to be relocatable
   is a position, not an identity.
3. **§3 gives `{app-id}` a claim sentence and `{window_id}` none, in the same section.**
   *"`{app-id}` is a **per-instance identifier** that the app claims by writing to it."* Nothing
   comparable is said of `{window_id}`. Where the guide meant an identifier to be durably claimed
   it said so, four paragraphs above.
4. **§4.1.1's promotion ladder and §4.2's slot table key portability on type names throughout.**
   No cross-impl contract anywhere in the guide is keyed on a window ordinal.

**Corroboration (last, and it changes nothing above):** `entity-browser-rust`'s
`WindowManager::new` restarts at 1 each boot; `entity-workbench-go`'s `console/workspace.go:88`
does `ws.nextID++` on a `workspace` struct freshly zero-valued by `newWorkspace` each process
start. Two impls, independently, session counters.

**So the guide is not ambiguous — it is silent, and the silence reads as permission for the one
use the field cannot support.** Saying it plainly is most of the value, exactly as filed.

---

## §2 Q6 — the contradiction underneath the filing seat's §1, found on the way

The filed proposal opens its tension with *"**§1 conformance table — MUST:** action wire shape
with `(window_id, event, value)` semantics."* That row exists. **§6 retracts it:**

> *"the wire shape `(window_id, event_name, value)` **are speculative / schema-anchor** … NOT a
> load-bearing runtime requirement today"* … *"Implementations are **NOT REQUIRED** to emit named
> events."*

**§1 is an index; §6 owns the surface, and §6 wins** — L19's rule, and §6 is additionally the
later text, absorbed from the cross-impl alignment cycle, carrying a measured three-impl negative
(*"no implementation … currently emits named action events as distinct wire-shape triples"*).
§1's row is stale and is corrected to SHOULD.

**The finding survives losing this leg, and is cleaner for it.** The action shape is why a window
id is *small and dense*. It was never why the id is *unstable* — the instability is that nothing
constrains the lifetime, which is Q1 and stands on its own. An argument that needed the MUST would
have died here; this one does not.

*Recorded because it is the same trap L8's fourteenth form names: a conclusion that survives its
own justification being wrong is the hardest kind to notice, because nothing downstream breaks.*

---

## §3 Q2 — the slot already exists, in two guides the authority does not read

The filing seat proposed a new roster slot. **Before minting one, the corpus was searched (L7).**
`app/state/layout` is declared in **`GUIDE-PEER-CONCERNS-AND-NAMESPACES` §5.1** and
**`GUIDE-SDK-PATTERNS` §2 Convention Types**, both canonical and both published. It appears
**nowhere else in the corpus** — no schema, no prose, no consumer — and, decisively, **it is absent
from `GUIDE-ENTITY-WORKBENCH-APP` §4.2**, the table §4.1.1 designates as the authority
(*"The slot table (§4.2) enumerates the current cross-impl canonical types"*).

**This is L23 on our own text: one rule, three homes, one updated.** §4.2's three-impl consensus
assigned arrangement — *"slot/split ratios"* — to **`renderer-specific decoration … per-impl, not
portable`**. That ruling retired the layout slot and nobody removed the row from the two sibling
guides, so the corpus publishes a canonical type name that its own authority has decided against.

**Ruling: retire `app/state/layout`, do not repurpose it.** Renaming a row two guides gloss as
*"Layout"* into a window roster would mislead every reader who met it there first. And the
distinction is the useful part:

> **Membership is portable; arrangement is not.** *Which windows exist, of what content type, on
> whose peer* is a fact about application state and belongs in the tree. *How they are split,
> sized, and stacked* is renderer decoration and stays per-impl, per §4.2. The new slot carries
> the first and must not grow the second.

**`app/state/window-index`** is added to §4.2, instances at
`app/{app-id}/workspace/window-index`. The name is unused anywhere in the corpus (searched).

**This also answers the "cosigned MUST" the filing seat withdrew a design on — see §5.**

---

## §4 Q3/Q4 — what the index carries, and the fallback that has no discriminator at all

### §4.1 The index schema

Type `app/state/window-index`, one entity, an array of rows:

```
windows        [ { id, content_type, peer_id } ]     the live window set

id             uint          the session-scoped slot address (§3)
content_type   text          what this window is showing. OPAQUE — see below
peer_id        text          which peer's tree this window is bound to. Empty = the host peer
```

**Field names are the corpus's own, not new ones.** `peer_id` is §5.4's spelling; `content_type`
matches §4.2's `app/state/{content_type}` vocabulary; all three are snake per
`STYLE-NAMING-CONVENTIONS` §2 (*"Field / map-key name — snake_case"*). This **corrects two live
divergences**, both cheap today and expensive after a second impl ships resumption:

| Seat | Writes | Should write |
|---|---|---|
| `entity-browser-rust` `2a1725a` `window_index.rs` `to_entity` | `{id, type, peer}` | `{id, content_type, peer_id}` |
| `entity-workbench-go` `bf48191` `workspace_state.go` `SaveWindowContent` | map key `"content-type"` | `"content_type"` — kebab is the namespace axis, not the key axis |

**`content_type` is opaque and does not wait on the catalog.** The filing seat's scoping is
adopted verbatim and is right: the value is a string the writing app uses to find its own factory.
Cross-impl agreement on the *value* would only matter if one impl reconstructed another's windows,
which nobody is asking for. So this does **not** depend on §11 item 1 (the content-type catalog,
in flux).

**This does not un-retire §5.4's `content_type`.** §5.4 rule 1/2 forbid `content_type` in
**`app/state/selection`** payloads, and the reason is specific — it was *source attribution*, made
unnecessary by the per-panel-slot model. Here the field names **the window's own content type**,
which is the fact the index exists to record. Different slot, different subject. Readers gating on
legacy fields must scope that gate to `app/state/selection`, which `entity-workbench-go`'s
`legacySelectionFields` already does correctly.

### §4.2 The fallback is where this actually bites — and it is not a small thing

The filing seat filed this as *"§5.4 One small thing in the slot table."* **It is the largest item
in the set**, and the reason is structural rather than a matter of one missing field.

Their mitigation for cross-window adoption is a type-discriminator check —
`if entity.entity_type != STATE_TYPE { return no persisted state }` — which works because they use
the **long-term** slot, `app/state/{content_type}`, so each window type's state entity carries a
distinct `entity_type`.

**On the transitional slot that discriminator does not exist.** Every window's state entity is
typed `app/state/window`, so the field is identical for all of them and there is nothing to
compare. Verified live in `entity-workbench-go` `bf48191`: `updateWindowState`
(`entitysdk/workspace_state.go:426`) is read-modify-write — it reads the persisted map at
`windows/{id}/state`, which on a fresh session was written by **whatever window held that ordinal
last session**, merges the new key, and writes it back under the same `app/state/window` type.
Stale keys accumulate under the new window's identity, and `SaveWindowContent` overwrites the one
field that would have revealed it.

**So the guide's easy on-ramp is strictly more dangerous than its long-term shape, and says
nothing about it.** §4.2 offers the fallback to *"let new windows land before T2 ratifies their
schema"* — sound reasoning that hands the taker a slot with no discriminator.

**Ruling: `app/state/window` MUST carry `content_type`.** This is a **rename, not new work**, for
the one impl on the fallback — `entity-workbench-go` already writes exactly this fact, at
`console/application.go:284` and `:303`, under the kebab spelling. *An impl reached for the field
before the guide had it, which is the best evidence available that the shape is right.*

---

## §5 The withdrawal is upheld; its reasoning is replaced

The filing seat considered re-keying the path to `windows/{type}/{ordinal}/state` and **withdrew
it on finding the path is a cosigned MUST.** The module doc-comment repeats this — *"without
touching a cosigned MUST"* — and it is the stated reason the index exists in the shape it does.

**The MUST they withdrew on does not say that.** §1 tiers *"`app/{app-id}/workspace/...`
**namespace prefix**"* — the prefix. The cosigned block is §4.1.1's disambiguation, which
separates the **type-name** namespace from the **instance-path** namespace and, on the path side,
pins only that instances live *"under `app/{app-id}/...` paths per §3."* Neither pins the segments
beneath `workspace/`. §3.1 settles it outright by blessing two different sub-shapes — flat
`workspace/windows/{id}/state` and nested `workspace/screens/{id}/windows/{id}/state`. Both seats
already extend under the prefix without objection: `entity-workbench-go` `bf48191` carries
`workspace/panels/{panelID}/selection` and `workspace/shells/aliases/{alias}`, neither in §3's
listing.

**The withdrawal is nonetheless correct, for reasons that are about the design:**

1. It does not solve the case that motivated it. Two windows of the same type on one peer collide
   on the ordinal exactly as before; the problem moves one segment inward.
2. It writes a **portable type name into a per-app instance path**, creating a second source of
   truth for a fact the `entity_type` field already carries — the two can disagree, and §4.1.1's
   cosigned split exists to keep those axes apart.
3. It breaks the one thing §1 does pin: `window_id` would no longer be sufficient to locate a
   window's state, so the action addressee and the state address diverge.

**Recorded because the conclusion was right and the stated reason was not** — and a design
withdrawn for a reason that does not hold is one a later session will reopen, correctly, and get
wrong. The index is the better answer on its own merits and does not need the MUST to defend it.

---

## §6 Q5 — the persist arm becomes a checkable disjunction

§8's arm currently permits taking persistence with nothing to identify what was persisted. Filed
as a SHOULD-sweep; ruled as a three-way choice, so a reviewer can tell which one an impl took:

> An application persisting per-window state **MUST** be able to determine, at startup, which
> windows a persisted state entity belongs to. It satisfies this by **either** maintaining an
> `app/state/window-index` (§4.2a), **or** sweeping `app/{app-id}/workspace/windows/` at startup
> before allocating any window id. An application doing neither **MUST NOT** persist per-window
> state — §8's ephemeral arm is the correct and conformant choice, not a lesser one.

**Why the sweep is a real option and not a booby prize:** it is cheap, it is correct, and its cost
is bounded and legible — the user loses last session's per-window state, which is precisely what
an app that does not offer session resumption was never promising. The failure the guide currently
permits is the third case: persisting, restoring, and silently getting it wrong.

**Three implementation warnings, adopted from the filing seat's §4 verbatim** — they were paid for
in a built implementation and all three are portable:

- **A session that re-opens nothing must not persist an empty index.** Unclaimed prior entries are
  retained, or the next boot's sweep deletes every slot. *A window nobody re-opened is not a window
  that was closed.*
- **Only a read you can vouch for may authorize deleting.** `no-index` and `malformed` must not
  merge — *"you never had one"* and *"you have one and cannot read it"* differ in exactly the way
  that decides whether a sweep is safe.
- **A malformed row fails the whole index**, never yielding a short one. A partial index still
  authorizes a sweep, and a sweep driven by a half-read list deletes live state.

---

## §7 Q7 — `GUIDE-SHELL-FRAMING` §7.1, found by the L23 sweep

Enumerating the rule's homes **by subject** turned up a fourth document. §7.1's worked example:

```
Selection { path: "…", kind: "entity", origin: { surface: "shell", window_id: "..." } }
```

Three defects against `GUIDE-ENTITY-WORKBENCH-APP` §5.4, in a canonical, published guide:

1. **`origin` is source attribution**, which §5.4 rule 1 makes a `MUST NOT` — *"MUST NOT emit
   `source_window`, `source_panel`, or `source_*` fields."*
2. **`kind` where §5.4 rule 2 pins `type`** as a MUST.
3. **A `window_id` inside a persisted selection payload** — which Q1 makes meaningless to any
   later reader by construction.

**This is L23's second shape exactly, and it is why the sweep was run by subject rather than by
token.** `origin` shares not one character with `source_window` / `source_panel` / `content_type`,
so the grep the first shape prescribes would never have found it — and neither does the live gate:
`entity-workbench-go`'s `legacySelectionFields` is `{content_type, source_window, source_panel,
paths}`, so an impl that read `GUIDE-SHELL-FRAMING`, emitted `origin`, and ran against
workbench-go's violation logger would be **non-conformant and silently green.**

It is also **L25**: the rule's own section is fine, and its *example* three documents away
contradicts it.

---

## §8 §7-delta table — the edits, each verified against the tree

| # | File | § | Edit |
|---|---|---|---|
| **D1** | `guides/GUIDE-ENTITY-WORKBENCH-APP.md` | §3 | State `{window_id}` is a session-scoped slot address, not stable across sessions; add `window-index` to the path block |
| **D2** | `guides/GUIDE-ENTITY-WORKBENCH-APP.md` | §4.2 | Add the `app/state/window-index` row; amend the `app/state/window` row to require `content_type` |
| **D3** | `guides/GUIDE-ENTITY-WORKBENCH-APP.md` | §4.2a (new) | The index schema, the membership-vs-arrangement boundary, the three implementation warnings |
| **D4** | `guides/GUIDE-ENTITY-WORKBENCH-APP.md` | §8 | The persist-arm disjunction (§6 above) |
| **D5** | `guides/GUIDE-ENTITY-WORKBENCH-APP.md` | §1 | Correct the action-wire-shape row from MUST to SHOULD, pointing at §6 |
| **D6** | `guides/GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` | §5.1 | Retire `app/state/layout`; add `app/state/window-index`; name §4.2 the authority |
| **D7** | `guides/GUIDE-SDK-PATTERNS.md` | §2 Convention Types | Same as D6 |
| **D8** | `guides/GUIDE-SHELL-FRAMING.md` | §7.1 | Rewrite the `Selection` example — drop `origin`, `kind` → `type` |

---

## §9 Cohort impact — and who is told, by the routing model

**Filing seat first (L15). It is already built there, which is what lets this go wider in the same
session.**

| Seat | What is owed | Path |
|---|---|---|
| `entity-browser-rust` | Field rename `{id, type, peer}` → `{id, content_type, peer_id}`; type-name promotion `app/entity-browser/window-index` → `app/state/window-index` per §4.1.1; the cosigned-MUST correction (§5) | **Direct** — app tier |
| `entity-workbench-go` | Map key `"content-type"` → `"content_type"`; `content_type` on `app/state/window` is now required (already written, needs the rename); the read-modify-write finding (§4.2) is theirs to judge, not arch's to fix; their three §6 questions are answered | **Direct** — SDK/apps tier |
| Core tier | **Nothing.** `entity-core-{go,py}`, keystone, formalization: zero occurrences. `entity-core-rust` carries `workspace/windows/1/state` in `bindings/sdk/src/sdk.rs` **doc-comment examples only** — no behaviour, no owed work | Not routed |

**Two corrections travel with the packets, both L22 — a peer's true sentence about their own
artifact, re-pointed at a different artifact:**

1. The filed proposal reads *"workbench-go's slot is 'today write-only; future restore last
   session … features read from it', which is the same defect not yet triggered."* **That comment
   is on `SaveAlias`** (`workspace_state.go:349`), the **shell-alias** slot at
   `workspace/shells/aliases/{alias}` — keyed by a user-chosen alias name, which is durable, so it
   has none of this defect.
2. **The real workbench-go evidence is different and stronger.** `workbench/log_model.go:143`
   reads `ReadWindowSetting(m.windowID, "log-display-level")` — a live read of per-window persisted
   state keyed by a session-scoped ordinal — and `updateWindowState` merges into last session's
   map. So workbench-go is **not** *"the same defect not yet triggered"*; it is the same defect,
   live, on a read path, in the impl that cannot use the type discriminator. The gap is portable —
   which is what the proposal set out to establish, and it is now established on two trees rather
   than one.

*The obligation to re-derive travelled with whoever moved the sentence, and that was arch as much
as the filing seat — the packet that carries this says so.*

---

## §10 What this proposal does not claim

- **Not that the index shape is settled.** `(content_type, peer_id)` does not distinguish two
  windows of the same type on the same peer; the filing seat names this as a deliberate limit and
  arch agrees it is the right limit *for now*. It is recorded in §4.2a as a known bound, not hidden.
- **Not that this is urgent.** The filing seat measures the collision inert in their production
  profiles on coincidences. Nothing here blocks a release. It is cheap now and expensive after a
  second impl ships resumption — and per §9 one already reads the slot.
- **Not that arch found it.** A user-visible symptom was reported to `entity-browser-rust`, an
  audit went looking, and they built the answer before asking. Arch's contribution is the corpus
  sweep the filing seat could not run: the slot that already existed, the two guides that
  disagreed, and the contradiction in §1 that their own argument was standing on.
