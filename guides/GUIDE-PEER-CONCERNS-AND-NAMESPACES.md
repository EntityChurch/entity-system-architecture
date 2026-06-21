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

| Prefix | Meaning | Who uses it |
|--------|---------|-------------|
| `system/` | Protocol infrastructure. Handlers, types, identity, capabilities, extension state, config. | Every peer. Application code reads via SDK operations but doesn't write directly. |
| `system/attestation/` | (When EXTENSION-ATTESTATION registered.) Substrate signed-claim entities. **Application code MUST NOT write directly via `tree:put`** — go through `system/attestation:create` / `:supersede` / `:revoke` per `EXTENSION-ATTESTATION.md` v1.2 §6 and the closed-namespace ownership rule §7. | Substrate-aware peers. |
| `system/quorum/` | (When EXTENSION-QUORUM registered.) K-of-N node entities + self-event attestations. Closed-namespace ownership per `EXTENSION-QUORUM.md` v1.2 §3.4 — other extensions MUST use their own top-level namespace (`system/history/...`, `system/role/...`), NOT nest under `system/quorum/{q}/`. Use `system/quorum:create` / `:update` / `:publish` / `:verify` for writes. | Quorum-aware peers. |
| `system/identity/` | (When EXTENSION-IDENTITY registered.) Identity infrastructure. `peer-config`, identity-cert attestations (per audience tier `internal/public/relationships`), identity-binding helpers, controller-events stream. Application code MUST NOT write directly — go through `system/identity:configure` / `:create_attestation` / `:supersede_attestation` / `:revoke_attestation` / `:publish_attestation` per `EXTENSION-IDENTITY.md` v3.5 §6. Bootstrap (the first `:configure` call) goes through the L0 helper library defined in `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` §7. | Identity-aware peers. |
| `system/role/` | (When EXTENSION-ROLE registered.) Role infrastructure. Role definitions, assignments, exclusions, derived-token linkage. Per role v2.0, role-definition writes go through `system/role:define` (default) — direct `tree:put` to role-namespace paths is reserved for system-extension and administrative use. | Role-extension-aware peers. |
| `system/group/` | (When EXTENSION-GROUP registered, v1.5 sweep landing post-substrate.) Group infrastructure. Member entities, subgroup entities, governance entities, acting-on-behalf-of attestations. Same write discipline as identity. | Group-extension-aware peers. |
| `system/runtime/` | Runtime-instantiated machinery that doesn't fit any single extension's namespace. Per-call, ephemeral; system-privileged; unfitted-elsewhere. Parallel to `app/{app-id}/` for application content. Specs describing particular purposes enumerate sub-purposes. See `SDK-OPERATIONS.md` §11.6.7. | SDK / kernel runtime. |
| `host/` | Machine resources. Filesystem, processes, network, hardware. | Peers exposing the local machine (device archetype). |
| `app/{app-id}/` | Per-application state. Workspace, settings. | Peers running applications. Scoped by app ID so multiple apps can coexist. |
| `local/` | Device-local data and handlers. Used by handlers like `local/files`, with associated config at `system/config/local-files/...` (or similar per the handler's convention). | Device-local extensions. |
| `bridge/` | External system connections. Git, HTTP, etc. | Peers with external integrations. |
| `storage/{identity}/` | Identity-scoped persistent data on a shared peer. | Peers providing multi-user storage (device archetype). |

### 4.2 Content Domains

Beyond the reserved prefixes, peers can use whatever top-level paths make sense. Common conventions:

- `knowledge/` — user's knowledge base, articles, notes
- `projects/` — project-scoped content
- `public/` — content intentionally shared with other peers
- `temp/` — ephemeral working state

These are open. Applications define their own content domains.

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
    app/{app-id}/                Application state
        workspace/               UI state
        settings/                Config
    knowledge/                   Content
    projects/                    Content
    public/                      Shared
    temp/                        Ephemeral
```

As the system matures, these paths distribute across peers naturally. Same paths, different peers owning them.

---

## 5. Application State Conventions

### 5.1 Type Names

| Concept | Recommended type name |
|---------|----------------------|
| Generic setting | `app/state/setting` |
| Selection state | `app/state/selection` |
| Window configuration | `app/state/window` |
| Layout | `app/state/layout` |

`app/state/` prefix is language-neutral. Not `rust_workspace/settings`. Not `go_workspace/state`.

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

1. **Content domain organization.** Should there be a container for user content on session-archetype peers, or is the peer's tree implicitly "the user's space"?
2. **Bridge placement.** Identity-bound bridges (my git repo) vs machine-bound bridges (git repo on disk). `bridge/` on application peer vs `host/` on device peer?
3. **Session persistence.** How much state does a managing application cache between sessions?
4. **Peer discovery.** How do peers in a user's network find each other? Bootstrap config, network discovery, or capability chain?

---

*These are conventions, not requirements. The SDK operations spec (SDK-OPERATIONS.md) defines the normative interface. These patterns help peers compose well in multi-peer scenarios.*
