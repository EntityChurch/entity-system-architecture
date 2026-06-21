# Guide: Shell Framing

**Status**: Active
**Source:** `EXPLORATION-SHELL-FRAMING.md` (reasoning record); `PROPOSAL-CLI-ENTITY-TREE-TOOLS.md` v5.0 (foundational verb vocabulary); `GUIDE-ENTITY-WORKBENCH-APP.md` §5.12 (CLI as render context); `EXPLORATION-PROGRAMMING-ERGONOMICS-COMPARATIVE.md` L7 (named behaviours).

---

## What This Guide Covers

The **shell** is a named layer above the SDK that exposes the entity system's coordination primitives — tree operations, handler dispatch, capability and identity management, revision and subscription surfaces — as a stable, named-verb vocabulary suitable for *interactive* use.

This guide names that layer, pins what implementations must agree on, documents the axes that should remain per-implementation choices, and connects the shell to existing architectural threads (CLI-as-render-context, named-behaviour ergonomics, SDK Layer 3 promotion).

It applies to any application that wraps the SDK in a verb surface — whether shipped as an interactive REPL (`dsh`, `entity-shell`), a one-shot CLI (`dls`, `dexec`), an embedded GUI window or panel, a command palette, a parameter form, or a library in a test driver. All are *render contexts* of the same shell.

The exploration that feeds this guide (`core-protocol-domain/explorations/EXPLORATION-SHELL-FRAMING.md`) carries the reasoning, inheritance from prior shell traditions (Unix, Lisp REPL, IDE palette, Smalltalk image, structured shells), and the design-axis derivation. This guide carries only the conclusions.

---

## 1. What the Shell Layer Is

The shell is *its own layer* between the SDK and the application's render contexts. The layering claim:

> **Shell : SDK :: SDK : protocol.**

Each layer takes the layer below as substrate, adds named conventions, and exposes a smaller, more opinionated surface that handles the common cases ergonomically.

| Layer | Surface | Audience | What it adds |
|---|---|---|---|
| Wire / protocol | bytes, envelopes, capability tokens, content hashes | protocol implementers | Coordination semantics. |
| SDK L0–L2 | typed peer handles, tree ops, handler dispatch | application developers | Language-native binding; opinionated defaults. |
| SDK L3 (formalized guides) | named composition functions | application developers | Multi-extension recipes as one callable function. |
| **Shell** | **named verbs over the SDK** | **end users and developers** | **Interactive surface: stable verb vocabulary, session state, observability of in-flight work.** |
| Render contexts | text / form / panel / window | the user in their chosen modality | Modality variation over the shared verb vocabulary. |

The shell earns its layer status by adding three things the SDK does not:

1. **Named verbs as a stable vocabulary.** The verb is the *contract*; the SDK call is the *implementation*. SDK functions may refactor; verb names persist.
2. **Interactive-session state** — working directory, history, scrollback, active connections, aliases. Belongs neither in the SDK nor duplicated in each render context.
3. **Observability of in-flight system work.** `continuation ls`, `tail`, `revision history` — the shell is where the system's *temporal* dimension becomes visible.

---

## 2. Render Contexts — One Shell, Many Modalities

Per `GUIDE-ENTITY-WORKBENCH-APP.md` §5.12: CLI / TUI / GUI / canvas / web / game-engine are **render contexts** of one application. The shell's verb vocabulary is canonical; render contexts vary the modality.

A shell verb has identical name, arguments, and semantics across all render contexts:

| Render context | What the user does | What renders |
|---|---|---|
| Terminal REPL | Types `cat /peer/foo` | Text scrollback |
| One-shot CLI | Runs `dcat /peer/foo` | stdout, exits |
| Embedded GUI window | Types into window's text input | In-window line buffer |
| Embedded multi-pane panel | Types into panel's input | Panel-shaped output (`OutputLine[]`) |
| Command palette | Selects `cat` from filter list; fills path parameter | Form result rendered as inline panel |
| Test driver library | Calls `shell.dispatch("cat", ["/peer/foo"])` directly | Programmatic `Result` returned |

This is the principle that makes the layer non-trivial: implementations **MUST NOT** diverge verb names or argument shapes across render contexts within a single application. The same `cat` works everywhere.

---

## 3. Normative Pins

These are the points where implementations **MUST** converge. Each pin is justified in `EXPLORATION-SHELL-FRAMING.md` Part III / IV / VI as cited.

### 3.1. Layer position

The shell **MUST** sit above the SDK and below render contexts. Shell implementations consume the SDK; they do not extend or modify the protocol. Code below the SDK (handler code, extension code, protocol code) **MUST NOT** consume the shell.

### 3.2. Verb vocabulary tiers

The shell vocabulary is organized into three tiers (§4 below catalogues them):

- **Tier C (core).** Every shell **MUST** ship these. Tracks the SDK's tree + handler + connection surface.
- **Tier E (extension-derived).** Shell **SHOULD** ship verbs for every extension the SDK supports.
- **Tier A (application-derived).** The application **MAY** register verbs over its own handlers.

### 3.3. Result shape

A verb returns a **presentation-neutral, streamable sum-type** with categorized variants. Implementations **MUST** support the following variants:

| Variant | Carries |
|---|---|
| `path` | A resolved tree path |
| `listing` | A list of entries (children of a path, query results, peer list) |
| `entity` | One entity (the result of `cat`, `get`, `info`) |
| `tree` | A subtree (the result of `tree`) |
| `dispatch` | The result of `exec` — handler-defined |
| `info` | Generic structured info (connection details, version, identity) |
| `lines` | A stream of textual lines (the result of `tail`) |
| `message` | A short status message (no data) |
| `error` | An error with code, message, and optional cause |

**Error-channel idiom (per-implementation surface choice).** The `error` variant is a *contractual* part of the result shape — error information **MUST** be surfaced uniformly to the render adapter. Whether the implementation expresses it as a sum-type variant (e.g., one variant of an enum) or via the language's idiomatic error channel (Go's `(T, error)` tuple, Rust's outer `Result<T, E>`) is a per-implementation surface choice. What matters is that the render adapter receives error information through a stable path; whether that path travels in-variant or out-of-band is implementation idiom. The guide does not pin in-variant; it pins the uniform-error-surface contract.

Streaming **MUST** be supported for at least `lines` and `dispatch` — the verb may produce intermediate output before completing.

The concrete streaming primitive (a stream type, an async receiver, a callback) is a per-implementation choice. Implementations **SHOULD** document the choice in the shell library's API. Implementations building against the *same* shared shell library (e.g., the proposed `entity_shell` Rust crate consumed by both egui and Godot) **MUST** pick one shape and hold it across consumers — diverging primitives across consumers of one library forces breaking changes at extraction time. Language-neutral choices the streaming shape **MUST** support: an unbounded sequence of `StreamChunk` values terminated by either a final result variant or an error; the consumer can drain at its own cadence (drain-on-tick, await-loop, callback) without blocking the verb.

### 3.4. Path syntax — `@alias` as a peer-id substitution sigil

The shell's path syntax does **not** introduce new path-grammar constructs. It honors the protocol's canonical absolute path form (`/{peer_id}/path`) and the entity-URI form (`entity://{peer_id}/path`) directly, and adds **one** substitution sigil that operates wherever a peer-id appears.

**The `@alias` sigil.** `@alias` is a substitution that expands to an identifier — either a peer-id (for peer-aliases) or a group-id (for group-aliases; see §6.4). It is **not** path syntax; it is a peer-identifier convenience that composes with every existing site where peer-ids appear in the protocol:

