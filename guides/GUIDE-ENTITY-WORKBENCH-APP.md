# Guide: Entity Workbench App

**Status**: Active
**Source:** A workbench-go four-pass UI-conventions reconciliation review; a three-pass workbench-go response (with §13 synthesis); an egui-side UI-alignment review; the Godot app architecture doc; and workbench-go Stage 5 implementation feedback (three findings — §5.3 reframed, §5.4 schema refocused, new §10.1).

---

## What this guide covers

This is a **general-purpose application namespace convention** built around `app/{app-id}/...`. The entity workbench — the application class that makes the entity system visible and operable to a human user (tree browsing, handler dispatch, peer/identity management, knowledge-base authoring, and so on) — is the first concrete instance and the reason the convention exists. Future apps using `app/{app-id}/...` (a calculator, an expanded knowledge-base, third-party apps) inherit the conventions for free.

In scope:

- Architectural pattern (model → renderer-neutral output → renderer).
- Path namespace under `app/{app-id}/...`.
- Type-name vs path distinction.
- Structural schema vocabulary — the slot table for what entity types describe what application state.
- Path-position-invariance for state schemas.
- Selection scope semantics — per-panel local + per-presentation-context propagated.
- Action wire shape vs in-memory shape.
- Change-detection styles.
- Persistence interaction surface (cross-link to GUIDE-PERSISTENCE).
- Multi-peer-per-app and multi-app composition.
- CLI / TUI / GUI as render contexts of one application, not separate application classes.

Out of scope:

- Renderer code, widget primitives, layout serialization (per-impl).
- Cross-impl test harness for UI behavior.
- Identity SDK shape (covered by `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md`).
- Content-type catalog ratification — currently in flux pending extension-driven direction; the four content-types whose schemas are stable across the transition (`tree-browser`, `entity-detail`, `execute-console`, `event-log`) progress at T2 regardless.

Cross-links: `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md`, `SDK-OPERATIONS.md` (especially §6.4 generation-counter, §6.5 notification access levels, §11.6 handler registration, §15 configuration directory), `GUIDE-SDK-PATTERNS.md`, `GUIDE-PERSISTENCE.md`, `ENTITY-NATIVE-TYPE-SYSTEM.md` §11.1 (type entity path convention).

---

## 1. Conformance philosophy

This is an application-layer convention, not an SDK contract. Wire-format invariants live in V7; SDK invariants live in SDK-OPERATIONS. This guide describes how applications under the `app/{app-id}/...` namespace pattern compose well — three impls of the workbench should be able to read each other's `app/workbench-{id}/...` state without reverse-engineering language-native struct names; future apps using the same pattern should pick up the same conventions for free.

| Tier | Items |
|---|---|
| **MUST** | `app/{app-id}/workspace/...` namespace prefix; bundled per-window state as CBOR maps; action wire shape with `(window_id, event, value)` semantics. |
| **SHOULD** | Schema field names per content-type; canonical action vocabulary; per-presentation-context selection propagation. |
| **Idiomatic per impl** | Window abstraction; renderer; layout primitives; in-memory action shape; change-detection style; renderer-specific decoration. |

App state vs app runtime: "everything under `app/{app-id}/...`" means **state**, not the app's binary or runtime image. Future "transferable apps" addresses runtime as a separate concern; this guide is about state.

---

## 2. Architectural pattern: model → output → renderer

The load-bearing convention. Each window/panel/view is composed of three layers:

- **Model** — language-native object holding the panel's working state. Reads inputs from the entity tree (own subscriptions or pulls); accepts incoming actions; emits an output struct.
- **Output struct** — renderer-neutral data describing what the panel should display. Plain language-native types (Go struct, Rust struct, Godot Resource/Dictionary). Field names align cross-impl per the schema vocabulary in §4.
- **Renderer** — language-and-framework-native code that consumes the output and draws (egui canvas, raylib canvas, tview lines, DOM tree, Godot scene). Per-impl. Not portable.

The pattern is TEA-shaped (The Elm Architecture). The renderer-neutrality of the output struct is what makes the same model run under multiple renderers — the Go workbench's `workbench/` is shared between raylib (`canvas/`) and tview (`console/`); Rust's Knowledge Base similarly.

