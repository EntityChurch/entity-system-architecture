# Clock Extension — Normative Specification

**Version**: 1.3
**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.3+)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. Overview

The clock extension provides a unified timing mechanism for the entity system. It formalizes the `system_clock_ms()` function used by the history extension (EXTENSION-HISTORY.md §5.1, §5.2) and the revision extension (EXTENSION-REVISION.md §6.1), and layers additional clock disciplines on top: logical clocks for causal ordering within a peer, vector clocks for concurrency detection across peers, and hybrid logical clocks (HLC) for total ordering correlated with real time.

### 1.1 Scope

This extension covers:

- **Wall-clock timestamps** — formalizing `system_clock_ms()` as a defined operation
- **Logical clocks** — Lamport-style causal ordering within a peer
- **Vector clocks** — concurrency detection across peers
- **Hybrid logical clocks** — total order with wall-clock correlation
- **Emit integration** — clock increment on tree writes
- **Sync integration** — clock merge during peer synchronization
- **Tick subscriptions** — periodic clock events for timed execution

This extension does **not** cover:

- Consensus timers or leader election timeouts (application-level, uses clock primitives)
- Real-time scheduling guarantees (OS-level concern)
- Clock synchronization protocols (NTP, PTP — external to the entity system)
- Distributed agreement on time (no global clock — peers maintain local clocks)

### 1.2 Relationship to Other Extensions

**History extension.** History transitions record a `timestamp` field using `system_clock_ms()`. This extension formalizes that function. History does not change — it continues to call `system_clock_ms()`, which this extension defines as returning a `system/clock/timestamp` value. When the clock extension is present, transitions MAY additionally record the logical clock value for causal ordering.

**Revision extension.** Revision entries record a `timestamp` field using `system_clock_ms()`. During sync, clock state is merged alongside version DAGs (§5). The `lww` merge strategy (EXTENSION-REVISION.md §2.3) compares version timestamps — with HLC, this comparison becomes causally meaningful rather than wall-clock dependent.

**Subscription extension.** Tick subscriptions (§3.3) deliver periodic clock events through the subscription extension's notification mechanism. Tick events are tree change events on `system/clock/tick/` paths, firing standard subscription notifications.

### 1.3 Design Principles

**Wall-clock is the baseline.** Every peer has a wall clock. `system_clock_ms()` works without the clock extension installed — it's a host function. The clock extension formalizes it and layers causal clocks on top.

**Opt-in complexity.** A peer that only needs timestamps uses wall-clock mode (the default). A peer that needs causal ordering enables logical clocks. A peer that needs concurrency detection enables vector clocks. A peer that needs total ordering enables HLC. Each level adds cost — no peer pays for what it doesn't use.

**Clock is local state.** Each peer maintains its own clock. There is no global clock. Cross-peer ordering emerges from clock merge during sync, not from clock synchronization.

---

## 2. Type Definitions

### 2.1 Timestamp

Wall-clock milliseconds since Unix epoch. This is what `system_clock_ms()` returns.

```
system/clock/timestamp := {
  fields: {
    ms: {type_ref: "primitive/uint"}
         ; Milliseconds since the Unix epoch
  }
}
```

`system_clock_ms()` is formally defined as:

```
system_clock_ms():
  return host_wall_clock_ms()
  ; Returns the host system's wall-clock time as milliseconds since Unix epoch.
  ; The value is a primitive/uint suitable for use in timestamp fields.
  ; Monotonicity is not guaranteed — the host clock may be adjusted.
```

The value returned by `system_clock_ms()` is the `ms` field of a `system/clock/timestamp` entity. When existing extensions write `timestamp: system_clock_ms()`, they are writing this value.

### 2.2 Logical Clock

Lamport-style scalar counter. Increments on each tree write. Provides a total order within a single peer and a partial causal order across peers (when merged during sync).

```
system/clock/logical := {
  fields: {
    counter: {type_ref: "primitive/uint"}
              ; Monotonically increasing event counter.
              ; Increments on every tree write.
  }
}
```

### 2.3 Vector Clock

Peer-indexed counters for concurrency detection. Each entry tracks the last known counter for a peer. Element-wise max merge during sync.

