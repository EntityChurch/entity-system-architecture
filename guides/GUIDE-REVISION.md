# Revision Extension: User Guide

**Status**: Draft
**Audience:** Application developers who know git and want version control over entity tree subtrees. No assumption that they've read the spec.
**Prerequisite for:** GUIDE-REVISION-AUTO-VERSION (auto-versioning, cross-peer sync composition)
**Spec reference:** EXTENSION-REVISION.md v2.8

---

## 1. What revision is (and isn't)

The revision extension turns a subtree of the entity tree into a content-addressed version DAG. Each version entry records a trie root (structural snapshot of all bindings under a prefix) and parent pointers. It is git's object model applied to the entity tree.

**What it IS:**

- A version history for tree bindings (path to hash mappings) under a configured prefix
- Content-addressed and immutable once shared — version hashes include their ancestry
- Prefix-scoped — you version what matters, not the entire tree
- Merge-only — no rebase, no history rewrite (content addressing makes rewrite structurally impossible)

**What it is NOT:**

- Not a file-level VCS — it versions tree bindings, not file contents. The entities at those bindings can be anything.
- Not a sync protocol — cross-peer sync is a composition of subscription + continuation + fetch + merge (see GUIDE-REVISION-AUTO-VERSION)
- Not automatic by default — auto-version is opt-in via config. This guide covers the manual workflow.

**Relationship to history.** The history extension (EXTENSION-HISTORY) records every individual write as a per-path linked list of transitions. Revision records named snapshots you share with collaborators. History is your undo log. Revision is your commit graph. They are independent — you can use either without the other, or both together. When both are active, history captures the fine-grained per-path journal; revision captures the coarse-grained DAG of complete-subtree snapshots.

---

## 2. The basics: commit, log, status, diff

The manual workflow with `auto_version: false` (the default):

### 2.1 Write entities to the tree

Write entities to tree paths under your prefix using `system/tree` put operations. These writes are not versioned until you commit.

### 2.2 Check current state — `status`

```
EXECUTE system/revision  operation: "status"
  params: {prefix: "project/"}
```

Returns:
- `head` — current version hash (null if no versions exist yet)
- `conflicts` — number of unresolved conflict entities
- `pending` — number of path changes since last version
- `remotes` — map of remote peer IDs to their last known version hash

**Git equivalent:** `git status`

### 2.3 Create a version — `commit`

```
EXECUTE system/revision  operation: "commit"
  params: {prefix: "project/", message: "Add login feature"}
```

Builds a Merkle trie from the current tree state under the prefix, creates a version entry (`{root, parents}`), and advances the head pointer.

Returns the version hash and the trie root hash.

**Git equivalent:** `git commit` — but there is no staging area. Commit captures everything under the prefix (minus excludes). There is no `git add`. If you want to exclude paths, configure them in the revision config's `exclude` list (see [Configuration](#9-configuration)).

**Messages.** The `message` field is optional. The version entry itself is purely structural — `{root, parents}` — and does not store the message. The handler MAY store messages as metadata entities in the tree by application convention. The version system is agnostic to what is in the trie.

### 2.4 Walk version history — `log`

```
EXECUTE system/revision  operation: "log"
  params: {prefix: "project/", limit: 20}
```

BFS walk from HEAD through the parent chain. Returns version hashes in reverse chronological order, with version entities included in the result envelope.

The `since` parameter stops the walk at a known version — useful for incremental history display.

**Git equivalent:** `git log`

### 2.5 Compare two versions — `diff`

```
EXECUTE system/revision  operation: "diff"
  params: {prefix: "project/", base: <hash_a>, target: <hash_b>}
```

Computes the Merkle trie diff between two versions' snapshots. Returns added, changed, removed, and unchanged paths.

The diff is efficient — it recursively descends the trie, skipping entire subtrees where hashes match. Deep trees with localized changes are fast.

**Git equivalent:** `git diff <commit1> <commit2>`

---

## 3. Branching and switching context

Branches are hash pointers stored at `system/revision/branches/{prefix}{name}`. Creating a branch is a single tree write — there is no copy.

### 3.1 Create a branch

```
EXECUTE system/revision  operation: "branch"
  params: {prefix: "project/", action: "create", name: "feature-login"}
```

Creates a named pointer to the current head (or to a specific version via the optional `from` parameter).

### 3.2 List branches

```
EXECUTE system/revision  operation: "branch"
  params: {prefix: "project/", action: "list"}
```

Returns all branch names with their version hashes, plus the active branch name.

### 3.3 Switch branches — `checkout`