Why the output struct is the convention (not the model interface): the model's *shape* is per-impl idiomatic (Go uses struct + interface, Rust uses trait + associated type, Godot uses Node + signal); the model's *output* is data, and data crosses idiom boundaries cleanly.

---

## 3. Path namespace

The `app/{app-id}/...` namespace is general-purpose. `{app-id}` is a **per-instance identifier** that the app claims by writing to it — there is no central registry, no reservation mechanism. Apps coexist by claiming different `{app-id}`s. Multiple instances of the same app type are also supported by giving each instance its own `{app-id}`.

Concrete examples on a single peer's tree:

```
/{peer_id}/app/workbench-01/workspace/...        first workbench instance
/{peer_id}/app/workbench-02/workspace/...        second workbench instance
/{peer_id}/app/workbench-dev/...                 development instance
/{peer_id}/app/calculator/workspace/...          a calculator app using the same convention
/{peer_id}/app/calculator-02/workspace/...       second calculator instance
/{peer_id}/app/knowledge-base/...                a KB app using the same convention
```

Within an app's namespace, the binding decisions on path shape:

```
app/{app-id}/
    workspace/
        windows/{window_id}/state              bundled per-window state (CBOR map)
        screens/{screen_id}/...                per-presentation-context state (optional; flat apps omit)
        selection                              propagated selection (§5)
    settings/{key}                             global app settings
    manifest                                   OPTIONAL — app metadata, if needed
```

