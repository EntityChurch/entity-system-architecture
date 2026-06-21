# Local Files Domain — Normative Specification

**Version**: 1.3
**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.51+), EXTENSION-CONTENT.md (v3.5+), EXTENSION-TREE.md (v3.1+), EXTENSION-SUBSCRIPTION.md (v3.4+)
**Optional**: EXTENSION-REVISION.md (v2.0+) — versioning and cross-peer sync
**Encoding**: ENTITY-CBOR-ENCODING.md (ECF)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. Overview

The local files domain maps a host filesystem subtree into the entity tree, making filesystem content accessible through the entity protocol. Files become entities; directories become listings. The handler is the translation layer between two worlds: the filesystem and the entity tree.

The local files handler stores file content via the CONTENT extension's blob/chunk substrate (`system/content/blob` + `system/content/chunk` per `EXTENSION-CONTENT.md` v3.5). File entities carry a `system/hash` reference to the blob; the bytes themselves live in the content store. This unblocks files of any size, enables cross-handler chunk deduplication (the same bytes through any handler produce the same chunk entities at the substrate level), and shifts the cost of an edit from "re-encode the whole file" to "re-chunk via FastCDC and ship the ~1 new chunk."

This domain defines:

- File and content entity model on the CONTENT substrate (file metadata entity carrying a blob hash)
- Handler pattern and operations (`read`, `write`, `list`, `delete`, `watch`)
- Root mapping configuration (multiple independent filesystem roots; per-root descriptor publication)
- Reverse write via subscription: tree changes produce filesystem writes (sync → disk)
- File watcher: filesystem changes produce tree writes (disk → sync)
- Optional descriptor publication for public/hash-addressable consumption (CONTENT v3.5 §5.3)

### 1.1 Scope

This domain covers files and directories. It does **not** cover:

- Symlinks, device nodes, sockets, or FIFOs (future extension)
- Extended attributes, ACLs, or POSIX permissions beyond basic metadata (future extension)
- Content-type detection or indexing beyond the optional `media_type` field (application-level)
- **Concurrent collaborative edits to the same path** — concurrent writes from multiple peers to the same `local/files/...` path are NOT a CRDT use case for this domain. The §5 reverse-write convergence model is **substrate-level tree convergence** (last-arrival wins under the §5.5 circuit-breaker; both writes hashed identically as blobs and recorded at distinct chain positions; the filesystem reflects the last write to land). This is intentional. **For concurrent collaborative edits, use `EXTENSION-REVISION` instead** (versioned semantics, three-way merge, conflict resolution via revision metadata). The local files domain is for filesystem-backed content surfaces, not for collaborative-edit semantics. Per workbench-go WB-25 closure (Stage 4 round-3).

### 1.1a CRDT-semantics framing (v1.3 Amendment 4)

`local/files` content-mode under concurrent same-path writes is **not** a CRDT. The substrate guarantees:

1. **Tree convergence** — both peers' writes produce blob entities at distinct content hashes (because bytes differ); the tree records both as distinct chain events; subscriptions deliver both.
2. **Filesystem partial swap** — the §5 reverse-write pipeline applies the most-recently-arrived write to disk; the on-disk file reflects the last arrival.
3. **No automatic merge** — bytes are not three-way-merged at the substrate level. Operators wanting merge semantics layer it via the revision extension.

This is the right default for filesystem-backed content (matches the rsync / git-checkout mental model — most-recent-write wins at the FS surface). Applications needing collaborative-edit semantics MUST layer `EXTENSION-REVISION` on top; `local/files` does not silently provide it. The substrate-level "tree convergence + FS partial swap" guarantee is what workbench-go's WB-25 case empirically validated (Stage 4 round-3).

### 1.2 Design Principles

**Substrate-backed content.** File bytes live in the CONTENT v3.5 substrate as a blob + chunks. The file entity carries only metadata + a hash reference. Identical bytes through any handler (local/files, media, dataset, …) produce identical chunks and identical blob entities — cross-handler deduplication is structural, not a separate mechanism.

**Filesystem as external state.** The entity tree is the source of truth for protocol operations. The filesystem is external state that the handler translates. Reads project filesystem state into entities. Writes project entity state into the filesystem. The two can diverge — reconciliation is explicit, not automatic (except when the file watcher is running).

**Handler owns both directions.** The handler translates in both directions: filesystem → tree (via `read` and file watcher) and tree → filesystem (via reverse write subscription). All filesystem I/O goes through the handler. The tree handler (`system/tree`) does not touch the filesystem — it only manages entities. When sync writes a file entity into the tree via `tree put`, the handler's subscription detects the change and writes the file to disk.

**Root isolation.** Each root mapping is independent. A peer can expose `/home/alice/shared/` at `local/files/shared/` and `/var/data/` at `local/files/data/` with different capabilities. Root mappings cannot overlap in the entity tree (the handler rejects overlapping prefixes at configuration time).

---

## 2. Type Definitions

### 2.1 File Entity

```
local/files/file := {
  fields: {
    path:        {type_ref: "primitive/string"}
                 ; Relative path within the root mapping (e.g., "readme.md", "src/main.rs")
    size:        {type_ref: "primitive/uint"}
                 ; File size in bytes
    modified_at: {type_ref: "primitive/uint", optional: true}
                 ; Filesystem mtime, ms since epoch
    content:     {type_ref: "system/hash"}
                 ; Hash of the system/content/blob carrying the file's bytes.
                 ; The blob's chunk list is the manifest; chunks live in the
                 ; content store under their own hashes (CONTENT v3.5 §2.1–§2.2).
    media_type:  {type_ref: "primitive/string", optional: true}
                 ; IANA media type — handler-aware fast path per
                 ; CONTENT v3.5 A2 §6.5. Optional; absent when the handler
                 ; cannot derive one from the path extension or other source.
                 ; Drift between this field and any published descriptor's
                 ; media_type is consumer policy (CONTENT v3.5 §6.5).
    written:     {type_ref: "primitive/bool", optional: true}
                 ; true in write confirmation responses
  }
}
```

The `path` field stores the relative path within the root mapping, not the full tree path or filesystem path. The tree path is `{prefix}{path}` and the filesystem path is `{root}{path}`.

The `content` field is a hash reference to a `system/content/blob` entity in the content store. The blob entity describes the file's chunked content (chunk list, total size, chunking algorithm); the chunks themselves are stored under their own content hashes. See CONTENT v3.5 §1.2 for the content-pattern rationale.

Two files with identical bytes share the same blob entity and the same chunks — deduplication is structural at the substrate level. This applies across handlers: a `local/files/file` and an image-handler entity referencing the same image data share chunks automatically.

### 2.2 Directory Entity

```
local/files/directory := {
  fields: {
    path:        {type_ref: "primitive/string"}
                 ; Relative directory path
    children:    {array_of: {type_ref: "local/files/directory/entry"}, optional: true}
                 ; Populated on list operations
    modified_at: {type_ref: "primitive/uint", optional: true}
                 ; Directory mtime, ms since epoch
  }
}
```

### 2.3 Directory Entry

```
local/files/directory/entry := {
  fields: {
    name:        {type_ref: "primitive/string"}
                 ; Entry name (filename or subdirectory name)
    entity_path: {type_ref: "system/tree/path"}
                 ; Full tree path to the entity
    entry_type:  {type_ref: "primitive/string"}
                 ; "file", "directory", or "symlink"
    size:        {type_ref: "primitive/uint", optional: true}
                 ; File size (absent for directories)
    modified_at: {type_ref: "primitive/uint", optional: true}
                 ; Entry mtime
  }
}
```

### 2.4 Deleted Entity

