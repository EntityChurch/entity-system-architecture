# Revision Extension: Auto-Version User Guide (Exploratory)

**Status**: Draft

**Audience:** Application developers integrating the revision extension. Not a spec. The spec (EXTENSION-REVISION.md + SYSTEM-COMPOSITION.md + EXTENSION-TREE.md) is normative; this guide is descriptive.

**Scope:** Auto-versioning plus the subscription + continuation composition for cross-peer sync (§4). Manual commits, branches/tags, cherry-pick/revert are covered in EXTENSION-REVISION itself. Capability/authorization is out of scope — see EXTENSION-ROLE and the capability model for how peers authorize each other to subscribe, fetch, and push.

---

## 1. The one-paragraph mental model

The revision extension turns a tree subtree into a content-addressed version DAG — a git-like history of structural states. `auto_version: true` makes that history update automatically on every write. Multiple peers editing the same subtree converge to a shared DAG under sync. You get collaborative editing (think Figma, Google Docs) at the tree layer, not per-document.

If you just want git-style history where you commit manually, leave `auto_version: false` and use `revision/commit`. If you want live collaborative editing with auto-convergence, turn `auto_version: true` and accept that every write produces a DAG entry.

---

## 2. The starter config (copy this)

```yaml
# system/revision/config/prefixes/your-app
type: system/revision/config
data:
  prefix: "your-app/"                  # whatever subtree your app owns
  auto_version: true
  merge_order: "deterministic"         # DO NOT pick caller-perspective (§3)
  exclude:
    - "system/**"                      # covers all engine-owned paths
    - "**/*.tmp"
    - "**/cache/**"
  exclude_types:
    - "your-app/ephemeral/**"          # transient entities you don't want in history
```

This config is safe to deploy on any number of peers. They will converge.

**Do NOT deploy this config:**

```yaml
data:
  prefix: "/"
  auto_version: true
  # merge_order omitted (was: caller-perspective default — DON'T)
  # exclude omitted → cascade explosion
```

This is the footgun configuration. The spec rejects it at config-write time (proposal §6D.4), but you should also recognize it on sight.

**Note on cross-peer sync.** The revision extension does not manage synchronization between peers. There is no `auto_sync` flag, no `remotes` list on the config. If you want Peer B to follow Peer A's versions, B sets up a subscription on A's `system/revision/head/{prefix}` with a continuation that calls `revision:fetch-diff` + `tree:merge`. See §4 for the pattern.

---

## 3. Why deterministic merge ordering (and never caller-perspective)

The `merge_order` knob has two values. Under the new default, you get `"deterministic"`. You can still pick `"caller-perspective"` but you almost certainly shouldn't.

**Caller-perspective** means "when A merges from B, A's branch is labeled 'local' and B's is 'remote'." It produces git-like history where your perspective is preserved. In a single-authority deployment (one server, many clients that push to it), this is fine — the server is always the caller.

**In peer-to-peer,** each peer is its own caller, so the same merge computed on A and B labels the branches differently, and non-commutative merge strategies produce different results. The DAG diverges silently — the hashes differ, sync sees them as different versions, and the system tries to merge the merges. In mesh topologies this oscillates.

**Deterministic** hashes the branch contents to decide which is "local" and which is "remote" regardless of who's computing the merge. All peers get the same answer. The tradeoff: your history doesn't read "A merged B"; it reads "whichever-hash-sorts-lower merged whichever-hash-sorts-higher." Cosmetic, not semantic.

**Rule of thumb:** `auto_version: true` → `merge_order: "deterministic"`. Always. If you want caller-perspective, turn auto-version off and do manual commits.

---

## 4. Following another peer's versions

The revision extension doesn't manage cross-peer sync. It provides the primitives; **subscription + continuation** orchestrate when they fire. If you want Peer B to follow Peer A's `/project/` versions, the model is: B subscribes to A's revision head, and on each head advance B pulls the **content delta** from A and applies it locally.

