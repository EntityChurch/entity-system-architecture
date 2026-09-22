# Guide: Peer Concerns and Namespace Conventions

**Status**: Active
**Source:** Grounded in the web-team SDK namespace-and-peer model. The Rev 6 SDK-Identity-Infrastructure pass added §4.1 reserved-prefix entries for `system/attestation/`, `system/quorum/`, `system/identity/`, `system/role/`, `system/group/`, and the cross-impl UI conventions reconciliation (Godot third-impl contribution) added the §3 multi-peer-per-app variant + §4.3 app-id parameterization SHOULD.

---

## What This Guide Covers

How to think about what a peer is, what defines it, and how to organize its tree. These are conventions — the SDK doesn't enforce them. A peer is whatever you configure it to be: its identity, its handlers, its capability grants, and what's in its tree.

---

## 1. What Defines a Peer

A peer is NOT defined by a "type" label. A peer is defined by:

- **Its handlers.** What operations it can process. A peer with `system/tree` + `system/query` + `local/files` serves files and answers queries. A peer with just `system/tree` stores entities.
- **Its capability grants.** What it allows connecting peers to do. A peer that grants `{handlers: ["system/tree"], operations: ["get", "list"], resources: ["public/*"]}` is read-only on `public/`. A peer that grants everything is wide open.
- **Its tree contents.** What's actually in the tree. Handlers installed, types registered, data stored.
- **Its identity.** Ed25519 keypair. Cryptographic root for capability chains.

When we talk about "device peer" or "session peer" below, those are shorthand for common capability/handler/tree combinations. They're archetypes — patterns that recur — not a type system.

---

## 2. The Concern Matrix

Every peer sits somewhere on these dimensions. The dimensions are independent — any combination is valid.

### 2.1 Lifecycle

How long does this peer exist?

| Lifecycle | Description | Example |
|-----------|-------------|---------|
| **Persistent** | Survives restarts. Lives as long as the hardware/service. | Peer on a home server |
| **Long-running** | Extended lifetime, not hardware-bound. | Desktop application's peer |
| **Session** | Created for active use, cached between uses. | User's login context |
| **Ephemeral** | Created for a task, discarded after. | One-off sync worker |

### 2.2 Autonomy

Who controls this peer's lifecycle?

| Autonomy | Description | Example |
|----------|-------------|---------|
| **Autonomous** | Self-managing. Starts on boot, runs until stopped. | Server peer |
| **Managed** | Another peer controls creation/teardown. | Session peer managed by an application |
| **Embedded** | Runs inside an application as a library. | App using entity system internally |

### 2.3 Visibility

Does the user know this peer exists?

| Visibility | Description | Example |
|------------|-------------|---------|
| **Transparent** | User interacts directly. | Peer in entity shell |
| **Abstracted** | User interacts through a UI layer. | Peer behind a desktop app |
| **Invisible** | User has no idea the entity system is involved. | Internal data layer |

### 2.4 Capability Surface

What does this peer allow?

| Surface | Description |
|---------|-------------|
| **Wide** | Broad grants — full tree access, many operations. Development, single-user. |
| **Scoped** | Grants restricted by path prefix, operation set, handler set. Multi-user, production. |
| **Minimal** | Read-only access to specific subtrees. Public-facing, untrusted connections. |

### 2.5 Persistence

How is storage configured?

| Storage | Description |
|---------|-------------|
| **Durable** | SQLite or equivalent. Survives process restarts. |
| **Cached** | Durable but expendable. Can be rebuilt from other peers. |
| **In-memory** | Lost on shutdown. Tests, ephemeral workers. |

---

## 3. Common Archetypes

These are recurring combinations of the dimensions above. They're useful vocabulary, not protocol categories.

### Device Archetype

Persistent + autonomous + durable storage + wide capability surface (for local use).

Owns the machine's resources. Provides persistent storage. Other peers connect to it.

**Typical handlers:** `system/tree`, `system/query`, `local/files`, extensions (subscription, history, revision, etc.)

**Typical tree:**
```
system/                          Protocol infrastructure
host/                            Machine resources
    files/{mount}/               Filesystem bridges
    processes/                   Process tree
storage/{identity}/              Identity-scoped persistent data
```

`host/` is what makes it a "device" archetype — it exposes the machine.

### Application Archetype

Long-running + autonomous or embedded + variable visibility.

A program using the entity system. Ranges from invisible library usage to fully entity-native interface.

| Integration depth | Visibility | SDK depth |
|-------------------|------------|-----------|
| Library | Invisible | L0-L1 |
| Connected | Abstracted | L1-L2 |
| Native | Transparent | L1-L4 |