```
local/files/deleted := {
  fields: {
    path:    {type_ref: "primitive/string"}
             ; Relative path of the deleted file
    existed: {type_ref: "primitive/bool"}
             ; true if the file existed before deletion
  }
}
```

### 2.5 Root Mapping Configuration

```
local/files/root-config := {
  fields: {
    prefix:              {type_ref: "system/tree/path"}
                         ; Tree prefix (e.g., "local/files/shared/")
    filesystem_root:     {type_ref: "primitive/string"}
                         ; Filesystem path (e.g., "/home/alice/shared/")
    read_only:           {type_ref: "primitive/bool", optional: true}
                         ; If true, handler rejects write/delete operations
    exclude:             {array_of: {type_ref: "primitive/string"}, optional: true}
                         ; Glob patterns to exclude (e.g., [".git", "*.tmp"])
                         ; Applies to files AND directories: matched
                         ; directories are skipped entirely, pruning their
                         ; subtree.
    include:             {array_of: {type_ref: "primitive/string"}, optional: true}
                         ; Glob patterns to include (filename match)
                         ; Applies to files only — never refuses directory
                         ; descent (otherwise an "*.md" filter would skip
                         ; subdirs containing matching files).
                         ; Admission rule: a file is admitted iff
                         ; !matches(exclude) && (include empty || matches(include)).
                         ; Empty include = no positive filter (all
                         ; non-excluded files admitted).
    publish_descriptors: {type_ref: "primitive/bool", optional: true}
                         ; When true, on read the handler publishes a
                         ; system/content/descriptor at
                         ; /{local_peer}/system/content/descriptor/{B_hex}/{D_hex}
                         ; carrying the file's media_type (CONTENT v3.5
                         ; §2.4 + §5.3). Default false.
  }
}
```

Root mapping entities are stored at `system/config/local/files/{name}` where `{name}` is a human-readable identifier for the mapping (e.g., "shared", "data").

### 2.6 Watcher Configuration

```
local/files/watcher-config := {
  fields: {
    root_name:      {type_ref: "primitive/string"}
                    ; Name of the root mapping to watch (matches config entity name)
    status:         {type_ref: "primitive/string"}
                    ; "active", "stopped", "error"
    debounce_ms:    {type_ref: "primitive/uint", optional: true}
                    ; Debounce window for coalescing rapid changes (default: 2000)
    error_message:  {type_ref: "primitive/string", optional: true}
                    ; Present when status is "error"
  }
}
```

Watcher entities are stored at `system/config/local/files/watch/{name}`.

---

## 3. Handler

### 3.1 Handler Manifest

```
system/handler := {
  pattern:        "local/files"
  name:           "local-files"
  operations: {
    read:   {input_type: null,                        output_type: "local/files/file"}
    write:  {input_type: "local/files/write-request", output_type: "local/files/file"}
    list:   {input_type: null,                        output_type: "local/files/directory"}
    delete: {input_type: null,                        output_type: "local/files/deleted"}
    watch:  {input_type: "local/files/watch-request", output_type: "local/files/watcher-config"}
  }
  internal_scope: [
    {handlers: {include: ["system/tree"]},         resources: {include: ["local/files/*"]},               operations: {include: ["get", "put"]}},
    {handlers: {include: ["system/subscription"]}, resources: {include: ["local/files/*"]},               operations: {include: ["subscribe", "unsubscribe"]}},
    {handlers: {include: ["system/content"]},      resources: {include: ["system/content"]},              operations: {include: ["ingest", "get"]}},
    {handlers: {include: ["system/tree"]},         resources: {include: ["system/content/descriptor/*"]}, operations: {include: ["put"]}}
  ]
}
```

The handler uses domain operation names (`read`, `write`, `list`, `delete`, `watch`), not entity system names (`get`, `put`). The handler translates between the filesystem domain and the entity tree — its operations belong to the filesystem domain.

The handler entity binds at the literal prefix path `local/files` per `GUIDE-EXTENSION-DEVELOPMENT.md` §4.9; the manifest's `pattern` field is identical advertisement.

The `internal_scope` declares four grants:
- `system/tree` get/put on `local/files/*` — for binding file/directory entities into the tree.
- `system/subscription` subscribe/unsubscribe on `local/files/*` — for the reverse-write subscription (§5).
- `system/content` ingest/get on the `system/content` namespace — for persisting blob + chunks during `read`/`write`/`watcher` and for fetching missing chunks during reverse-write.
- `system/tree:put` on `system/content/descriptor/*` — for descriptor publication (§4.1 + §2.5 `publish_descriptors`). Always declared in the manifest as potential authority; actual use is gated per-root by `RootConfigData.publish_descriptors`. Deployments that never enable descriptor publication never exercise this grant; the cap is declared-but-unused with no observable behavior.

### 3.2 Write Request

```
local/files/write-request := {
  fields: {
    bytes:       {type_ref: "primitive/bytes", optional: true}
                 ; Raw payload — handler chunks via FastCDC and stores
                 ; the blob + chunks under the CONTENT v3.5 substrate.
    content:     {type_ref: "system/hash",     optional: true}
                 ; Existing blob hash — handler resolves the blob in the
                 ; content store, reassembles bytes per CONTENT §3.4, and
                 ; writes to disk. No re-transmit; dedup write path.
    media_type:  {type_ref: "primitive/string", optional: true}
                 ; Optional override; defaults to handler's media-type
                 ; guess from the path extension when absent.
    create_dirs: {type_ref: "primitive/bool",   optional: true}
                 ; Create parent directories if they don't exist (default: false)
  }
}
```

**Presence rule.** Exactly one of `bytes` / `content` MUST be present. Both-set returns error 400 `invalid_params` ("ambiguous_input"); neither-set returns error 400 `invalid_params` ("missing_input"). The shape parallels CONTENT v3.5 §6.3's envelope-or-entity input-mode discipline.

**Bytes-mode size constraint.** Bytes-mode carries the full payload in `WriteRequestData.bytes` inside a single EXECUTE envelope. Per `ENTITY-CORE-PROTOCOL.md` §1.1.4, the transport layer applies a frame limit (default ~16 MiB on TCP, negotiable). Bytes-mode write is therefore bounded by the negotiated frame max — a payload exceeding it returns a transport-level error (typically `frame too large`). This is **not** a v1.3 capability ceiling; it is the manifestation of the wire-layer constraint at the local/files surface.

**Larger-file pattern (content-mode).** For files exceeding the transport frame max, use content-mode: (1) caller chunks bytes via the shared CONTENT chunker (or a CONTENT-aware client); (2) caller calls `system/content:ingest` with the envelope of chunks + blob per CONTENT v3.5 §6.3 (envelope mode handles chunked ingestion); (3) caller calls `local/files:write` in content-mode with `{content: blob_hash}` — no bytes traverse the wire, only the hash reference. The content store handles the chunked persistence; the file entity references the blob; cross-handler dedup applies as usual. This is the principled path for arbitrary file sizes; bytes-mode is the ergonomic affordance for "small enough to fit in one envelope."

**SDK / client-library affordance (informative).** The two-mode shape is honest at the protocol layer — bytes-mode and content-mode each express a different transport reality. It is not the right shape at the application layer, where the user-mode call is uniformly *"write these bytes to this file."* A client library wrapping `local/files` SHOULD provide a single `write_file(path, bytes, …)` affordance that routes internally: payloads fitting comfortably under the negotiated frame max go through bytes-mode in one call; payloads above route through the SDK's local FastCDC chunker → `system/content:ingest` (envelope mode) → `local/files:write` in content-mode. The caller never sees the routing. This is the GUIDE-EXTENSION-DEVELOPMENT.md §2 layering: *"An extension is NOT an application. Applications consume extensions through the SDK; they don't redefine system semantics."* The protocol exposes both modes precisely so the SDK can route between them; the SDK papers over the distinction. Deployments without an SDK fall back to caller-managed mode selection per the rules above.