App-side path helpers SHOULD parameterize `{app-id}` per `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §4.3. App-id is permissive — claimed labels, renameable, no registration.

### 3.1 Path hierarchies — flat or nested

Both shapes are permitted:

```
flat:    app/{app-id}/workspace/windows/{id}/state
nested:  app/{app-id}/workspace/screens/{id}/windows/{id}/state
```

Per-presentation-context state (the `screens/` layer in egui terminology, "workspaces" in Godot terminology) is optional. Apps that don't differentiate use the flat form. Both impls' state schemas share fields.

### 3.2 Path-position-invariance

The state schema for a given content-type is **invariant under repositioning**. The state entity at `app/workbench-01/workspace/windows/3/state` of type `app/state/tree-browser` carries the same fields as the state entity at `app/workbench-01/workspace/screens/0/windows/3/state` of the same type. Impls migrating between flat and nested hierarchies (or composing deeper, like Godot's `workspace → layer → slot → panel`) do not have schema-rewrite work — only path-rewrite.

Bundled-vs-per-key state: **bundled** for persisted state (one entity per window, multi-field CBOR map). Per-field outputs may apply at T3 for high-frequency outputs (deferred — see §10).

---

## 4. Structural schema vocabulary

### 4.1 Type names vs paths

A clarifying frame. `app/state/tree-browser` is a **type name**. The corresponding type entity lives at `system/type/app/state/tree-browser` per `ENTITY-NATIVE-TYPE-SYSTEM.md` §11.1 ("type paths shown without the `system/type/` prefix for brevity; resolution prepends this prefix").

- Type names are **global** — the same `app/state/tree-browser` type is reusable across any app that wants tree-browser state.
- Paths to **instances** of those types are **per-app**: `/{peer_id}/app/workbench-01/workspace/windows/3/state` is an instance of type `app/state/tree-browser` belonging to the `workbench-01` app.

### 4.1.1 Namespace split (normative; absorbed from the cross-impl alignment cycle)

**SHOULD (cross-impl portability rule for the `entity_type` *field*).** Type names written into the `entity_type` field of entities follow this split:

- **`app/state/{type_name}`** — **cross-impl portable canonical types**. Implementations consuming these types MUST follow the canonical schema. The slot table (§4.2) enumerates the current cross-impl canonical types.
- **`app/{app-id}/{type_name}`** — **app-internal types** that are NOT cross-impl portable. Schema is the implementing app's own contract. Cross-impl observers MUST NOT assume schema portability of types whose name begins with `app/{app-id}/`.

> **Disambiguation (load-bearing — egui + workbench-go cosigned).** This rule scopes the `entity_type` *field* (the type-name string written into entities), NOT the instance-path prefix. The two namespaces parallel each other but apply to different concepts:
> - **Type-name namespace (this §4.1.1 rule)**: governs strings written into `entity_type`. `app/state/...` = portable; `app/{app-id}/...` = app-internal.
> - **Instance-path namespace (§3 / §4.2)**: governs tree paths where instances of those types live. ALL instances — both portable-type instances and app-internal-type instances — live under `app/{app-id}/...` paths per §3. The instance path is per-app regardless of whether the type itself is portable.
>
> Example: an instance of the portable type `app/state/selection` lives at `app/workbench-01/workspace/selection` (per-app *path*, portable *type*). An instance of the app-internal type `app/entity-browser/connection` lives at `app/entity-browser/connections/{id}` (per-app *path*, app-internal *type*).
>
> The slot table (§4.2) describes type-name → instance-path conventions; this rule (§4.1.1) describes the type-name conventions independently.

**Implementations writing app-specific types** SHOULD place them under `app/{app-id}/{type_name}` rather than appending app-suffixes within `app/state/{type_name}`. The canonical pattern from egui: `app/entity-browser/connection`, `app/entity-browser/listener-state`, etc. (verified cross-impl).

**Type-name promotion** — when a previously-app-specific type proves convergeable across impls (e.g., `connection` shape stabilizes), promote it from `app/{app-id}/{type}` to `app/state/{type}` and update the slot table. Pre-publication, migration is natural-overwrite acceptable; in-process parsing unaffected. Post-publication this requires a spec amendment with a published cutover schedule (peers running older codebases would not recognize a renamed type without coordination).

### 4.2 Slot table

Three-impl consensus across Go + egui-Rust + Godot.

| Slot | Type name (full type entity at `system/type/{name}`) | Status |
|---|---|---|
| Per-content-type window state | `app/state/{content_type}` (e.g. `app/state/tree-browser`) | **Long-term shape.** Schemas defined per content-type at T2. Describes **portable** content-type state only — what the content-type *is*, not how a particular renderer decorates it. |
| Generic per-window fallback | `app/state/window` | **Transitional.** For windows whose content-type isn't yet schema'd; lets new windows land before T2 ratifies their schema. Track usage rather than pin a sunset — landing a per-content-type schema graduates that content-type out of the fallback. |
| Per-screen state | `app/state/screen` | **Optional.** Multi-screen / multi-workspace apps use it; flat-window apps omit. Schema TBD. |
| Selection | `app/state/selection` | **Two-layer model** per §5; schema in §5. |
| Settings (key/value) | `app/state/setting` | Existing convention. Generic `{key, value}` pair under `app/{app-id}/settings/{key}`. |
| App manifest | `app/state/manifest` | **Optional, if needed.** Lives at `app/{app-id}/manifest` *under the app's own namespace*. Speculative; not reserved unless an impl needs it. |
| Top-level entity outside the namespace | **none** | Nothing top-level outside `app/{app-id}/...`. The "AppState" working name was dropped during consensus. |
| T3 outputs | `app/ui/output/{content_type}` | **Reserved namespace.** Deferred until SDK affordances (S1 local-only namespace, S2 watch-with-diffs) land. Runtime path is `app/{app-id}/ui/output/...`. |
| Renderer-specific decoration | **per-impl, not portable** | Theme overrides, mouse filters, layer z-index, slot/split ratios, etc. Lives in per-impl runtime/view state. NOT in the cross-impl `app/state/{content_type}` schema. |

### 4.3 Guidance the slot table commits to

- **Portable vs decoration.** `app/state/{content_type}` is the cross-impl portable contract; renderers add their own decoration outside it.
- **Bundled state per window** (one CBOR map per window, multi-field) — settled. Per-field-entities at T3 outputs only.
- **ViewState vs RenderState** distinction is **per-impl**, not per-convention. Some impls (egui) split persisted vs runtime view state; some don't. The convention names what's persisted and portable; impls split or bundle internally as they please.
- **Persisted-state vs T3-output namespace split** is pinned now (`app/state/...` vs `app/ui/output/...`) even with T3 deferred — prevents retrofit.

The slot table commits to what each slot *means*. Per-content-type field-by-field alignment happens through ongoing T2 alignment with concrete impl reference points (Go's `DetailOutput` / `ArticleViewOutput`, Rust's `KnowledgeBaseOutput`, Godot's day-one panels). T2 schemas may also ship as types in `entity-sdk` crate exports, reducing re-derivation cost across impls.

---

## 5. Selection scope: per-panel local + per-presentation-context propagated

Two layers exist; the convention accommodates both.

### 5.1 Per-panel selection (local truth)

Each panel's model holds whatever selection state it needs for its own purposes — `tree-browser` has a current path cursor, `event-log` has a current event row, `entity-detail` has a viewed entity. This is internal to the panel and not propagated unless the panel chooses to.

### 5.2 Per-presentation-context selection (system-level "what is being attended to")

A higher-level signal at `app/{app-id}/workspace/screens/{screen_id}/selection` (or `app/{app-id}/workspace/selection` for flat apps) carries the current focus the user has expressed at that level — typically the most recent action from any panel that *opted in* to propagate. Other panels in the same context **MAY** subscribe to this and co-orient (entity-detail follows the focused tree-browser's current path, for instance).

Two `tree-browser` panels in the same screen viewing different peer trees: each holds local selection; the most recently interacted with also writes to the per-context selection; entity-detail co-orients on the per-context one. Subscription-and-notify is the propagation mechanism.

### 5.3 Recommended publish-to-aggregate defaults per action

| Action | Recommended publish-to-aggregate |
|---|---|
| `navigate` | yes (context) |
| `select` | yes (context) |
| `submit` | no (panel) |
| `clear` | no (panel) |
| `set_filter` | no (panel) |
| `toggle_raw` | no (panel) |

**Convention for content-type authors, not runtime dispatch metadata.** Implementation experience across the three reference impls (workbench-go Stage 5, egui, Godot) converged on a per-panel-slot + per-context-aggregate model where the publisher already knows at publish time whether to write to the aggregate. The publisher writes:

- to its own slot at `app/{app-id}/workspace/panels/{panel_id}/selection` (always), and
- to the screen aggregate at `app/{app-id}/workspace/screens/{screen_idx}/selection` (iff the publisher's configuration says it should propagate).

Under this model, the per-panel decision collapses to a single boolean per publisher panel (typical name: `publishesToAggregate`). The action-name vocabulary above is still useful for **typing** the kind of selection on inspection and for guiding new content-type authors picking sensible defaults — it is **not** a runtime registry consulted at dispatch time. Future impls needing runtime-overridable propagation (e.g. `tree-browser` configured as "scratchpad navigator" that suppresses context propagation) express it via the publisher's own config — no registry needed.

### 5.4 Selection schema

Type `app/state/selection` (full type entity at `system/type/app/state/selection`). Fields:

```
path           text?         # focused path the user is attending to
paths          [text]?       # multi-select forward-compat hook (optional)
peer_id        text?         # which peer's tree (multi-peer apps)
type           text?         # type of the selected thing (e.g. "entity", forward-compat for "query-row", "event-row")
updated_at     uint          # epoch ms; staleness signal
```

- `path` is the focused single selection. Renderers that don't multi-select read this and ignore `paths`.
- `paths` is the wider selection set when the user has shift-clicked / ctrl-clicked. Optional; multi-select-aware renderers populate.
- `type` names the **type of the selected thing** (today typically `"entity"`; forward-compat for query-result rows, event-log rows, etc.). Load-bearing for filtering on aggregate-slot consumers when multiple selectable types coexist on the same screen.
- **Absence = cleared.** No empty-string sentinel. If there's no current selection, the entity isn't present (or `path` is unset). Empty-string `path` is malformed.

**Note (absorbed from workbench-go feedback and tightened per the cross-impl alignment cycle).** Earlier drafts of this schema included `content_type` and `source_window` fields for source attribution ("which content-type / which window emitted this aggregate-level selection"). Implementation experience surfaced that under the per-panel-slot model (§5.2 / §5.3), source attribution is unnecessary: per-panel slots are single-publisher by construction (the slot path identifies the publisher), and per-context aggregate consumers care about the current value rather than the most-recent writer (`updated_at` handles staleness/tie-breaking).

**Normative (absorbed from the cross-impl alignment cycle; cross-impl-aligned posture). Pre-publication stance — no legacy tolerance.**

1. **Emit-side (MUST).** Implementations MUST NOT emit `source_window`, `source_panel`, or `source_*` fields in `app/state/selection` entity payloads.
2. **Field rename (MUST).** Implementations writing this schema MUST use `type` (not legacy `content_type`). The semantic was refocused: `type` names the type of the *selected thing* (e.g., `"entity"`), not the panel that emitted the selection.
3. **Read-side (MUST log violation; MAY reject).** Pre-publication, implementations MUST log a violation (severity WARN minimum) when encountering legacy `source_window` / `source_panel` / `content_type` fields in received entities. Implementations MAY reject (return error / refuse to decode) such entities. Silent tolerance is NON-CONFORMANT pre-publication — the entire ecosystem is owned peers, and silent tolerance hides non-conformant emitters that should be fixed.
4. **Re-emit-on-write (MUST).** Readers that load a legacy entity and re-publish MUST NOT re-emit legacy fields. Drop them on the round trip.
5. **Post-publication policy (deferred).** Once the protocol publishes to uncontrolled peers, a "MUST tolerate (skip-unknown-fields per V7 §2.6 open-types)" rule with a published cutover date will apply. Until that cutover lands, pre-publication strictness is the canonical posture.

---

## 6. Action wire shape vs in-memory shape

**Status note (absorbed from the cross-impl alignment cycle).** The named action-event vocabulary below and the wire shape `(window_id, event_name, value)` are **speculative / schema-anchor for cross-impl wire persistence and replay** — NOT a load-bearing runtime requirement today.

**Verified cross-impl reality:** no implementation in the workbench ecosystem (egui, workbench-go, Godot) currently emits named action events as distinct wire-shape triples. Cross-panel co-orientation rides on **selection state updates** (writes to `app/{app_id}/workspace/selection`) in all three impls. The vocabulary below exists as schema documentation — egui's `src/action_event.rs` explicitly notes "zero in-repo call sites by design... no Rust callers until wire-shape persistence/replay lands."

**Normative position:**
- Named action events are **SHOULD-level for cross-impl wire persistence/replay**. Implementations that persist or replay action history SHOULD use the vocabulary below.
- For current cross-panel co-orientation, selection-state-only is the cross-impl-canonical pattern. Implementations are NOT REQUIRED to emit named events.
- **Vocabulary is open-extensible per V7 §2.6 open-types semantics.** New events MAY be added by content-types as needed (the vocabulary "is **not closed**" per §6.1). When the first implementation reaches for wire-shape persistence/replay and surfaces real consumer needs, the canonical vocabulary will be re-confirmed via D8 cadence; current entries are draft, subject to revision when a real consumer lands.

- **Wire shape (convention; aspirational pre-consumer)** — `(window_id, event_name, value)` triple, where `event_name` is from the canonical vocabulary and `value` is a typed CBOR value (string, number, bool, struct, or empty). This is what gets serialized when actions are persisted, replayed, or cross a process boundary — once an impl reaches for those surfaces.
- **In-memory shape (per-impl)** — typed enum (Rust), struct + interface (Go), Resource/Dictionary with Variant (Godot), object literal (TS). Renderers freely use whatever's idiomatic. Translation to wire shape happens at the persistence/cross-process boundary.

### 6.1 Initial canonical events

The list is **not closed** — content-types add events as needed; new events should be named and registered, not absorbed into a generic catch-all.

| Event | Value shape | Typical targets | Default propagation |
|---|---|---|---|
| `navigate` | `text` (path) | tree-browser | context |
| `select` | `text` (path) | tree-browser, query-console | context |
| `submit` | empty | execute-console, query-console | panel |
| `clear` | empty | event-log, execute-console | panel |
| `set_filter` | `text` (level name) | event-log | panel |
| `toggle_raw` | empty | entity-detail | panel |

A `set_field` catch-all was considered and **deprecated**. Force every action to be named; entity-typed action history (T3) treats two named actions cleanly while a generic `set_field` would produce ambiguous replay.

---

## 7. Change-detection styles

A panel declares which change-detection style it uses (factory-time content-type metadata). Three styles, all SDK-supported (SDK-OPERATIONS §6.4 + §6.5):

1. **Counter polling** — `generation()` checked each frame/tick. May suit immediate-mode UI with simple state where a per-frame check is already happening (raylib loops, egui native).
2. **Path-targeted observation** — `store().watch(prefix)` returning a local event stream. May suit retained-mode UI with many panels watching disjoint subtrees (Godot signal bridge, DOM with retained state).
3. **Dispatched subscription** — `subscribe(pattern, callback)` via `system/subscription`. Required for cross-peer reactivity. May suit any panel watching another peer's tree.

None is the default; panels pick per their rendering paradigm and observation needs. Panels watching their own peer's `app/{app-id}/workspace/...` state typically use (1) or (2). Panels watching network peer trees typically use (3). The convention names the styles; the SDK supports all three.

---

## 8. Persistence interaction

Forward to `GUIDE-PERSISTENCE.md`, which reads `SDK-OPERATIONS.md` §15 (Configuration Directory) + `SDK-IDENTITY-INFRASTRUCTURE.md` §8.4 (identity bundle layout) into developer-readable form.

**App state lives in the host peer's tree.** There is no separate "app store" on disk. When an application writes to `app/{app-id}/workspace/windows/3/state`, that's a write into the peer's tree; the on-disk effect is whatever the peer's storage backend does. Apps don't manage filesystem layout — peers do.

Persistent vs ephemeral line, per state kind:

- **Settings** (`app/{app-id}/settings/{key}`) — persist across restart by default.
- **Per-window state** (`app/{app-id}/workspace/windows/{id}/state`) — persist if the application offers session resumption; MAY be ephemeral otherwise.
- **Per-screen / workspace state** (`app/{app-id}/workspace/screens/{id}/...`) — persist if multi-screen layout is meaningful across sessions.
- **Selection** (`app/{app-id}/workspace/selection` or per-screen) — ephemeral by default; recompute on restart from active panel state. Apps that persist "where I left off" do so explicitly.
- **T3 output entities** (`app/{app-id}/ui/output/...`) — ephemeral / non-syncing. They are the rendering peer's view, not data.

The line between persistent and ephemeral is per-namespace, not per-extension. This guide names the line; `GUIDE-PERSISTENCE.md` owns the configuration-directory layout.

**Dispatch-index re-registration** (`SDK-OPERATIONS.md` §11.6.6, callout in `GUIDE-PERSISTENCE.md` §3.6) — applications **MUST** re-register their handlers in startup code. The handler manifest in the tree survives restart; the in-memory callable does not.

---

## 9. Multi-peer-per-app and multi-app composition

### 9.1 Multi-peer-per-app

Forward to `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §3 multi-peer-per-app variant.

