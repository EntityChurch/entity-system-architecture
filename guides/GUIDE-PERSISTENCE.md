# Guide: Persistence

**Status**: Active
**Source:** A three-impl response surfaced the persistence-grounding gap; this guide is rewritten to defer to the existing specs rather than redefine.

---

## What this guide covers

How peers persist on disk. The on-disk layout for entity-core peers is **already normatively specified** by `SDK-OPERATIONS.md` §15 (Configuration Directory) and extended for identity-aware peers by `SDK-IDENTITY-INFRASTRUCTURE.md` §8.4 (identity bundle layout). This guide reads those specs back into developer-readable form, fills small alignment gaps (storage filename, env-var name), and elevates the dispatch-index re-registration callout from spec footnote to first-class guidance.

In scope:

- The three-layer mental model — configuration directory, per-peer state, runtime state in the tree.
- Platform mappings for the configuration directory (Linux / macOS / Windows / Godot).
- Storage backend selection (memory / SQLite / future).
- What survives restart vs what's recomputed.
- Identity-aware vs V7-only modes.
- Application state on disk — there isn't any direct representation.
- Application startup sequence including handler re-registration.

Out of scope:

- **Redefining the on-disk layout.** SDK-OPERATIONS §15 is normative; this guide reads it back.
- Specific SQLite schema design (implementation detail of `core/store/sqlite.rs`).
- Cross-peer sync at startup (handled by sync engines).
- Backup / disaster recovery beyond a one-line "back up the configuration directory."
- Cross-impl entity-tree exchange formats (covered by V7 wire-format).

---

## 1. The three layers

Persistence sits in three layers, each with a clean responsibility.

### 1.1 Configuration directory — where stuff lives on disk

`~/.entity/` (SDK-OPERATIONS §15 default; env-var/CLI-overridable per §15.3).

```
~/.entity/
├── identities/{name}/          Keypair material (V7-only) or identity bundle (identity-aware)
└── peers/{name}/               Per-peer state
    ├── keypair                 Runtime peer keypair
    ├── config.toml             Startup config (listen addr, storage backend, extensions)
    ├── identity.toml           OPTIONAL — present iff peer is identity-aware
    └── store.db                The peer's tree (when storage_backend = "sqlite")
```

The configuration directory is **per-user**, not per-app. Multiple apps coexisting on a machine share `~/.entity/` the same way they share `~/.ssh/` or `~/.gnupg/` — by convention, by user filesystem permissions, and by per-peer naming inside the directory.

### 1.2 Per-peer state — one peer, one tree

Peers are indexed by **name** (a local user-chosen alias) inside `~/.entity/peers/{name}/`. The peer-id is derived from `peers/{name}/keypair` and is not the directory name.

A peer is one tree (one DB), one keypair, one optional identity binding, one config. There is no separate "app peer" vs "network peer" at the persistence layer — there are just peers. Whether a peer's tree happens to host application namespaces (`app/workbench-01/...`, `app/calculator/...`) or only system + protocol state is a property of the tree's contents, not a property of the on-disk structure.

If a user runs multiple impls of the same conceptual application (Go workbench, egui-Rust workbench, Godot workbench), they spin up peers under different names — `peers/workbench-go/`, `peers/workbench-egui/`, `peers/workbench-godot/`, or whatever the user picks. Different names = different keypairs = different peer-ids = no interference. They can also share, by pointing multiple impls at the same `peers/{name}/`, but that means running the same peer through different impls — uncommon and not generally recommended (concurrent live runs of the same peer-id is not supported behavior).

### 1.3 Runtime state lives in the tree, not on disk

Once a peer is up, its operational state — known peers, transports, configuration parameters, peer aliases, active connections — lives **in its own tree** as entities under `system/transport/*`, `system/config/*`, `system/peer/*`, `system/connection/*` (per `proposals/PROPOSAL-OPERATIONAL-STATE-TYPES.md`). A handler that needs the keepalive interval reads `system/config/connection`. On crash restart, the peer reads its own tree to recover known peers and their transports — **the tree is the recovery mechanism.**

What's on disk, then, is:

- **Bootstrap state** — keypair, minimal `config.toml` with enough to start the peer (storage backend, listen addr, extensions list). Read once at startup.
- **The tree itself** (`store.db`) — once the peer is up, all further state changes go through the tree, persisted by the storage backend.
- **Identity bundle** (when identity-aware, per SDK-IDENTITY-INFRASTRUCTURE §8.4) — controller/agent/quorum keypairs, identity manifest. Loaded at startup, referenced by `peers/{name}/identity.toml`.