The content-transfer primitive is **`revision:fetch-diff`** (EXTENSION-REVISION §4.4.19): given a `base` version that B already has, A returns the closure of everything that changed between `base` and A's current head — O(diff) bandwidth, one round trip. The follower applies it with `tree:merge`.

**Two distinct things you might mean by "follow" — keep them separate:**

- **Content mirror** (this section): B wants A's *files/content* mirrored locally. `revision:fetch-diff → tree:merge`. B's local tree under `/{B}/project/...` ends up matching A's content. B does **not** necessarily grow a version DAG. This is the common case.
- **DAG mirror**: B also wants A's *version history* (the commit DAG) integrated into B's own revision DAG. That is `revision:merge` (or the `RevisionConverge` composition), a separate, heavier concern. Don't reach for it unless B genuinely needs to branch/merge/log A's history locally.

This section is about the content mirror. The two are independent on purpose — entangling them is what made an earlier `tree:extract`-based design go wrong.

### 4.1 The one correctness rule: `base` is the *follower's* current head

`revision:fetch-diff(prefix, base)` computes the diff from **`base`** to A's current head and bundles only that closure. For B to converge correctly, **`base` MUST be the version whose content B actually has right now** — i.e., B's current local head for the prefix. The diff fills the gap between what B has and what A has; if `base` is *ahead* of what B actually has, the diff is **incomplete** and the merge leaves B in a corrupt mixed state (some paths at A's latest, others stale at B's real position). This is a silent data-correctness bug, not a performance one — get `base` right.

This rule drives everything below. The two ways to supply `base`:

1. **Notification `previous_hash` — the reliable-delivery shortcut.** A head-advance notification carries `previous_hash` (A's head *before* this change). **When B has missed no notifications, A's prior head equals B's current head**, so `base = $notification.previous_hash` is correct — and it is a single dynamic field, so the whole thing is expressible as a pure continuation chain (no custom handler). This is the recipe the cross-impl POC validated.
2. **B's own local head — the robust form.** When notifications can be dropped, reordered, or rate-limited (the general case across an unreliable link), `previous_hash` may be *ahead* of B's real position, so it is **not** a safe `base`. The robust `base` is B's actual local revision head, read at fire time. The continuation transform vocab cannot read B's local mutable state mid-chain, so the robust form uses a **thin custom handler** on B's inbox that reads B's head and dispatches `fetch-diff(base = that)`. (This is the same "read receiver-local state" gap that deferred `revision:diff-since-local-head`; for a standing follow it is the legitimate handler off-ramp.)

Pick form 1 when delivery is reliable and ordered (LAN, single writer, B keeps up). Pick form 2 when drops are possible and silent corruption is unacceptable.

### 4.2 The canonical chain (reliable delivery)

The two-step standing chain, form 1:

```
; B subscribes to A's revision head
system/subscription @ Peer A:
  pattern:    "system/revision/head/project"
  deliver_to: {uri: "system/inbox/follow-a"}        ; on B

; Step 1 — continuation on B's inbox: fetch the content delta from A.
;          base = previous_hash (single dynamic field → inject mode); prefix static.
system/continuation @ system/inbox/follow-a:
  target:           "system/revision"               ; dispatched cross-peer to A
  operation:        "fetch-diff"
  params:           {prefix: "project/"}            ; static scaffold
  result_transform: {extract: "previous_hash"}      ; navigate to the notification's prior head
  result_field:     "base"                          ; inject it as fetch-diff's `base`
  deliver_to:       {uri: "system/inbox/follow-a-merge"}   ; chain to step 2

; Step 2 — continuation on B's merge inbox: apply the returned envelope locally.
;          fetch-diff returns a system/envelope; inject it as tree:merge's source_envelope.
system/continuation @ system/inbox/follow-a-merge:
  target:       "system/tree"
  operation:    "merge"
  params:       {}                                  ; no static fields needed
  result_field: "source_envelope"                   ; inject the fetch-diff envelope (pass-through-ish inject)
```

One cross-peer round trip per advance (step 1), O(diff) bytes; step 2 is local. `tree:merge` writes the mirrored content under B's own namespace (`/{B}/project/...`) — this is content mirroring, not DAG advancement. (Note: step 1 uses **inject** mode — a single dynamic field. An op needing *two* dynamic fields, e.g. an explicit `(base, target)`, would use **merge** mode, `result_merge: true` — EXTENSION-CONTINUATION §2.1.)

### 4.3 Initial sync / bootstrap (first subscribe, no prior state)

Subscription fires on *changes*, not current state, and a fresh follower has no `base`. Bootstrap with a full closure: call `revision:fetch-diff(prefix, base = <zero hash>)` — `base = zero` means "diff against the empty trie," i.e. the entire current closure (EXTENSION-REVISION §4.4.19). Then install the standing chain.

```
1. B writes the subscription entity (no notifications yet).
2. B calls revision:fetch-diff(prefix, base=ZERO) on A → full closure → tree:merge. B now mirrors A's current state.
3. The standing chain (§4.2) keeps B current from here.
```

Skip step 2 and B mirrors nothing until A's next commit. Easy to forget; do it in setup.

### 4.4 Bursts, drops, and reconnect

**Bursts** (A commits V2→V3→V4→V5 faster than B processes): content addressing debounces naturally. If B is on form 1 and processes notifications in order, each fetch-diff is the small per-commit delta. If several notifications queue, processing them in order still converges; an already-applied delta re-applies as a content-addressed no-op. No timers needed.

**Drops / reordering** are where the §4.1 rule bites. If B misses the V3 and V4 notifications and only sees V5's (`previous_hash = V4`), form 1 would diff `V4 → V5` and **miss the V2→V4 deltas B never got** — corrupt merge. Form 2 (base = B's actual head = V2) diffs `V2 → V5`, transferring the full gap: bigger envelope, **correct** convergence. So: **if your transport can drop or reorder notifications, use form 2.** Bandwidth degrades gracefully toward a full closure as the gap grows; correctness holds.

