# PROPOSAL — the sub-peer isolation model (where a hosted app runs, and its boundary to the primary)

**Status:** DRAFT — 2026-07-20.
**Depends on:** `PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM` (the descriptor + generic host it isolates);
`APP-CONVENTION-EMBED` (the `sandbox-constraint` vocabulary + gate G1 this reuses, §7); `PROPOSAL-SYSTEM-DEVICE`
(`offered`/admission — the capability gate); `EXTENSION-CAPABILITY` (prefix-scoped grants). **No wire change;
no new sandbox model — this reuses EMBED's `sandbox-constraint` + G1 at app-peer scale.**
**Scope:** an **L5 applications-domain convention** — the **hosting envelope**: how a generic-host-mounted
program (or an app-peer) gets its **own peer + sub-tree**, its **capability boundary to the primary**, and
its **transport to the host**. It standardizes the *isolation seam*, never the payload.
**Research (design record):** `EXPLORATION-L5-APP-HOSTING-UNIFICATION.md` §3/§5 (the two principles + the
open decisions this closes); `EXPLORATION-BRIDGE-HOST-AND-ANY-NATIVE-COMPUTE.md` §4a (the ③β transport the
isolated app-peer speaks).
**Direction: confirmed by the operator (2026-07-20)** — the sub-peer isolation model is intended design and
the biggest gap; this proposal turns L5 §5.2/§5.3/§5.4 into a ratifiable convention.

---

## §1 Concept — the hosting envelope is separable from the payload

The L5 synthesis established that an entity app is a choice on three independent axes — **payload** (opaque
UI / compute program / WASM entity peer), **isolation** (system peer / user peer / sandboxed iframe /
native sub-peer), and **boundary contract** (postMessage UI / entity-protocol-over-transport / EMBED
active-embed). This proposal specifies **the isolation axis and its boundary to the primary** — the
*envelope* — and deliberately leaves the payload and the wire-contract to their own conventions (the
compute-program descriptor; the entity-app postMessage contract). One envelope, several payloads.

The two load-bearing principles it enforces (L5 §3):

- **P1 — the host is blind to the payload.** The envelope handles one thing: an isolated unit boots, emits
  state, and the host manages that state. What runs inside is opaque.
- **P2 — application compute runs isolated; the system peer is reserved.** The concern is *isolation*; the
  sandboxed iframe is browser-rust's chosen mechanism, a user peer is acceptable, and the **system peer is
  never** a host for transferred/untrusted compute.

The `sandbox-constraint` EMBED already defines for active *embeds* (`read_only_within`,
`deny_external_handlers`) is the **same instinct at widget scale**; this proposal is that instinct at
**whole-app scale**, and (per L5 §5.5) **they SHOULD share one mechanism and one gate (G1)** — the security
gate is done once for a widget and a whole app.

---

## §2 The isolation unit — an app gets its own peer + sub-tree

> **Normative (I-1).** A hosted application MUST run in an **isolation unit** distinct from the host's
> **system peer**. The isolation unit is one of the profiles in §3. Its state lives under a **single
> sub-tree root** the host assigns; the application has authority **only** within that root.

The sub-tree is the unit of everything — placement, capability scoping, teardown, and (P1) the opaque state
the host persists. Concretely:

```
<host tree>
  system/…                      ; the system peer — RESERVED; no app compute here (P2)
  apps/<app-id>/                 ; the isolation unit's sub-tree root (host-assigned)
    descriptor                   ; the mounted program's descriptor (compute-program payload)
    state                        ; the app's state_path — the opaque {__v,data} / program state (P1)
    ports/…                      ; the app's input/output ports (incl. byte-stream ports, bridge doc §2)
    save                         ; the persisted opaque save the host manages
```

**The app reads/writes only within `apps/<app-id>/`.** That prefix is `read_only_within`'s default ("the
prefix the embed's compute may read — default: its own subtree", `APP-CONVENTION-EMBED §3`), promoted here
from a widget's subtree to the app's sub-tree root. Anything outside is denied unless an explicit grant
(§4) says otherwise.

---

## §3 The four isolation profiles (one envelope, per-host mechanism)

The isolation *mechanism* is the host's choice; the *contract* (I-1 + §4 + §5) is uniform. Named profiles,
mapped to the L5 axis-② options:

| Profile | Mechanism | Trust / failure boundary | Typical host |
|---|---|---|---|
| **P-iframe** | `<iframe sandbox="allow-scripts">` (browser) | OS/browser process + origin sandbox; a crashing app can't touch the host tree except via the save | **entity-browser-rust** (preferred pattern) |
| **P-user-peer** | a distinct **user peer** (not the system peer) | peer-level: own tree root, own capability set | browser-rust / workbench-go when a full peer is wanted |
| **P-native-subpeer** | a native sub-process / sub-peer | OS process boundary | workbench-go, a godot host, a native bridge host |
| **P-in-process** | in-process mount, sub-tree-scoped only | **weakest** — sub-tree + capability scoping, no process/VM boundary | workbench-go / godot when the payload is trusted |

- **The system peer is never a profile.** (P2; I-1.) The rushed-alignment failure the L5 doc names — compute
  likely mounted onto the system peer — is exactly the absence of I-1; **the concrete "back on track" fix is
  to move any current mount into one of these profiles.** (Verify the browser mount's current placement — L5
  §6 step 4 — and relocate if it is on the primary/system peer.)
- **P-iframe is browser-rust's uniform mechanism** for *all* payloads (L5 §2, isolation column): a dumb JS
  app, a plain WASM app, and a full entity-core WASM peer all sit in `<iframe sandbox>` — the host can't
  tell them apart (P1), and *that is the point*. The **"different startup mode"** (L5 §3b) is the P-iframe
  case for the ①c payload: the embedded instance is **entity-browser-rust booted minimal** (no
  window-manager, no peer roster — just the state-emission handler + generic-host code + the program's
  interfaces), not a separate codebase.
- **A program-native descriptor is profile-agnostic.** The same `(descriptor + step IR + initial_state)`
  mounts under any profile; the profile is the *host's* deployment decision, matched by admission (§4). This
  is what preserves transferability (L5 §3 payoff).

---

## §4 The capability boundary — deny-by-default, prefix-scoped, admitted over `offered`

The boundary between the isolation unit and the primary is a **capability boundary**, and it reuses EMBED's
G1 vocabulary rather than inventing a second model (L5 §5.2/§5.5):

> **Normative (I-2 — deny-by-default).** An isolation unit has **no ambient authority.** It may read/write
> only within its sub-tree root (§2), dispatch only handlers within that root, and reach a capability of the
> primary or another peer **only through an explicit prefix-scoped grant** the host mints at mount.

Grounded in the existing fields:
- **`read_only_within`** (`APP-CONVENTION-EMBED §3`) = the sub-tree prefix the app's compute may read;
  **default = its own sub-tree root** (§2).
- **`deny_external_handlers`** (ibid., **default TRUE**) = refuse `handler_targets` outside the sub-tree.
  This is EMBED condition **C3** ("cross-peer dispatch from bundled compute refuse-by-default") at app scale.
- **Granted capabilities** are the app's `imports` (bridge doc §3) — e.g. a scoped **byte-stream** to one
  named inter-peer (the ③β connectivity, §5), a read grant on a shared sub-tree, a `system/device` domain.
  Each is an explicit grant, matched at **admission** against the host's `system/device/host/offered`
  (`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §5`, `PROPOSAL-SYSTEM-DEVICE §7`): a unit whose grants the host
  cannot supply **is not admitted, and says why.**

> **Normative (I-3 — one admission, one gate).** Admission of an isolation unit matches its declared shapes
> + imports + cross-sub-tree grants against `offered`; the **security review is gate G1**
> (`APP-CONVENTION-EMBED §7`, conditions C1–C5), which this proposal **shares** — an active embed (widget)
> and an isolated app-peer are the same capability at two scales, so G1 gates both. Do it once.

**G1 conditions, at app scale** (verbatim intent from EMBED §7, C1–C5): audit is shell-inspectable (C1);
per-front-end audit-prompt policy hook (C2); **cross-peer dispatch from bundled compute refuse-by-default**
(C3 = `deny_external_handlers`); per-site policy keyed by `(consuming-peer, sub-tree-root-hash)` (C4); a
**distinct install surface** for a site/app-scoped isolation unit, not a side door of generic handler
registration (C5). Owners: arch + the consuming L5 teams (browser-rust, workbench-go).