```
system/clock/vector := {
  fields: {
    entries: {map_of: {type_ref: "primitive/uint"}}
              ; peer_id (string) → counter (uint).
              ; Each peer increments its own entry on tree writes.
              ; Merge takes element-wise max.
  }
}
```

Two vector clocks can be compared to detect concurrency: if neither dominates the other (some entries higher in each), the events are concurrent.

### 2.4 Hybrid Logical Clock

Combines physical time with a logical counter. Provides total order correlated with real time. O(1) size regardless of peer count.

```
system/clock/hlc := {
  fields: {
    physical: {type_ref: "primitive/uint"}
               ; Physical component (ms since epoch).
               ; Always ≥ the local wall clock at time of creation.
    logical:  {type_ref: "primitive/uint"}
               ; Logical component. Disambiguates events with the same physical time.
               ; Resets to 0 when physical advances.
    peer:     {type_ref: "system/hash"}
               ; Content hash of the peer's identity entity.
               ; Breaks ties when physical and logical are equal.
  }
}
```

### 2.5 Clock Configuration

Peer-level clock configuration:

```
system/clock/config := {
  fields: {
    mode:          {type_ref: "primitive/string"}
                    ; "wall" | "logical" | "vector" | "hlc"
                    ; Determines which clock type is maintained.
                    ; Default: "wall"
    wall_clock:    {type_ref: "primitive/bool", optional: true}
                    ; Whether to include wall-clock timestamps alongside
                    ; logical/vector/hlc clocks. Default: true.
                    ; When mode is "wall", this field is ignored (always true).
    tick_interval: {type_ref: "primitive/uint", optional: true}
                    ; Milliseconds between periodic tick events.
                    ; Absent = no periodic ticks.
  }
}
```

Stored at `system/clock/config`.

### 2.6 Clock State

Current clock state for the peer:

```
system/clock/state := {
  fields: {
    mode:      {type_ref: "primitive/string"}
                ; Active clock mode
    timestamp: {type_ref: "system/clock/timestamp", optional: true}
                ; Current wall-clock timestamp (present when wall_clock is true)
    logical:   {type_ref: "system/clock/logical", optional: true}
                ; Current logical clock (present when mode is "logical", "vector", or "hlc")
    vector:    {type_ref: "system/clock/vector", optional: true}
                ; Current vector clock (present when mode is "vector")
    hlc:       {type_ref: "system/clock/hlc", optional: true}
                ; Current HLC (present when mode is "hlc")
  }
}
```

### 2.7 Compare Types

```
system/clock/compare-params := {
  fields: {
    a: {type_ref: "primitive/any"}
        ; First clock value (timestamp, logical, vector, or hlc)
    b: {type_ref: "primitive/any"}
        ; Second clock value (same type as a)
  }
}

system/clock/compare-result := {
  fields: {
    order: {type_ref: "primitive/string"}
            ; "before" | "after" | "concurrent" | "equal"
            ; "before" = a happened before b
            ; "after"  = a happened after b
            ; "concurrent" = a and b are concurrent (vector clocks only)
            ; "equal"  = a and b represent the same clock value
  }
}
```

### 2.8 Tick Event

```
system/clock/tick := {
  fields: {
    sequence: {type_ref: "primitive/uint"}
               ; Monotonically increasing tick counter
    state:    {type_ref: "system/clock/state"}
               ; Clock state at tick time
  }
}
```

Tick entities are written to `system/clock/tick/latest` at each tick interval. The subscription extension fires notifications for subscribers watching this path.

---

## 3. Handler

### 3.1 Handler Manifest

```
system/handler := {
  data: {
    pattern:    "system/clock"
    name:       "clock"
    operations: {
      now:     {output_type: "system/clock/state"}
      compare: {input_type: "system/clock/compare-params", output_type: "system/clock/compare-result"}
      tick:    {input_type: "system/subscription/request"}
    }
  }
}
```

Handler entity at pattern path `system/clock`. Index entry at `system/handler/system/clock`. Handler grant at `system/capability/grants/system/clock`.

### 3.2 now

Returns the current clock state. Reads the peer's clock without advancing it.

```
EXECUTE system/clock  operation: "now"
  resource: {targets: ["system/clock"]}
```

```
handle_now(ctx):
  config = tree.get("system/clock/config") or default_config()
  state = read_current_clock_state(config)
  return {type: "system/clock/state", data: state}
```

