# PROPOSAL — the published-root front door: one tree path, and discovery is not convention

**Status:** **DRAFT — folded at authoring.** `EXTENSION-TREE` **4.2 → 4.3** and `EXTENSION-NETWORK`
**1.7 → 1.8**; §6 is the verified delta.
**Target:** `EXTENSION-TREE.md` §3.3a (the head-pointer path) · `EXTENSION-NETWORK.md` §6.5.3
(`signed_pointer` vs `manifest_url_prefix`).
**Tier:** `extensions/` — the seats are `entity-core-{go,rust,py}` (path) and the app tier (fetch).
**Scope:** **where** the published root is bound in the tree, and **how** a consumer finds it at an
origin. **Not** the entity's shape, its signature carriage, `prefix`, `seq`, or the walk.
**Source:** `entity-workbench-go` `b6639e9`, `reviews/CROSSIMPL-PUBLISH-CONSUME-2026-08-18.md` — a Go
publisher emission read against `entity-browser-rust`'s consumer source, with a seeded reproducible
fixture (`publish/cmd/crossimpl-fixture`).
**Cohort review:** the *emission* is measured and the *reader* is source-read. The end-to-end run has
**not** happened — workbench-go says so explicitly and it is the right call to state it.

---

## 1. What the cross-check actually produced, because the headline is the good news

Three of four surfaces align between a Go publisher and a Rust consumer **with no shared code on the
path**: content sharding (`{hex[0:2]}/{hex[2:4]}/{hex}`), bare-hashable (2-key) blob bodies, and the
two-hop signature keyed on the **published-root entity hash**. Two of those three are recorded in
browser-rust's own docs as *"divergences from upstream."* **They are not divergences from
workbench-go** — both seats landed the same shapes independently.

**That is the result ADR-0012 says a same-language pair cannot produce.** Every signed-root result
either seat held before this was self-verified: workbench-go's by their own reader, browser-rust's by
their own projector's output. A publisher and consumer written by one author agreeing is
**cohort-consistent, not independent convergence**. This is the first evidence on this surface that is
neither.

**And it stopped at hop 0**, before reaching any of it. That is the finding.

## 2. Defect A — the head-pointer tree path was never pinned, and three impls picked

Searched: **no path form for the published-root head pointer appears anywhere in `specs/` or
`guides/`.** §3.3a defines the entity, its `prefix` field and its signature carriage; it never says
where the pointer is bound.

| Seat | symbol · path | Form |
|---|---|---|
| `entity-core-rust` `a701e13` | `published_root_head_path` · `core/peer/src/published_root.rs` | `/{peer}/system/peer/published-root` |
| `entity-core-py` `9ba439b` | `http_server.py` / `peer.py` | `/{peer}/system/peer/published-root` |
| `entity-core-go` `5b86b2b` | `PublishedRootStoragePath` · `core/types/published_root.go` | `system/peer/published-root/{base58_peer_id}` |

**Ruled: no peer-id segment.** The decisive argument is one of this corpus's five foreground
invariants — **absolute paths at every layer; do not double-qualify** (`ENTITY-CORE-PROTOCOL` §1.4).
The peer namespace is already the first segment of every absolute path, so
`/{peer}/system/peer/published-root/{peer}` names the peer twice. core-go's stated rationale — *"so a
consumer can locate the current published-root for peer X without enumerating the type's
content-addressed siblings"* — describes a real need that **the peer prefix already satisfies**. Two
of three impls, the `signed_pointer` string in `EXTENSION-NETWORK` §6.5.3, and §3.3a's own prose all
carry the un-suffixed form.

**The part that should not pass without comment.** core-go's source states that this path *"supersedes
the legacy `signed_pointer: "system/peer/published-root"` string in NETWORK §6.5.3,"* citing *"§4
cross-ref Q5."* **No such supersession exists in this corpus** — NETWORK's only Q5 is Amendment 8's
priority-selection ordering, unrelated. **A comment asserting that an implementation supersedes a spec
is a spec change with no proposal, recorded where no reviewer of the spec will ever read it.** It then
propagated: workbench-go builds on core-go, so the extra segment reached a published site.

**Why an unpinned path is not a free choice.** The extra segment makes the pointer's path a
**directory** on a filesystem-backed origin (the parent of `{peer}.bin`), so a consumer reading the
pointer as a file fails with `EISDIR` at hop 0 — **before** any of §1's three aligned surfaces is
touched. An unpinned path is a divergence with a delay, and this one hid behind agreement.