### 3.3 Watch Request

```
local/files/watch-request := {
  fields: {
    root_name:   {type_ref: "primitive/string"}
                 ; Name of the root mapping to watch
    action:      {type_ref: "primitive/string", optional: true}
                 ; "start" (default) or "stop"
    debounce_ms: {type_ref: "primitive/uint", optional: true}
                 ; Override default debounce window (default: 2000)
  }
}
```

---

## 4. Operations

### 4.1 Read

Read a file from the filesystem, chunk it via FastCDC, persist blob + chunks to the content store, bind the file entity into the tree, and return the file entity. The response envelope's `included` map carries the blob entity always; chunks are included when the blob's `total_size ≤ 64 KiB` (CONTENT v3.5 §4.3).

```
EXECUTE local/files  operation: "read"
  resource: {targets: ["local/files/shared/readme.md"]}
  params: null
```

```
handle_read(ctx):
  tree_path = ctx.resource.targets[0]
  root      = find_root_mapping(tree_path)
  if root is null: return error(404, "no_root_mapping")

  relative_path = strip_prefix(tree_path, root.prefix)
  fs_path       = root.filesystem_root + relative_path

  if not exists(fs_path):   return error(404, "file_not_found")
  if is_directory(fs_path): return error(400, "use_list_for_directories")

  raw_bytes = read_file(fs_path)

  ; CONTENT v3.5 §3.6 — chunk + persist via the shared substrate
  blob, chunks = build_blob_fastcdc(raw_bytes, DEFAULT_CHUNK_SIZE)
  for chunk in chunks:
    content_store.put(chunk)
  blob_hash = content_store.put(blob)

  stat = file_stat(fs_path)
  file_entity = {
    type: "local/files/file"
    data: {
      path:        relative_path
      size:        stat.size
      modified_at: stat.mtime_ms
      content:     blob_hash
      media_type:  guess_media_type(relative_path)    ; optional; absent when unknown
    }
  }

  ; Emit to tree — caches the read result; subsequent tree get returns
  ; the cached entity without touching the filesystem.
  ctx.emit(tree_path, file_entity)

  ; CONTENT v3.5 §5.2 MUST: blob always included
  ctx.include(blob_hash, blob)
  ; CONTENT v3.5 §4.3 SHOULD: chunks included when blob fits the inline threshold
  if blob.data.total_size <= MIN_CHUNK_SIZE:
    for chunk in chunks:
      ctx.include(chunk.content_hash, chunk)

  ; Descriptor publication — gated on per-root flag (§2.5)
  if root.publish_descriptors and file_entity.data.media_type is not null:
    publish_descriptor(blob_hash, file_entity.data.media_type)

  return {status: 200, result: file_entity}
```

`MIN_CHUNK_SIZE` is the CONTENT v3.5 constant (64 KiB = 65536 bytes per CONTENT §10.1). The inline-include boundary at `total_size ≤ MIN_CHUNK_SIZE` is a v1.2 MUST (see §10 conformance) — making the inline-include boundary cross-impl-uniform avoids divergent wire shapes for the same file across implementations.

`guess_media_type(path)` derives an IANA media type from the path extension via the platform's mime registry (Go's `mime.TypeByExtension`, Python's `mimetypes.guess_type`, equivalent in Rust). Returns absent when the extension is unknown; the `media_type` field stays absent rather than carrying a guessed-wrong value.

`publish_descriptor(blob_hash, media_type)` constructs a `system/content/descriptor` entity per CONTENT v3.5 §2.4 with `content = blob_hash`, computes its content hash to determine `D_hex`, and binds at the canonical path `/{local_peer}/system/content/descriptor/{B_hex}/{D_hex}` via `system/tree:put`. The descriptor body's `content` field carries the blob hash; the path embeds it again — CONTENT v3.5 §5.3's MUST integrity check on the consumer side gates against any path-corruption mismatch.

After a `read`, the file entity is available at the tree path. A subsequent `tree get` at the same path returns the cached entity without touching the filesystem. The entity tree is a projection of the filesystem, not the filesystem itself. The projection is updated by `read`, by the file watcher (§6), or by external writes (sync, remote operations) that the reverse write subscription (§5) translates back to disk.

### 4.2 List

List directory contents from the filesystem.

```
EXECUTE local/files  operation: "list"
  resource: {targets: ["local/files/shared/"]}
  params: null
```

```
handle_list(ctx):
  tree_path = ctx.resource.targets[0]
  root      = find_root_mapping(tree_path)
  if root is null: return error(404, "no_root_mapping")

  relative_path = strip_prefix(tree_path, root.prefix)
  fs_path       = root.filesystem_root + relative_path

  if not exists(fs_path): return error(404, "directory_not_found")

  children = []
  for entry in read_dir(fs_path):
    if matches_exclude(entry.name, root.exclude): continue
    ; Include filter applies to files only — never refuses directory
    ; descent (otherwise a "*.md" filter would skip subdirs containing
    ; matching files).
    if not is_directory(entry) and not matches_include(entry.name, root.include): continue

    child = {
      name:        entry.name
      entity_path: tree_path + entry.name
      entry_type:  "directory" if is_directory(entry) else "file"
      size:        entry.size if is_file(entry) else null
      modified_at: entry.mtime_ms
    }
    children.append(child)

  dir_entity = {
    type: "local/files/directory"
    data: {path: relative_path, children: children}
  }
  return {status: 200, result: dir_entity}
```

### 4.3 Write

Write content to the filesystem and update the tree. The write request carries either raw `bytes` (handler chunks via FastCDC) or a `content` blob hash (handler reassembles from chunks already in the content store — first-class dedup write path; no re-transmit).

```
EXECUTE local/files  operation: "write"
  resource: {targets: ["local/files/shared/readme.md"]}
  params: {
    type: "local/files/write-request"
    data: {bytes: h'23 48 65 6c 6c 6f 0a'}
  }
```

```
handle_write(ctx, params):
  tree_path     = ctx.resource.targets[0]
  root          = find_root_mapping(tree_path)
  if root is null: return error(404, "no_root_mapping")
  if root.read_only: return error(403, "read_only_root")

  relative_path = strip_prefix(tree_path, root.prefix)
  fs_path       = root.filesystem_root + relative_path

  has_bytes   = params.bytes is not null    ; presence, not non-emptiness
  has_content = params.content is not null
  if has_bytes == has_content:
    return error(400, "invalid_params", "exactly one of bytes / content must be set")

  ; NOTE: presence test is `params.bytes is not null`, NOT `len > 0`. A zero-byte
  ; write (empty file via bytes-mode) is a legitimate operation; the previous
  ; pseudocode shape (`len > 0`) rejected it as missing_input — a 3-of-3 latent
  ; bug Rust + Python + Go all inherited from the v1.2 pseudocode. Wire encoding
  ; carries the distinction: absent field decodes to null, present-but-empty to
  ; an empty byte string. The presence check honors that distinction.

  if has_bytes:
    raw_bytes = params.bytes
    blob, chunks = build_blob_fastcdc(raw_bytes, DEFAULT_CHUNK_SIZE)
    for chunk in chunks:
      content_store.put(chunk)
    blob_hash = content_store.put(blob)
  else:
    blob_hash = params.content
    blob      = content_store.get(blob_hash)
    if blob is null: return error(404, "content_not_found")
    raw_bytes = reassemble_content(blob)              ; CONTENT v3.5 §3.4

  if params.create_dirs:
    ensure_parent_dirs(fs_path)
  atomic_write_file(fs_path, raw_bytes)               ; SHOULD per §4.3 below

  stat = file_stat(fs_path)
  file_entity = {
    type: "local/files/file"
    data: {
      path:        relative_path
      size:        stat.size
      modified_at: stat.mtime_ms
      content:     blob_hash
      media_type:  params.media_type or guess_media_type(relative_path)
      written:     true
    }
  }
  ctx.emit(tree_path, file_entity)

  ctx.include(blob_hash, blob)
  if blob.data.total_size <= MIN_CHUNK_SIZE:
    for chunk in chunks_of(blob):                     ; bytes-mode: see note below
      ctx.include(chunk.content_hash, chunk)

  return {status: 200, result: file_entity}
```