**Reconnect after offline** (EXTENSION-NETWORK replays buffered notifications): same distinction. Form 2 catches up correctly on the first post-reconnect fire (one `fetch-diff` from B's real head to A's current). Form 1 is only safe if every buffered notification is delivered and processed in order — which post-outage replay does not guarantee. Another reason form 2 is the safer default across real networks.

**GC interaction:** if A has garbage-collected the version B passes as `base`, `fetch-diff` returns `404 base_not_found`. B's recovery is to re-bootstrap (§4.3, `base=ZERO`) — a full closure — or pass a more recent `base` it shares with A. Handle the 404 in the follow handler; don't let it silently stall the chain.

### 4.5 What NOT to build

- Don't push versions on commit, don't maintain a "remotes" list, don't invent an auto-sync flag — subscription already does the "who's watching" fan-out with better guarantees.
- Don't use `base = $notification.previous_hash` on an unreliable link (§4.1 / §4.4) — it silently corrupts under drops.
- Don't reach for `revision:merge` / DAG mirroring when you only need content (§4). It's heavier and couples two concerns the system deliberately separates.
- Don't build a follow chain into a reactive cycle (B follows A while A follows B *on the same prefix* with no convergence guard). Content addressing makes re-merge a no-op, so simple bidirectional content mirroring is fine; but see EXPLORATION-PEER-COMPOSITIONS on no-reactive-cycle topology discipline before wiring anything more intricate.

### 4.6 Capability note

Cross-peer follow requires A to have granted B: the right to subscribe to `system/revision/head/{prefix}`, and the right to call `revision:fetch-diff` on the prefix (revision-read scope). The `tree:merge` step is local to B and uses B's own authority. None of this is follow-specific — it's the standard capability model (EXTENSION-ROLE). Implementations that skip the grants pass single-user testing and fail the first time a real unauthorized peer is involved.

---

> **Historical note.** Earlier drafts of this section described a `revision/fetch` + `revision/merge` recipe. That recipe transferred DAG metadata (depth-1 trie nodes), not content, and silently left content unmaterialized for trees deeper than one level — it conflated content mirroring with DAG fetching. A brief `tree:extract.since` design (EXTENSION-TREE v3.14) then placed the content-diff in the tree layer and was withdrawn (v3.15) for the layering violation. The current recipe — `revision:fetch-diff → tree:merge`, with `base` = the follower's head — is the corrected, layer-clean form. See `proposals/implemented/PROPOSAL-TREE-EXTRACT-SINCE.md` (Amendment 1) for the full history.

