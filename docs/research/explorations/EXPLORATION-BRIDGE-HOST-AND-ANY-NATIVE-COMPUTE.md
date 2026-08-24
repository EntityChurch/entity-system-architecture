# EXPLORATION — the bridge host & any-native compute (byte streams, FFI, and an entity core that runs as a compute program)

**Status:** Exploration / design — 2026-07-20. **NOT** a proposal, **NOT** ratified. Feeds a future
`proposals/` entry once the transport-shape ABI and the import-provider contract are pinned against a
built bridge host. Author: arch workspace at the operator's request.

**Sibling tracks.** This doc is the **runtime seam** of the stack — how the descriptor-driven generic host
reaches *reality* (sockets, crypto, codecs, the clock). It sits between the two synthesis docs:
- `EXPLORATION-MACHINE-BOUNDARY-AND-FULL-STACK-TRAJECTORY.md` (the **bottom**: how little substrate is
  needed; the determinism boundary as `compute/apply`) — this doc makes that boundary **concrete on the
  runtime side**: it is the exact line a byte-stream driver and a crypto import sit on.
- `EXPLORATION-L5-APP-HOSTING-UNIFICATION.md` (the **top**: what runs where, in whose peer) — this doc
  supplies the piece that top doc's "full WASM entity peer" payload (①c) needs to actually *be a peer*:
  a peer with no network I/O is not a peer, and the generic host today has no network seam.

**The question this doc answers** (the operator's, stated precisely): *we have input/output ports on the
generic host; **what does a generic host that supports a TCP bridge / a byte stream look like?** Is it
custom handlers with bridges? FFI? And running on what?* The short answer, developed below: it is **the
same generic host plus a driver/import profile on the far (native) side of the determinism boundary** —
byte flow rides the existing **stream-port** mechanism, transforms and connection control ride the existing
**`imports`** mechanism, and both are matched by the existing **`offered`/admission** contract. Almost
nothing is a new mechanism; the new work is a **transport-shape ABI** and an **import-provider contract**,
plus honestly naming what is unbuilt.

---

## §0 TL;DR

- **The determinism boundary is the whole design.** Entity-compute is a deterministic CPU tree-walk
  (`compute/apply`, `MAX_DEPTH`-bounded, `MaxOps`-budgeted). Anything that touches reality — a socket, the
  clock, a hardware RNG, a GPU — is **non-deterministic** and MUST live on the **native side** as a driver
  or an import. A "TCP bridge" is not a compute change; it is native code the host binds *around* a pure
  step. This is exactly Paper 04's Category-C native-handler boundary, reached from the runtime side
  (`MACHINE-BOUNDARY §3c`).
- **Two mechanisms already exist; the design is assigning capabilities to them.**
  - **Byte flow = stream ports.** A `kind:"stream"` port is an inbox/subscription — "ordered, lossless,
    backpressured" (`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §3`, which already names *"send this packet"*
    as output-port intent). A TCP connection is a **bidirectional pair of stream ports**; a native
    **byte-stream driver** owns the socket and shuttles bytes across the boundary. This is a new **I/O
    lineage — `transport`** — added to the `(role, shape)` vocabulary the same way `display-list` was.
  - **Transforms + control = imports.** `imports: [app/program/import]` already exists as "capability
    handlers the program calls," matched at admission against `system/device/host/offered`
    (`§5` + `PROPOSAL-SYSTEM-DEVICE §7`). SHA-256, Ed25519, CBOR encode/decode, and connection lifecycle
    (`connect`/`listen`/`accept`/`close`) are **import handlers** — the FFI bridges. This is the operator's
    "custom handlers with bridges / FFI," and it is already the spec'd seam.
- **The bridge host = the generic host with a transport+crypto+codec provider profile.** "Bridge host" is
  the native-provider side; "generic host" is the descriptor-driven mount loop. They are one binary; the
  bridge is the set of drivers/imports it *offers*.
- **An entity core can run as a compute program on the bridge host** — this is what "write an entity core in
  any-native compute" means concretely. A peer's core is `state₀ + step` (state = tree + connection table;
  step = protocol-handler dispatch); the reality-touching bits (TCP, SHA-256, Ed25519, CBOR) are the bridge
  host's imports/drivers. The dizzying recursion (a host running a program that is a core that hosts a
  program…) is **tamed by the determinism boundary**: at every level the pure core is compute and the
  reality is a native bridge.