**Isolation scopes the *path* namespace, not the content store (I-2 clarification).** `read_only_within`
bounds an app's authority over **tree paths** (`path → hash`). It does **not** and **cannot** bound
content-addressed reads (`lookup/hash`): the content store is **global by dedup** (load-bearing invariant #2,
`AGENTS.md`), and possession of a hash *is* the read capability for that content — that is content-addressing,
not a leak. So an isolation unit reads any content whose hash it holds, but can only *name* (path-address)
things inside its sub-tree. The boundary is over the namespace it can traverse and mutate, which is the
authority that matters; it is never a claim that the app cannot see a blob it already has the address of.

---

## §4a App-peer identity — its own key, never the host's (the impersonation edge)

The moment an isolation unit is a **peer** (③β, §5) and can `crypto/ed25519/sign` (bridge doc §3), *whose
identity does it sign as?* If it borrowed the host's — or worse the **system peer's** — key, a hosted,
possibly-untrusted app could **sign as the host**, forge its cross-peer messages, and mint capabilities in
its name. That is the isolation model failing at its most important seam. So:

> **Normative (I-5 — distinct identity).** An isolation unit that acts as a peer MUST have its **own peer
> identity** — its own Ed25519 keypair, its own PeerID — **minted at mount and distinct from the host and
> from the system peer.** (Canonical PeerID is the **identity-multihash** form: the digest *is* the public
> key — `PeerId::encode_raw`, `entity-core-rust/core/crypto/src/lib.rs`, `HASH_TYPE_IDENTITY`; the older
> `SHA-256(pubkey)` form is **legacy-decode-only**. Say "own keypair," not "hash of a pubkey.") The
> `crypto/ed25519/sign` import available to the unit signs with **that unit's** secret key only; the host's
> / system peer's secret key **never enters** an isolation unit's compute or import surface. Tearing down
> the unit (§6) discards its identity.

**This is net-new — it does not exist today (prove-the-negative done).** A verification sweep of
`entity-browser-rust` + `entity-core-rust` found **no sub-identity / per-app-keypair mechanism anywhere**:
the three app types (`AppCatalog`/`AppBundle`/`AppSave`, `src/apps/format.rs`) carry **no** key/identity
field (`AppSave` is literally `{ state: String }`, an opaque blob); hosted apps run in
`sandbox="allow-scripts"` iframes with **no identity at all** — dumb state-emitters — and saves are written
under the **host** peer's tree keyed by the host `peer_id` (`app_save_path`, `src/app_paths.rs`); and the
only keypair minting is a *host* peer's own (`mint_identity`, `cmd/entity-peer`). So I-5 is a **capability
this proposal must add** (mint a per-unit peer identity + scope `sign` to it), not a reference to something
extant. Framing it as existing would be an overclaim.

Consequences:
- **The secret key stays native, on the far side of the boundary.** `sign` is "native by necessity"
  (bridge doc §1a): the key is held by the host's native crypto provider *scoped to this unit*, and the
  step only ever gets a signature back, never the key. (This is why `sign` is a capability, deny-by-default,
  not a pure builtin.)
- **Isolation is peer-granular — because the connection table is keyed by remote peer-id, one-per-peer.**
  `entity-core-rust` tracks connections in `HashMap<peer_id, endpoint>` with at most one per remote peer-id
  (`RemoteState`, `core/peer/src/remote.rs` — verified). So two units talking to the same inter-peer X
  **cannot share the host's connection table**; each unit must be a **full peer with its own identity and
  its own connection pool.** The cost (N peers → N connections to X) is the price of the boundary — and it
  is *why* a wire-speaking isolation unit is a real peer, not a sub-thing. (Bridge doc §2c/B2.)
- **Sub-identity vs fresh identity is an open (§9).** Whether the unit's identity is an independent fresh
  keypair or a *derived/attenuated* sub-identity of the host (an ocap-style attenuation, cf. the keystone
  revocation-as-OR-Set hand-off) is a real design fork — fresh is simpler and maximally isolating; derived
  buys accountability (the host can prove which unit it spawned). **Neither exists yet** (no sub-identity
  minting anywhere); pin at ratify.

## §5 The host↔app-peer transport (the ③β contract)