---

## 5. What gets versioned vs what doesn't

The version entry captures the tracked subtree's full structural state as a Merkle trie root. When you do `revision/log`, you're walking DAG entries whose `root` fields point at snapshots of your subtree.

**What's in the snapshot:** every binding under `prefix` not matched by `exclude` or `exclude_types`.

**What's NOT in the snapshot:**
- Content-store entities that aren't bound in the tree. If you write an entity to the content store and never reference it from a tracked tree path, it's not versioned.
- The revision extension's own metadata (`system/revision/**`). Always excluded.
- Engine state paths (`system/clock/**`, `system/history/**`, etc.). Required excludes.
- Anything under `exclude` or `exclude_types` per your config.

**Common mistake:** setting `exclude_types: ["system/revision/entry"]` thinking it'll exclude your version entries from versioning. It won't — version entries live in the content store, not at tracked paths. `exclude_types` only filters entities bound *at tracked paths*.

---

## 6. What a merge looks like under auto-version

Two peers have diverged:
- Peer A's head: `V_A` with root R_A
- Peer B's head: `V_B` with root R_B
- Common ancestor: `V_ancestor`

When A merges B (via `revision/merge` or automatic sync resolution):

1. Merge op computes the merged binding set `R_new` using the configured merge strategies.
2. Merge op creates `V_merge` with `parents: [V_A, V_B]` and `root: R_new`. Advances head to `V_merge`.
3. Merge op writes the N binding changes to the tree. Each write triggers auto-version.
4. Each trigger creates a single-parent intermediate: `V_1 → V_2 → ... → V_N`.
5. After write N, head is at `V_N`, `V_N.root == V_merge.root == R_new`.

**Net result:** N+1 commits (V_merge + N intermediates). This is expected. The intermediates capture real partial-merge states that the tree passed through — your tree went from R_A through R_1, R_2, ..., R_N = R_new, not instantly from R_A to R_new. The CRDT contract preserves those transitions.

**Displaying this in a UI:** walk the parent chain from V_N backward. When you hit a multi-parent ancestor (V_merge), collapse the linear descendants between them into "1 merge commit (expand to see intermediates)." The spec note in proposal §4 "Multi-parent versions" describes the exact structural filter.

---

## 7. When auto-version fires (and when it doesn't)

**Fires:** every tree write to a path matching the tracked prefix and not matching exclude patterns. Including writes from merge, cherry-pick, revert, and checkout operations. Including writes from other peers arriving via sync.

**Does NOT fire:**
- On pure content-store writes that don't bind to a tree path.
- On writes that produce no change to the tracked root (deduplication — e.g., writing the same hash to the same path).
- On writes to paths in the exclude list.
- Before structural summaries have settled (fires at emit position 7, after position 6).

**Fires once per matching config.** If two of your configs overlap (e.g., `/project/` and `/project/src/` both tracked), a write to `/project/src/foo` produces two version entries — one per DAG. This is intentional; use non-overlapping prefixes if you want single-DAG coverage.

---

## 8. Common patterns

Each pattern below shows the revision config. Cross-peer sync (where applicable) is set up separately via subscription + continuation — see §4.

### 8.1 Collaborative document editing (Google Docs style)

```yaml
prefix: "docs/{doc_id}/"
auto_version: true
merge_order: "deterministic"
exclude: ["system/**", "**/cursor/**"]   # exclude cursor positions, etc.
```