- Application processes **MAY** run an internal app peer for UI state and zero-or-more user-managed network peers for data.
- App peer owns `app/{app-id}/...`; network peers own their own trees.
- Routing between them uses standard SDK primitives (`execute()`, `watch()`, `subscribe()`).
- Panels typically declare which peer they target. Settings/workspace panels target the app peer; tree/event/execute panels target network peers (which one is user-controlled).

This composition lets UI ephemera (window state, selection, draft inputs) stay local without leaking to the network.

### 9.2 Multi-app composition on one peer

A single peer **MAY** host multiple apps in its tree, each under its own `app/{app-id}/...` namespace. This is the same mechanism that supports multiple instances of the same app — the convention doesn't distinguish "different instances of one app type" from "different apps."

```
/{peer_id}/
    app/workbench-01/...                        first workbench
    app/workbench-02/...                        second workbench
    app/calculator/...                          a calculator app
    app/knowledge-base/...                      a KB app
```

Each app's content-types use the same global type names; each app's instances live at per-app paths. Apps coexist by claiming different `{app-id}`s.

Future-tracked, not specified here: capability boundary between apps; discovery between apps; rendering primitive sharing across apps. The convention permits coexistence today; richer multi-app stories belong to a future capability/multi-app guide.

---

## 10. CLI / shell as a render context