**Typical tree (native):**
```
system/                          Protocol infrastructure
app/{app-id}/                    Application state
    workspace/                   UI state
    settings/                    Configuration
```

#### Multi-peer-per-app variant

Native applications **MAY** run more than one peer in a single process. The recurring shape:

- One **app peer** owning `app/{app-id}/workspace/...` and `app/{app-id}/settings/...`. Typically no TCP listener and minimal sync engines so UI ephemera (window state, draft inputs, selections) don't broadcast to the network. Always present, regardless of network state.
- Zero or more **network peers** connected to the entity network. Full sync / replication / TCP listener. Browsed by application UI as data sources. User-managed lifecycle (connect, disconnect, add, remove).

This composes cleanly with the namespace conventions: `app-id` already scopes UI state away from network peers, and the L0/L1 access model treats each peer as an independent tree. Cross-peer routing inside the application uses the same SDK primitives as cross-process routing — `execute()` against the chosen peer, `watch()` against its tree.

**When to use this variant:** native applications where UI state must persist independently of network connectivity (most desktop apps), where the user manages multiple independent peer connections (multi-account UIs), or where embedding the app's own peer at the top of the L0 boundary simplifies capability reasoning. The single-peer-playing-multiple-roles shape (§4.4) remains valid and simpler for less differentiated cases.

### Session Archetype

Session lifecycle + managed + abstracted + scoped capabilities.

Created when a user authenticates. Gets capability grants derived from the user's identity. Connects to device peers to access data.

**Typical tree:**
```
system/                          Protocol infrastructure
app/{app-id}/                    Application state
knowledge/                       Synced content
projects/                        Synced content
```

### Service Archetype

Persistent or long-running + autonomous + scoped capabilities.

Provides functionality to connecting peers. Content at top level, access controlled by grants.

**Typical tree:**
```
system/                          Protocol infrastructure
(whatever the service provides)
```

---

## 4. Namespace Conventions

### 4.1 Reserved Prefixes

> ⭐ **Read this before the table: `app/…` names TWO different namespaces and they are the same bytes.**
>
> - **A type-tag namespace.** `app/feed/entry` is a value in an entity's `type` field. It is not a
>   location and nothing resolves it in a tree.
> - **A tree-path namespace.** `/{peer}/app/…` is a location in a peer's tree.
>
> **They share a spelling, they are not the same space, and a document pinning one MUST say which**
> (`GUIDE-APPLICATION-DEVELOPMENT.md` §2.3). This is stated because it is not a theoretical hazard:
> a static analyzer written by the party that owns this vocabulary has confused the two in four
> distinct shapes — a prefix with a trailing slash, a complete pinned path with none, a parametric
> type family, and a comment *explaining the confusion*, parsed as an emission. **A heuristic is the
> only available discriminator between them and it fails on exactly the case a convention pins.**
>
> **And inside the tree-path space there are two opposite purposes**, which §4.1b separates: an
> application's **private** working state, and a convention's **published** index addressed to a
> stranger.

Each extension's own spec is the authority for the namespace it owns and the layout inside it; the rows below are the reserved prefixes an implementer meets first, not the whole set. **Installation** at `system/*` is governed by two different rules on two different paths: `ENTITY-CORE-PROTOCOL.md` §6.2 (the dispatch path — reserved, `403 forbidden_pattern`) and `SDK-OPERATIONS.md` §11.6 (the in-process path — how a peer's own standard extensions are installed, and not constrained by §6.2). The `(When EXTENSION-X registered.)` rows below are the output of that second path.