```
read_current_clock_state(config):
  state = {mode: config.data.mode}

  if config.data.mode == "wall" or config.data.wall_clock != false:
    state.timestamp = {ms: system_clock_ms()}

  if config.data.mode == "logical":
    state.logical = tree.get("system/clock/logical") or {counter: 0}

  if config.data.mode == "vector":
    state.logical = tree.get("system/clock/logical") or {counter: 0}
    state.vector = tree.get("system/clock/vector") or {entries: {}}

  if config.data.mode == "hlc":
    state.logical = tree.get("system/clock/logical") or {counter: 0}
    state.hlc = tree.get("system/clock/hlc") or {physical: system_clock_ms(), logical: 0, peer: local_peer_id}

  return state
```

### 3.3 compare

Compare two clock values. Both values MUST be the same clock type.

```
EXECUTE system/clock  operation: "compare"
  resource: {targets: ["system/clock"]}
  params: {
    type: "system/clock/compare-params"
    data: {
      a: {ms: 1709000000000}
      b: {ms: 1709000001000}
    }
  }
```

```
handle_compare(ctx, params):
  a = params.data.a
  b = params.data.b

  order = compare_clocks(a, b)
  return {type: "system/clock/compare-result", data: {order: order}}
```

Comparison algorithms are defined in §6.

### 3.4 tick

Creates a subscription for periodic clock events. This is a convenience operation that creates a subscription on the `system/clock/tick/latest` path through the subscription extension.

```
EXECUTE system/clock  operation: "tick"
  resource: {targets: ["system/clock/tick/*"]}
  params: {
    type: "system/subscription/request"
    data: {
      deliver_to:     {uri: "system/inbox/clock-watcher"}
      deliver_token:  <hash>
    }
  }
```

The clock handler delegates to the subscription handler, subscribing to `system/clock/tick/latest`. The tick interval is determined by the peer's clock configuration (§2.5).

---

## 4. Emit Integration

### 4.1 Clock as Emit Pathway Consumer

The clock extension integrates with the emit pathway — the same integration point used by the history extension (EXTENSION-HISTORY.md §5.1) and the subscription extension (EXTENSION-SUBSCRIPTION.md §4.1).

Clock's position in the emit pathway consumer ordering (first emitting consumer, after query indexes) is specified in SYSTEM-COMPOSITION.md §2.2. Clock persistence frequency options (every-emit, periodic, tick-only) are discussed in SYSTEM-COMPOSITION.md §6.2.