The workbench application is **one application**; CLI / TUI / GUI / canvas / web / game-engine are **render contexts** of that application, varying in interaction affordances and interaction model. A CLI workbench uses the same `app/{app-id}/...` namespace, the same content-types, the same action vocabulary, the same model→output pattern — it's a thinner renderer over the same models.

What the convention commits to (for any render context):

- Same `app/{app-id}/...` namespace as GUI surfaces. A CLI and a GUI workbench on the same peer co-orient on the same state.
- Same content-types, action vocabulary, state schemas.
- Same model→output→render pattern; the renderer being text formatting vs widget rendering doesn't change the contract.

What the convention explicitly does **not** commit to:

- Whether the CLI is REPL, command-mode, streaming, or mixed.
- Whether subscriptions are user-explicit (`tail`) or implicit per content-type.
- Whether content-types render the same way in CLI as in GUI (some may need pagination, alternate text representation, or be flagged not-surfaced-in-CLI).

The interaction model is in active discovery — long-term we expect CLI surfaces to vary the way `bash`, `zsh`, and `fish` vary on top of POSIX.

### 10.1 Action events vs commands — two vocabularies, one render context

A CLI render context surfaces **two distinct vocabularies** that coexist. The application's action vocabulary (§5.3, §6) is the pub-sub plane shared by every render context; a CLI additionally exposes a **command vocabulary** for direct invocation. They are not the same vocabulary:

| Domain | What it is | Scope |
|---|---|---|
| **Action events** (§5.3, §6) | Pub-sub: "user attended to X", "user submitted." Wire shape `(window_id, event, value)`. Named vocabulary (`navigate`, `select`, `submit`, `clear`, ...). | **Application-agnostic.** Two distinct apps use the same action names with the same semantics. |
| **Commands** (CLI-only) | Imperative ops: `cd`, `ls`, `connect`, `mount`, `exec`, `revision config put`. Args + typed `Result`. Dispatched through a registry. | **Application-directed.** Each app's CLI render context has its own command vocabulary. |

How they intersect:

- **A command may emit one or more actions as a side effect.** `cd /peerA/notes/` (command) produces a `navigate` action with `path=/peerA/notes/`. The command returns a typed `Result`; the action publishes to subscribers via the app's selection slot/aggregate.
- **Not every command emits an action.** `mount`, `connect`, `revision config put` are pure configuration writes; their observable consequence is on the entity tree's event stream, not the action channel.
- **Not every action has a command counterpart.** `toggle_raw` (entity-detail), `set_filter` (event-log) are panel-internal gestures. They *could* gain a verb, but the value is low until panels are shell-addressable.
- **The user is a publisher from either side.** Typing `cd` in the shell and pressing arrow keys in tree-browser are equivalent inputs to the same `navigate` event surface.