```
Canonical absolute path:   /{peer_id}/foo/bar         →  /@alice/foo/bar
Entity URI:                entity://{peer_id}/foo/bar →  entity://@alice/foo/bar
Cap-grant peer-ref:        grant {peer_id} ...        →  grant @alice ...
Identity ref:              {peer_id}                  →  @alice
```

After alias resolution, the form is structurally identical to its peer-id equivalent — the alias is expanded at the parser layer, not preserved through dispatch. This means downstream code (handler, capability check, wire serialization) sees only peer-ids; aliases never leak below the shell's parse step.

**Implementations MUST support these forms in shell input:**

- **Absolute path** — starts with `/`. Resolved in the bound peer's namespace. The first path segment **MAY** be a peer-id literal OR `@alias` (which expands to one).
- **Absolute path with alias substitution** — starts with `/@`. Canonical form when using an alias. `/@alice/foo/bar` is the canonical absolute form.
- **Typing shortcut** — starts with `@`. `@alice/foo/bar` is accepted as a shell-input shorthand for `/@alice/foo/bar`. The shell adds the leading slash internally before resolution. This is an *input ergonomic*, not a separate grammar.
- **Relative path** — does not start with `/`, `@`, or other reserved sigils. Resolved against the shell's working directory (`wd`). Standard `.` and `..` semantics.
- **Entity URI** — starts with `entity://`. Per protocol spec; `@alias` substitution is permitted in the peer-id position (`entity://@alice/foo/bar`).

**The `:` character is reserved for the protocol's `<handler-path>:<op>` op-naming convention** — used universally across V7 (`system/handler:register`, `system/tree:extract`, `system/type:validate`, `revision:fetch-diff`, etc.; see `ENTITY-CORE-PROTOCOL.md` §3.7 and §6.2). Implementations **MUST NOT** repurpose `:` for path-syntax constructs (alias prefixes, separators, or other path-grammar uses). The protocol's op namespace is load-bearing; preserving `:` as a single-meaning operator across the system is a normative requirement of this guide.

**The `:` reservation is grammatical, not character-level.** Entity content — names of entities stored at tree paths — **MAY** contain `:` (e.g., an entity named `handler:register` stored as a tree node). Shell parsers **MUST** handle `:` non-structurally within path segments: only `:` at known structural positions (currently none in path syntax, after the §3.4 reframe) is reserved; `:` inside a path segment is just a character. Implementations **MUST** support addressing entities whose names contain `:` without escaping.

**Rationale for `@`-as-sigil.** `@` is a single-meaning sigil across the developer landscape — email, SSH/SCP, git remotes, GitHub, Mastodon, Twitter, mention syntax — where `@x` means "x is an identity/handle." Adopting it consistently for peer-aliases honors that convention. Crucially, `@alias` is *not* a path-syntax construct (which is where prior framings stumbled); it is a peer-identifier substitution that composes with whatever syntax the surrounding context already has. The shell does not invent path grammar; it composes existing grammar.

### 3.4.1. Federation (informative)

The `@alias@org` form (email/Mastodon precedent) extends the singleton-alias case to *org-namespaced aliases* cleanly: `@alice` is a direct local alias (priority); `@alice@core-org` and `@alice@acme-corp` are org-controlled namespaces. The parser splits on `@`: first part is the alias, second part (if present) is the namespace. v1 ships flat aliases; federation extends without breaking the grammar. Not normative in this version of the guide; surfaced so the framing is future-ready.

### 3.5. Cross-surface co-orientation

When an application surface exists that hosts multiple presentations of the same workspace, the shell **MUST** coordinate with sibling presentations via **state-slot publish**, not via direct callbacks. The shell publishes a `Selection` (or analog) to a workspace-aggregate slot on `cd` or path-changing verbs; sibling panels configured to follow the aggregate slot co-orient.

State-slot publish is loop-safe by construction (consumers are gated) and composes with whatever else the application uses for state co-orientation. Direct callback wiring **SHOULD NOT** be used for this purpose.

This pin is **conditional on the deployment having a sibling-presentation surface**. A pure-REPL or one-shot CLI deployment has no co-orientation to perform; the pin is trivially satisfied.

**The publish hook is an explicit construction-time affordance.** The shell library **MUST** expose the publish hook as a constructor parameter on the shell or dispatcher — not as a deployment-time toggle, configuration flag, or fork. The hook's binding shape is per-implementation idiom: a Go impl may use a callback function field (`Shell.OnWDChanged` style); a Rust impl may use a sink type (`Option<SelectionSink>`) or a publisher trait. *What matters is the publish-to-slot semantics and the no-direct-cross-listener wiring* — the shell publishes a state change to one slot; subscribers watch the slot — not the specific binding mechanism. A pure-REPL embedding constructs the shell with no hook (or `nil`/`None`) and the pin is trivially satisfied; an embedded-window embedding constructs with the application's aggregate slot wired in; a palette embedding may construct with its own scoped sink. Multiple shells in one application with different sinks **MUST** be supportable without forking the library. This is what realizes the "trivially satisfied" framing in practice.

### 3.6. Render-context principle

Within one application, the same verb name **MUST** mean the same operation across every render context (terminal, window, panel, palette, library). Implementations **MUST NOT** rename or re-shape verbs per context.

### 3.7. Verb implementations are pure

Verb implementations are functions of `(Shell, args) → Result`. They **MUST NOT** assume a terminal, a readline, a GUI, or any specific render context. All session state they need is reached via the `Shell` handle; all output is the returned `Result`.

This is what makes the same verb work in all five embedding modes (library, binary, window, panel, palette).

### 3.8. Privilege is capability-gated

Authorization for any shell verb's underlying operation is enforced **inside the SDK call the verb makes**, via the capability system. The shell **MAY** surface a hint to the user (e.g., "this verb requires a system capability") for UX, but **MUST NOT** be the enforcement point.

Privilege levels in the shell layer are a UX affordance, not a security boundary.

### 3.9. Implementation-language neutrality (Python parity)

The normative pins (3.1–3.8) **MUST** be implementable in any reference implementation language without requiring a GUI or application architecture from another implementation. A language reference implementation achieves shell-level feature parity with other reference implementations by implementing the shell against these pins — not by re-implementing the other implementations' frontends.

This pin is what makes the shell layer a *coherent participation surface* across the three current reference implementations (Go, Rust, Python). See §11 below.

---

## 4. Verb Vocabulary

### 4.1. Tier C — Core (required)

Every shell ships these. Names and argument shapes are converged.

| Verb | Op | Arguments |
|---|---|---|
| `connect` | Establish session with a peer (handshake → capabilities) | `<addr>` |
| `disconnect` | Close a session | `<peer-or-alias>` |
| `cd` | Change working directory | `<path>` |
| `pwd` | Print working directory | (none) |
| `ls` | List children at a path | `[<path>]` |
| `cat` | Display entity bound at a path | `<path>` |
| `tree` | Recursive listing | `[<path>] [--depth N]` |
| `exec` | Dispatch a handler operation | `<target-uri> <op> [json-params]` |
| `info` | Show connection / version / identity details | `[<peer-or-alias>]` |
| `help` | Verb discovery and usage | `[<verb>]` — no args lists verbs; with arg shows that verb's Usage + Help |