The `clock` field on the execution context is an **extension-contributed context field** per SYSTEM-COMPOSITION.md §1.5. CLOCK is the owning extension: it registers the `clock` field at peer initialization (type `system/clock/state`; §2.6), advances the field only during its own consumer position (SYSTEM-COMPOSITION.md §2.2 position 2), and is the authoritative source for current clock state during a cascade. Downstream consumers (HISTORY, optionally COMPUTE and SUBSCRIPTION) read `ctx.clock` by name; access subfields by mode (e.g., `ctx.clock.logical.counter`, `ctx.clock.vector.entries`, `ctx.clock.hlc`). Consumers MUST handle the null case (CLOCK not installed) and the absent-subfield case (CLOCK in a mode that doesn't populate the requested subfield).

On each tree write, the emit pathway advances the clock before other consumers (history, subscriptions) run:

```
emit_entity(path, new_entity, execution_context):
  ; --- Clock advancement (this extension) ---
  if is_clock_engine_path(path):
    ; Clock engine output — do not advance (prevents infinite recursion)
  else:
    advance_clock(execution_context)

  ; --- Normal emit pathway continues ---
  ; Content store put, tree binding update, history recording,
  ; subscription notification — as defined by core protocol and other extensions.
  ; execution_context.clock is available to downstream consumers.

is_clock_engine_path(path):
  ; Guard only engine-written paths, not the entire system/clock/ namespace.
  ; Config paths (system/clock/config) are handler-written and SHOULD advance the clock.
  return path starts with "system/clock/logical"
      or path starts with "system/clock/vector"
      or path starts with "system/clock/hlc"
```

### 4.2 Clock Advancement Algorithm

```
advance_clock(execution_context):
  config = tree.get("system/clock/config") or default_config()
  mode = config.data.mode

  ; Build a system/clock/state value for this cascade step.
  ; Subfields populated per mode; mode field always set.
  state = {mode: mode}

  if mode == "wall":
    state.timestamp = {ms: system_clock_ms()}
    execution_context.clock = state
    return

  ; --- Logical clock (used by all non-wall modes) ---
  current_logical = tree.get("system/clock/logical") or {counter: 0}
  new_counter = current_logical.counter + 1
  new_logical = {type: "system/clock/logical", data: {counter: new_counter}}
  tree.set("system/clock/logical", content_store.put(new_logical))
  state.logical = {counter: new_counter}

  if mode == "vector":
    ; --- Vector clock ---
    current_vector = tree.get("system/clock/vector") or {entries: {}}
    new_entries = copy(current_vector.entries)
    new_entries[local_peer_id] = new_counter
    new_vector = {type: "system/clock/vector", data: {entries: new_entries}}
    tree.set("system/clock/vector", content_store.put(new_vector))
    state.vector = {entries: new_entries}

  if mode == "hlc":
    ; --- Hybrid logical clock ---
    current_hlc = tree.get("system/clock/hlc") or {physical: 0, logical: 0, peer: local_peer_id}
    new_hlc = hlc_local_event(current_hlc)
    tree.set("system/clock/hlc", content_store.put({type: "system/clock/hlc", data: new_hlc}))
    state.hlc = new_hlc

  if config.data.wall_clock != false:
    state.timestamp = {ms: system_clock_ms()}

  execution_context.clock = state
```

### 4.3 Clock State Writes Do Not Advance the Clock

Writing to clock engine output paths (`system/clock/logical`, `system/clock/vector`, `system/clock/hlc`) MUST NOT trigger clock advancement. This prevents infinite recursion — clock persistence writes are produced by the clock consumer itself during emit processing.

Configuration paths (`system/clock/config`) are handler-written via explicit EXECUTE and do not create recursion risk. Configuration changes SHOULD advance the clock like any other tree mutation. See SYSTEM-COMPOSITION.md §6.1 for the engine-output vs configuration path distinction.

---

## 5. Sync Integration

### 5.1 Clock Exchange During Sync

During sync (EXTENSION-REVISION.md §7.1), peers exchange clock state alongside version heads. The clock merge happens after version fetch and before version integration.

```
sync_clocks(local_clock_state, remote_clock_state, mode):
  if mode == "logical":
    merge_logical(local_clock_state.logical, remote_clock_state.logical)
  elif mode == "vector":
    merge_logical(local_clock_state.logical, remote_clock_state.logical)
    merge_vector(local_clock_state.vector, remote_clock_state.vector)
  elif mode == "hlc":
    merge_logical(local_clock_state.logical, remote_clock_state.logical)
    merge_hlc(local_clock_state.hlc, remote_clock_state.hlc)
```

### 5.2 Logical Clock Merge

```
merge_logical(local, remote):
  merged_counter = max(local.counter, remote.counter)
  new_logical = {type: "system/clock/logical", data: {counter: merged_counter}}
  tree.set("system/clock/logical", content_store.put(new_logical))
```

After merge, the next local event increments from the merged value, ensuring that post-sync events have a counter greater than all events from both peers.

### 5.3 Vector Clock Merge

```
merge_vector(local, remote):
  merged_entries = copy(local.entries)
  for (peer_id, counter) in remote.entries:
    if peer_id not in merged_entries or counter > merged_entries[peer_id]:
      merged_entries[peer_id] = counter
  new_vector = {type: "system/clock/vector", data: {entries: merged_entries}}
  tree.set("system/clock/vector", content_store.put(new_vector))
```

### 5.4 HLC Merge

```
merge_hlc(local, remote):
  ; Standard HLC receive algorithm (Kulkarni et al.)
  wall = system_clock_ms()
  new_physical = max(wall, local.physical, remote.physical)

  if new_physical == local.physical and new_physical == remote.physical:
    new_logical = max(local.logical, remote.logical) + 1
  elif new_physical == local.physical:
    new_logical = local.logical + 1
  elif new_physical == remote.physical:
    new_logical = remote.logical + 1
  else:
    new_logical = 0    ; physical advanced past both — reset logical

  new_hlc = {
    type: "system/clock/hlc"
    data: {physical: new_physical, logical: new_logical, peer: local_peer_id}
  }
  tree.set("system/clock/hlc", content_store.put(new_hlc))
```

### 5.5 Version Clock Field

Version entities (EXTENSION-REVISION.md §2.1) MAY include the clock value at commit time. This is an optional field on `system/revision/entry` via open types (ENTITY-CORE-PROTOCOL.md §2.7):

```
clock: {type_ref: "primitive/any", optional: true}
```

**Extends**: `system/revision/entry`

**When present**: Contains the clock value (`system/clock/logical`, `system/clock/vector`, or `system/clock/hlc`) at the time the version was created. Enables causally-aware version comparison — the `lww` merge strategy (EXTENSION-REVISION.md §2.3) can use the clock field instead of the wall-clock `timestamp` field.

**When absent**: Version ordering uses the `timestamp` field (wall-clock). This is the default behavior without the clock extension.

---

## 6. Clock Algorithms

### 6.1 Lamport Increment

```
lamport_increment(current):
  return {counter: current.counter + 1}
```

### 6.2 HLC Local Event

Advance the HLC for a local event (tree write):

```
hlc_local_event(current_hlc):
  wall = system_clock_ms()
  new_physical = max(wall, current_hlc.physical)

  if new_physical == current_hlc.physical:
    new_logical = current_hlc.logical + 1
  else:
    new_logical = 0    ; physical advanced — reset logical

  return {physical: new_physical, logical: new_logical, peer: local_peer_id}
```

### 6.3 HLC Receive

Advance the HLC on receiving a remote clock value (sync merge). Defined in §5.4.

### 6.4 Clock Comparison

#### 6.4.1 Timestamp Comparison

```
compare_timestamps(a, b):
  if a.ms < b.ms: return "before"
  if a.ms > b.ms: return "after"
  return "equal"
```

#### 6.4.2 Logical Clock Comparison

```
compare_logical(a, b):
  if a.counter < b.counter: return "before"
  if a.counter > b.counter: return "after"
  return "equal"
```

#### 6.4.3 Vector Clock Comparison

Standard vector clock partial order: `a ≤ b` iff `∀i: a[i] ≤ b[i]`. Strict ordering: `a < b` iff `a ≤ b` and `a ≠ b`. When neither `a ≤ b` nor `b ≤ a`, the events are concurrent.

```
compare_vector(a, b):
  all_peers = union(keys(a.entries), keys(b.entries))
  a_leq_b = true
  b_leq_a = true
  equal = true

  for peer_id in all_peers:
    a_val = a.entries.get(peer_id) or 0
    b_val = b.entries.get(peer_id) or 0
    if a_val > b_val:
      a_leq_b = false
      equal = false
    if b_val > a_val:
      b_leq_a = false
      equal = false

  if equal: return "equal"
  if a_leq_b: return "before"     ; a happened before b
  if b_leq_a: return "after"      ; a happened after b
  return "concurrent"              ; neither dominates
```

#### 6.4.4 HLC Comparison

```
compare_hlc(a, b):
  if a.physical < b.physical: return "before"
  if a.physical > b.physical: return "after"
  if a.logical < b.logical: return "before"
  if a.logical > b.logical: return "after"
  ; Physical and logical equal — compare peer identity for total order
  if a.peer < b.peer: return "before"
  if a.peer > b.peer: return "after"
  return "equal"
```

HLC comparison produces a total order — never "concurrent". The peer identity tiebreaker is deterministic but arbitrary.

---

## 7. Security Considerations

### 7.1 Clock State Access

Clock state at `system/clock/*` is local peer state. Access follows standard capability scoping — the clock handler grant controls who can read clock values via the `now` operation. Clock state entities in the tree are accessible to any capability that covers the `system/clock/` prefix.

### 7.2 Wall-Clock Trust

Wall-clock timestamps are untrusted across peers. A peer can report any value for `system_clock_ms()`. This is an acknowledged limitation, not something the clock extension solves. Defenses:

- **Logical and vector clocks** are trustworthy within honest peers — they increment monotonically and merge correctly. A dishonest peer can lie about its counter, but this only affects ordering decisions that include that peer's events.
- **HLC bounds** — the HLC physical component MUST NOT advance more than `MAX_HLC_DRIFT_MS` beyond the local wall clock (§8). This bounds the damage from a peer with a skewed clock. Implementations SHOULD reject received HLC values where `physical` exceeds `system_clock_ms() + MAX_HLC_DRIFT_MS`.
- **For high-trust ordering**, use logical or vector clocks rather than wall-clock timestamps. The `lww` merge strategy with HLC clock field provides causally-correct ordering without relying on synchronized wall clocks.

### 7.3 Vector Clock Growth

Vector clock entries grow with the number of peers that have written to the tree. For systems with many transient peers, the vector clock may grow unboundedly. Implementations SHOULD prune entries for peers that are no longer active (no writes for a configurable period). Pruning loses ordering information for pruned peers but bounds storage cost.

The `MAX_VECTOR_ENTRIES` constant (§8) provides a hard bound. When the limit is reached, the oldest entry (lowest counter) is evicted.

### 7.4 Tick Subscriptions

Tick subscriptions respect the subscription extension's capability model (EXTENSION-SUBSCRIPTION.md §9). A tick subscription requires capability for the `system/clock` handler with the `tick` operation and a valid deliver token for the inbox URI.

---

## 8. Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `DEFAULT_TICK_INTERVAL_MS` | 1000 | Default tick interval in milliseconds |
| `MAX_VECTOR_ENTRIES` | 1024 | Maximum entries in a vector clock before eviction |
| `MAX_HLC_DRIFT_MS` | 60000 | Maximum allowed HLC physical drift from wall clock (ms) |
| `DEFAULT_CLOCK_MODE` | `"wall"` | Default clock mode when no configuration is present |

---

## 9. Write Authorization

Clock handler operations (now, compare, tick) are read-only — no tree writes.

Clock advancement (§3.3) is an autonomous background operation. Writes to `system/clock/*` paths are authorized by the clock handler's own grant. The local peer identity is the author.

Note: `system/clock/*` paths are not excluded from history recording by the history spec (the recursion prevention in EXTENSION-HISTORY.md §3.2 only covers `system/history/*`). If history is configured for `system/clock/*` paths, advancement writes will be recorded with the clock handler grant as the authorizing capability.

See ENTITY-CORE-PROTOCOL.md §6.8 for the general write authorization model.

---

## 10. Conformance

### 10.1 MUST Implement

- `system_clock_ms()` function returning wall-clock milliseconds since Unix epoch (§2.1)
- Clock handler at `system/clock` with `now` operation (§3.2)
- `compare` operation for timestamp values (§3.3, §6.4.1)
- Logical clock increment on tree writes when mode is `"logical"`, `"vector"`, or `"hlc"` (§4.2)
- Exclusion of `system/clock/*` paths from clock advancement (§4.3)
- HLC physical drift bounded by `MAX_HLC_DRIFT_MS` (§7.2)

### 10.2 SHOULD Implement

- Vector clock merge during sync when mode is `"vector"` (§5.3)
- HLC merge during sync when mode is `"hlc"` (§5.4)
- Logical clock merge during sync when mode is `"logical"` (§5.2)
- Vector clock comparison with concurrency detection (§6.4.3)
- HLC comparison with total ordering (§6.4.4)
- Vector clock entry pruning for inactive peers (§7.3)
- Rejection of received HLC values exceeding drift bound (§7.2)

### 10.3 MAY Implement

- Tick subscription via `tick` operation (§3.4)
- `clock` field on `system/revision/entry` entities (§5.5)
- Clock state available in `emission_context` for downstream consumers (§4.2)
- `wall_clock` configuration for dual timestamps with causal clocks (§2.5)

### 10.4 Implementation-Defined

- Wall-clock source (host system call, monotonic variant)
- Tick scheduling precision and jitter tolerance
- Vector clock entry eviction policy (age-based, count-based)
- Clock state persistence across peer restarts
- Logical clock initial value on first boot

### 10.5 Types Installed

```
system/clock/timestamp
system/clock/logical
system/clock/vector
system/clock/hlc
system/clock/config
system/clock/state
system/clock/compare-params
system/clock/compare-result
system/clock/tick
```

### 10.6 Handler Registered

```
system/handler/system/clock := system/handler {
  pattern:    "system/clock"
  name:       "clock"
  operations: {now, compare, tick}
  ; See §3.1 for full operation-spec map with input/output types
}
```