Two boundary contracts, by payload — **and the isolation unit does not change which one is used**, only
where it terminates:

- **③α — postMessage UI/state** (the dumb-client entity-app contract): `ready-for-init → init{state} →
  viewport/audio → ready → state{…} → request-state → destroy` (L5 §1, `entity-apps-htmljs`). The payload is
  **not a peer**; the host is dumb key→blob storage. Unchanged by this proposal — it is the P-iframe
  contract for opaque-UI payloads.
- **③β — entity protocol over a transport** (the app *is* a peer): the isolated app-peer speaks the entity
  protocol to the host over a transport. **In-browser (P-iframe): tunnelled over a `MessagePort`;** natively
  (P-native-subpeer): a local socket / the **byte-stream transport shape** (bridge doc §2a). This is where
  the two lineages meet — Lineage A's postMessage and Lineage B's peer model.

> **Normative (I-4 — the tunnel).** For a ③β app-peer in P-iframe, the host↔app-peer wire is
> **`EXTENSION-NETWORK` framing over a `MessagePort`** — the same frames (`HELLO`/`EXECUTE`/…), the same
> CBOR ECF, carried by postMessage instead of a socket. A byte-stream transport port (bridge doc §2a) whose
> `scene.protocol = message-port` is the in-browser driver for exactly this. So ③β is **not a new wire** —
> it is the locked wire core over a `MessagePort` driver, which keeps a WASM app-peer in an iframe and a
> native peer over TCP **byte-identical on the wire** (the re-encode interop hazard is avoided: the tunnel
> preserves canonical ECF, `AGENTS.md`).

This is the payoff the operator named (bridge doc §4a): once the app-peer speaks ③β over a granted
byte-stream, it can **browse an inter-peer's tree and dispatch to it** — connectivity to interpeers from a
hosted app, gated by the §4 capability boundary (the app reaches only the peers its grants name).

---

## §6 Lifecycle & state — the envelope owns boot, state, teardown (P1)

- **Boot.** The host creates `apps/<app-id>/`, seeds the descriptor's input ports (F-E1,
  `PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §3`), mints the §4 grants, and starts the mount loop / boots the
  minimal app-peer (the "different startup mode", §3). Run-state (start/stop/restart/reseed) is the host's,
  outside the program (ibid. §5).
- **State.** The app emits its state as output-port state / the ③α `state{…}` message; the host persists it
  as the opaque save **within the sub-tree** and MUST NOT inspect or migrate it (P1; the EMBED "dumb
  key→blob" rule). *(The monolithic→chunked emission optimization — L5 §5.7 — is **out of scope** here, per
  the operator's deprioritization; the envelope is written so a later chunked `state` message drops in
  without reshaping the sub-tree, since `apps/<app-id>/state` is already `path→hash` sub-tree-scoped.)*
- **Teardown.** Destroying the isolation unit is **deleting its sub-tree root and revoking its grants** —
  one operation, because everything the app had authority over lived under one prefix (§2). Transitive grant
  revocation follows the capability model (an OR-Set, per the keystone convergence hand-off).
  - **In-flight cross-peer work must not dangle (teardown edge).** An app-peer may have **standing
    continuations** (W-CONTINUATION) rooted in its sub-tree, or open byte-stream connections (bridge doc
    §2b). Teardown MUST: (1) cancel/collect the unit's in-flight continuations — the same **crash-mid-flight
    collector** the keystone hand-off names as open (`MACHINE-BOUNDARY §9`), here fired deliberately rather
    than on crash; and (2) close its connections (entity-protocol close **before** the WebSocket/TCP close,
    per `EXTENSION-NETWORK §6.5` close-ordering). A continuation *targeting* the torn-down unit from another
    peer resolves via the lost-error-marker MUST (`PROPOSAL-CONTINUATION-LOST-ERROR-MARKER-MUST`) — the
    remote sees a clean error, not a silent hang. **Deleting the sub-tree is necessary but not sufficient;
    the cross-peer edges leaving it must be severed too.**

---

## §7 The JS lifecycle contract's place in the stack (L5 §5.4, resolved)