`chunks_of(blob)` enumerates the blob's chunk list from the content store. In the **dedup-write branch** this is the only available source (the chunks were already there from sync or prior writes). In the **bytes-mode branch** the freshly-built `chunks` collection is still in scope from `build_blob_fastcdc` — implementations MAY reuse that binding directly instead of re-fetching from the content store. Both shapes are conformant; the spec uses `chunks_of(blob)` uniformly for readability.

**Atomic write (SHOULD).** Implementations SHOULD use an atomic write pattern that ensures a power loss leaves the prior content intact or commits the new content in full. The reference pattern is: open a sibling temp file in the same directory as the target; write all bytes; fsync the temp file; close; rename atomically to the target path; **on POSIX, open the parent directory, fsync it, close — this ensures the directory-entry update for the rename is durable, not merely queued by the kernel.** Without the parent-dir fsync, a power loss within the kernel writeback window can silently drop the rename even though the temp file is durable; per POSIX `fsync(2)` and ext4/xfs/btrfs semantics, `rename()` returning success means the kernel has *queued* the directory-entry update, not that it is on stable storage. On Windows, `MoveFileEx` with `MOVEFILE_REPLACE_EXISTING` provides equivalent semantics without the parent-dir fsync step. The non-atomic alternative (truncate + write in place) is non-conformant for sync deployments because a partial-write surfaces as legitimate file content on subsequent reads. The reverse-write path (§5.3) carries the same SHOULD.

**Streaming for large files (SHOULD).** Implementations SHOULD provide streaming reassembly for blob sizes above a threshold (RECOMMENDED: 64 MiB). The reference pattern: instead of materializing the full payload via `reassemble_content(blob)` into a single in-memory buffer, stream chunk-by-chunk from the content store directly into the FS write pipeline. The atomic-write SHOULD above still applies (sibling temp file, fsync, rename, parent-dir fsync on POSIX); the difference is the write phase consumes a stream rather than a buffer.