This is the ten-command core. Verbs `connect/disconnect/cd/pwd/ls/cat/tree/exec/info` were independently converged across three CLI implementations (Go, Rust, Python) per `PROPOSAL-CLI-ENTITY-TREE-TOOLS.md` §2. `help` is added in this guide as a universal discoverability verb (every shell from `bash` to `psql` to `git --help` to our REPL has one); cross-impl convergence record carries it.

### 4.2. Tier E — Extension-derived (per extension)

Verbs **SHOULD** be shipped when the corresponding extension is loaded.

| Verb | Extension | Op |
|---|---|---|
| `tail` | SUBSCRIPTION | Stream notifications matching a pattern |
| `query` | QUERY | Run a query, return matching entities |
| `count` | QUERY | Run a query, return count |
| `find` | QUERY / SCAN | Find by type, tag, or property |
| `grep` | QUERY / SCAN | Find by content match |
| `revision` | REVISION | `commit`, `branch`, `merge`, `cherry-pick`, `revert`, `tag` |
| `history` | HISTORY | Walk the audit history of a path |
| `identity` | IDENTITY | Show / rotate / recover identity |
| `role` | ROLE | List / grant / revoke roles |
| `peer` | PEER MGMT | `list`, `create`, `delete`, `rename` |
| `continuation` | CONTINUATION | `ls`, `inspect`, `cancel` — the "DPS of the shell" |
| `put` | TREE | Write an entity |
| `rm` | TREE | Remove an entity |
| `cp` | TREE | Copy an entity (handler-driven) |
| `mv` | TREE | Move (rename) an entity |
| `has` | TREE | Test existence |
| `open` | APPLICATION | Open a window / panel / pane (when render context supports) |
| `mount` | NAMESPACE | Mount a remote subtree under a local path |
| `unmount` | NAMESPACE | Reverse of mount |
| `mounts` | NAMESPACE | List active mounts |
| `subscription` | SUBSCRIPTION | List / inspect / cancel active subscriptions (sub-ops: `ls`, `inspect`, `cancel`) |

### 4.3. Tier A — Application-derived

The application's own handlers may expose verbs. A version control system built on EXTENSION-REVISION may register `commit`, `branch`, `log` as application verbs that wrap the underlying `exec` calls with ergonomic argument shapes.

Tier A verbs **MAY** override or hide Tier C/E verbs *within their application*, but **MUST NOT** rename them. A scoped VCS app (§9) might hide `peer` (irrelevant to the VCS user) but **MUST NOT** rename `cd` to `goto`.

### 4.4. Verb registration

The shell library **SHOULD** expose a registration API allowing the application to:
- Register additional Tier A verbs.
- Hide specific Tier E verbs not relevant to the application.
- Override help text for any verb.

The registration shape is per-implementation (Part III §7.3 axis); see §5 below.

### 4.5. Verb naming principle

Verb names are not chosen freely. They follow two rules, in order. **The MUST rule applies to top-level verb names only**; sub-op vocabularies inside a compound verb are owned by the verb owner (see §4.6).

1. **Extension-operation match (MUST) — top-level verbs.** Where a Tier C or Tier E *top-level* verb directly invokes an extension primitive, the verb name **MUST** match the extension operation name. Examples: `put`/`get`/`rm` from EXTENSION-TREE; the top-level verb `revision` corresponds to EXTENSION-REVISION as the extension namespace. The shell verb is the extension's named handle made interactive; the top-level names must align.
2. **Cross-implementation convergence (fixed by record) — top-level verbs.** Where a top-level verb has no direct extension counterpart — `info`, `cd`, `pwd`, `tree`, `find`, `grep`, `open`, `help` — the name is fixed by cross-implementation convergence as recorded in `PROPOSAL-CLI-ENTITY-TREE-TOOLS.md` §12 and the three-impl 10-core. These names are stable by precedent; future shells **MUST** adopt them.