- **Running on what:** **workbench-go first** (has compute + trivial stdlib TCP/crypto/CBOR), **entity-
  browser-rust second** (has compute; but the browser cannot do raw TCP — its bridge offers WebSocket/WebRTC
  byte-streams only, and cleanly *refuses* raw-TCP programs via admission). **Python** is a third viable
  compute host. **Keystone peers cannot host** — they have core-protocol but **no `EXTENSION-COMPUTE`**, so
  they can be *reached* as peers but cannot *mount* a compute program.
- **Honest status: almost none of this is built.** The mechanisms are spec'd; the transport shape, the
  import-provider contract, and every byte-stream/crypto driver are **unbuilt**. `imports` has a descriptor
  slot and an admission rule but **zero providers**. This doc is the design, not a report.

---

## §1 The determinism boundary, made concrete (the load-bearing line)

The generic host mount loop is (from `EXPLORATION-GENERIC-HOST-AND-DESCRIPTOR §2`, as amended 07-17):

```
mount(descriptor):
  seed  input ports
  bind  output-driver by (role, shape) ← reads port path
  bind  input-driver  by (role, shape) → writes port path
  loop at tick.rate_hint:  eval(step) → put(state_path); refresh output ports
```

`eval(step)` is **pure and deterministic** — same inputs, byte-identical state hash on any conformant
engine (proven Life/Snake/Asteroids vs the Go oracle). Everything a bridge adds — a socket read, a signed
handshake, the wall clock — is **non-deterministic** and therefore CANNOT be inside `eval`. It must be
either:

- a **driver** bound to a port (it runs *between* ticks, filling/draining port state), or
- an **import** the step dispatches to via the `compute/apply` seam (it runs *during* a tick but its result
  is treated as an opaque input, not re-derived).