## 3. Defect B — `signed_pointer` is a tree path and was read as a fetch location

`DirFetcher::manifest()` joins the origin base with `PUBLISHED_ROOT_REL` (`"system/peer/published-root"`)
and reads it. **That treats a *tree* path as a *transport* path.**

The spec already separates them and workbench-go's reading is the correct one:

| Field | Answers | §6.5.3.1 |
|---|---|---|
| `manifest_url_prefix` | **where to GET it** — terminal, no suffix, no trailing slash | *"This slot is reserved for the `system/peer/published-root` entity and nothing else (MUST)"* |
| `signed_pointer` | **what the origin is asserting** — the §3.3a entity path, so a consumer knows a signed root exists and what to verify | advertised in the profile |

**Ruled: the manifest's location is DISCOVERED from `manifest_url_prefix`, never derived by convention
from the tree path.** `{origin}/manifest` and `{origin}/{peer}/system/peer/published-root` are equally
conformant; only the advertised one is findable. **A consumer MUST NOT join `signed_pointer` onto an
origin.** workbench-go advertises `manifest_url_prefix = {origin}/manifest` in its emitted
transport-profile and is conformant as it stands.

**The tree path may not be servable at all** — it names a binding in a trie, not a byte range at an
origin. A profile field that exists to be read is not a default to be assumed.

## 4. Why the publisher was right not to move

workbench-go declined to relocate their layout to match one consumer's reader: *"a publisher that
moves its front door to match one consumer's fixture reader is how the next divergence gets built."*
**That is correct and it is the reason this reached a ruling instead of a silent fix.** Had they
moved, both defects above would still exist — unpinned path, conventional-vs-discovered unresolved —
and the next pair of implementations would rediscover them.

## 5. Rejected alternative

**Serve the published root at the tree path as well, so both readers work.** It removes the symptom
and keeps both defects: the path stays unpinned (core-go's extra segment still diverges) and
"conventional" stays alive as an undocumented second discovery mechanism that every static origin must
now support forever. **Two front doors is not compatibility; it is the divergence, ratified.**

## 6. Spec delta — verified against the tree before the versions moved (L3)

| # | File | § | Change | Verified |
|---|---|---|---|---|
| D1 | `EXTENSION-TREE.md` | §3.3a | `[MUST, v4.3]` — head pointer at `{peer}/system/peer/published-root`, **no peer-id segment**, with the double-qualification argument | ✅ present |
| D2 | `EXTENSION-TREE.md` | §3.3a | The three-way measurement, the unratified-supersession note, and *an unpinned path is a divergence with a delay* | ✅ present |
| D3 | `EXTENSION-NETWORK.md` | §6.5.3 | `signed_pointer` inline comment re-labelled **tree path, not a URL and not a suffix** | ✅ present |
| D4 | `EXTENSION-NETWORK.md` | §6.5.3 | `[MUST, v1.8]` — discovered-not-conventional; a consumer MUST NOT join `signed_pointer` onto an origin; the hop-0 rationale | ✅ present |
| D5 | headers | — | **TREE 4.2 → 4.3**, **NETWORK 1.7 → 1.8** | ✅ present |

**No wire change, no new entity type, no new field, no new error code.**

## 7. Cohort impact — routed in `docs/status/ROUTING-2026-08-18-p-*` (L13)

| Seat | Owed |
|---|---|
| `entity-core-go` `5b86b2b` | **`PublishedRootStoragePath` drops the peer-id segment**, and the "supersedes NETWORK §6.5.3" comment goes with it. This is the change that unblocks the cross-impl run |
| `entity-workbench-go` `b6639e9` | **Nothing to change by choice** — your emission follows core-go's helper and moves with it. Your `manifest_url_prefix` advertisement is conformant and stays. Re-cut the fixture after core-go lands |
| `entity-browser-rust` | `DirFetcher::manifest()` reads `manifest_url_prefix` from the transport profile instead of joining `PUBLISHED_ROOT_REL`. **Then run workbench-go's fixture** — §1's three surfaces are waiting behind hop 0 |
| `entity-core-{rust,py}` | **Conformant, no change.** Named so nobody "fixes" a correct path toward go's |

**The end-to-end run is the deliverable, not the spec edit.** Both defects are cheap; the value is the
first genuinely independent publish/consume result on this surface, and it is one `EISDIR` away.