**Sub-op naming is owned by the verb's extension.** Sub-ops *inside* a compound verb (`revision commit`, `revision sync`, `revision find-ancestor`, `identity bootstrap`, `history rollback`, `continuation inspect`) follow extension-internal conventions:
- Sub-ops that map 1:1 to extension primitives **SHOULD** match the extension op name (`revision commit` ↔ EXTENSION-REVISION's `commit` op).
- Sub-ops that are shell-level *compositions* with no single extension counterpart (`revision sync`, `revision find-ancestor`, `identity bootstrap`) are named by the verb owner using cross-impl convergence record as the fallback.

This rule resolves name disputes by reference to a relevant extension spec or the convergence record, rather than by fiat. Where neither rule applies (a genuinely new top-level verb), the proposing implementation **SHOULD** surface the name to the architecture team and the other reference impls for explicit convergence before adoption, and the chosen name becomes a precedent under rule 2.

### 4.6. Sub-op convention for compound verbs

Some Tier E verbs bundle sub-ops (`revision commit`, `identity rotate`, `peer create`, `continuation inspect`). The convention:

- **Surface syntax:** *space-separated*. The first token after the verb name names the sub-op. `revision commit -m "msg"` and `peer create alice` are both `<verb> <sub-op> [args...]`.
- **Dispatcher shape:** the parametric dispatcher (per §5.2) surfaces the sub-op shape via the verb's parameter schema — the verb declares its sub-op vocabulary alongside its argument schema. Implementations using a closed-match dispatcher **MUST** still honor the same surface syntax.
- **Vocabulary ownership:** extension owners **MAY** define the sub-op vocabulary for their extension's verb. The sub-op names **MUST** follow §4.5's extension-operation match rule (`revision commit` because EXTENSION-REVISION defines a `commit` operation).
- **Single-shot verbs:** verbs with no sub-ops (`cat`, `cd`, `ls`, `tail`) take their arguments directly with no sub-op token. The presence or absence of a sub-op tier is per-verb, declared in the parameter schema.
- **Verb-op layer factoring (per-impl).** At the verb-op layer (§8.1), sub-ops **MAY** be factored as one verb-op per sub-op (e.g., separate `revision_commit_op`, `revision_branch_op`, `revision_merge_op` functions), or as one verb-op accepting a sub-op enum/discriminant. Both are valid; the choice is per-implementation idiom. What this guide pins is the *surface syntax* (`<verb> <sub-op> [args]`) and the §8.1 verb-op signature constraints (no `Shell`, typed outputs); the internal factoring at the op layer is per-impl.

### 4.7. Verb semantic notes

Specific semantic clarifications for verbs where the surface name does not fully determine the operation:

- **`disconnect <peer-or-alias>`** — Idempotent against an already-closed connection: a `disconnect` against a peer with no active connection **MUST** return a `message` variant (not an error), conveying that no action was taken. Default behavior **MUST** gracefully drain in-flight operations against the target peer before tearing down the connection; implementations **MAY** offer a flag (e.g., `--force`) to abort in-flight operations. A `disconnect` against an unknown peer-or-alias returns an `error` variant. The verb does not affect the bound peer (window-bound deployments) or current peer (session-current deployments) state — disconnecting a peer that happens to be the bound/current peer leaves the binding intact but with the connection closed.

Additional clarifications will be added as cross-impl review surfaces verbs with under-specified semantics.

---

## 5. Axes Documented but Left Per-Implementation

The following choices are legitimate per-implementation. The guide documents the trade-offs; it does not pin.

### 5.1. Peer-binding model

The shell's "binding model" is the answer to *what does it mean for the shell to have a local peer*, separate from how cross-peer addressing works in path syntax (the `@alias` sigil per §3.4 is orthogonal — alias-prefix targeting is path syntax, not a binding mechanism).

Three legitimate binding models:

- **Session-current-peer:** the shell has one *mutable* "current local peer." `connect <peer>` changes which peer is the shell's local peer. Cross-peer access uses `@alias`/peer-id substitution in paths. Long-lived REPLs that need to *switch* local context favor this.
- **Window-bound peer:** the shell is constructed against *one fixed local peer*; that's the local peer for its lifetime. Switching local context means opening another shell. Cross-peer access uses `@alias`/peer-id substitution in paths. GUI windows and per-window panel embeddings favor this. *Workbench-go's `shellcmd/` and egui's window-bound shell are both this model;* they differ in path syntax mechanics (per §3.4) but not in binding mechanics.
- **Multi-peer multiplexed:** the shell has *no single local peer* — it constructs against a set of peer connections, every command targets one explicitly via `@alias`/peer-id. The set is mutable via `connect`/`disconnect` operations.

In every model, cross-peer access in path syntax uses the same `@alias` mechanism (§3.4). The binding-model axis is about *whether the shell has one local peer, and whether that is mutable*; it is not about how cross-peer addressing works. This separation resolves an enumeration ambiguity in earlier drafts: workbench-go's `shellcmd/` is **window-bound** (one local peer per shell-instance, set at construction, not mutable), with `connect`/`disconnect` operating on the *remote alias table* rather than on the local-peer binding. The local-peer-vs-remote-peer distinction is what the binding-model axis names; the alias-prefix is the path-syntax mechanism that operates orthogonally.

### 5.2. Dispatch surface

- *Registry of typed `Command{}` values:* extensible at runtime; ceremony.
- *Closed match statement on verb name:* simple; harder to grow.
- *Parametric command definitions* with declarative parameter schemas: needed if a palette/form render context renders parameters from the schema.

Where a palette render context is among the targets of a shared shell library, the parametric form is **MUST**, not SHOULD — a parametric palette cannot consume a closed-match dispatcher without a translation shim that re-derives the parameter schema, and any such shim defeats the layering. Outside the palette case (e.g., a shell library targeting only terminal + window adapters), parametric is recommended but not required.

### 5.3. Session persistence

- *Ephemeral:* `wd`, history, scrollback in process memory; gone on exit. Fits one-shot CLI.
- *Tree-backed:* state lives as entities in the peer tree (`app/state/shell`). Survives restart. Fits embedded windows that participate in workspace persistence.

One-shot binary mode is ephemeral by construction; embedded/REPL mode is tree-backed by convention.

**Namespace ownership for tree-backed state.** The shell library **SHOULD** provide path *helpers* for canonical state-location patterns (`{root}/state/shell`, `{root}/state/shell/history`, `{root}/state/shell/aliases`); the embedding application **MUST** supply the path *root* (e.g., `app/{app-id}/...`). The shell library does not own absolute paths — it owns the *shape* of its persistence subtree relative to a root the embedding provides.

This division allows:

- Multiple shells in one embedding (e.g., two windows, each its own session) to have independent persistence locations without library cooperation — the embedding supplies different roots per shell.
- The same shell library to participate cleanly in different application namespace conventions (`app/godot/...` vs `app/entity-browser/...` vs `app/repo/...` for a scoped `entity-repo` binary) without absolute-path conflicts.
- Test drivers and scoped binaries to direct persistence to ephemeral or test-scoped roots without the library knowing.

The shell library's contract: *given a root, I will read/write the standard subtree shape under it.* The embedding's contract: *here is the root.*

### 5.4. Lifecycle

- *One-shot:* run one command, exit (like `git`).
- *REPL:* read commands forever (like `bash` or `psql`).
- *Embedded window/panel:* host application lifetime governs.
- *Library:* invoked programmatically by a containing program.

The same verb implementations **MUST** work in all four (per 3.7 above).

### 5.5. Specific verb set choice

Beyond Tier C, which Tier E verbs an implementation ships is a function of which extensions it loads. Which Tier A verbs an application defines is its own decision.

---

## 6. Path Syntax in Detail

### 6.1. The `@alias` sigil — resolution

`@alias` resolves to an identifier (peer-id or group-id) via the following lookup order (normative):

1. **`@self`** — the bound peer (in window-bound deployments) or local peer (in window-bound + path-multiplexed deployments; see §5.1).
2. **`@primary` / `@system`** — the application's primary / system peer.
3. **Label** — case-insensitive match against `PeerMetadata.label` over local + connected peers + known groups.
4. **Identifier prefix** — first peer or group whose id starts with the alias.

The resolved identifier is substituted into the surrounding syntax at parse time. Forms after substitution are structurally identical to their literal-identifier equivalents (`/@alice/foo` and `/{alice_peer_id}/foo` are the same path after resolution).

**Alias name grammar.** Alias names are **Unicode strings** (any codepoint sequence). The system places only the minimum restrictions required to keep shell-input parsing unambiguous:

Alias names **MUST NOT** contain:
- `@` (reserved as the alias sigil; also the federation separator per §3.4.1)
- `/` (path separator)
- `:` (protocol op-namespace separator)
- Whitespace characters (token boundaries)
- Control characters (`U+0000`–`U+001F`, `U+007F`)
- Shell quote/escape characters in surface positions: `'`, `"`, `\` (these depend on the eventual shell-grammar; implementations **SHOULD** reject these in alias names as a precaution)

Beyond these structural restrictions, alias names **MAY** contain **any Unicode codepoint**, including letters from non-Latin scripts (Cyrillic `@Алиса`, Chinese `@愛麗絲`, Arabic `@علي`, etc.), emoji, combining marks, and other non-ASCII content. ASCII-only restrictions on alias names are **explicitly NOT** part of this guide's contract.

**Unicode normalization.** Implementations **SHOULD** apply Unicode normalization (NFC) to alias names at the alias-table-write boundary (e.g., on `peer rename`, `connect`-with-alias) so that visually-identical-but-byte-different inputs resolve to the same alias. This prevents trivial duplication (e.g., `café` written with precomposed `é` vs `e` + combining acute) and reduces homograph confusion. Full homograph defense (e.g., Cyrillic `а` vs Latin `a`) is out of scope; NFC normalization is the minimum bar.

**Recommendation (not normative).** For portability of help text, tab-completion ergonomics, and shell-script reuse across environments with varying terminal Unicode support, ASCII alias names (`[A-Za-z0-9_-]+`) are *recommended* for everyday peer/group naming. The grammar permits broader Unicode for users whose local conventions require it.

### 6.2. Standalone `@alias`

`@alias` is accepted on its own (without surrounding path syntax) wherever a peer reference is expected — `peer delete @foo`, `peer rename @foo bar`, `open Shell @primary`. The resolution rules from §6.1 apply.

### 6.3. Path semantics in the bound peer's namespace

When a shell is on peer A and the user types `cat /@bob/foo` (or `cat /{bob_peer_id}/foo`, equivalent after alias resolution), that path is interpreted in **A's namespace** — A's local view of B's content (which may or may not be mirrored). To see B's actual tree directly, open a shell bound to B (in window-bound deployments) or `connect` to B and address by the resulting peer-id (in window-bound + path-multiplexed deployments).

Implementations **MUST** interpret paths in the bound peer's namespace; cross-peer reach is explicit, via `connect` and an `@alias`/peer-id in the path.

### 6.4. Groups vs. peer-aliases at the sigil level

Per §3.4, `@alias` resolves to *one identifier*. When the alias names a group, the resolved value is the **group's own identifier** (group-id), not the member set. Group-membership *expansion* (fan-out across members) is a separate concern from alias resolution — see §14.5 below. The sigil's contract stays simple: one `@alias`, one resolved identifier.

**Dispatching against a group-resolved path outside a fan-out context.** When a user types a path like `cat /@compgroup1/foo`, the dispatcher resolves `@compgroup1` to the group-id, then the verb-op attempts to operate against a tree rooted at the group-id (which is not a peer). Implementations **MUST** return a `Result::error` with a clear indication that the target is a group and fan-out is required — e.g., *"group target requires foreach or equivalent fan-out verb; address group members directly or wait for fan-out operator support."* The error **MUST NOT** silently succeed against an empty or non-existent tree; the distinction between "this is a group, you wanted fan-out" and "this peer-id doesn't exist" is load-bearing for UX.

### 6.5. Working-directory display and alias reverse-resolution

After `cd @alice/foo` (or `cd /@alice/foo`), the shell's working directory **MUST** store the *resolved* form (`/{alice_peer_id}/foo`) — this is the single source of truth for dispatch, and storing both forms invites state divergence if the alias table changes between `cd` and a subsequent operation.

For display (`pwd` and any other surface that shows the working directory to the user), implementations **SHOULD** attempt to *reverse-resolve* the peer-id against the alias table at display time:

- If an alias maps to the current peer-id in the WD, display the alias form (`/@alice/foo`).
- Otherwise (no alias or alias unmapped), display the resolved form (`/{peer_id}/foo`).

This keeps dispatch deterministic (resolved-form WD) while presenting a user-friendly form at display time. Reverse-resolution at display also handles the case where the alias table changes after `cd` — the display reflects the current alias state, not a stale snapshot.

**Multi-alias tiebreak (informational, not normative).** When multiple aliases map to the same peer-id, reverse-resolution must pick one for display. The guide does not pin a tiebreak rule; the choice is per-implementation. First shipped reference: `entity-workbench-go` uses *most-recent-wins* implicitly (the workspace's `peerMap[peerID]` keeps only the most-recently-added binding under `addConn`/`removeConn`; display reads whatever the map currently reports). This is one valid choice, not the only one — implementations **MAY** keep an ordered list and return first-set, prefer a user-pinned alias, or use any other deterministic rule. Cross-impl conformance work (§11.3) may pin a convergent rule if real divergence surfaces as a UX problem; until then, each implementation should pick a deterministic and stable rule and document it.

---

## 7. Co-orientation in Detail

### 7.1. The state-slot pattern

In a multi-panel application, the workspace has a state-aggregate slot (typically named `App aggregate` or `panel-source`). The shell publishes a `Selection` to this slot on `cd` or path-changing verbs:

```
Selection {
  path: "/peer-a/notes/note-001",
  kind: "entity",
  origin: { surface: "shell", window_id: "..." },
}
```

Sibling panels configured to follow the aggregate slot pick up the selection and co-orient their views.

### 7.2. Why state-slot, not callback

State-slot publish is:
- **Loop-safe by construction.** Consumers are gated; no action replay.
- **No-action-event.** State change, not message. Idempotent re-application is safe.
- **Composable.** Whatever else the application uses for state co-orientation hooks the same slot.

Direct callbacks couple the shell to specific listeners and require ad-hoc loop prevention. State-slot is the right pattern.

### 7.3. When this pin doesn't apply

Pure-REPL and one-shot CLI deployments have no sibling panels. The pin is satisfied trivially by *not publishing* (there's nowhere to publish to). Per §3.5, the publish hook is an explicit construction-time affordance — an embedding without a workspace constructs the shell without a sink, and no publishing occurs. The shell library **MUST NOT** require a sink at construction; absent sink **MUST** be a supported configuration.

The same library — same crate, same dispatcher, same verbs — supports three publishing postures simultaneously across distinct embedding instances:

- **Sink supplied, default-on:** embedded window/panel that lives in a multi-panel workspace.
- **Sink supplied, scoped:** palette overlay that publishes to a palette-local slot only, not the global workspace.
- **No sink:** standalone binary, one-shot CLI, library invocation, headless test driver.

The library's contract is to accept any of these without modification or fork.

---

## 8. Lifecycle and Embedding

### 8.1. The four-layer factoring

The shell library factors into four layers between the render adapter and the SDK. Naming the **verb-op seam** explicitly — the boundary between the thin CLI-string-args parser and the typed, callable op — is what makes non-shell panel consumers and SDK Layer 3 promotion (§12) tractable.

```
+--------------------------------+
| Render adapter (per modality)  |   readline / window input / panel input / palette form
+--------------------------------+
| Dispatcher                     |   verb registry, parsing, completion, history
+--------------------------------+
| Verb-parsers                   |   thin: parse string args, call op, project to Result
|                                |   pure functions of (Shell, args) → Result
+--------------------------------+
| Verb-ops (exported, typed)     |   structured-input callable: (PeerCtx, typed-args) → Response
|                                |   reusable by non-shell consumers
+--------------------------------+
| SDK                            |
+--------------------------------+
```

A binary is `verb-parsers + dispatcher + readline adapter + main()` over the underlying verb-ops. A window is `verb-parsers + dispatcher + window adapter + window-manager registration`. A palette is **verb-ops + form-render adapter + palette registration** — *the palette does not need verb-parsers* because palette forms have typed inputs, not CLI strings; the palette drives the verb-op directly. A test driver is `verb-parsers + dispatcher` for verb-by-verb-name testing, or **verb-ops alone** for structured-input testing. A non-shell panel (revision admin, mount manager, identity ceremony UI, …) **drives verb-ops directly** without ever assembling CLI args.

**Why this seam matters.** Without it, every non-shell consumer of a shell verb has to either (a) fake CLI string-args and call the verb-parser, or (b) duplicate the SDK-call logic. Workbench-go discovered this seam during their Stage-4 work (the execute-console panel needed to drive `exec` without a CLI string, and the resolution was to extract the op from the verb-parser). Godot's palette will hit it immediately on first use. Egui's crate extraction will hit it the first time a non-shell panel needs to drive a verb. SDK Layer 3 promotion (§12) implicitly assumes this seam: `revision follow` shell verb → `sdk.revision.follow()` L3 function is a trivial rename of the *verb-op* into the SDK namespace, not a refactor of the verb-parser's string-handling code.

**Implementations MUST factor verb-parsers from verb-ops.** The two need not live in separate files or modules — a verb-parser may be a thin function in the same module as its op — but the *function call* between them is the seam. Verb-ops are pure callables over typed inputs and the underlying SDK; verb-parsers wrap them with arg-string parsing and Result projection.

**Verb-op signature constraints (normative):**

- **Verb-ops MUST NOT take a `Shell` handle.** Session state (`wd`, history, alias table) belongs to the verb-parser tier; verb-ops take only the typed inputs they require plus the peer context (`PeerCtx` or equivalent). If a verb-op needed `Shell`, non-shell consumers (palette forms, panels, library callers) could not drive it — the whole point of the seam fails. Implementations that find themselves wanting to thread `Shell` into an op are signaling that an arg should be made explicit at the op signature instead.

- **Verb-op outputs are typed; they MAY be stream-typed for streaming verbs.** Non-streaming verbs produce a typed `Response` value the verb-parser projects to a `Result` variant. Streaming verbs (`tail`, future `subscribe`) produce a stream-typed value (per §3.3's per-impl streaming primitive — `mpsc::Receiver`, `chan`, async iterator, callback) that the verb-parser projects to `Result::lines` or streamed `Result::dispatch`. Non-shell consumers (palettes, panels) consume the typed stream directly without the parser's projection.

- **Alias resolution happens at the dispatcher tier**, between argument parsing and verb-parser dispatch. The dispatcher uses the `Shell` handle's alias table to expand `@alias` (and `@alias@org` future-federation form) into peer-ids and group-ids; verb-parsers receive already-resolved paths and identifier-typed args. This keeps the alias-table-plumbing in one place (the dispatcher) rather than duplicated across every verb-parser, and keeps verb-ops alias-unaware (consistent with their no-`Shell` constraint).

  **Minimum dispatcher metadata required.** For the dispatcher to do alias pre-resolution, it must know *which positional args of each verb are path-typed*. Closed-match dispatchers (per §5.2) **MUST** therefore carry at least a per-verb path-arg-position table — a minimal metadata layer that names which positional args are paths — even when they do not adopt the full parametric schema needed for palette consumption. The parametric form satisfies this requirement intrinsically (the parameter schema includes type info); the minimal-metadata form satisfies it explicitly. What is **not** acceptable is a closed-match dispatcher with *no* arg-type metadata, which would force alias resolution inside verb-parsers and violate the dispatcher-tier pin.

  **Resolution function idempotency (load-bearing).** Implementations' alias-resolution function (the operation that expands `@alias` and `@alias@org` to identifiers in a path) **MUST** be **idempotent on already-resolved inputs**: applying it to an input that contains no alias sigils returns the input unchanged. This property is what makes the minimum-metadata path tractable for impls with flag-interleaved verbs (e.g., `ls -l @alias/path`, `tree --depth 3 @alias/foo`). The dispatcher resolves declared path-arg positions; flag-interleaved positions or sub-op-position args may continue to call the resolution function inside the verb-parser without conflict, because re-resolving an already-resolved path is a no-op. Validated end-to-end by workbench-go's `TestDispatcher_ResolveArgs_IdempotentWithHandlerResolve`. Cross-impl conformance work (§11.3) **SHOULD** test for this property explicitly.

### 8.2. The supported embedding modes

| Embedding | Description | Example |
|---|---|---|
| Standalone binary | Own process, own main | `dsh`, `entity-shell`, `dls` |
| GUI window | One window in an egui-style app | egui's 10th window |
| Multi-pane panel | One panel within a multi-pane workspace | workbench-go's `tview` shell panel |
| Command palette | Visual form interface consuming the verb schema | Godot palette |
| Library | Programmatic dispatch from another program | Go test harness, scoped CLI app |

The verb implementations **MUST** work in all five.

---

## 9. The Scoped Standalone Application Pattern

A general-purpose shell exposes the full vocabulary. A **scoped standalone app** instantiates a peer, runs a fixed set of verb compositions, presents domain-specific output, and exits. It is *built on the shell* but presents only what its domain needs.

### 9.1. Pattern

```
1. Process start.
2. Construct peer with the extensions the app needs.
3. Restore persisted state.
4. Parse the user's invocation (e.g., `entity-repo log`, `entity-repo commit -m ...`).
5. Dispatch the corresponding shell verb composition (via library embedding mode).
6. Render the result in the app's chosen presentation.
7. Persist state changes.
8. Shut down the peer.
9. Exit.
```

### 9.2. Worked example: `entity-repo` as standalone VCS

A binary that does for entity content what `git` does for files:

- **Extensions used:** REVISION, HISTORY, INBOX (for push/pull), possibly QUERY.
- **Invocation surface:** `entity-repo init`, `commit`, `log`, `branch`, `merge`, `pull` (and `push` as future work — see below). All thin wrappers over Tier E + Tier A verbs.
- **No interactive REPL** — every invocation is one-shot.
- **No GUI** — output is git-style text.
- **The peer is the engine.** It provides content addressing, revision DAG, history. The user doesn't see the peer; they see a VCS.
- **Network surfaces only when the user calls `push`/`pull`.**

**Current impl status.** No `entity-repo` binary exists yet; the verb composition is viable against existing reference impls. `entity-workbench-go`'s `shellcmd` package implements the verbs an `entity-repo` would need: `commit`, `log`, `status`, `diff`, `branch`, `tag`, `checkout`, `merge`, `cherry-pick`, `revert`, `find-ancestor`, `show`, `resolve`, `config`. `pull` is implemented as `revision sync` (one-shot pull) plus the V-1 closure incremental-sync recipe (`subscribe head → fetch-diff → tree:merge`). **`push` is deferred** in all reference impls (workbench-go's `cmd_revision_follow.go:48` explicitly marks `push / fetch / fetch-entities` as deferred cross-peer ops); building `entity-repo push` requires the push side of cross-peer revision sync to land first. The archetype itself stands; the push verb is the missing piece.

The scoped app is the entity-system's equivalent of `git`-as-CLI: a focused tool built on shell primitives, presenting one task cleanly. See `EXPLORATION-SHELL-FRAMING.md` Part VI §17 for full discussion.

### 9.3. Pre-shell scaffolding

Per `entity-core-go/docs/architecture/proposals/active/DESIGN-IDENTITY-CONFIG-AND-PEER-MANAGEMENT.md` §3.1–§3.2: a scoped app composes *peer-manager* scaffolding (construct peer, install identity, configure persistence) with shell-layer verb dispatch. The peer-manager is pre-shell; the shell runs against the resulting peer.

The shell library **SHOULD NOT** assume the peer already exists; it **SHOULD** compose cleanly with peer-construction code so scoped apps can build the peer per-invocation.

---

## 10. Application Archetypes Built on the Shell

A catalogue of valid deployment shapes:

| Archetype | Lifetime | Embedding | Verb scope | Examples |
|---|---|---|---|---|
| General-purpose shell binary | REPL | standalone | full vocabulary | `dsh`, `entity-shell` |
| One-shot CLI | one-shot | standalone | full vocabulary | `dls`, `dcat`, `dexec` |
| Scoped CLI app | one-shot | standalone + library | restricted | `entity-repo`; hypothetical `entity-journal`, `entity-log` |
| Embedded REPL window | long-lived | GUI window | full | egui's 10th window |
| Embedded panel | long-lived | multi-pane GUI | full | workbench-go's shell panel |
| Command palette | event-driven | GUI overlay | full + restricted | Godot palette |
| Test driver | scripted | library | full + scripted | Go test harness invoking shell verbs |

For substantial classes of user (server admins, scripted automation, power users), the shell will be the **dominant** interface; GUI surfaces are a bonus that gets the full feature space because the shell already encapsulated it.

---

## 11. Cross-Implementation Alignment

### 11.1. Recommended adoption path

1. **Rust:** `entity-core-rust` extracts a shared shell crate, consumed by both `egui-entity-core-rust` and `godot-entity-core-rust`. This is the testbed for this guide's pins.
2. **Go:** `entity-workbench-go` reviews its `shellcmd/` against the extracted crate's surface; differences beyond Go/Rust idiom surface as feedback into this guide.
3. **Python:** `entity-core-py` implements the shell against the same pins. Per §3.9, this brings Python to feature parity at the verb-vocabulary level without requiring GUI parity.
4. **This guide is amended** via proposal (per `[Spec changes via proposal + layering]`) if a pin proves wrong under build.

### 11.2. Python's positioning

The shell layer turns out to provide a clean *participation surface* for Python. Python does not need to ship egui-equivalent web frontends, Godot-equivalent game integrations, or workbench-equivalent multi-pane GUIs to be a first-class reference implementation. It implements the shell. From the user's perspective:

> *"I can do everything in the Python `entity-shell` that I can do in Go's `dsh` or Rust's terminal."*

This parity is scope a Python team can carry — building shell verbs is much smaller work than building a GUI app — and it cleanly fits Python's actual deployment contexts (scripting, CI integration, server-side automation, notebook-style exploration). Python becomes a *peer* implementation rather than a *trailing reference*. See `EXPLORATION-SHELL-FRAMING.md` Part VI §20.

### 11.3. Cross-implementation conformance

Cross-impl conformance has historically been measured at the protocol/extension level. The shell layer adds a new conformance dimension: *do your shell verbs produce identical observable behavior given identical peer state?*

A cross-impl shell conformance test suite — same verb sequence against Go, Rust, and Python shells, against shared peer state, assert results match — is the natural validation surface for this guide. Scoped as future work once the three shells exist.

---

## 12. Promotion Path to SDK Layer 3

Stable shell verbs are SDK Layer 3 promotion candidates, in parallel with the protocol-guide → L3 promotion path documented in `guides/index.md`.

When a shell verb has:
- Identical semantics across three reference implementations, and
- A recurring underlying SDK composition that justifies a callable function,

…it is a candidate for promotion to `sdk.namespace.function()`. **The promotion is structurally simple because of the verb-parser/verb-op seam (§8.1):** the verb-op is already a typed, callable function over structured inputs. Promotion to L3 is *renaming/relocating the verb-op into the SDK namespace*, not refactoring CLI parsing logic out of the verb-impl. The shell verb persists (UX surface for interactive use, the verb-parser still wraps the now-SDK-L3-function); the L3 function exists (programmatic surface for application code); both are named the same and share the underlying op.

The first likely promotions:

- `revision follow` shell verb → `sdk.revision.follow()` L3 function (when G1, G2 close).
- `exec --await` (cross-peer execute with result) → `sdk.execute.await()` L3.
- `continuation ls` → `sdk.continuation.list()` L3.

Tracked as candidate work, not part of this guide.

---

## 13. Spec Gaps to Be Aware Of

Spec gaps surface in the current shell work. They are out of scope for this guide; they belong to the respective extensions / substrate. Naming them here so the shell layer's open issues are visible.

| Gap | Status | Source | What it blocks |
|---|---|---|---|
| **G1 / G2** | **Closed** (V-1 closure) | `EXPLORATION-PROGRAMMING-MODEL-GO-FORWARD.md` §2.1–§2.2 (original framing); closed by EXTENSION-CONTINUATION v1.15 + EXTENSION-TREE v3.15 + EXTENSION-REVISION §4.4.19 `fetch-diff` | Cross-peer follow at any tree size is now achievable declaratively via `revision:fetch-diff` (incremental closure transport at the revision layer) + `collect_keys` transform (chain-expressibility for projecting diff results into `tree:extract.paths`) + `tree:extract` / `tree:merge` composition. Validated end-to-end by a workbench-go incremental-sync-chain POC. The earlier G1/G2 framing (cross-peer compositions cannot be expressed declaratively) is *historical*; the gap is closed. |
| **K1** | Open | Substrate status notes §161 | Substrate's non-blocking emit liveness invariant is not yet normative. Shell verbs whose chain depth exceeds the stall threshold can deadlock under load. The shell layer cannot paper over this; substrate-level fix. |
| **G-OBS** | Open | Workbench-go memory (V-1 closure context); not yet in a central spec gap doc | Handshake-ingest path for revision-follow's first-delivery is under-specified — the chain machinery works once a follower is hot, but the cold-start (first-handshake) path runs through different code that the V-1 closure work surfaced as needing tightening. Not a shell-layer concern; flagged here because revision-follow shell verbs depend on it. |

---

## 14. Out of Scope

The following are deferred to future work and **MUST NOT** block adoption of this guide:

1. **Verb-result-as-entity.** Whether shell results are themselves typed entities (under `system/shell-result/...`), making them content-addressable, replayable, and composable. Future direction.
2. **Pipe semantics** between shell verbs (structured pipes, not byte pipes). Future direction; depends on shell-result-as-entity.
3. **Notebook render context** (Jupyter-style cells, structured outputs interleaved with prose). Possible future render context.
4. **`dsh` vs `entity-shell` — binary naming convergence.** This guide pins the layer, not a binary name. *Both* `dsh` (per `v3.0-core-revision/SYSTEM-ROADMAP.md` §154) and `entity-shell` (workbench-go's shipped binary) name the same archetype: the general-purpose standalone shell binary. They are the same kind of thing under different names; this guide does not pin one. Future convergence on a canonical binary name is open work, separate from the layer-level pins this guide carries. Implementations **MAY** ship either name (or both as aliases) until cross-impl convergence settles.
5. **Alias groups + fan-out (parallel dispatch across group members).** Groups exist as protocol entities (`EXTENSION-GROUP.md`); the `@alias` sigil per §3.4 cleanly resolves to a group's *own identifier* in any single-identifier context (cap-grants, URIs, identity refs). What this guide does **not** specify is the shell-level mechanism for *fan-out*: running one verb against every member of a group and aggregating results. Fan-out has its own semantics — aggregation strategy, partial-failure handling, ordering, parallelism — that warrant explicit ergonomic control rather than implicit alias-expansion. The likely future shape is an explicit fan-out verb or operator (e.g., `foreach @compgroup1: ls /system`, or a `multi` Tier C verb), with the singleton `@alias` semantics from §3.4 unchanged. Tracked as future work; surfacing the seam here so implementations don't accidentally couple group-expansion into alias resolution.

6. **System-wide internationalization (i18n) review.** The Unicode-permissive alias-name grammar in §6.1 surfaced that we have not done a system-wide review of where ASCII assumptions may have leaked into user-facing strings (alias names, peer labels, identity labels, capability scopes, role names, application-tier verb names, application path segments, error messages, etc.). The shell-framing work touches only one slice of this. A broader i18n review across the protocol, SDK, and application surfaces is owed before v1 ships any user-facing surface that would be costly to retrofit. Tracked as future architectural work; not in scope for this guide. Implementations **SHOULD** default to Unicode-permissive handling in user-facing strings (with structural constraints applied only where ambiguity in parsing would result), pending the broader review.

---

## 15. Companion Documents

- `EXPLORATION-SHELL-FRAMING.md` (core-protocol-domain/explorations/) — the reasoning record this guide stands on. Part III (axes), Part IV (layering), Part VI (scoped apps + Python parity) are the load-bearing sections.
- `PROPOSAL-CLI-ENTITY-TREE-TOOLS.md` (v5.0 proposals) — foundational verb vocabulary and dutils enumeration; predates and substantively aligns with this guide.
- `GUIDE-ENTITY-WORKBENCH-APP.md` — §5.12 (CLI as render context). The principle that motivates the render-context normative pin (§3.6).
- `GUIDE-SDK-PATTERNS.md` — the SDK surface the shell layer stands on.
- `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` — peer namespace conventions the shell honors.
- `EXPLORATION-PROGRAMMING-ERGONOMICS-COMPARATIVE.md` — L7 (named behaviours); shell verbs as the most direct named-behaviour surface the system ships.
- `EXPLORATION-CONTINUATION-PROGRAMMING-DEV-FEEDBACK.md` — the "DPS of the shell" thread; shell as continuation observability surface.

---

*Authored draft, further amended after workbench-go review. Review absorption record:*

- *Rust-team reviews (egui and Godot shell-framing reviews) absorbed: seven consolidated asks folded into §3.3 (streaming), §3.5/§7.3 (publish hook), §4.5 (naming principle), §4.6 (sub-op convention), §4.7 (`disconnect` semantics), §5.2 (palette MUST), §5.3 (namespace ownership). Both Rust reviews converged on every load-bearing pin.*

- *Workbench-go review (filed in the workbench-go repo per their project guidance) absorbed: the substantive item — **§3.4 path-syntax rewrite** — surfaced via architecture-team analysis (not vote-counting) of the `<handler-path>:<op>` convention used universally in V7 (`ENTITY-CORE-PROTOCOL.md` §3.7, §6.2). The earlier `@alias/path` path-syntax framing was reframed as `@alias` is a peer-id substitution sigil, not a path-grammar construct — composes with canonical `/{peer_id}/path` and `entity://{peer_id}/path` directly; the protocol's `:` op-operator stays clean. Other workbench-go absorbs: §3.3 error channel (Go `(T, error)` / Rust outer `Result<T, E>` are isomorphic to an in-variant error), §3.5 publish hook idiom (callback / sink / trait — what matters is publish-to-slot semantics), §4.1 added `help` to Tier C (now 10-core), §4.2 added `mounts` + `subscription` to Tier E, §4.5 naming MUST scoped to top-level verbs (sub-ops owned by verb owner), §5.1 binding-model rewrite (alias-prefix targeting recognized as path-syntax, orthogonal to binding), §8.1 **four-layer factoring with explicit verb-op seam** (verb-parsers + verb-ops as distinct layers — ripples to §12 promotion language), §13 G1/G2 closure (V-1 closure — `revision:fetch-diff` + `collect_keys`), §14.4 dsh/entity-shell naming-convergence-open, new §14.5 alias-groups+fan-out future work.*

- *`entity-core-py` review pending per the Python-parity positioning in §3.9.*

- *Egui pre-sign-off seam-level read absorbed: seven additional pins landed without further review cycle. §8.1 verb-op signature constraints (verb-ops MUST NOT take Shell; outputs are typed and MAY be stream-typed for streaming verbs; alias resolution at dispatcher tier, not at verb-parser or verb-op). §6.4 group-resolved-path-error shape (MUST return Result::error indicating fan-out required; MUST NOT silently succeed). §6.5 (new) pwd reverse-resolution at display time. §3.4 clarification that the `:` reservation is grammatical not character-level (entities may be named `handler:register`; parsers MUST handle non-structurally within segments). §6.1 alias-name grammar — **Unicode-permissive with structural-only restrictions** (the original ASCII-only proposal was reframed against the i18n-first principle); NFC normalization SHOULD be applied at alias-table-write boundary. §4.6 sub-op factoring at verb-op layer named as per-impl choice. §14.6 (new) broader system-wide i18n review flagged as future architectural work surfaced by the alias-grammar discussion.*

- *Workbench-go final critical pass + sign-off: all seven pre-sign-off pins surveyed against shipped behavior — six already-correct / trivially-satisfied / half-hour-fix; one substantive (§8.1 alias-resolution-at-dispatcher forces structural change to their `Registry.Dispatch` flow). Workbench-go's analytical move: §8.1's dispatcher-tier pin makes parametric dispatch the path of least resistance, so they planned an upgrade from "registry of typed Commands" toward parametric dispatch. After the post-sign-off §8.1 softening (closed-match-with-minimum-metadata explicitly permitted), they revised to the smaller path. Owed-1 (parser migration) + Owed-4 (pwd reverse-resolution) + Owed-5 (NFC normalization) bundled and shipped in commit `fcae99b`; Owed-3 (dispatcher-tier alias resolution, minimum-metadata variant) shipped in commit `5db125d`; closure addendum in commit `4696e7d`. All five owed items ✓ shipped (Owed-2 verb-op extractions remain opportunistic by design). `make test-shellcmd test-shell test-shellpanel test-shellboot` green; `make build` clean. Filed as a closure addendum to the workbench-go shell-framing review plus a workbench-go status note. **Signed off + implementation shipped.***

- *Workbench-go implementation-phase findings absorbed: two architectural items pulled back from build validation. (1) The `sh.Resolve` idempotency property in §8.1 — alias-resolution functions MUST be idempotent on already-resolved inputs; this is what makes the minimum-metadata path tractable for impls with flag-interleaved verbs (`ls -l @alias/path`, etc.). Validated by workbench-go's `TestDispatcher_ResolveArgs_IdempotentWithHandlerResolve`. (2) Multi-alias reverse-resolution tiebreak informational note in §6.5 — workbench-go shipped most-recent-wins via `peerMap[peerID]` ordering as the first cross-impl reference; guide records this as informational without normatively pinning, leaving cross-impl convergence to §11.3 conformance work.*

- *Egui Rust pilot validation (one-verb migration of `cat` through the four-layer factoring + dispatcher-tier alias resolution): all crate tests green; WASM build clean; pin set held without architectural surprise. Per-impl factoring shape diverged from workbench-go's central-table approach — egui used per-verb `dispatch_<verb>` helper functions as an incremental migration pattern (vs Go's central `Command.ResolveArgs + PathArgs(positions...)`). Both shapes satisfy §8.1 because the pin specifies the location of resolution (dispatcher tier), not the dispatcher's internal shape. **Two-language, two-shape, same-pin convergence is itself signal that the §8.1 abstraction level is correct.** Streaming-shape validation (§8.1 "verb-op outputs MAY be stream-typed") remains pending across all impls — coverage lands when the first streaming verb (`query`, `tail`) migrates through the four-layer factoring; flagged as an open cross-impl validation point but not blocking. Egui continues with the remaining 17-verb sweep in parallel with workbench-go's already-shipped refactor; no coordination dependency between them per the layer's design.*