Application state (`app/{app-id}/...`) is part of the tree, persists with it, and isn't a separate concern at the on-disk layer.

---

## 2. Platform mappings for the configuration directory

`~/.entity/` is the SDK-OPERATIONS default; deployments often want platform-native data dirs. SDK-OPERATIONS §15.3 MUSTs env-var/CLI override support; the recommended env-var is `ENTITY_DATA_DIR`.

| Platform | Typical resolution | Mechanism |
|---|---|---|
| Linux native | `$XDG_DATA_HOME/entity/` (default `~/.local/share/entity/`) | XDG Base Directory; `directories` crate / equivalent |
| macOS native | `~/Library/Application Support/entity/` | Apple HIG; `directories` crate |
| Windows native | `%APPDATA%\entity\` | Known-folder API; `directories` crate |
| Godot project | `user://entity/` | Godot's `user://` resolves per-OS |
| Server / containerized | `/var/lib/entity/` or env-var-set path | Operator deployment choice |
| Test / hermetic | `$TMPDIR/entity-test-{n}/` | Per-process, cleaned up |

The default `~/.entity/` is fine for development; production deployments override.

---

## 3. Storage backend selection

| Backend | When | Caveats |
|---|---|---|
| **In-memory** | Tests, ephemeral workers, library embedding where the host owns persistence. | Lost on shutdown. |
| **SQLite** | Native desktop apps, server peers, anything user-persistent. | Single-process write access; WAL mode handles concurrent readers. Recommended `store.db` filename. |
| **Future backends** | (Reserved) — RocksDB, FoundationDB, browser IndexedDB for WASM, custom. | The store trait is normative; backends are implementation-defined per SDK-OPERATIONS §16.4. |

Per peer, picked at construction time via `config.toml` `storage_backend = "memory" | "sqlite" | ...` (current Rust convention) or equivalent.

Decision flow:

- Native desktop apps → SQLite by default.
- Server peers → SQLite.
- CLI tools → memory unless persistence is meaningful for the use case.
- Tests → memory; hermetic temp dir for SQLite tests.
- Library embedding → ask the host application.

---

## 4. What survives restart, what's recomputed