The `init/viewport/state/destroy` lifecycle is a **profile of this envelope, not a legacy adapter.** Its
`viewport`/`audio`/`teardown` messages are **substrate concerns** (any embedded surface has a viewport and a
teardown), not JS concerns — so a WASM app-peer in P-iframe uses the *same* lifecycle envelope, with its
wire being ③β (I-4) instead of ③α. Resolution: **the lifecycle envelope (boot/viewport/state/destroy) is
shared across payloads; the *state-transfer contract inside it* is ③α for opaque-UI and ③β for app-peers.**
One envelope; the payload picks the inner contract.

---

## §8 What is convention vs. impl (the fence)

- **Convention (this proposal):** I-1..I-4 — the isolation unit + sub-tree root, deny-by-default capability
  boundary, admission-over-`offered`, the ③β tunnel = wire-core-over-MessagePort, the shared lifecycle
  envelope, sub-tree teardown. All cross-peer/cross-host-observable.
- **Impl (per host, N-class):** *which* profile a host offers (browser → P-iframe; workbench-go →
  P-native-subpeer / P-in-process), the iframe/process mechanics, the `MessagePort` plumbing. Not specified.
- **Non-goals:** not a new sandbox model (reuses `sandbox-constraint` + G1); not a new wire (reuses the
  locked wire core over a transport driver); not the payload conventions (compute-program descriptor,
  entity-app postMessage contract stay their own docs); not chunked state emission (deferred, §6).

---

## §9 Open questions (honest)

1. **Grant enumeration shape.** §4 mints prefix-scoped grants at mount; the exact enumeration (which
   `imports` / cross-sub-tree reads / named-peer byte-streams a host offers to an app, and how the user
   consents) rides EMBED G1 C4's per-site policy — needs the G1 audit design to land (shared owner).
2. **P-in-process is the weak profile — when is it allowed?** No process/VM boundary means sub-tree +
   capability scoping is the *only* guard. Pin: P-in-process is for **trusted** payloads only (the host's
   own compute, not transferred/untrusted code); untrusted/transferred code MUST use a boundary-bearing
   profile (P-iframe / P-user-peer / P-native-subpeer). Confirm the trust classification input.
3. **Which peer is the "user peer"?** P-user-peer says "not the system peer"; browser-rust runs many peers
   (L5 §3 P2 correction). Define whether the app-peer is a *fresh* peer per app or a shared user peer with
   per-app sub-trees. (Leaning: fresh peer per app for untrusted; shared user peer for the user's own apps.)
4. **`MessagePort` framing details (I-4).** Confirm `EXTENSION-NETWORK` framing rides a `MessagePort`
   unchanged (HELLO handshake, flow control) vs. needing a reduced local-transport profile — route to the
   browser-rust team as the impl question (L5 §5.3).
5. **Remote mount ≠ remote control.** If an isolation unit is mounted on a *remote* peer's host, run-state
   is the *receiving* host's (bridge/generic-host §8). Pin that the granting/consenting party and the
   run-state owner can differ, and the capability boundary (§4) still binds on the receiver.
6. **Fresh vs derived app-peer identity (I-5).** Independent fresh keypair (simplest, maximally isolating)
   vs an attenuated sub-identity derived from the host (accountable — the host can prove which unit it
   spawned, ocap-style; cf. the keystone revocation-OR-Set hand-off). Decide at ratify. Ties to F-PQ /
   `PROPOSAL-MULTIKEY-MULTIHASH-ALIGNMENT` if unit keys ever need rotation/migration.

---

## §10 Sequence (design-level, matching L5 §6)

1. **This envelope + the compute-program convention ratify together** — the descriptor is the payload, this
   is the envelope; they lock Lineage B as an L5 convention (L5 §6 step 2–3).
2. **Confirm/correct the browser mount placement** (L5 §6 step 4, §3 above) — the concrete back-on-track fix.
3. **P-iframe + ③β (I-4)** is the convergence point — gated on the two wasm32 compute stubs upstream
   (L5 §4) and the bridge doc's `MessagePort` byte-stream driver.
4. **G1 is done once** for widget-embeds and app-peers (§4, I-3).

*Authored in the arch workspace at the operator's request; ratifiable via proposal → ratify → fold. Sibling
to `EXPLORATION-BRIDGE-HOST-AND-ANY-NATIVE-COMPUTE` — that doc supplies the transport shape this envelope's
③β grants ride; this doc supplies the peer/sub-tree/capability boundary that doc's app-peers run inside.*