| Prefix | Meaning | Who uses it |
|--------|---------|-------------|
| `system/` | Protocol infrastructure. Handlers, types, identity, capabilities, extension state, config. | Every peer. Application code reads via SDK operations but doesn't write directly. |
| `system/attestation/` | (When EXTENSION-ATTESTATION registered.) Substrate signed-claim entities. **Application code MUST NOT write directly via `tree:put`** — go through `system/attestation:create` / `:supersede` / `:revoke` per `EXTENSION-ATTESTATION.md` v1.2 §6 and the closed-namespace ownership rule §7. | Substrate-aware peers. |
| `system/quorum/` | (When EXTENSION-QUORUM registered.) K-of-N node entities + self-event attestations. Closed-namespace ownership per `EXTENSION-QUORUM.md` v1.2 §3.4 — other extensions MUST use their own top-level namespace (`system/history/...`, `system/role/...`), NOT nest under `system/quorum/{q}/`. Use `system/quorum:create` / `:update` / `:publish` / `:verify` for writes. | Quorum-aware peers. |
| `system/identity/` | (When EXTENSION-IDENTITY registered.) Identity infrastructure. `peer-config`, identity-cert attestations (per audience tier `internal/public/relationships`), identity-binding helpers, controller-events stream. Application code MUST NOT write directly — go through `system/identity:configure` / `:create_attestation` / `:supersede_attestation` / `:revoke_attestation` / `:publish_attestation` per `EXTENSION-IDENTITY.md` v3.5 §6. Bootstrap (the first `:configure` call) goes through the L0 helper library defined in `SDK-IDENTITY-INFRASTRUCTURE.md` §7. | Identity-aware peers. |
| `system/role/` | (When EXTENSION-ROLE registered.) Role infrastructure. Role definitions, assignments, exclusions, derived-token linkage. Per role v2.0, role-definition writes go through `system/role:define` (default) — direct `tree:put` to role-namespace paths is reserved for system-extension and administrative use. | Role-extension-aware peers. |
| `system/group/` | (When EXTENSION-GROUP registered, v1.5 sweep landing post-substrate.) Group infrastructure. Member entities, subgroup entities, governance entities, acting-on-behalf-of attestations. Same write discipline as identity. | Group-extension-aware peers. |
| `system/runtime/` | Runtime-instantiated machinery that doesn't fit any single extension's namespace. Per-call, ephemeral; system-privileged; unfitted-elsewhere. Parallel to `app/{app-id}/` for application content. Specs describing particular purposes enumerate sub-purposes. See `SDK-OPERATIONS.md` §11.6.7. | SDK / kernel runtime. |
| `host/` | Machine resources. Filesystem, processes, network, hardware. | Peers exposing the local machine (device archetype). |
| `app/{app-id}/` | **Private** per-application state. Workspace, settings. **Addressed to nobody** — not published, and a publisher's projection excludes it. | Peers running applications. Scoped by app ID so multiple apps can coexist. `{app-id}` is claimed by writing (`GUIDE-ENTITY-WORKBENCH-APP.md` §3), **subject to the closed exclusion set in `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §4.1b (this guide, below)**. |
| `app/{convention}/` | **Published** application-convention data — an index or well-known path a convention pins, **addressed to a stranger**. The declaring convention's own spec is the authority for the layout inside it. | Every peer publishing under that convention. **The set is closed and enumerated in §4.1b**; it is not open to an application. |
| `local/` | Device-local data and handlers. Used by handlers like `local/files`, with associated config at `system/config/local-files/...` (or similar per the handler's convention). | Device-local extensions. |
| `bridge/` | External system connections *reached across a boundary* — version control, mail, a package store, a third-party web origin. **Reserved; no occupants yet.** | Peers with external integrations. |
| `storage/{identity}/` | Identity-scoped persistent data on a shared peer. | Peers providing multi-user storage (device archetype). |

#### 4.1a `host/` versus `local/` versus `bridge/` — what the split means

**Three of the prefixes above reserve space for the world outside the protocol, and the line between them is *what we are standing on* versus *what we are reaching out to*.** It was arrived at by practice before it was written down, and it is worth stating because it is not obvious from the names:

- **A filesystem or process bridge is assumed present.** The system runs **on** it. It is substrate as well as adapter, not an optional integration a deployment may or may not have. That is what `host/` and `local/` are for.
- **A version-control, package-store or mail bridge is an optional integration reached across a boundary.** That is `bridge/`.

⚠ **Three caveats, because this area is less settled than the table implies.**

1. **`host/` currently has no occupants and `local/` holds the filesystem** — the one case `host/` names explicitly. **A reserved prefix lost its declared subject to a neighbour**, because reserving a prefix and specifying one are different acts and only the second leaves a document. Treat `local/files` as the shipped fact and the two prefixes as unreconciled.
2. **A name that asserts *locality* is falsified by a network-mounted filesystem**, which is reached through the same interface and is not local. A prefix naming *whose machine* survives that case; one naming *how near the bytes are* does not. The reconciliation is expected to be **additive** — a new namespace alongside the old — rather than a rename.
3. **`bridge/` is reserved and empty while the corpus's forward references to bridges spell `system/bridge/...` and `app/bridge/...`.** See `GUIDE-BRIDGE-EXTENSION-DEVELOPMENT.md` §6.1 — **the first bridge specification authored decides this, and no precedent here is binding on it.**

#### 4.1b The declared convention namespaces under `app/` — a closed set

**A published application-convention namespace under `app/` is drawn from this set, and this set is
the whole of it.** A convention that pins a tree path declares its namespace in its own specification
(`GUIDE-APPLICATION-DEVELOPMENT.md` §2.3) and it is listed here.

| Namespace | Declared by | What is under it |
|---|---|---|
| `app/feed/` | `APP-CONVENTION-FEED.md` §4.2 | the index head and its pages; any further well-known path that convention pins |

**[MUST NOT] An application-chosen `{app-id}` is equal to a namespace in this table.** Everything else
about claiming an app-id is unchanged: there is still no registry and no reservation step, because the
forbidden set is small, closed and published, so an app can check it without asking anyone.

**Why this rule and not a separate root for published data.** *Partition by root* — a reserved
top-level segment for convention data, leaving `app/{app-id}/` unambiguously private — is the cleaner
design, and it makes *"what does this peer publish"* answerable by looking rather than by rule.
**It is rejected on cost and not on merit:** it moves both of the feed convention's pinned paths,
which shipping implementations already publish *and* read, to buy a partition this table delivers
without moving a byte. ⚠ **If this set ever grows past a handful, revisit that** — a rule scales by
enumeration and a root scales by construction.

**What the collision was, so the rule is not mistaken for tidiness.** `{app-id}` is claimed by
writing, with no mechanism preventing any particular name. An application claiming the app-id of a
convention writes its **private** workspace over that convention's **published** index path. Both
documents were correct in isolation and neither acknowledged the other.

### 4.2 Content Domains

Beyond the reserved prefixes, peers can use whatever top-level paths make sense. Common conventions:

- `knowledge/` — user's knowledge base, articles, notes
- `projects/` — project-scoped content
- `public/` — content intentionally shared with other peers
- `temp/` — ephemeral working state

These are open. Applications define their own content domains.

#### 4.2a The user-content domain — `files/` and `imports/`

**Two content domains are named rather than open, because their whole value is that every host uses
the same one.**

| Domain | What is in it |
|---|---|
| `files/` | **A person's own content.** Documents, saves, working files — the things that belong to the person rather than to an application. |
| `imports/` | **Staging for a subgraph that arrived from elsewhere.** `imports/{stamp}/`. |

**[MUST] A person's own content is addressed under `files/`, not under an application namespace.**
`app/{app-id}/` is *per-application* state by definition, and a person's documents are not an
application's working state — they outlive any particular application and a second host should open
the same ones. A person's save file living under one application's id is what this rule exists to
stop.

**[MUST NOT] A host projects `files/` into the published root as a side effect of a write to it.**
A user-content domain is addressed to nobody until its owner addresses it to someone; publication of
anything under it is a separate explicit act.

> **This is a rule about publication and deliberately not about the grant table.** A deployment's
> grant posture is the deployment's choice and is not a convention's to mandate. What this says is
> what the domain **means** — and a deployment that wants its connections to exclude a person's
> files now has a declared name to express that against, which is the half that did not exist.
>
> ⛔ **It is not privacy, and MUST NOT be described as privacy, while a content read resolves bytes
> without regard to namespace.** A grant can withhold a file's *name* and not its *bytes*. That is a
> substrate gap, it is open, and nothing in this section closes it.

**[MUST] An imported subgraph lands under `imports/`, never directly in `files/`.** The reason is
provenance: an import merges a subgraph somebody else authored, and once it is indistinguishable from
the person's own content, no later act can recover the difference. Staging keeps *I made this* and
*this arrived* separable until a person places it.

**On a shared peer these nest inside the identity scope** — `storage/{identity}/files/…` — rather than
sitting at the top level. One rule, two deployments. *(Derived; no implementation runs this today.)*

⚠ **`local/` is not this.** That is the host-disk mount reached through its own handler — *the
machine's* files. These are entities the peer is the authority for. Two different things with a
similar-sounding name.

### 4.3 Naming Principles

- Full words: `host/` not `/dev`, `temp/` not `tmp/`, `bridge/` not `mnt/`.
- `system/` is protocol infrastructure, not a role.
- Application state at `app/{app-id}/` — scoped so multiple apps coexist.
- No language-specific prefixes (`rust_workspace/` → `app/{app-id}/workspace/`).
- App-side path helpers **SHOULD** parameterize `{app-id}` rather than hardcoding it as a constant. The dominant deployment today is one app per peer, but a constant turns multi-app coexistence on a peer into a refactor; a parameter keeps the door open at no cost. Default values are fine; the helper signature should accept an override.

### 4.4 The Current Reality

All implementations currently run a single peer playing multiple roles. Use all conventions in one tree:

```
/{peer-id}/
    system/                      Protocol infrastructure
    host/                        Machine resources (if mounted)
    bridge/                      External systems (if connected)
    app/{app-id}/                Application state — PRIVATE, not published
        workspace/               UI state
        settings/                Config
    app/{convention}/            Convention data — PUBLISHED (the §4.1b set)
    files/                       The person's own content
    imports/                     Staged arrivals, awaiting placement
    knowledge/                   Content
    projects/                    Content
    public/                      Shared
    temp/                        Ephemeral
```

**The two `app/` lines are the ones to read twice.** They are adjacent in the tree, they are
indistinguishable by shape, and one is addressed to a stranger while the other is addressed to
nobody. §4.1b's table is what sorts them, and it is the only thing that does.

As the system matures, these paths distribute across peers naturally. Same paths, different peers owning them.

---

## 5. Application State Conventions

### 5.1 Type Names

| Concept | Recommended type name |
|---------|----------------------|
| Generic setting | `app/state/setting` |
| Selection state | `app/state/selection` |
| Window configuration | `app/state/window` |
| The live window set | `app/state/window-index` |

`app/state/` prefix is language-neutral. Not `rust_workspace/settings`. Not `go_workspace/state`.

**`GUIDE-ENTITY-WORKBENCH-APP.md` §4.2 is the authority for this set**; the rows above are the common ones, not the whole table. An earlier `app/state/layout` row is **retired**: window *arrangement* — splits, ratios, sizing, stacking — is renderer decoration and stays per-implementation. What is portable is window *membership*, which `app/state/window-index` carries (`GUIDE-ENTITY-WORKBENCH-APP.md` §4.2a — **not** §4.2a of this guide, which is the user-content domain).

### 5.2 Path Convention

```
app/{app-id}/
    workspace/                   UI state (windows, layout, selection)
    settings/                    Application configuration
```

The `{app-id}` scoping lets multiple applications coexist on one peer without colliding.

---

## 6. Sync

**What syncs:** Identity-scoped data on shared peers (`storage/{identity}/`), content across a user's peer network, app state (configurable).

**What doesn't sync:** `host/` (machine-specific), `system/` (each peer's own), `temp/` (ephemeral).