This is not a restriction the bridge imposes; it is the property that lets a program **save-state, replay
deterministically, and transfer by hash** (`§3`: "the program never performs an effect; it emits effect
*intent* as output-port state; a driver actuates it"). The bridge host's entire job is to be the honest
native executor of those intents while keeping the core pure. Stated as the invariant:

> **B1 — Nothing that touches reality is entity-compute.** Sockets, clocks, RNG, GPU, and secret keys live
> on the native side as drivers/imports. `eval(step)` sees only their *results as declared port/import
> values*, never their execution. The determinism boundary = `compute/apply` = the CPU/GPU line =
> Paper 04 Category-C. (`MACHINE-BOUNDARY §3c`, `GUIDE-CORE §5`.)

### §1a Two reasons a thing is native — keep them distinct

Not all native imports are native for the same reason, and the difference matters for what's portable:

| Reason | Examples | Nature | Portability consequence |
|---|---|---|---|
| **Native by *necessity*** (touches reality/secrets/time) | TCP send/recv, `accept`, wall clock, hardware RNG, `Ed25519.sign` (needs the host's secret key) | genuinely non-deterministic or effectful | MUST be a driver/import; can never move into compute |
| **Native by *choice*** (deterministic, but FFI'd for speed) | `SHA-256(bytes)→hash`, `CBOR.encode/decode`, `Ed25519.verify(msg,sig,pk)→bool` | pure functions of their inputs | *could* be pure compute (Unison peer hand-rolled Ed25519 — `MACHINE-BOUNDARY §3b`); FFI'd only because a tree-walk SHA-256 is absurd |

The "by choice" row is why the 43-peer cohort's *no-C-FFI* result holds: those functions are in the floor
as **semantics**, not as **FFI** — a peer with no FFI implements them slowly in-language; a bridge host
FFIs them for performance. **The wire and hashes are byte-identical either way** — that equivalence is what
makes a bridge host and a pure peer interoperate. (A useful conformance handle: a bridge host's FFI'd
SHA-256 MUST agree bit-for-bit with the reference tree-walk SHA-256, or content addressing forks.)

### §1b The necessity/choice split IS the builtin/drop-down split — and wasm32 proves it (verified)

The distinction is not academic; it lands on two *different code paths* in the evaluator, and the browser
forces the issue. Verified in `entity-core-rust`:

- **`compute/apply` handler-mode is the drop-down/import seam.** With a `path`+`operation` (no inline
  `fn`) it leaves the evaluator through a `dispatch_execute` callback into a native handler
  (`eval_apply_handler`, `extensions/compute/src/eval/apply.rs`; `DispatchExecuteFn`,
  `eval/mod.rs`). **On wasm32 this callback is stubbed to
  `InvalidExpression("Handler dispatch not available in WASM context")`** (`compute/src/lib.rs`,
  `build_dispatch_execute`, `#[cfg(target_arch="wasm32")]`) — because the native handler is `async` and the
  bridge is `block_in_place`+`block_on`, which has **no equivalent on the single-threaded browser event
  loop** (you cannot block to await). So in a browser, **any import reached via `compute/apply` returns an
  error value.**
- **But `system/compute/builtins/*` are intercepted *before* dispatch and run in-process on *all* targets**
  (`is_builtin_path`/`dispatch_builtin_alias`, `eval/apply.rs`) — they never touch the wasm32 stub.

> **B0 (design ruling that falls out) — pure imports MUST be *builtins*, not drop-downs.** The
> **native-by-choice** set (`crypto/sha-256`, `crypto/ed25519/verify`, `codec/cbor/*`) is pure, so it
> belongs on the **builtin** path (`system/compute/builtins/*`) — which runs everywhere, **including
> wasm32/browser.** The **native-by-necessity** set (`transport/*`, `crypto/ed25519/sign`, `clock/now`) is
> the only set that genuinely needs the `compute/apply` drop-down — and that set is exactly what a browser
> **cannot do anyway** (no raw TCP; the host key is native). So the wasm32 stub is **not a blocker for a
> browser bridge host's *pure* needs** (make them builtins) and is a **non-issue for its *impure* needs**
> (the browser can't serve them regardless — it offers `byte-stream{ws}` and defers `sign` to a native
> co-peer). The stub only bites a design that wrongly ships SHA-256/CBOR as drop-downs. *(Caveat: this
> assumes those functions exist as blessed builtins; today that's an ask on the compute-standardization
> track — SHA-256/CBOR as builtins — not a given. Route it.)* Separately note: on wasm32 the recursion
> stack-guard (`stacker::maybe_grow`) is compiled out while `MAX_DEPTH` stays 1024
> (`compute/src/eval/mod.rs`, `types.rs`), so a deep non-tail policy spine has a fixed browser stack — keep
> the wire-serving step shallow.

---

## §2 Byte flow is a stream-port pair (the dataplane)

A TCP connection carries two byte streams (in, out). Map each to a `kind:"stream"` port — the inbox/
subscription mechanism that is "ordered, lossless, backpressured" (`§3`). The step reads received bytes
from an **input** stream port and writes bytes-to-send to an **output** stream port; a native **byte-stream
driver** owns the socket and moves bytes across the boundary each tick (or reactively).

```
program (pure)                          bridge host (native)
  in-port  ⟵ (bytes received)  ⟵  byte-stream driver ⟵ socket.recv()
  out-port ⟶ (bytes to send)   ⟶  byte-stream driver ⟶ socket.send()
```

**Why determinism survives a socket.** Within a tick the step sees a *fixed* input — the bytes the driver
had already delivered to the inbox before this tick's `eval` — and produces a *fixed* output. The
non-determinism (when bytes arrive, how the socket is doing) is entirely in the driver, outside `eval`.
Replay a `(state₀, recorded-inbox-stream)` and you get the same state-hash sequence — the same property
Snake's scripted-replay test already proves, now with packets instead of keypresses. The inbox's
lossless/ordered/backpressured semantics are exactly a reliable byte stream's, which is why `stream` (not
`snapshot`) is mandatory here: you may not drop a byte.

### §2a The transport shape — a new I/O lineage in the `(role, shape)` vocabulary

The generic-host exploration grounds shapes in the **lineage of computer I/O** and makes the set
**extensible by a rule** (`§4`/`§6`): name the device class, pin what the port carries + its `scene`,
define what a driver does. The I/O lineages so far are output {text, framebuffer, display-list, audio} and
input {key, pointer, event}. **Transport is the missing lineage** — the network interface, as fundamental
as the teletype:

| lineage | classic device | `role` | `shape` | port `kind` | carries (`type_ref`) | `scene` fields |
|---|---|---|---|---|---|---|
| **transport (reliable)** | NIC / socket | `transport` | `byte-stream` | `stream` | `primitive/bytes` (chunk) | `protocol` (tcp\|ws\|webrtc-dc\|serial\|message-port), `endpoint`, `conn-role` (client\|listener) |
| **transport (datagram)** | UDP / packet radio | `transport` | `datagram` | `stream` | `primitive/bytes` + peer-addr | `protocol` (udp\|…), `mtu?` |

**Framing is NOT a `scene` field — it is per-transport, and already pinned.** Do not invent a framing hint:
`ENTITY-CORE-PROTOCOL §1.6` states *"framing is per-transport — other transports define their own
framing,"* and `EXTENSION-NETWORK §6.5` fixes each one — **TCP** carries the **4-byte length prefix**
(a byte-oriented medium with no message boundaries; the driver reads the prefix), **HTTP** frames natively
(Content-Length/chunked; the §1.6 prefix MUST NOT be applied), **WebSocket** reuses the §1.6 prefix per its
V7 v7.13 blessing, and an in-browser **`message-port`** is message-oriented (one postMessage = one frame).
So the byte-stream driver **delivers whole entity-protocol frames** to the inbox regardless of transport;
*how* it finds a frame boundary is the driver's per-transport concern, following the transport's blessed
framing — not a program- or descriptor-level choice. This is the message-vs-byte distinction handled once,
in the place the spec already put it.

Key properties, consistent with how `display-list` was added:
- **The shape is the ABI; the driver is per-host.** Every bridge host implements the *same* `byte-stream`
  shape; Go binds it to `net.Conn`, browser-rust binds it to a `WebSocket`/`RTCDataChannel`, a native Rust
  host to `tokio::net::TcpStream`. A program declares `shape:"byte-stream"` and never ships a driver — the
  driver-level analog of "the IR is the ABI" (`GENERIC-HOST §4`).
- **`protocol` is a `scene` field, not a shape.** TCP vs WebSocket vs WebRTC-datachannel vs a serial line
  are all `byte-stream`; they differ in the driver, declared in `scene.protocol`. This keeps the shape set
  small and lets **admission** do the matching: a browser bridge host offers `byte-stream{protocol:ws}` and
  **cleanly refuses** a program needing `byte-stream{protocol:tcp}` — "never a half-render" (`GENERIC-HOST
  §6`), applied to transport. Raw-TCP-in-the-browser is thus not a bug to fix but a capability the browser
  host honestly does not offer.
- **Datagram is a distinct shape, not a `scene` flag** — losslessness/ordering differ at the contract
  level (a `datagram` port MAY drop/reorder; a `byte-stream` port MUST NOT), and that difference is
  cross-peer-observable, so it is pinned as a shape per the house rule (pin the observable seam;
  `AGENTS.md` load-bearing invariants).

### §2b Connection lifecycle — where does open/accept/close live?

A byte-stream port is a *connected* stream; something must connect it. Two candidate owners, and the
resolution:

- **Run-state (host verbs).** `start/stop/restart/reseed` are already "runtime verbs outside the program"
  (`§5`). A *client* connection whose endpoint is fixed in the descriptor's `scene.endpoint` can be opened
  by the host at mount (like seeding an input port) and closed at unmount — **no program involvement**.
  This covers the common case (a program that talks to one known peer).
- **Imports (program-driven).** A program that opens connections *dynamically* (a server that `accept`s
  many, a peer that dials addresses it learns at runtime) cannot have them baked in `scene`. It calls a
  **connection-control import** — `transport/connect(endpoint) → conn-id`, `transport/listen`,
  `transport/accept → conn-id`, `transport/close(conn-id)` — and the host materializes a byte-stream port
  per live `conn-id` under a `fragment_base`-style path the program reads. This is `imports`, i.e. the FFI
  bridge, used for control while the bytes still flow over stream ports.

**Ruling-candidate:** static single-endpoint connections are **run-state** (host-opened, descriptor-
declared); dynamic/many connections are **imports** (program-opened, capability-gated). A pure tick-loop
game needs neither; a *peer* needs the import form (it accepts inbound and dials outbound). This is the
first real forcing function that pushes the design past games.

### §2c Connection identity must be *logical*, not a native handle (the replay leak — B2)

The sharpest determinism edge case, and it is easy to get wrong: a `conn-id` is part of the program's
**state** (the connection table, §4), so it is hashed and replayed. A native socket handle
(`net.Conn` pointer, an fd number, a `RTCDataChannel` object) is **non-deterministic** — it changes every
run and differs across hosts. Putting it in state **forks the state hash** and breaks replay/transfer.

> **B2 — the connection table is keyed by the *authenticated remote peer-id*, not by a native handle.** The
> logical connection identifier is the **remote peer-id established by the HELLO handshake** — which is
> deterministic and verified *in-compute* (B5), so it is safe to hash and replay. The host keeps a side
> table `remote-peer-id → native socket`; that table is **not** part of program state and is **not** hashed.
> This *matches the reference impl exactly*: `entity-core-rust` tracks connections in
> `HashMap<peer_id_string, endpoint>` (`RemoteState.conns`/`inbound`, `core/peer/src/remote.rs`), addressed
> by remote peer-id with **no numeric handle** — verified. A native socket handle would fork the state hash
> and MUST never enter state (the file-descriptor-table shape, program side deterministic).

Two consequences of keying by remote peer-id (both verified against the impl):
- **A provisional id is needed only during the pre-`Established` handshake window.** Before HELLO+
  authenticate completes, the remote peer-id is unknown (`ConnectionState: AwaitingHello →
  AwaitingAuthenticate → Established`, `core/protocol/src/connect.rs`), so a mount-local provisional id
  carries the half-open connection until it's promoted to peer-id keying on `Established`. After that, the
  peer-id is the stable key.
- **One connection per remote peer-id per peer — so isolation is *peer*-granular.** The impl holds at most
  one live inbound + one pooled outbound per remote peer-id (a second inbound overwrites the map entry,
  `remote.rs`). That means two hosted app-peers talking to the same inter-peer X **cannot share one host
  connection table** — each must be its **own peer with its own `RemoteState`** (its own identity, its own
  connection pool). This is not a limitation to route around; it is *why* an isolation unit that speaks the
  wire is a full peer, not a sub-thing sharing the host's table (`PROPOSAL-SUB-PEER-ISOLATION-MODEL §4a`).

Two consequences that fall out of B2:
- **Cross-port ordering is descriptor-declared, not arrival-order.** Within a tick a step reads a fixed
  snapshot of each byte-stream inbox (each internally ordered — the inbox guarantee, §2). *Across* ports,
  "which socket delivered first globally" is non-deterministic, so the step MUST process ports in
  **descriptor-declared order**, never wall-clock arrival order. (Same discipline as Asteroids' multiple
  heterogeneous output ports; now load-bearing for replay.)
- **A peer that accepts inbound is *event-driven*, and the cascade rule bites here.** A wire-serving core
  advances when a frame arrives, so it is `tick.mode = event-driven` — a reactive install keyed on the
  **byte-stream input port**, not on `state_path`. `PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §4`'s cascade
  rule is exactly the trap: the step reads *and writes* `state_path`, so the reactive install MUST be keyed
  on the **input port** (the inbound frame), and the step's writes to `state_path` MUST NOT be a reactive
  trigger (only output-port projections are reactive on state, and they never feed back). Get this wrong and
  every handled frame re-triggers to `cascade_limit`. Pin: **event-driven wire-serving keys reactivity on
  the transport input port; state writes are host-clocked/step-internal, never a reactive source.**

---

## §3 Transforms and control are imports (the controlplane / FFI)

`imports: [app/program/import]` is the descriptor's WIT-world analog — "the capability handlers the
program's drop-downs require, matched against the peer's offered capabilities at admission" (`§2`/`§5`).
Today it is a declared slot **with no providers and no vocabulary**. The bridge host is what fills it. The
first import set — the minimum to run a *peer* as compute:

| import | signature (semantics) | native reason (§1a) | notes |
|---|---|---|---|
| `crypto/sha-256` | `bytes → hash` | by choice (perf) | MUST equal the reference tree-walk hash bit-for-bit (content addressing) |
| `crypto/ed25519/verify` | `(msg, sig, pubkey) → bool` | by choice (perf) | pure; verifies a peer's signature |
| `crypto/ed25519/sign` | `(msg) → sig` | **by necessity** | signs with the *host's* secret key — the key never enters compute; this is a capability, deny-by-default |
| `codec/cbor/encode` | `value → bytes` | by choice (perf) | canonical ECF; MUST be bit-identical canonical form (the re-encode interop hazard — `AGENTS.md`); **flag the known cbor2 float16-minimization Rule-4 gap** before any float-carrying entity ships through a Python bridge host |
| `codec/cbor/decode` | `bytes → value` | by choice (perf) | MUST-ignore unknowns (ADR-0002) |
| `transport/connect` … `close` | connection control (§2b) | by necessity | returns/consumes `conn-id`; gates a byte-stream port |
| `clock/now` | `→ timestamp` | by necessity | non-deterministic; a program that reads it is not replayable *at that read* — flag it |

Two design rulings fall out:

- **A pure program routes effects through ports; a service program MAY use impure imports** (`§3` already
  carves this: "an impure `compute/apply` drop-down remains available for service work that doesn't need
  replay"). Crypto-verify/CBOR are pure (replay-safe); `sign`/`connect`/`clock` are impure (replay
  boundaries). The descriptor should **mark impure imports** so a host knows a program's replay is only
  valid up to its first impure call. (Open — §6.) **Purity also decides the code path (B0):** the pure rows
  above are **builtins** (`system/compute/builtins/*`, run everywhere incl. wasm32); only the impure rows
  are **`compute/apply` drop-downs**. So in the table, `sha-256`/`verify`/`cbor/*` are builtin-track;
  `sign`/`transport/*`/`clock` are drop-down-track.
- **The import vocabulary is a registry extended by a rule, exactly like shapes** (`GENERIC-HOST §6`): name
  the capability, pin its signature, define the provider contract, register it in `offered`. `crypto/*`,
  `codec/*`, `transport/*` are the first three families; `storage/*` (durable put beyond the tree),
  `random/*`, `gpu/*` (as a native drop-down, never compute) come the same way.

### §3a Admission ties it together — `offered` is the bridge's manifest

The seam is already load-bearing and already has one reader (`PROPOSAL-SYSTEM-DEVICE §7`,
`ANALYSIS-SUBSTRATE-CONVERGENCE §2`): `system/device/host/offered` is "the set of host interfaces /
capability providers this peer can supply," and it **MUST advertise only currently-dispatchable handlers
(`discover_handlers`-backed), never aspirational hostability."** A bridge host's `offered` is therefore its
**honest capability manifest** — the shapes it can drive *and* the imports it can dispatch:

```
system/device/host/offered  ⊇  { shape:byte-stream{tcp}, shape:byte-stream{ws}, shape:datagram{udp},
                                  import:crypto/sha-256, import:crypto/ed25519/verify,
                                  import:codec/cbor/*, import:transport/*, import:clock/now }
```

Admission = match a program's declared `shapes` + `imports` against `offered`; a program the host cannot
serve **is not admitted, and says why** (deny-by-default, the WIT-world/policy-allowlist check). This is
the same machinery for a browser host refusing raw TCP and for a headless host refusing `display-list` —
**one admission rule over one `offered` set**. The bridge is not a new subsystem; it is more entries in
`offered`.

**But admission is coarse — it gates *capability class*, not *reach* (B3, a security edge).** Matching
`import:transport/connect` against `offered` only proves "this host can open TCP." It does **not** bound
*which endpoints* a program may dial — and a hosted, possibly-untrusted app-peer with an unrestricted
`transport/connect` can **phone home anywhere.** So reach is a **second, finer gate**: `transport/connect`
is capability-gated **per endpoint (or per endpoint-set / per named peer-id)**, granted at mount by the
host, enforced at the connect call — a connect to an ungranted endpoint is **denied**, not just unadmitted.
Mount-time admission answers "can this host do TCP at all"; the runtime grant answers "to whom." This is
the same deny-by-default capability boundary the sub-peer isolation model mints
(`PROPOSAL-SUB-PEER-ISOLATION-MODEL §4`) — the byte-stream a hosted app gets is a grant to *one named
inter-peer*, not the open internet. Listener imports (`transport/listen`/`accept`) are gated the same way
(which interface/port a program may bind).

---

## §4 The recursion, tamed — an entity core as a compute program

"We're approaching where we could start to attempt writing an entity core in any-native compute." Here is
what that is, structurally, and why the layering is not circular.

A peer's **core** is a program: `state₀ + step`.
- **state** = the peer's tree (`path → hash`, content-store-deduped) **+** a connection table (the live
  `conn-id`s and their read cursors).
- **step** = the protocol handler dispatch: read the next inbound frame (from a byte-stream input port),
  `codec/cbor/decode` it (import), check the capability (pure compute), run the handler
  (`get`/`put`/`execute` — pure compute over the tree), `codec/cbor/encode` the reply (import),
  `crypto/ed25519/sign` it if needed (import), write it to a byte-stream output port. That is a peer
  serving the wire, expressed as one deterministic step over impure imports.

This lands **Paper 04's four fixed points** on concrete mechanisms (`MACHINE-BOUNDARY §3a`):

| Paper-04 fixed point | mechanism here |
|---|---|
| tree writes | `put(state_path)` — the step's output |
| cross-peer exchange | byte-stream ports + `codec/cbor/*` + `crypto/*` imports |
| capability checks | pure compute in the step |
| handler transitions | the `eval(step)` itself |

And **the seven machine primitives** split cleanly across the boundary: byte-ops / string-compare /
integer-arithmetic / allocation are *inside* compute (pure); **SHA-256, CBOR, and I/O are the bridge's
imports/drivers** (native). That split *is* the machine boundary — the generic host is the deterministic
side, the bridge host is the seven-primitives-that-touch-reality side.

**Honesty refinement — "core as compute" is the *policy/dispatch spine* in compute, not every handler as a
tree-walk (B4).** Two forces push the same way. (1) *Budget:* a step evals under the ~100k-op `MaxOps`
ceiling (`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §4`); a heavy handler (a big `put`, a broad query) would
blow it, and a protocol handler is not obviously shardable the way a `map` is. (2) *Determinism:* a tree
write and a store read **touch reality** — they are Paper-04 fixed points / Category-C, i.e. native by
B1 anyway. So the realistic decomposition of an entity-core-as-program is: **framing, decode-dispatch,
capability check, and routing are compute** (the deterministic policy spine — cheap, replayable, the part
worth having as content-addressed transferable logic); **the get/put/store ops and the crypto/codec are
native imports** (heavy or effectful). This is not a retreat from the thesis — it is the thesis stated
precisely: what belongs in the deterministic core is the *policy*, and the machine boundary was always
where the effects live. The compile-to-handler path (`GENERIC-HOST §8`, the compilation gradient) is how a
hot policy spine later compresses into a native handler without changing this line.

**Identity is established *in compute*, not by the driver (B5).** When a connection is accepted, the remote
peer's identity is proven by the **HELLO handshake** — a signature the step verifies via `crypto/ed25519/
verify` (a pure import). The byte-stream driver is **identity-blind**: it moves bytes and knows a native
socket, nothing about *who*. The authenticated `remote-peer-id` is written into the connection-table entry
(§2c) **by the step, after verifying** — so trust is derived inside the deterministic core, never asserted
by the native side. This keeps the capability check (which reads `remote-peer-id`) honest: the thing it
trusts was verified by compute, not handed over by an un-auditable driver.

**Why it's not circular.** The recursion "a host runs a program that is a core that hosts a program…" is a
**stack of determinism boundaries, not a loop.** At each level the pure core is compute and reality is a
native bridge; you never need "compute to run compute" because the *bottom* bridge host is ordinary native
code (Go/Rust) providing the seven primitives once. Self-hosting (Profile 4→5) is the far-horizon question
of shrinking that bottom bridge toward Paper 04's ~400–500 LOC bootstrap evaluator — **explicitly unbuilt
and unproven** (`MACHINE-BOUNDARY §4`: independence = fact; replacement = open). This doc is Profile-3: a
core that runs *as a program* on a *native* bridge host, which is buildable now and is the honest first
rung of the self-hosting ladder.

### §4a What this buys immediately (the operator's "new doors")

Even at Profile-3 and even before a full core-as-compute, the transport shape alone opens the doors the
operator named: **byte-stream ports let a hosted app-peer talk the entity protocol.** That is the L5 doc's
**③β contract** — "entity protocol over a transport, postMessage-tunnelled in-browser, WS/TCP natively"
(`L5 §2`). Concretely: entity-browser-rust (or any bridge host) can open byte-stream ports to inter-peers,
**browse their trees and dispatch to them**, and a hosted app can be given a *scoped* byte-stream to a peer
rather than being a dumb state-emitter. Connectivity to interpeers, tree-browsing, cross-peer
communication — all of it is "the transport shape, bound to a real socket, admitted by capability." The
snake/life/asteroids goal needs none of this (they're pure tick-loops); it is the *next* capability the
same host grows.

---

## §5 Running on what (the substrate choice, honestly)

| runtime | has compute? | native TCP? | crypto/CBOR FFI? | verdict as a bridge host |
|---|---|---|---|---|
| **entity-workbench-go** | ✅ (Stage-1 + Axis-1; generic host proven zero-symbol) | ✅ stdlib `net` | ✅ stdlib `crypto/*`, a CBOR lib | **first bridge host** — everything is stdlib; the generic host is furthest along here |
| **entity-browser-rust** | ✅ (browser mount passes Go oracle) | ❌ **no raw TCP** | ✅ if pure crypto/CBOR are **builtins** (B0) | **second** — offers `byte-stream{ws,webrtc-dc}` only; raw-TCP programs cleanly unadmitted. The wasm32 `compute/apply` stub blocks *drop-down* imports but **not builtins** (B0), so pure crypto/CBOR work in-browser once blessed as builtins; the impure drop-downs it can't dispatch are ones the browser can't serve anyway |
| **entity-core-py** | ✅ (py impl has compute) | ✅ stdlib `socket` | ✅ stdlib | viable third; useful as a cross-impl transport-shape oracle |
| **Keystone peers** | ❌ **no `EXTENSION-COMPUTE`** | (n/a) | (core-protocol only) | **cannot host** — reachable as a peer over the wire, cannot *mount* a compute program |

**Recommendation.** Start the bridge host on **workbench-go**: implement the `transport`/`byte-stream`
driver over `net.Conn` and the `crypto/*` + `codec/cbor/*` imports over stdlib, populate `offered`, and run
the **smallest possible core-as-program** — a peer whose `step` answers a `get` over one TCP byte-stream
port — against a Keystone peer as the wire counterparty (Keystone can't host, but it's a perfect
independent wire oracle). Then port the *shape ABI and import contract* (not the drivers) to browser-rust
over WebSocket. The falsification mirrors the generic-host one: **the same core-program, authored once,
serves the wire from Go and from a browser with no shared driver code, byte-identical on the wire.**

---

## §6 Open questions & the gap list (honest)

Named so they route, not tracked here as a checklist:

1. **Transport-shape ABI normative home.** Same question the generic-host doc left for shapes (`§8`): the
   `(role, shape)` + `scene` table for `transport` is a small new normative surface. Likely folds into the
   app-convention proposal's shape table alongside `display-list`/`text`. Pin `byte-stream` vs `datagram`
   contract (lossless/ordered vs may-drop) as the cross-peer-observable seam.
2. **Import-provider contract.** `imports` has a descriptor slot and an admission rule but **no provider
   contract** — the analog of the device proposal's §10 provider/conformance section, but for dispatchable
   capabilities. What's the signature encoding, how is purity declared, how does a host advertise an import
   in `offered`'s exact enumeration shape (device §11 open-Q 3)?
3. **Purity / replay marking.** A program's replay guarantee is valid only up to its first impure import
   (`sign`/`connect`/`clock`). The descriptor should mark impure imports and the host should record the
   replay boundary. Undesigned.
4. **Connection lifecycle ownership (§2b).** Confirm the static-endpoint=run-state / dynamic=imports split;
   define the `conn-id` → byte-stream-port materialization path convention.
5. **Backpressure differs by shape (corrected).** A `byte-stream` is **lossless — you never drop a byte**;
   its backpressure is **stall**, not error: when the inbox hits its bound the driver **stops reading the
   socket**, and TCP's own flow-control window closes upstream (lossless by stalling). Only a hard cap
   breach (a stalled peer that never drains) escalates to **closing the connection** — an observable error,
   not a silent drop. A `datagram` port, by contrast, MAY drop (that is the shape's contract). So the
   policy is: byte-stream = bounded-inbox + socket stall + close-on-hard-cap; datagram = drop-oldest. The
   *bound* (per-port buffer size) is the one number to pin.
6. **The clock as an import vs the tick.** The tick is host-clocked (`§4`); `clock/now` as a readable import
   is a *different* non-determinism (a timestamp *in* state). A peer needs timeouts/keepalive (W-NETWORK) —
   decide whether those read `clock/now` or are driven by host-side timers that write an input port (the
   latter keeps the step replayable). Ties to `PROPOSAL-NETWORK-LIVENESS-REACTIVE-BUILDOUT`.
7. **Nothing is built.** To be blunt for the honesty rule: `imports` has **zero providers**; there is **no
   transport shape**, **no byte-stream driver**, **no crypto/CBOR import** in any generic host today. The
   generic host is proven for *display/input* shapes only. Everything in this doc is design. The cheapest
   real evidence is §5's smallest-core-over-one-TCP-port demo on workbench-go.

**Deliberately deferred** (flagged, not opened): GPU/DSP as native drop-downs (they're `compute/apply`
Category-C like rendering — real, but a display-lineage concern, not transport); durable `storage/*` beyond
the tree; the full self-hosting descent (Profile 4→5, `MACHINE-BOUNDARY §5`); and the monolithic→chunked
state-emission problem (`L5 §5.7`) — the operator deprioritized it, and it is orthogonal to the bridge
seam.

---

## §7 Where this meets the other tracks (one paragraph, so the stack stays one story)

The **generic host** (`EXPLORATION-GENERIC-HOST-AND-DESCRIPTOR`) is the deterministic mount loop; the
**bridge host** (this doc) is that same host's *native provider profile* — the drivers and imports on the
far side of the determinism boundary. The **machine-boundary** doc's `compute/apply` line is the boundary
these drivers/imports sit on; the **L5 app-hosting** doc's "full WASM entity peer" payload (①c) and its
③β "entity protocol over a transport" contract are *what the transport shape enables*; and the
**sub-peer isolation** proposal (sibling, drafted alongside this) decides *in which peer/sub-tree* a
bridge-hosted app-peer runs and *what byte-streams it is granted* — admission over `offered` is the shared
gate for both "which shapes/imports" (this doc) and "which peer/capabilities" (that one). One host, one
`(role, shape)`+`imports` vocabulary, one `offered`/admission contract — told from the reality-facing end.

*Coordination synthesis authored in the arch workspace at the operator's request; the ratifiable output is
a `proposals/` entry (transport shape + import-provider contract) produced via proposal → ratify → fold,
not this file.*