**Implications for impls:**

- Action-event names belong in the SDK as application-agnostic cross-app constants.
- Commands belong per-app in app-specific code (separate from the SDK).
- A panel content-type declares its action surface (which events it produces, which it consumes) at registration time — content-types are the cross-app contract.
- **Command → action emission is the integration point.** Document the pattern in app code so future apps don't reinvent it.

---

## 11. What's not in this guide (yet)

Tracked as future work:

1. **Content-type catalog.** The current direction is shifting toward extension-driven panels — one canonical window per advanced-extension surface (revision, history, attestation, role, group, etc.) — with knowledge-base becoming a view over the revision system rather than a top-level content-type. The catalog stabilizes when the extension-driven design lands. Until then, the four content-types whose schemas are stable across the transition (`tree-browser`, `entity-detail`, `execute-console`, `event-log`) progress at T2.
2. **T3 entity-typed outputs** (`app/ui/output/{content_type}`). Deferred until SDK affordances S1 (local-only namespace) and S2 (watch-with-diffs) are designed.
3. **Cross-peer rendering.** Three latency regimes (pull-on-demand, watch-with-diffs, shared-output) likely; SDK affordances and capability scoping for cross-peer outputs are unsettled.
4. **Compute-backed UIs / handlers.** T3 + reactive-recompute territory; flag as future.
5. **Multi-app capability boundary.** Future capability/multi-app guide.