```
EXECUTE system/revision  operation: "checkout"
  params: {prefix: "project/", branch: "feature-login"}
```

Resolves the branch's version, computes the diff between the current tree state and the target version's trie, and applies the changes. Also advances head and sets the active branch.

You can also checkout by version hash instead of branch name:

```
EXECUTE system/revision  operation: "checkout"
  params: {prefix: "project/", version: <hash>}
```

This puts you in **detached HEAD** mode — the active branch is cleared. Commits still work but no branch pointer advances.

### 3.4 Active branch

The system tracks which branch you are "on" via `system/revision/active-branch/{prefix}`. Commits advance both the head pointer and the active branch pointer. Checkout-by-branch sets it; checkout-by-version clears it.

### 3.5 Stash

Just create a branch. Branching is a pointer write, not a copy. There is no separate stash mechanism and none is needed.

### 3.6 Uncommitted changes

Checkout applies the target version's trie regardless of uncommitted changes. If you have uncommitted modifications, they are overwritten. **Commit first to preserve them.** The handler MAY warn about uncommitted changes but will not refuse the checkout — the caller's intent is explicit.

---

## 4. Merging

Three-way merge using the common ancestor.

### 4.1 How merge works

```
EXECUTE system/revision  operation: "merge"
  params: {prefix: "project/", remote_version: <hash>}
```

The merge operation:

1. Finds the common ancestor of the current head and the remote version (bilateral BFS on the DAG).
2. Flattens both versions' tries into binding maps.
3. Walks all paths, comparing each against the ancestor:
   - Same on both sides: no conflict.
   - Only one side changed: take the changed side.
   - Both changed differently: apply the merge strategy (see [Merge strategies](#42-merge-strategies)).
4. Checks for oscillation (has this trie root appeared in recent ancestry?).
5. Creates a merge version entry with `parents: [head, remote_version]` (sorted by hash).
6. Applies the merged bindings to the tree.

Returns one of: `merged` (clean), `merged_with_conflicts` (some paths conflicted), `fast_forward` (head advanced without merge commit), `already_in_sync`, `already_ahead`, `converged_identical` (trie roots already match), or `oscillation_detected`.

### 4.2 Merge strategies

When both sides changed the same path, the merge strategy decides what happens. The resolution cascade:

1. **Per-type config** — check `system/revision/config/merge/type/{type_name}` for a type-specific strategy.
2. **Per-path config** — check `system/revision/config/merge/path/{name}` for a path-pattern strategy.
3. **Default three-way** — field-level diff3 on entity data.
4. **Conflict entity** — if nothing resolves it.

Built-in strategies:

| Strategy | Behavior |
|----------|---------|
| `three-way` | Diff3 with common ancestor. Auto-merge non-overlapping changes. Conflict on overlap. |
| `source-wins` | Always take the incoming (remote) version |
| `target-wins` | Always keep the local version |
| `lww` | Last-write-wins by version timestamp |
| `keep-both` | Store both versions (local at path, remote at `{path}.keep-both-{hash_prefix}`) |
| `manual` | Always create a conflict entity |

Custom merge handlers can be configured by setting `strategy: "handler"` with a handler path. The handler receives base, local, and remote entity hashes and returns a merged entity or conflict.

### 4.3 Merge ordering

The `merge_order` config controls how "local" and "remote" sides are assigned:

- **`deterministic`** (default): sides are assigned by hash comparison. All peers compute the same merge regardless of who initiates. Required for peer-to-peer convergence.
- **`caller-perspective`**: caller's head is "local", incoming is "remote". Git-like but can cause DAG divergence in P2P topologies with asymmetric strategies.

Use `deterministic`. Always. See GUIDE-REVISION-AUTO-VERSION §3 for the full rationale.

### 4.4 Conflict entities

When merge cannot resolve a path, a conflict entity is stored at `system/revision/conflicts/{prefix}{path}`. The entity at the conflicting path is NOT replaced — documents never disappear. Only version-aware handlers/UIs check the conflicts path to discover and present unresolved conflicts.

```
; After merge with conflict:
project/config.toml                                  -> hash_of_local_config  (unchanged)
system/revision/conflicts/project/config.toml        -> hash_of_conflict_entity
```

The conflict entity records: base hash, local hash, remote hash, strategy attempted, and version references for both sides.

**Conflicts are local.** Conflict entities live under `system/`, outside the versioned prefix. They are not included in version snapshots and not synced to other peers. The version DAG records the merge (both parents), but conflict metadata is for local resolution only.

**You can commit with unresolved conflicts** (unlike git). The merge version exists in the DAG; conflicts are side-channel metadata waiting for resolution.

### 4.5 Resolving conflicts

```
EXECUTE system/revision  operation: "resolve"
  params: {prefix: "project/", path: "config.toml", resolved: <hash_of_resolved_entity>}
```

The resolve operation:
1. Verifies the resolved entity hash exists in the content store
2. Puts the resolved entity at the original path
3. Removes the conflict entry from `system/revision/conflicts/`
4. Returns `remaining_conflicts` — the number of unresolved conflicts still under this prefix

**Resolve by deletion.** Pass `resolved: null` to delete the path entirely. Valid for create/delete conflicts where neither side should win.

Resolve does NOT create a new version. When `remaining_conflicts` reaches zero, the caller SHOULD commit to record the fully-resolved state. Under `auto_version: true`, the tree.put from resolve triggers auto-version automatically. Under `auto_version: false`, commit manually via `revision/commit`.

### 4.6 Dry-run merge

```
EXECUTE system/revision  operation: "merge"
  params: {prefix: "project/", remote_version: <hash>, dry_run: true}
```

Returns `would_merge` or `would_conflict` with conflict paths listed, without modifying any state. Use this to preview what a merge will do before committing to it.

### 4.7 Fast-forward

When the current head is an ancestor of the remote version, merge is a fast-forward — head advances to the remote version without creating a merge commit. Returned status: `fast_forward`.

When the remote version is an ancestor of the current head, there is nothing to merge. Returned status: `already_ahead`.

### 4.8 Oscillation detection

Before committing a merge version, the handler checks whether the proposed trie root has appeared in recent ancestry (configurable depth, default 4). If so, the merge is producing no forward progress — the system falls through to conflict entities instead of committing a cycling version.

Oscillation requires all four conditions simultaneously: bidirectional sync, genuine conflict, asymmetric resolution strategy, AND caller-perspective ordering. With the default `deterministic` ordering, oscillation cannot occur.

---

## 5. Cherry-pick and revert

### 5.1 Cherry-pick

```
EXECUTE system/revision  operation: "cherry-pick"
  params: {prefix: "project/", version: <hash>}
```

Applies a specific version's changes onto the current head. Internally, this is a three-way merge where:

| Merge role | Cherry-pick assignment |
|------------|----------------------|
| ancestor | Cherry-picked version's parent's trie root |
| local | Current trie root |
| remote | Cherry-picked version's trie root |

The result is "what the version changed relative to its parent" applied on top of your current state. A new version is created with a single parent (current head) — the cherry-picked version is NOT added as a parent.

**Merge versions.** When cherry-picking a merge version (2+ parents), you MUST specify which parent to use as the ancestor via the `parent` parameter:

```
EXECUTE system/revision  operation: "cherry-pick"
  params: {prefix: "project/", version: <merge_hash>, parent: <parent_hash>}
```

The handler rejects merge versions without an explicit `parent` — unlike git's `-m 1` default, the entity system does not assign semantic ordering to hash-sorted parents.

### 5.2 Revert

```
EXECUTE system/revision  operation: "revert"
  params: {prefix: "project/", version: <hash>}
```

Undoes a specific version's changes. The inverse of cherry-pick:

| Merge role | Revert assignment |
|------------|-------------------|
| ancestor | Version's trie root (what we are undoing) |
| local | Current trie root |
| remote | Version's parent's trie root (what it was before) |

A new version is created — history is never rewritten. The original version stays in the DAG.

**Merge versions.** When reverting a merge version (2+ parents), you MUST specify which parent to use as the ancestor via the `parent` parameter — same requirement as cherry-pick:

```
EXECUTE system/revision  operation: "revert"
  params: {prefix: "project/", version: <merge_hash>, parent: <parent_hash>}
```

**Revert vs rollback.** Revert (revision extension) undoes a version's changes across all paths it touched, creating a new version in the DAG. Rollback (history extension) restores a single path to a previous state. Revert is version-level undo; rollback is path-level undo.

---

## 6. Tags

Immutable named pointers to versions. Once created, a tag cannot be moved to a different version (unlike branches, which advance on commit). Tags can be deleted.

```
EXECUTE system/revision  operation: "tag"
  params: {prefix: "project/", action: "create", name: "v1.0"}
```

Tags are stored at `system/revision/tags/{prefix}{tag_name}`.

Use for release markers, milestones, known-good states.

```
; List all tags
EXECUTE system/revision  operation: "tag"
  params: {prefix: "project/", action: "list"}

; Delete a tag
EXECUTE system/revision  operation: "tag"
  params: {prefix: "project/", action: "delete", name: "v1.0"}
```

---

## 7. What a version entry actually is

The version entry is minimal:

```
system/revision/entry {
  root:    <hash of trie root node>
  parents: [<hash of parent version>, ...]   ; sorted by hash value
}
```

Two fields. The trie root is a Merkle tree of the prefix's bindings at commit time. Parents are sorted lexicographically so that any peer computing the same merge produces the same version hash.

**Content-addressed.** The version's hash = SHA-256("system/revision/entry" + CBOR(data)). Because parents are in the data, the hash includes ancestry — the DAG has cryptographic integrity.

**Location-independent.** The version entry contains no prefix, author, timestamp, or message. The same content at different mount points shares version entries and version history. The prefix lives externally in the head pointer path: `system/revision/head/{prefix}`.

**Structural sharing.** Two versions with identical tree state produce identical trie roots. Comparing two versions starts with one hash comparison (are the roots equal?). If not, recursive descent into the trie visits only subtrees where hashes differ. Unchanged subtrees are skipped entirely. This is why merge is efficient even over large trees.

---

## 8. History extension interaction

If the history extension is configured for paths under your revision prefix, every tree write — including those from commit, checkout, merge, cherry-pick, and revert — produces history transition entries. This gives you:

- **Per-path undo** via `history/rollback`, independent of revision
- **Audit trail** with author, capability, and operation context per transition
- **Between-commit visibility** — what happened between snapshots, not just the snapshots themselves

History and revision are complementary. History is the fine-grained journal (every write, per-path, local). Revision is the coarse-grained DAG (named snapshots, shared between peers). History tells you "what happened between commits." Revision tells you "what the meaningful checkpoints are."

The `pending` field in `revision/status` leverages history when available — it reports how many path changes have occurred since the last version.

---

## 9. Configuration

The revision config entity at `system/revision/config/prefixes/{name}`:

```yaml
type: system/revision/config
data:
  prefix: "project/"               # subtree to version
  auto_version: false              # manual commits (true = auto, see advanced guide)
  merge_order: "deterministic"     # always use this for multi-peer
  exclude:                         # path patterns to skip (glob)
    - "system/**"
    - "**/*.tmp"
    - "**/cache/**"
  exclude_types:                   # entity types to skip
    - "your-app/ephemeral"
  oscillation_depth: 4             # DAG levels to check for merge cycling (default 4)
```

### 9.1 Writing config

Config writes MUST go through the `revision/config` handler operation, not direct tree.put:

```
EXECUTE system/revision  operation: "config"
  params: {
    name: "my-project",
    action: "set",
    config: {
      type: "system/revision/config",
      data: {
        prefix: "project/",
        auto_version: true,
        merge_order: "deterministic",
        exclude: ["system/**"]
      }
    }
  }
```

The handler validates before writing:

| Rule | What it checks |
|------|---------------|
| V1 | `prefix` is a valid absolute path |
| V2 | Auto-version excludes include all required system patterns (see §9.2) |
| V3 | Auto-version excludes include `system/tree/root/**` when prefix encompasses it |
| V4 | `merge_order` is `"deterministic"` or `"caller-perspective"` |
| V5 | `oscillation_depth` >= 2 |

The config operation also coordinates tracking-config writes. When enabling auto-version, tracking-config is written FIRST (so reads during the window find a populated root). When disabling, the revision config is written FIRST (so auto-version stops before tracking is removed).

To delete a config: `action: "delete"`. For optimistic concurrency, pass `expected_hash` (CAS guard on the existing config binding).

### 9.2 Required excludes

When `auto_version: true`, certain system paths MUST be excluded to prevent reentrancy:
- `system/revision/**` — version metadata
- `system/tree/root/**` — trie root hashes (circular dependency)
- `system/tree/tracking-config/**` — auto-version tracking state
- `system/history/**` — history transitions
- `system/clock/**` — clock state

The config handler rejects invalid configs at write time.

### 9.3 Exclude patterns

`exclude` filters by path pattern (glob). `exclude_types` filters by entity type. Both apply when building the trie for a commit. Excluded paths and types are not included in version snapshots and not transferred during sync.

### 9.4 Overlapping prefixes

If two configs overlap (e.g., `project/` and `project/src/`), a write to `project/src/foo` produces two version entries — one per DAG. Use non-overlapping prefixes if you want single-DAG coverage.

---

## 10. Storage layout

All paths are peer-qualified at rest (`/{peer_id}/...`). This guide uses peer-relative notation.

```
system/revision/
  head/{prefix}                          -> hash of current version entry
  active-branch/{prefix}                 -> branch name string
  branches/{prefix}{name}                -> hash (branch tip pointer)
  tags/{prefix}{name}                    -> hash (immutable version pointer)
  config/prefixes/{name}                 -> revision config entity
  config/merge/type/{type_name}          -> merge config for type
  config/merge/path/{name}               -> merge config for path pattern
  conflicts/{prefix}{path}               -> conflict entity
  remotes/{peer_id}/{prefix}             -> hash (their last known head)
```

Version entries live in the content store (content-addressed, immutable). The tree paths above are pointers into the content store.

---

## 11. Cross-peer operations

The revision extension provides the primitives for cross-peer sync. The orchestration is a separate composition (see GUIDE-REVISION-AUTO-VERSION §4).

### 11.1 Fetch

```
EXECUTE system/revision  operation: "fetch"
  params: {prefix: "/{peerD}/", since: <last_known_hash>, depth: 50}
```

Gets version DAG metadata and root trie nodes from a remote peer. After fetch, you have the version structure but not the full data entities. Use `fetch-entities` to incrementally retrieve missing content by walking the trie.

### 11.2 Push

```
EXECUTE system/revision  operation: "push"
  params: {remote: "peer-uri", prefix: "project/"}
```

Sends local versions to a remote peer. The handler:
1. Gets the remote's current head
2. Determines the relationship (ahead/behind/diverged/in_sync)
3. If ahead: computes the trie diff and sends missing entities
4. If behind or diverged: returns status telling you to pull first

Push is a one-shot operation — for standing sync, use subscription + continuation.

### 11.3 Pull

Fetch + incremental fetch-entities (trie walk) + merge in one operation. A convenience over doing the three steps manually.

### 11.4 Remote head tracking

After push or fetch, the handler updates `system/revision/remotes/{peer_id}/{prefix}` with the remote's latest known head. This is bookkeeping — it lets `status` report what you know about each remote.

---

## 12. Comparison to git

| Concept | Git | Entity Revision |
|---------|-----|----------------|
| Staging area | `git add` | None — commit captures the full prefix (minus excludes) |
| Commit | `git commit` | `revision/commit` |
| Log | `git log` | `revision/log` |
| Status | `git status` | `revision/status` |
| Diff | `git diff` | `revision/diff` (between versions) |
| Branch | `git branch` | `revision/branch` |
| Checkout | `git checkout` / `git switch` | `revision/checkout` |
| Merge | `git merge` | `revision/merge` (three-way, per-prefix) |
| Rebase | `git rebase` | Not possible (content-addressed DAG, and that is fine) |
| Cherry-pick | `git cherry-pick` | `revision/cherry-pick` |
| Revert | `git revert` | `revision/revert` |
| Tag | `git tag` | `revision/tag` (immutable pointers) |
| Stash | `git stash` | Just create a branch (single pointer write) |
| Remote | `git remote` | Peer identity in URI |
| Fetch | `git fetch` | `revision/fetch` (per-prefix, not all-refs) |
| Push | `git push` | `revision/push` |
| `.gitignore` | Text file | `exclude` / `exclude_types` in config |
| History rewrite | `filter-branch`, `rebase -i` | Not possible (content addressing prevents it) |
| GC | `git gc` | Not yet specified |

**Key differences:**

1. **No staging area.** Commit is all-or-nothing for the prefix. Exclude patterns are the filtering mechanism.
2. **No rebase.** Content-addressed version entries cannot be rewritten without changing every descendant's hash. Merge is the only integration strategy. The "cascading rebase conflict" problem does not exist.
3. **Per-prefix scoping.** Git fetch pulls all refs. Entity revision is per-prefix — you only sync what you care about.
4. **Conflicts as side-channel.** Git blocks the working tree on conflict. Entity revision stores conflicts separately — documents stay visible, and you can commit with unresolved conflicts.
5. **Structural metadata only.** Version entries are `{root, parents}` — no author, timestamp, or message in the version hash. Metadata lives in the tree as regular entities, managed by application convention.

---

## 13. Where to go next

- **Auto-versioning, cross-peer sync, burst handling** -> GUIDE-REVISION-AUTO-VERSION
- **Spec details** -> EXTENSION-REVISION.md
- **Merge strategy framework** -> EXTENSION-REVISION.md §5
- **Capability setup for cross-peer operations** -> EXTENSION-ROLE
- **Per-path undo and audit** -> EXTENSION-HISTORY