Implementations SHOULD also provide streaming ingest for the read / bytes-write side: chunk the file via a streaming FastCDC scanner (e.g., the `fastcdc` Rust crate's `v2020::StreamCDC`, or equivalent windowed scanner) instead of `read_file()` + chunker-over-buffer. The cross-impl conformance gate (CONTENT v3.5 §3.6.5 / §3.7 byte-identical chunk boundaries) is unchanged — the same FastCDC algorithm running over windowed input produces the same boundaries as over a full buffer; streaming is wire-conformant.

The 64 MiB threshold is RECOMMENDED, not normative. Implementations MAY stream for all blob sizes; deployments with tight latency requirements on small files MAY in-memory-buffer below their own threshold. Per CONTENT v3.5 §3.4 reassembly is Reference-classified — both in-memory and streaming shapes are conformant.

### 4.4 Delete

Remove a file from the filesystem and the tree.

```
EXECUTE local/files  operation: "delete"
  resource: {targets: ["local/files/shared/old-file.txt"]}
  params: null
```

```
handle_delete(ctx):
  tree_path     = ctx.resource.targets[0]
  root          = find_root_mapping(tree_path)
  if root is null: return error(404, "no_root_mapping")
  if root.read_only: return error(403, "read_only_root")

  relative_path = strip_prefix(tree_path, root.prefix)
  fs_path       = root.filesystem_root + relative_path

  existed = exists(fs_path)
  if existed:
    delete_file(fs_path)

  ctx.remove(tree_path)

  deleted_entity = {
    type: "local/files/deleted"
    data: {path: relative_path, existed: existed}
  }
  return {status: 200, result: deleted_entity}
```

Deletion of the file entity from the tree leaves the blob + chunks in the content store (CONTENT v3.5 §6.6 persistence-by-default). A subsequent `write` of new content at the same path produces a new blob; both blobs coexist in the content store. The EXTENSION-GC graduation will define the normative reachability contract; until then, the content store grows monotonically by design.

### 4.5 Watch

Start or stop filesystem monitoring for a configured root mapping. The operation manages the watcher configuration entity — creating or updating it at `system/config/local/files/watch/{root_name}`. See §6 for the watcher behavior.

```
EXECUTE local/files  operation: "watch"
  resource: {targets: ["local/files"]}
  params: {
    type: "local/files/watch-request"
    data: {root_name: "shared"}
  }
```

```
handle_watch(ctx, params):
  root_name = params.root_name
  action    = params.action or "start"

  config_path = "system/config/local/files/" + root_name
  root = ctx.entity_tree.get(config_path)
  if root is null: return error(404, "root_mapping_not_found")

  watcher_path = "system/config/local/files/watch/" + root_name

  if action == "stop":
    existing = ctx.entity_tree.get(watcher_path)
    if existing is null: return error(404, "watcher_not_found")
    stop_platform_watcher(root_name)
    watcher_entity = {
      type: "local/files/watcher-config"
      data: {root_name: root_name, status: "stopped"}
    }
    ctx.entity_tree.put(watcher_path, watcher_entity)
    return {status: 200, result: watcher_entity}

  ; action == "start"
  debounce_ms = params.debounce_ms or 2000
  start_platform_watcher(root.data.filesystem_root, root.data.exclude)
  watcher_entity = {
    type: "local/files/watcher-config"
    data: {root_name: root_name, status: "active", debounce_ms: debounce_ms}
  }
  ctx.entity_tree.put(watcher_path, watcher_entity)
  return {status: 200, result: watcher_entity}
```

---

## 5. Reverse Write (Tree → Filesystem)

When an entity is stored in the tree at a path matching a root mapping prefix — by sync, by a remote peer, by any tree put — the handler detects it via subscription and writes the corresponding file to the filesystem.

This is the mechanism that makes sync produce actual files on disk.

### 5.1 Subscription Setup

On startup, for each configured root mapping, the handler subscribes to tree changes at the mapping's prefix:

```
setup_reverse_write(root):
  subscribe(
    pattern:       root.prefix + "*"
    events:        ["created", "updated", "deleted"]
    deliver_to:    {uri: "system/inbox/local/files/reverse/" + root.name}
    deliver_token: <handler's deliver token>
  )
```

If EXTENSION-CONTINUATION is installed, a continuation at the inbox path handles the notification automatically. Without it, the handler processes inbox deliveries directly.

**Subscription-mechanism flexibility.** The §10.1 MUST ("Reverse write via subscription on configured root mapping prefixes") pins observable behavior — reverse-write fires for tree changes within configured root prefixes — not the subscription wiring. Implementations MAY satisfy it by (a) calling `system/subscription:subscribe` per-root as shown above, or (b) consuming a single global tree-change event stream and filtering by configured root prefixes in the handler. Both shapes are conformant. The pseudocode shows the per-root form because it composes naturally with the inbox-delivery path; the global-stream form is the simplest shape for a handler that already has tree-change observability for other reasons.

### 5.2 Notification Handler

```
on_tree_change_notification(notification):
  tree_path = notification.uri
  event     = notification.event

  if event == "deleted":
    reverse_delete(tree_path)
    return

  entity = content_store.get(notification.hash)
  if entity is null: return
  if entity.type != "local/files/file": return    ; Only reverse-write file entities

  root = find_root_mapping(tree_path)
  if root is null: return
  if root.read_only: return

  reverse_write(root, tree_path, entity)
```

### 5.3 Write Algorithm

```
reverse_write(root, tree_path, file_entity):
  relative_path      = strip_prefix(tree_path, root.prefix)
  fs_path            = root.filesystem_root + relative_path
  incoming_blob_hash = file_entity.data.content

  ; Fetch the incoming blob entity early — its chunk_size is required for the
  ; circuit-breaker recompute (§5.5) and we'll need the entity for reassembly.
  blob = content_store.get(incoming_blob_hash)
  if blob is null: return    ; Blob entity not yet arrived; await next sync delivery
  incoming_chunk_size = blob.data.chunk_size         ; CONTENT v3.5 §3.5 — per-blob

  ; Loop prevention — early-out when local FS already matches.
  ; The circuit-breaker recompute MUST use the incoming blob's chunk_size
  ; (per §5.5), not the consumer's local default; otherwise a peer running
  ; a different chunk-size default re-chunks the on-disk file at the wrong
  ; size, hashes diverge, and the circuit-breaker spuriously rewrites
  ; identical content.
  if exists(fs_path):
    current_bytes      = read_file(fs_path)
    current_blob, _    = build_blob_fastcdc(current_bytes, incoming_chunk_size)
    current_blob_hash  = content_hash(current_blob)
    if current_blob_hash == incoming_blob_hash:
      return    ; Filesystem already matches — skip

  raw_bytes = reassemble_content(blob)               ; CONTENT v3.5 §3.4
  if raw_bytes is null: return    ; Chunks missing from content store;
                                  ; await next subscription event (do NOT
                                  ; assert / panic — partial sync is expected)

  ensure_parent_dirs(fs_path)
  atomic_write_file(fs_path, raw_bytes)              ; SHOULD per §4.3
```

Same atomic-write SHOULD as §4.3 applies — including the parent-directory fsync on POSIX after the rename, which is critical for the reverse-write path. The reverse-write path is the more critical one for atomicity, since the input is incoming sync content (not a local user action); a power loss mid-reverse-write produces a half-applied sync, and a lost rename (no parent-dir fsync) silently undoes the sync without a detection signal.

The streaming-reassembly SHOULD from §4.3 applies symmetrically and is **especially load-bearing** for reverse-write: incoming sync payloads can be arbitrarily large (multi-GB syncs are part of the Stage 3 use case), and the reverse-write path is the more common large-payload site than user-initiated writes. Streaming reassembly directly into the FS write pipeline avoids materializing the full blob in memory; CONTENT v3.5 §3.4 Reference-classification covers both shapes.

If chunks are missing from the local content store (a partial sync delivery), `reassemble_content` returns the canonical `503 blob_pending_sync` error per `EXTENSION-CONTENT.md` §3.4 (sync-state-visibility predicate applies — peers with active subscription return 503, peers without return 404). The reverse-write completes when subsequent chunk arrivals fill the gap and the subscription fires again. CONTENT v3.5 §7.1 / §7.2 (handler-mediated and system-content transfer patterns) govern how missing chunks arrive; reverse-write is the consumer, not the fetcher. Reverse-write MUST treat `blob_pending_sync` as retry-eligible without operator intervention.

### 5.4 Reverse Delete

```
reverse_delete(tree_path):
  root = find_root_mapping(tree_path)
  if root is null: return
  if root.read_only: return

  relative_path = strip_prefix(tree_path, root.prefix)
  fs_path       = root.filesystem_root + relative_path

  if exists(fs_path):
    delete_file(fs_path)
```

### 5.5 Loop Prevention

The reverse write and the file watcher (§6) can create a feedback loop:

```
file change → watcher → tree put → subscription fires → reverse write → file change → ...
```

The blob-hash comparison in §5.3 is the primary circuit breaker. When the handler writes to the tree (via `read`, `write`, or watcher), the filesystem already has that content. The reverse write detects `current_blob_hash == incoming_blob_hash` and skips. Same bytes ⇒ same FastCDC chunks ⇒ same blob hash — the comparison is sufficient **as long as both sides chunk with the same parameters**.

**Circuit-breaker recompute MUST use the incoming blob's `chunk_size` field (normative).** When the reverse-write handler recomputes the on-disk file's blob hash for the circuit-breaker comparison, it MUST chunk the on-disk file using the `chunk_size` value carried in the incoming blob entity (CONTENT v3.5 §3.5 — recorded on every `system/content/blob` entity), **not** the consumer's local `DEFAULT_CHUNK_SIZE`. Using the local default produces hash divergence whenever a peer running a different chunk-size default ingests the same content; this breaks cross-peer dedup and triggers unnecessary rewrites that loop the content back to the sender. The pseudocode in §5.3 reflects this — `build_blob_fastcdc(current_bytes, incoming_chunk_size)` reads `chunk_size` off the fetched blob entity and uses it for the recompute. The local `DEFAULT_CHUNK_SIZE` is only relevant for forward chunking (the watcher path and user writes when the consumer is producing the blob), never for the circuit-breaker recompute.

For tree deletes → filesystem deletes → watcher detects deletion → tree delete: the second tree delete is a no-op (path already unbound).

Implementations MAY additionally track which tree paths were recently written by the handler itself and skip reverse write for those paths within a short window, as a performance optimization that avoids the cost of read-and-rechunk on the loop-back path. This is not required for correctness — the blob-hash circuit breaker handles it.

---

## 6. File Watcher (Filesystem → Tree)

The file watcher monitors a filesystem directory for changes and translates them into entity tree writes. This is the mechanism that makes local edits propagate through the entity protocol.

### 6.1 Configuration

The watcher is managed through the `watch` operation (§4.5), which creates and updates configuration entities at `system/config/local/files/watch/{name}`. The configuration entity persists the watcher state across peer restarts.

On startup, implementations MAY scan for existing watcher configuration entities with `status: "active"` and automatically resume those watchers. This behavior is implementation-defined.

### 6.2 Mechanism

The watcher uses platform-native filesystem notification APIs:

| Platform | API |
|----------|-----|
| Linux | inotify |
| macOS | FSEvents / kqueue |
| Windows | ReadDirectoryChangesW |

On detecting a filesystem change:

```
on_fs_change(fs_event_type, fs_path):
  root = find_root_for_fs_path(fs_path)
  if root is null: return

  relative_path = strip_prefix(fs_path, root.filesystem_root)
  if matches_exclude(relative_path, root.exclude): return
  ; Same include-filter discipline as list: applies to files only
  if not is_directory(fs_path) and not matches_include(relative_path, root.include): return

  debounce_buffer.add(root.name, relative_path, fs_event_type)
```

After the debounce window closes:

```
on_debounce_flush(root_name, changes):
  root = get_root_mapping(root_name)

  for (relative_path, event_type) in coalesce(changes):
    tree_path = root.prefix + relative_path

    if event_type == "deleted":
      entity_tree.remove(tree_path)
      continue

    fs_path = root.filesystem_root + relative_path
    if is_directory(fs_path): continue    ; Directories tracked implicitly

    ; CONTENT v3.5 §3.6 — re-chunk via FastCDC. Edit stability means
    ; small edits produce ~1 new chunk; the rest reuse identically.
    raw_bytes = read_file(fs_path)
    blob, chunks = build_blob_fastcdc(raw_bytes, DEFAULT_CHUNK_SIZE)
    for chunk in chunks:
      content_store.put(chunk)
    blob_hash = content_store.put(blob)

    stat = file_stat(fs_path)
    file_entity = {
      type: "local/files/file"
      data: {
        path:        relative_path
        size:        stat.size
        modified_at: stat.mtime_ms
        content:     blob_hash
        media_type:  guess_media_type(relative_path)
      }
    }
    entity_tree.put(tree_path, file_entity)
```

**Edit-stability payoff.** A 1-byte edit on a 1 GiB FastCDC-chunked file with the default 4 MiB target chunks produces ~256 chunks; FastCDC's edit-stability property means typically 1–3 of them shift and the rest reuse identically. The watcher stores 1–3 new chunks + 1 new blob entity; the other ~253 chunks are already in the content store. Compared to a re-encode-as-string watcher, the cost reduction is roughly two orders of magnitude for large files with localized edits.

### 6.3 Debounce

Rapid filesystem changes (e.g., editor save-then-reformat, build tools writing multiple files) are coalesced into a single batch of tree writes. The debounce window is configurable per watcher (default: 2000 ms).

Coalescing rules for changes to the same path within one debounce window:

| Sequence | Result |
|----------|--------|
| created → updated | created (final content) |
| created → deleted | no-op |
| updated → updated | updated (final content) |
| updated → deleted | deleted |
| deleted → created | updated (content changed) |

### 6.4 Deletion Detection

When a file is removed from the filesystem, the watcher removes the corresponding tree path binding. The blob and chunks remain in the content store (CONTENT v3.5 §6.6 persistence-by-default; EXTENSION-GC will define the normative reachability contract when it lands).

---

## 7. Capability Model

### 7.1 Handler Capability

The files handler uses the standard capability model (ENTITY-CORE-PROTOCOL.md §5):

```
system/capability/grant-entry := {
  handlers:   {include: ["local/files"]}
  resources:  {include: ["local/files/shared/*"]}
  operations: {include: ["read", "list", "write", "delete"]}
}
```

Grant dimensions:

| Dimension | Scoping |
|-----------|---------|
| `handlers` | `local/files` — the handler pattern |
| `resources` | Path patterns under the handler prefix (e.g., `local/files/shared/*`) |
| `operations` | `read`, `list`, `write`, `delete`, `watch` |

### 7.2 Read-Only Access

```
system/capability/grant-entry := {
  handlers:   {include: ["local/files"]}
  resources:  {include: ["local/files/shared/*"]}
  operations: {include: ["read", "list"]}
}
```

### 7.3 Revision Capability

For file sync between peers, grants are needed on the tree and revision handlers. The files handler is not directly involved in sync — the revision extension operates on the tree. The reverse write subscription translates tree changes into filesystem writes automatically.

```
; Tree access for sync
system/capability/grant-entry := {
  handlers:   {include: ["system/tree"]}
  resources:  {include: ["local/files/shared/*"]}
  operations: {include: ["get", "put"]}
}

; Sync handler access
system/capability/grant-entry := {
  handlers:   {include: ["system/revision"]}
  resources:  {include: ["local/files/shared/*"]}
  operations: {include: ["commit", "status", "log", "fetch", "pull", "push"]}
}
```

The remote peer does not need grants on the `local/files` handler for sync. Sync writes through the tree handler; the local files handler's subscription handles the translation to disk.

When the remote peer's `local/files/file` entity references a blob hash, the receiving peer fetches the referenced chunks per CONTENT v3.5 §7. The standard transfer pattern uses the sender's content handler under the standard cap; no local/files-specific grants on the sender side are required for chunk fetch.

---

## 8. Path Resolution

### 8.1 Root Mapping Resolution

```
find_root_mapping(tree_path):
  ; Iterate configured root mappings, find longest prefix match
  best_match = null
  for config in list_entities("system/config/local/files/"):
    if tree_path starts with config.data.prefix:
      if best_match is null or len(config.data.prefix) > len(best_match.data.prefix):
        best_match = config
  return best_match
```

### 8.2 Path Translation

```
Tree path:       local/files/shared/src/main.rs
Root prefix:     local/files/shared/
Filesystem root: /home/alice/shared/
Relative path:   src/main.rs
Filesystem path: /home/alice/shared/src/main.rs
```

### 8.3 Path Security

The handler MUST validate that the resolved filesystem path is within the configured root directory. Two classes of escape MUST be defended against: parent-traversal via `..` segments, and same-component escape via a symlink at the resolved target.

**Parent-traversal defense (MUST).** Path traversal attacks using `..` segments MUST be rejected:

```
validate_path(root, relative_path):
  canonical = canonicalize(root.filesystem_root + relative_path)
  if not canonical starts with canonicalize(root.filesystem_root):
    return error(403, "path_traversal_rejected")
```

**Leaf-symlink defense (MUST).** Implementations MUST reject leaf symlinks at the resolved target path. The parent-traversal check above defends against `..` segments and parent-component symlinks (when the resolver canonicalizes the parent); it does NOT defend against a symlink at the leaf itself. An authenticated writer with `local/files/shared/* write` capability could otherwise have a symlink at the target path — placed by a prior write through the same handler, or by an attacker with concurrent access to the configured root — and the subsequent open would follow it to anywhere on disk.

The protocol pins the property — leaf-symlink rejection — but not the mechanism. Reference platform recipes:

- **POSIX baseline.** Open the leaf with `O_NOFOLLOW`; an attempt to open through a leaf symlink fails with `ELOOP`, which the handler surfaces as `path_traversal_rejected`. NOTE: `O_NOFOLLOW` rejects only the *leaf* — intermediate-component symlinks are still followed. Intermediate-component TOCTOU defense requires the kernel-enforced atomic walk-down primitives below.
- **Linux 5.6+ (RECOMMENDED).** `openat2(rootfd, relative_path, &how)` with `how.resolve = RESOLVE_BENEATH | RESOLVE_NO_MAGICLINKS` performs the entire path walk atomically and returns `-EXDEV` on any escape, closing both leaf and intermediate-component TOCTOU windows.
- **Rust ecosystem.** `cap-std` (Bytecode Alliance; used by Wasmtime) provides `cap_std::fs::Dir` over `openat`-based syscalls with `RESOLVE_BENEATH` / `O_NOFOLLOW` semantics — kernel-enforced, no TOCTOU.
- **Windows.** `CreateFileW` with `FILE_FLAG_OPEN_REPARSE_POINT` refuses reparse-point traversal; `GetFinalPathNameByHandleW` normalizes for prefix comparison.

Implementations SHOULD pick the strongest defense their platform / language ecosystem supports. `O_NOFOLLOW` on the leaf is the minimum acceptable mitigation; intermediate-component TOCTOU defense (`openat2(RESOLVE_BENEATH)` or equivalent) is RECOMMENDED for production deployments.

A property-level non-atomic mitigation (`Lstat` the resolved path; reject if symlink; then open) closes the trivial leaf-symlink exploit at the cost of a narrow TOCTOU window between the check and the subsequent open. This is acceptable as an interim defense — pinning the platform-specific primitives is the next pass.

**Normative error code (MUST).** Both the parent-traversal defense and the leaf-symlink defense MUST surface failure as **HTTP-style status 403 with error code `path_traversal_rejected`**. The mechanism varies by platform (`O_NOFOLLOW`, `openat2(RESOLVE_BENEATH)`, `cap-std`, `FILE_FLAG_OPEN_REPARSE_POINT`); the wire response shape is uniform. Validate-suite cross-impl behavioral gates assert the specific code. This pin lets sibling impls converge on the same error surface without re-litigation.

**All path-resolving callsites MUST apply both defenses.** The two defenses above MUST be applied at every callsite that resolves a tree path to a filesystem path — `read`, `write`, `list`, `delete`, the watcher's debounce-flush ingest, and the reverse-write / reverse-delete handlers. A common error surfaced by Go's L5 audit (and likely affecting Rust + Python as well, pending their audit): reverse-write and reverse-delete using bare path-join to derive the FS path, bypassing the canonical resolver entirely and skipping both the parent-traversal and leaf-symlink checks. Reverse-write is the more critical path (input is incoming sync content, not local user action); the defenses MUST apply.

**Implicit dependency on the tree layer (informative).** The local-files path defenses derive the relative path from the tree path via `strings.TrimPrefix(tree_path, root.prefix)` (or equivalent). The defenses assume the tree path itself is free of traversal segments — `./` and `../` rejected by ENTITY-CORE-PROTOCOL.md §1.4 + `CleanPath` discipline. If a future V7 change introduces a `..`-permitting path form (directory-relative resolution has come up), the local-files defenses silently weaken. Implementations and reviewers SHOULD treat the tree-layer traversal rejection as a load-bearing dependency.

---

## 9. Write Authorization

The local files handler operates in mixed mode — some writes are caller-authorized, others are handler-authorized or autonomous.

| Operation | Write target | Authorization | Capability recorded |
|-----------|-------------|---------------|-------------------|
| read (lazy populate) | `local/files/{path}` | Handler grant | Handler grant |
| read (content store: blob + chunks) | content store | Handler grant (`system/content:ingest`) | Handler grant |
| read (descriptor publish, when enabled) | `system/content/descriptor/{B_hex}/{D_hex}` | Handler grant (`system/tree:put`) | Handler grant |
| write | `local/files/{path}` | Caller capability | Caller capability |
| write (content store: blob + chunks) | content store | Handler grant (`system/content:ingest`) | Handler grant |
| delete | `local/files/{path}` | Caller capability | Caller capability |
| watch | `system/config/local/files/watch/{name}` | Handler grant | Handler grant |
| reverse write (background) | `local/files/{path}` | Autonomous — handler grant | Handler grant |
| reverse write (content store fetch, when chunks missing) | content store | Handler grant (`system/content:get`) | Handler grant |

**Caller-authorized writes** (write, delete): The caller specifies the file path. The handler MUST verify the caller's capability covers the write path. The root mapping's `read_only` check (§4.3) is an additional domain constraint — it is not a substitute for capability checking. The handler-side `system/content:ingest` for blob/chunk persistence is performed under the handler's own grant — the caller does not need a content-handler cap.

**Handler-authorized writes** (read lazy populate, descriptor publish, watch config, content store writes during read/watcher): The handler decides to cache a file entity, publish a descriptor, store watcher configuration, or land blob/chunk entities. These are handler-managed — the handler's grant authorizes them (the four `internal_scope` grants in §3.1).

**Autonomous writes** (reverse write): Triggered by filesystem events from a remote source (sync notification), not by an external caller. The handler acts under its own grant with the local peer as author. The handler's grant was issued at registration time with scope covering `local/files/*` plus the `system/content` and `system/content/descriptor/*` additions.

See ENTITY-CORE-PROTOCOL.md §6.8 for the general write authorization model.

---

## 10. Conformance

### 10.1 MUST Implement

- Handler at `local/files` with `read`, `write`, `list`, `delete` operations (§3, §4). The `watch` operation is conditional — see the next bullet.
- **If the handler manifest exposes the `watch` operation, the handler MUST monitor filesystem changes** for the configured root mappings via a platform-native filesystem-notification API (inotify / FSEvents / kqueue / ReadDirectoryChangesW). A `watch` operation that returns success but does not propagate on-disk edits to the tree within the configured debounce window is non-conformant. Implementations without a watcher implementation MUST omit `watch` from the manifest's `operations` map — a deliberate visible signal (callers receive `unknown_operation`) is preferable to an undetectable silent failure of a published capability.
- File entity v1.2 shape (§2.1): `content` is a `system/hash` to a `system/content/blob`; no inline `content: string`; no `encoding` field.
- Write-request two-mode shape (§3.2): exactly one of `bytes` / `content` MUST be present.
- Chunking via CONTENT v3.5 §3.6 FastCDC (the protocol default), producing byte-identical chunk boundaries on the same input across implementations.
- Blob and chunk entities persisted to the content store per CONTENT v3.5 §6.
- Response envelope `included` on `read` and `write`: blob always (CONTENT v3.5 §5.2 MUST); chunks when `total_size ≤ MIN_CHUNK_SIZE` (CONTENT v3.5 §4.3 SHOULD applied as MUST at this surface so the cross-impl inline-include boundary is deterministic).
- Reverse write via subscription on configured root mapping prefixes (§5). **Reverse-write fires eventually after the producing tree-write commits; implementations MAY satisfy this synchronously or asynchronously. The producing tree-write's emit cascade MUST NOT be blocked on filesystem work — synchronous implementations dispatch the FS work in a way that does not stall the cascade (executor, thread pool, task queue, dedicated goroutine driven by a buffered channel); asynchronous implementations post a notification and complete the FS write when the executor reaches it.** Both shapes are conformant. The §10.2 stat-cache SHOULD makes the FS-work cost cheap in the common case but is not normatively prerequisite for the MUST — the non-blocking property holds regardless of cache presence.
- Blob-hash comparison for loop prevention (§5.5).
- Path traversal prevention (§8.3) — parent-traversal AND leaf-symlink defense, both required, both MUST be applied at every path-resolving callsite including reverse-write / reverse-delete.
- Capability enforcement per standard dispatch chain (§7).
- Handler `internal_scope` covers the four grants in §3.1.
- Root mapping configuration via `system/config/local/files/*` entities (§2.5).
- **Stat-cache for reverse-write circuit breaker and watcher fast-path (promoted from SHOULD to MUST).** Implementations MUST maintain a path-keyed cache: `path → (dev, ino, mtime_ns, ctime_ns, size, mode_bits, blob_hash)`. The cache-hit predicate is "all stat fields match AND `mtime_ns < cache_write_time_ns`" (Git's racy-clean rule per `git-scm.com/docs/index-format`). When writing a cache entry whose `mtime_ns >= cache_write_time_ns`, the implementation smudges `size = 0` so the next stat is a forced miss. Reverse-write and watcher circuit breakers consult the cache before rechunking; cache hit + stat match ⇒ skip rechunk; cache miss OR stat-changed ⇒ full rechunk + cache update. **MUST-promotion preconditions met (per spec close-out):** (a) Go landed L7 stat-cache (commit `e375d19`; 106× empirical on 64 KiB, matches Python's 138,510× on larger files); (b) cross-impl behavioral gate green (V4 PASS three-way, commit `8e396a4`). Cache shape is normative; persistence and eviction policies are implementation-defined.

### 10.2 SHOULD Implement

- File watcher SHOULD be implemented (deployments without filesystem-change propagation lose key sync semantics). When implemented, the watcher MUST follow §10.1 — filesystem monitoring via a platform-native notification API is mandatory for any impl that exposes `watch` in its manifest. When not implemented, the impl MUST omit `watch` from the manifest per §10.1 rather than ship a config-only handler that returns success without monitoring.
- Debounce with configurable window (default 2000 ms).
- Atomic FS write (§4.3 and §5.3): temp + fsync + rename + parent-dir fsync (POSIX) pattern. The parent-dir fsync after rename closes a POSIX crash-consistency hole — `rename()` returning success means the kernel has *queued* the directory-entry update, not that it is durable on stable storage. Without the parent-dir fsync, a power loss within the kernel writeback window can silently drop the rename even though the renamed file was fsync'd. Windows `MoveFileEx` provides equivalent semantics without the parent-dir step.
- `create_dirs` support on write.
- Auto-start watcher based on persisted configuration (§6.1).
- `media_type` derivation from path extension via the platform mime registry when the extension is well-known.
- *(Stat-cache promoted to MUST — see §10.1.)*
- **Watcher overflow recovery.** Filesystem-notification APIs have bounded internal queues; under burst load (large directory copies, build outputs, recursive deletes) the queue may overflow and the kernel discards events without re-emitting them. **Implementations exposing the `watch` operation MUST detect overflow and recover via full rescan of the watched root** — a watcher that silently desynchronizes after overflow is functionally equivalent to no watcher at all, undermining the §10.1 MUST-iff-declared posture. (Promotion from SHOULD to MUST per Rust pushback; aligned with §10.1.) Implementations that do not expose `watch` are unaffected. Platform overflow signals: Linux inotify `IN_Q_OVERFLOW` event; macOS FSEvents `kFSEventStreamEventFlagMustScanSubDirs` flag; Windows ReadDirectoryChangesW `ERROR_NOTIFY_ENUM_DIR` from the I/O completion port. The rescan re-walks the configured root, seeding the debounce buffer with synthetic Updated events for each regular file. The stat-cache SHOULD above makes rescan cheap in the common case (most files unchanged ⇒ stat-cache hits ⇒ skip rechunk). Without overflow recovery, the watcher's in-memory model silently desynchronizes from disk.
- **Unicode normalization at the FS boundary (interim SHOULD; promotion to MUST tracked for Stage 3 cross-platform sync test).** Implementations SHOULD NFC-normalize filenames per Unicode UAX #15 before binding into the tree. Linux ext4 stores filename bytes verbatim; macOS APFS and Windows NTFS normalize at the VFS layer; a file written from a Linux peer with bytes `café` (NFC) is invisible to a macOS peer searching via the NFD variant. The protocol-level fix is to pin a single normalization at the boundary. NFC is the recommended form (matches W3C Character Model; renders identically across platforms). Non-UTF-8 filenames (Linux ext4 surrogate-escape strings from undecodable bytes) SHOULD be rejected as `invalid_filename` — ECF wire encoding requires strict UTF-8. **Promotion to MUST is deferred until Stage 3 cross-platform sync surfaces the actual divergence under deployed load**; the interim SHOULD lets early adopters converge without imposing UX cost (rejecting valid NFD filenames) on deployments that don't yet cross platforms.

### 10.3 MAY Implement

- Multiple simultaneous root mappings (§1.2).
- Read-only root mappings (§2.5).
- Glob-based `exclude` and `include` patterns on root mappings (§2.5).
- Descriptor publication (§2.5 + §4.1 `publish_descriptors`; gated per-root).
- Directory entity caching in tree (§4.2).
- Recently-written-path tracking for additional loop prevention beyond the blob-hash circuit breaker (§5.5).

### 10.4 Implementation-Defined

- Platform-specific filesystem notification API (inotify / FSEvents / ReadDirectoryChangesW).
- Default debounce window (recommended: 2000 ms).
- The mechanism implementing CONTENT v3.5 §4.4 step 1 (handler-manages-blob check): entity-tree walk, in-memory blob-hash set, or other strategies.
- `media_type` derivation source (mime registry, libmagic, configurable lookup table).
- Atomic-write recipe details (the §4.3 / §5.3 SHOULDs pin outcome, not mechanism — sibling temp + fsync + rename is the reference; alternatives that achieve the same atomicity guarantee on the target platform are conformant).

### 10.5 Cross-impl Conformance Gates (Phase 4)

The Go Track 1 validate-peer surface (37 checks; `entity-core-go/cmd/internal/validate/local_files.go`) is the reference shape. Critical cross-impl gates:

1. **Cross-handler blob-hash convergence.** Same file bytes through `local/files:read` and through any other handler that uses the CONTENT substrate produce byte-identical blob hashes. This is the load-bearing v1.2 property — CONTENT v3.5 §1.1 promised it, v1.2 is the first deployed surface that exercises it.
2. **Cross-impl chunk-boundary convergence.** Same file bytes chunked by Go, Python, and Rust implementations produce byte-identical chunk boundaries. This is the CONTENT v3.5 §3.6.5 / §3.7 Conformance class; local/files is a downstream consumer of the same gate.
3. **Edit-stability chunk reuse.** A 1-byte mid-file edit on a ≥6 MiB body retains ≥75% chunk reuse between pre-edit and post-edit blob chunk lists. The 75% floor catches catastrophic chunker bugs without locking in a specific FastCDC parameter regime; in practice reuse is far higher.
4. **Inline-include boundary at 64 KiB.** `read` of a file with `total_size ≤ 65536` returns blob + all chunks in `included`; `total_size == 65537` returns blob only (chunks excluded).
5. **Content-mode dedup write.** A `write` with `content: blob_hash` where the blob already exists in the content store produces a file entity whose `content` field equals the input `blob_hash` — no re-chunking, dedup preserved.

These gates are tracked in the cross-impl validation matrix (compare `reviews/VALIDATION-MATRIX-IDENTITY-FOUNDATIONS.md` for format reference) once Phase 4 starts. The architecture team does NOT pre-author conformance vectors; they emerge as a Stage-4 byproduct per GUIDE-EXTENSION-DEVELOPMENT.md §7.

**Behavioral gates (v1.3, Phase 4-bis).** Beyond the five consistency gates above, three behavioral checks land in the Go reference validate-peer suite (`entity-core-go/cmd/internal/validate/local_files.go`); sibling impls mirror. These catch silent-failure modes that consistency-only checks miss:

- **V1 — Oversized bytes-mode write returns graceful error.** A 20 MiB bytes-mode payload returns a graceful error (400 `invalid_params` or transport-layer `frame too large`) — **not a handler crash, panic, or hung connection.** Gates against unbounded buffer allocation regressions and validates the L1 documented constraint surfaces cleanly.
- **V2 — Watcher fires on disk edit.** Write a file directly to disk (bypassing the handler), start a watch, wait debounce + ~500ms safety margin, assert the tree contains the corresponding `local/files/file` entity. Skip-with-WARN if the impl declares no `watch` capability; FAIL if `watch` is in the manifest but propagation does not occur — the §10.1 conditional-MUST enforcement mechanism.
- **V3 — Descriptor publication exercised.** With `publish_descriptors: true` on a root, a `read` of a file with a known media-type produces a `system/content/descriptor` entity at the canonical `/{local_peer}/system/content/descriptor/{B_hex}/{D_hex}` path with the correct `media_type`. Gates against latent drift in the descriptor-publish path.

These behavioral gates are part of the production-readiness review surface per `GUIDE-EXTENSION-DEVELOPMENT.md` §9.1. They MUST pass for grade transition to Production-Ready.

---

## 11. Types Installed

The handler registers the following types at `system/type/`:

| Type | Description |
|------|-------------|
| `local/files/file` | File metadata entity carrying a `system/content/blob` hash |
| `local/files/directory` | Directory listing entity |
| `local/files/directory/entry` | Directory entry within a listing |
| `local/files/deleted` | Deletion confirmation entity |
| `local/files/root-config` | Root mapping configuration (including `publish_descriptors`) |
| `local/files/watcher-config` | File watcher state |
| `local/files/write-request` | Write operation parameters (two-mode: `bytes` / `content`) |
| `local/files/watch-request` | Watch operation parameters |

Domain types are not bootstrap types — they are registered at handler installation time.

The v1.1 type `local/files/content` is retired in v1.2. The CONTENT v3.5 substrate (`system/content/blob` + `system/content/chunk`) replaces the per-handler sidecar with substrate-level dedup that crosses handler boundaries.