---

## 11A. Self-Assessment Checklist for Workbench Implementations

**Status: tool, not normative.** Per the cross-impl alignment cycle (workbench-go feedback §2.E): use this for surfacing demand-driven roadmap candidates; not a SHOULD-level expectation. Useful for first-pass coverage audit by any workbench impl.

Workbench applications cover varying portions of the 14 application-architecture primitives enumerated in `entity-core-papers/09-application-architectures.md`:

| # | Primitive | Short description |
|---|---|---|
| 1 | **Pg — Pagination** | How the app surfaces large result sets in bounded views |
| 2 | **Ch — Change detection** | How the app notices state changes (polling / watch / subscription) |
| 3 | **Da — Data** | How data flows between substrate and app state |
| 4 | **Sh — Shape** | Type structure and field-spec conformance |
| 5 | **Ac — Access** | Capability-scoped access patterns |
| 6 | **Mu — Mutation** | Write patterns (L0 / L1 / handler op dispatch) |
| 7 | **Pr — Propagation** | Cross-panel / cross-context state propagation |
| 8 | **Co — Coherence** | Convergence across replicas / merges / rollback |
| 9 | **Hi — History** | Audit, transition chains, rollback surfaces |
| 10 | **Ev — Evaluation/compute** | Entity-native computation, expression evaluation, reactive subgraphs |
| 11 | **Pe — Perception** | Subscription, observation, watch surfaces |
| 12 | **Pn — Presentation** | Renderer / view-state / decoration surfaces |
| 13 | **Bn — Boundary** | Capability boundaries, namespace boundaries, app boundaries |
| 14 | **Au — Authority** | Identity stack, role assignment, attestation surfaces |

**Self-assessment pattern**: for each primitive, score current coverage on this scale:

| Score | Meaning |
|---|---|
| **1** | Absent — no surface in this impl |
| **2** | Ad-hoc — incidental code, no canonical pattern |
| **3** | Present but incomplete — partial surface, known gaps |
| **4** | Complete but untested — full surface, no real-app validation |
| **5** | Production-tested — full surface, validated under real workloads |

Identify the lowest-scoring primitives as candidates for the next demand-driven panel/feature work. The first impl to run this self-assessment (Godot) identified Ev=1/5 (Evaluation/compute) as their biggest gap and named "compute-graph viewer" as their highest-value gap-closing panel.

**Cross-impl visibility**: per `CROSS-IMPL-PARITY-MATRIX.md`, surface gaps that cluster across multiple impls become candidates for cross-impl coordinated work; gaps unique to one impl are that impl's roadmap.

---

## 12. References

External:

- Workbench-go four-pass cross-impl UI-conventions reconciliation review.
- Workbench-go three-impl response with §13 synthesis.
- Egui-side parallel UI-alignment review.
- `godot-entity-core-rust/docs/ARCHITECTURE.md` — Godot third-impl architecture.

Internal (this repo):

- `core-protocol-domain/specs/ENTITY-NATIVE-TYPE-SYSTEM.md` §11.1 — type entity path convention.
- `sdk-domain/specs/SDK-OPERATIONS.md` §2.7, §6.4, §6.5, §11.6, §15.
- `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` §8.4.
- `sdk-domain/guides/GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §3 (multi-peer-per-app variant), §4 (namespace conventions), §5 (application state).
- `sdk-domain/guides/GUIDE-SDK-PATTERNS.md` — entity-backed state, reactive cycle, multi-peer composition.
- `sdk-domain/guides/GUIDE-PERSISTENCE.md` — companion guide.
- `proposals/implemented/PROPOSAL-GUIDE-ENTITY-WORKBENCH-APP.md` — proposal-track artifact this guide graduated from.

Adjacent priors:

- The Elm Architecture (TEA), Model-View-Intent (MVI), Headless UI / React Aria, Cells / spreadsheet model, FRP literature (Elliott & Hudak; Rx).