| State | Persists? | Notes |
|---|---|---|
| Entity tree (the peer's data) | Yes (with persistent backend) | The whole point of persistence. App state, system/operational state, identity bindings — all in here. |
| Keypair | Yes (file in `peers/{name}/keypair`) | Independent of the tree's storage backend. |
| Identity bundle | Yes (files in `identities/{name}/...`) | Per SDK-IDENTITY-INFRASTRUCTURE §8.4. |
| Bootstrap config | Yes (`peers/{name}/config.toml`) | What's needed to start the peer; not the same as runtime `system/config/*` entities. |
| Runtime operational state (`system/transport/*`, `system/config/*`, `system/peer/*`, `system/connection/*`) | Yes (lives in the tree) | Per PROPOSAL-OPERATIONAL-STATE-TYPES; recovered on restart by reading the tree. |
| Query indexes | Implementation-defined per EXTENSION-QUERY | Some persist, some always-rebuild. |
| Subscription state | No | `subscribe()` is per-process; re-establish at startup. |
| `watch()` sessions | No | L0 watch is in-memory pattern matching. |
| **Dispatch index (handler bodies)** | **No** | Manifest in tree survives; in-memory callable doesn't. **Application MUST re-register at startup** per SDK-OPERATIONS §11.6.6. |
| In-flight continuations | If persisted via EXTENSION-CONTINUATION | Without that extension, restart loses in-flight work. |
| TCP listeners / sync engines | No | Re-established by peer construction. |

The dispatch-index row is the callout: **register your handlers in your startup path**, not just at install. The manifest in the tree is the *declaration* (handler exists, this op, this grant); the language-native callable that actually runs is held in the dispatch index in memory and is gone at restart.

---

## 5. Identity-aware vs V7-only modes

Both modes are valid per SDK-OPERATIONS §15.1 / §15.2 / §15.4.

### 5.1 V7-only mode

No identity extension. Flat keypair files under `identities/{name}/{public_key, private_key}`; `peers/{name}/` with `keypair` + `config.toml` + `grants.toml`. Suitable for development, simple deployments. Absence of `peers/{name}/identity.toml` signals V7-only.

### 5.2 Identity-aware mode

Identity extension installed. `identities/{name}/` becomes the bundle directory (controllers, agents, quorum constituents, `identity.toml`). `peers/{name}/identity.toml` references the bundle. See SDK-IDENTITY-INFRASTRUCTURE §8.4 for the full layout.

### 5.3 Migration

`BootstrapFromExistingKeypair` (SDK-IDENTITY-INFRASTRUCTURE §8.1) promotes a V7-only peer to identity-aware without changing the peer-id. Bundle is created in `identities/{name}/`, `peers/{name}/identity.toml` is added pointing at the bundle, the peer's keypair is unchanged. No retroactive migration of contacts.

---

## 6. Application state on disk — none directly

Worth saying explicitly because it counters a natural misconception:

**Application state has no direct on-disk representation.** When an application writes to `app/{app-id}/workspace/windows/3/state`, that's a write into the host peer's tree. The on-disk effect is "the peer's `store.db` got an updated row." There is no `app-peer.db` separate from any other peer's DB; there is no per-app filesystem directory. Apps live inside peer trees.

Implications:

- **Sync property** (per `GUIDE-ENTITY-WORKBENCH-APP.md`): pulling `app/{app-id}/*` out of the peer's tree gives you the app's portable state, regardless of where the tree is stored.
- **Multi-app on one peer** (workbench + calculator coexisting) shares one DB because they share a peer.
- **Multi-impl on one machine** (Go workbench + Godot workbench) means two different peers, two different DB files under different `peers/{name}/`.

---

## 7. Application startup sequence

Concretely:

1. **Construct the peer** — load keypair, open DB, replay operational state from tree.
2. **Re-register handlers** — for each handler the application provides, call `register_handler` so the in-memory dispatch index is populated. Steps 1 and 2 together produce a peer that's ready to serve requests.
3. **Re-establish subscriptions** — the application's panels / watchers re-subscribe to the paths they care about.
4. **Open listeners** (if the peer accepts incoming connections).

Steps 1 and 4 are SDK + peer-construction responsibility; steps 2 and 3 are application responsibility. Forgetting step 2 means handlers are silently unreachable until re-registration.

---

## 8. Operational notes

### 8.1 Filesystem permissions on Unix

Keypair files **MUST** be `0600` (existing convention; user-private). DB files: `0600` recommended (may contain identity-bound private data). Identity bundle files inherit the `0600` standard.

### 8.2 Backup and restore

Simple form: back up `~/.entity/` (or whatever the configuration directory resolves to). Restore by copying back; preserve the `0600` permissions on keypair files. Don't back up while the peer is live unless the storage backend supports hot backups (SQLite WAL mode does support this with the right flags; consult backend documentation).

Richer backup tooling (incremental backup, encrypted backup, off-machine custody) is operator territory and not specified here.

### 8.3 Web (egui-WASM) persistence

The configuration-directory abstraction maps to IndexedDB / OPFS / LocalStorage on browser WASM rather than a filesystem path. SDK-IDENTITY-INFRASTRUCTURE §8.0 covers the computation/I/O separation for identity bundles; the same pattern applies to the tree's storage backend — pure logic + platform-specific persistence layer. Specific WASM persistence specifics land when an impl commits to an approach.

### 8.4 Multi-peer per app

Per `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §3 multi-peer-per-app variant: a native application **MAY** run multiple peers in one process (an internal app peer for UI state plus zero-or-more user-managed network peers for data). All those peers live under `peers/{name}/` with different names; nothing special at the on-disk layer.

---

## 9. References

External:

- `entity-core-rust/core/store/sqlite.rs` — SQLite backend implementation reference.
- XDG Base Directory Specification.
- Apple HIG — Application Support directory.

Internal (this repo):

- `sdk-domain/specs/SDK-OPERATIONS.md` §15 (Configuration Directory — load-bearing on-disk layout); §11.6 / §11.6.6 (handler registration and re-registration); §16.4 (implementation-defined sections).
- `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` §8.0 (computation/I/O separation); §8.1 (`BootstrapFromExistingKeypair`); §8.4 (on-disk identity bundle layout).
- `sdk-domain/guides/GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §2.5 (storage dimension); §3 (archetypes — including multi-peer-per-app variant).
- `sdk-domain/guides/GUIDE-ENTITY-WORKBENCH-APP.md` — companion guide; cross-references this for the persistent-vs-ephemeral line on application state.
- `core-protocol-domain/specs/ARCHITECTURE-IDENTITY-INFRASTRUCTURE.md`.
- `proposals/PROPOSAL-OPERATIONAL-STATE-TYPES.md` — `system/transport`, `system/config`, `system/peer/*` types in the tree.
- `proposals/implemented/PROPOSAL-GUIDE-PERSISTENCE.md` — proposal-track artifact this guide graduated from.