Every keystroke (assuming your editor debounces to tree writes) produces a version. Each collaborator subscribes to the others' `system/revision/head/docs/{doc_id}` paths; a continuation on the subscription inbox dispatches **`revision:pull`** (`fetch` + incremental `fetch-entities` content walk + `merge` in one op — EXTENSION-REVISION §4.4.8) — this is the **DAG-mirror** case, since collaborators want each other's version history integrated, not just content. Collaborators converge. DAG is fine-grained; your UI collapses adjacent intermediates. (If a collaborator only needs the latest *content* and not the others' DAGs, the lighter content-mirror recipe — `revision:fetch-diff → tree:merge`, §4 — applies instead.)

> **Implementation status.** `revision:pull` is a **convenience** op (§4.2): spec'd at §4.4.8 but not guaranteed implemented on every peer — it MAY be exposed through an alternative interface or not yet built. core-go currently uses a custom `RevisionConvergeHandler` in its place. Before relying on `revision:pull` as a chain step, confirm your target peer implements it; otherwise use the handler-based DAG-convergence path. This is an implementation gap (build the spec'd op), **not** a missing-spec gap — no ratification is needed.

### 8.2 Project workspace with CI

```yaml
prefix: "projects/{name}/"
auto_version: true
merge_order: "deterministic"
exclude: ["system/**", "build/**", "**/*.o", "**/*.tmp"]
```

Local workspace is continuously versioned. CI or the user does `revision/push` at known-good points. Local DAG stays detailed; remote DAG gets coarse tags via manual commits or selective pushes. No standing subscription — sync is one-shot, caller-invoked.

### 8.3 Append-only audit log

```yaml
prefix: "audit/"
auto_version: true
merge_order: "deterministic"
exclude: ["system/**"]
```

Writes are expected to be append-only (new paths, no mutations). Auto-version produces one entry per log write. For remote peers to stream the log, each remote sets up a subscription on `system/revision/head/audit` with a fetch continuation. Merges are rare (would indicate concurrent writes to the same path — check your usage).

### 8.4 Milestone-only (no auto-version)

```yaml
prefix: "releases/"
auto_version: false                 # manual only
merge_order: "deterministic"        # still applies to manual merges
```

Classic git model. Use `revision/commit` at milestones, `revision/tag` for release markers, `revision/push` to publish. No auto-anything.

---

## 9. Debugging divergence

If two peers report different heads for the same prefix and aren't converging under sync:

1. **Check `merge_order`.** If either peer is on `"caller-perspective"`, stop here — that's the cause. Switch both to `"deterministic"` and re-sync. The previously-divergent DAG entries remain in the content store but are no longer head-reachable after the re-sync.

2. **Check `exclude` lists.** Two peers with different excludes compute different tracked roots for the same writes. One peer's version entry has `root: R_1`, the other's has `root: R_2`, both describe the same underlying tree state but look different. Harmonize excludes across peers.

3. **Check custom merge handlers.** A non-commutative handler with caller-perspective ordering is the classic divergence cause. Either make the handler commutative or switch to deterministic.

4. **Check for overlapping prefix configs.** If you meant one DAG but got multiple, you may be looking at different DAGs on each peer.

5. **Check the DAG depth under oscillation.** `oscillation_depth: 4` is the default; under a 3+ peer mesh with caller-perspective, raise to `2N`. With deterministic this is moot.

---

## 10. Performance notes

- **DAG growth rate** = tree-write rate. Under active editing, expect version entries to accumulate at the same rate as writes.
- **Content store growth** is bounded by unique `{root, parents}` tuples. Structural sharing of trie nodes means most of the content-store growth is new trie nodes, not new version entries or duplicated data.
- **Merge cost** scales with the number of differing paths between branches. A Merkle trie diff skips matching subtrees with one hash comparison; deep trees with localized changes are fast.
- **Subscription cost** under auto-version: every version head advance fires subscribers on `system/revision/head/{prefix}`. If you don't need live notification, don't subscribe.

---

## 11. What's still in flux (read before committing to a design)

This guide is exploratory because the following are still in motion:

- **Per-version metadata sidecar.** Today, version entries are structural-only (`{root, parents}`). Timing/author/capability live in history. A future amendment may add a structured sidecar entity; if you're planning UI that shows "who committed this when," know that the data source may shift.
- **Version-scoped tree reads.** There's no `tree.get(@version)` or `tree.snapshot(@version)` operation today. To inspect an old version, use `revision/fetch` + manual trie walk. A future EXTENSION-TREE amendment may add a read-at-version convenience.
- **Merge intermediate suppression.** Today, a merge of N paths produces N+1 DAG entries. Proposals to tag intermediates for UI filtering are not yet adopted.
- **Multi-head data model.** The current spec uses a single-head pointer per prefix. Multi-head (set-of-concurrent-heads) is possible but not yet spec'd. If you hit contention on head advance, you're using the single-head model with CAS+retry or single-writer serialization.
- **Checkout under auto-version policy.** The `checkout_under_auto_version: "allow" | "warn" | "deny"` config field exists but isn't required for implementations. Don't rely on `"deny"` for safety; treat checkout as always mutating under auto-version.
- **Cross-peer follow recipe (`revision:fetch-diff`).** §4's content-mirror recipe is new (EXTENSION-REVISION v3.4 / EXTENSION-CONTINUATION v1.16) and validated by one cross-impl POC (no-drop tight-leader-follower). Two areas want field experience: (a) the **drop-tolerant form** (§4.1 form 2 — `base` = follower's own head via a custom handler), which the chain-expressible form does not cover and which is the correctness-critical path under unreliable delivery; (b) **GC/`base_not_found` re-bootstrap** behavior under long divergence. Expect refinements as application teams report back, especially on the form-1-vs-form-2 boundary.

---

## 12. FAQ

**Q: Do I need to turn on auto-version to use the revision extension?**
No. `revision/commit`, `revision/merge`, `revision/branch`, etc. all work with `auto_version: false`. Manual mode is the full git model.

**Q: Can I mix auto-version and manual commits on the same prefix?**
Yes but the result is messy. A `revision/commit` under `auto_version: true` typically produces a redundant entry (same root as the current head, already captured by the prior auto-version). The spec (§5 in the proposal) adds a dedup check that suppresses these. Don't rely on `commit` to create a meaningful "checkpoint" under auto-version — use separate tooling (a named-checkpoint entity at a different path, tags, etc.).

**Q: What happens if two peers turn on auto-version for the same prefix independently?**
Fine, as long as both use `deterministic` merge_order and compatible excludes. Each peer builds its local DAG; when they sync, the DAGs converge via the shared ancestor (or through graft cost if they don't share an ancestor — see the version extension transfer protocol).

**Q: How do I delete a version I don't want?**
You don't. Content-addressed DAG entries are immutable. You can abandon them by moving the head elsewhere (`revision/reset` or equivalent), but the entries remain in the content store until GC'd. The deletion story is an open question (memory-note: deferred to a future GC proposal).

**Q: Does auto-version work across subtrees written by different peers?**
Yes. A tracked prefix like `/` (with excludes) captures writes from all peers' namespaces (`/{peer_A}/...`, `/{peer_B}/...`). The resulting version entries describe the universal-tree state from your peer's perspective. Other peers' authoritative views may differ (their version entries describe their perspective). Cross-peer convergence on the universal-tree DAG requires all peers to track the same prefix with compatible configs.

**Q: Can I change the exclude list on a live config?**
Yes, but the next write will produce a version entry whose root differs significantly from the previous head's root (because bindings that were excluded are now included, or vice versa). The DAG stays valid; the diff looks like a large bulk change. Plan config changes at quiescent moments if you care about DAG readability.

---

## 13. Status and TODO

This guide covers auto-version plus basic cross-peer sync composition, under the v2.5 proposal. Still to write (if application teams need them):

- Manual commit / branch / merge walkthrough
- Conflict resolution UI patterns
- Custom merge handler tutorial (with commutativity examples)
- Performance tuning guide for high-write-rate deployments
- Interaction with history extension (full capture of author/capability/clock per change)
- Capability setup walkthrough for cross-peer subscribe + fetch + push
- Operator guide for oscillation debugging in caller-perspective topologies (if anyone still needs caller-perspective)

Feedback from application teams welcome — this guide will be replaced by properly-scoped operator docs once the team knows what questions come up in practice.