- *Egui final pass + sign-off: all seven pins land cleanly and consistently across §3.4 / §4.6 / §6.1 / §6.4 / §6.5 / §8.1 / §14.6. One small tension flagged — §8.1 dispatcher-tier pin vs §5.2 closed-match per-impl option — independently surfaced by workbench-go and resolved by the same-day addition above (closed-match dispatchers MUST carry minimum per-verb path-arg-position metadata; parametric form satisfies intrinsically). Three small ambiguities flagged as cross-impl-conformance-future-work (§6.5 multi-alias-tiebreaker, §6.4 group-fanout error code identifier, Tier A verb registration arg-schema). **Signed off** with the §8.1+§5.2 clarification absorbed.*

- *Godot explicit post-amendment sign-off pending against the sign-off request to the Rust teams. Their original concerns are all addressed by the absorbed pins; pending response is procedural.*

- *`entity-core-py` review pending per the Python-parity positioning in §3.9.*

*Per `[Land-now-coordinate-during-build]`, the guide stands as normative direction; further cross-impl feedback during build amends via proposal. The three nice-to-have clarifying questions surfaced in the final passes — verb-op PeerCtx shape, pwd multi-alias tiebreaker, §6.4 group-fanout error code identifier, Tier A path-arg-schema convention — are deliberately left open as build-time discoverable; cross-impl conformance work (§11.3) will pin them if divergence becomes visible.*