**Granularity:** Per prefix, per peer, or on demand.

---

## 7. Multi-User

Handled through capability grants, not namespace mechanisms.

Each user identity gets grants scoped to their `storage/{identity}/` namespace on shared peers. Login creates a session with limited grants. No special multi-user infrastructure — the capability system provides it.

---

## 8. Progressive Usage

**Level 0 (CLI):** Run peers from command line. Entity shell. No application layer.

**Level 1 (Manual multi-peer):** Multiple peers, manual connections, CLI administration.

**Level 2 (Application):** Install an entity application. Login. Session peer handles everything. Infrastructure transparent.

**Level 3 (Organization):** Server peers with administered grants. Users connect through their applications.

Same protocol, same operations, same capability model at every level.

---

## 9. Mapping to Current Implementations

| Current | Archetype combination | Notes |
|---------|----------------------|-------|
| Rust PeerKind::System | Application + management concerns | Name should change — "system" collides with `system/` path |
| Rust PeerKind::Backend | Device-like concerns | WebSocket listener, provides connectivity |
| Rust PeerKind::Wasm | Application concerns | User-created, holds content |
| Go workbench | Application + management (fused) | Single peer per instance |
| Go entitysdk | SDK for any archetype | Executor, WorkspaceState, PeerContext |
| Godot | Application (native level), multi-peer variant | App peer for UI state + N user-managed network peers; SQLite from day one |

---

## 10. Open Questions

1. ~~**Content domain organization.** Should there be a container for user content on session-archetype peers, or is the peer's tree implicitly "the user's space"?~~ ✅ **ANSWERED — §4.2a.** There is a container and it is `files/`. *The peer's tree is implicitly the user's space* was the reading that let a person's documents sit under one application's id, which is how the question arrived back: from a shipping host that had to put them somewhere and had nothing declared to put them in.
2. **Bridge placement.** Identity-bound bridges (my git repo) vs machine-bound bridges (git repo on disk). `bridge/` on application peer vs `host/` on device peer?
3. **Session persistence.** How much state does a managing application cache between sessions?
4. **Peer discovery.** How do peers in a user's network find each other? Bootstrap config, network discovery, or capability chain?

---

*These are conventions, not requirements. The SDK operations spec (SDK-OPERATIONS.md) defines the normative interface. These patterns help peers compose well in multi-peer scenarios.*
