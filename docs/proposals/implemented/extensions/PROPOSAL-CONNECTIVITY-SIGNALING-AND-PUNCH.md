# PROPOSAL — Connectivity: the signaling service + punch coordination (native, TCP-simopen first)

**Status:** **RATIFIED + FOLDED (2026-07-31)** — landed as `specs/extensions/EXTENSION-SIGNALING.md` **v1.0**
(peer-side: §3 key derivation + four modes, §6 coordination messages, §7 the punch, §10 fallbacks) plus
`EXTENSION-NETWORK.md` **v1.6 Amendment 14** (the §10.3 live-establishment seam the punch registers behind).
**The spec is source of truth from here**; this document is the design record.
**Build state (2026-07-31):** key derivation and coordination are built in all three, with a `signalingMeet` cross-impl check in Go's validator; the punch (§7) is not built. §11.5's full gate has not run.
*Was: DRAFT (2026-07-22).*
**Target:** a new **`system/signaling`** handler spec (the pluggable-carrier rendezvous service) + the
coordination-message entity types (`system/nat/connect-request` / `-response` / `-punch-sync`) + a NETWORK note
wiring the punched connection into §10 as a live-transport establishment path (the live-punch rung reserved at the
**§10.2** seam, L1334). Consumes `PROPOSAL-NETWORK-REACHABILITY-FACTS` (the candidates + reflection + dial-back).
No V7/wire renumber.
**Provenance:** brought forward — *reconciled, not verbatim* — from the archived DRAFT
`PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY.md` §4/§5/§6/§7 + `PROPOSAL-EXTENSION-WEBRTC-TRANSPORT.md` (signaling
gap), read-only in the legacy meta-tree, under `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md`
Parts B/D + Part F steps 2–3 + Part H unit #2.
**Scope:** the **one genuinely new protocol** — coordinating a direct connection between two peers who cannot dial
each other (both behind NAT). It is unit #1's *facts* put in motion. **Browser/WebRTC is unit #3**
(`PROPOSAL-EXTENSION-WEBRTC-TRANSPORT`); our-QUIC substrate, RELAY Mode-C live circuit, and the async inbox are the
**opt-in heavier tier** (§7), not basic coordination.

---

## 0. The recorded operator decisions (this session, 2026-07-22)

Two calls the connectivity exploration deferred to the operator (Part G1/G2) were made this session and are
recorded here so the design record is on-disk, not in a transcript:

- **G2 — v1 signaling carrier = rendezvous (stateless), preferred.** The basic coordination tier is **registry-CDN
  + stateless reflector + a stateless rendezvous signaling endpoint** — cheap, horizontally scalable, near-zero
  state. **Out-of-band QR/short-code is a *secondary* carrier, flagged for review** (kept as the zero-infrastructure
  human-present option; not the primary path). **`relay-forward` (RELAY Mode-F), the live data-relay (Mode-C), and
  the async inbox (Mode-S) are the *opt-in* tier** — services a dedicated peer chooses to run *on top of* the basic
  coordination infra, never part of it (§7). *Rationale (operator):* keep the always-on infra to the cheapest
  possible coordination primitive; everything heavier is opt-in capacity that specific peers volunteer.
- **G1 — substrate build order = TCP-simultaneous-open first, our-QUIC as the sequenced upgrade** (§5). Substrate is
  additive; the coordination layer is substrate-agnostic, so this is a build-order call, not an architecture fork.

**The Mode-S clarification (it caused confusion, so it is pinned):** Mode-S is disqualified **only as the live
*signaling* carrier** — signaling is a real-time "both dial at T" handshake, and the Mode-S mailbox is *async
store-and-forward mail* (the wrong tool for a live rendezvous). **Mode-S is fully retained** as the opt-in async
*data* path (offline delivery), advertised as the optional `inbox_relay` in `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT`.
"Disqualified as a signaling carrier" ≠ "removed."

### 0.1 Additional operator decisions (2026-07-24 — the key-modes + deployment session)

- **The rendezvous key has four modes, and a shared secret is just one of them (§2.2).** `pair` (peer-ids),
  `tag` (a group/topic label), `secret` (an agreed string), `lobby` (a blind "connect me to anyone" constant). A
  **shared secret is any agreed term/string both peers use at the same time on the same service pool** — it needs
  no new mechanism; it is a `secret`-mode key, `H(string)`, mechanically identical to a `tag` and distinguished
  only by the secrecy/entropy of the string. *Rationale (operator): the modes are only how `K` is computed; the
  carrier, the provider selection, and the punch are invariant across them.*
- **The connection node is valuable at both ends of the deployment spectrum — both first-class (§2.3).** It is
  worth running **even if exposed only to the operator's own managed peers** (a private device mesh — phone,
  laptops, servers punching to each other anywhere), *and* it functions as **generic shared infrastructure anyone
  can use**. Same node, same wire surface; a private pool + optional capability gate vs. an open public pool is
  configuration, not a fork.

## 1. Motivation — the one missing case

`PROPOSAL-NETWORK-REACHABILITY-FACTS` §1 established that a NAT'd peer ↔ a *public* peer already works today
(`held_connection_client`). **The one genuinely missing case is two peers *both* behind NAT wanting a *direct*
connection** — neither can accept inbound, so neither can dial the other, and there is no public party in the
middle to hold a socket to. Two ways to serve it:

1. **Relay the data** through a public peer both reach outbound (RELAY Mode-C live / Mode-S async) — always works
   (needs only outbound), but consumes a dedicated peer's bandwidth. **Opt-in tier (§7).**
2. **Punch a direct hole** so they talk peer-to-peer with no server in the data path — lower latency, no relay
   cost, but fails on ~some NAT types (symmetric/CGNAT), so it *always* needs the relay behind it as fallback.
   **This proposal.**

So the job: **add the punch path, keep the relay as opt-in fallback, and let a peer try direct-first, relay-last**
(the prefer-cheap-path order already pinned in `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` §3).

## 2. The signaling service — one service, pluggable carriers, fixed message schema

Signaling is the one service two not-yet-connected peers need: a **mutually reachable third party that carries the
candidate exchange** between them (Part D). The coherent rule (Part D, now decided per §0):

> **One signaling service, a fixed per-substrate message schema (§3), and a *pluggable carrier*.** The carrier is
> how the opaque signaling messages travel; the message schema does not change with the carrier.

| Carrier | v1 posture | What it is |
|---|---|---|
| **rendezvous** (`system/signaling`) | **v1 — the basic tier** | A thin **stateless** endpoint. Both peers rendezvous-hash the shared key into the advertised `signaling` pool (`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` §3.1) → both meet at the **same** server for the seconds of the handshake. Transient per-handshake state only, TTL-reaped. |
| **out-of-band** (QR / short-code) | **v1 — secondary, flagged for review** | The zero-infrastructure human-present case (LAN pairing / paste-a-code). Kept as an option; **not** the primary path (operator reservation, §0). |
| **relay-forward** (RELAY Mode-F) | **opt-in tier (§7)** | Live signaling routed through a RELAY peer that offers Mode-F. A dedicated peer's service, not basic infra. Rides RELAY §9 opacity (the relay never decodes the signaling entity). |
| ~~Mode-S inbox~~ | **disqualified as a carrier** | Async mail, not a live handshake (§0). Retained as the async *data* path, not signaling. |

### 2.1 The `system/signaling` rendezvous handler (v1)

A minimal stateless carrier — net-new handler (no collision with `system/relay` / `system/network`):

```
system/signaling:offer(rendezvous_key, message) → { ok }   ; deposit an opaque signaling entity at the key
system/signaling:collect(rendezvous_key) → { messages: [<hash>, ...] }  ; peer retrieves entities at the key
system/signaling:advertise → { endpoint, limits }          ; announce the carrier + its rate/TTL limits
```

- `rendezvous_key` — **both peers derive it identically**, and it is the shard key
  (`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` §3.1). Its derivation is **mode-dependent (§2.2)** — a peer-id pair, a
  group tag, a shared secret, or a blind-lobby constant — but the carrier is **mode-blind**: it sees only the
  opaque key and holds a TTL-reaped bucket per key, with **no cross-server state** (a hot key is one handshake, not
  a hot shard). Provider selection (rendezvous-hash into the configured pool, §3.1) is **invariant across modes**;
  only the key input changes.
- `message` is an **opaque signed entity** (§3) — the signaling server, like a relay, **never decodes it**
  (carrier opacity, mirroring RELAY §9). It sees a content-addressed blob and a rendezvous key, nothing more.
- **Stateless + rebuildable:** the only state is the transient per-key bucket; losing a server drops in-flight
  handshakes (peers retry), never durable data. This is what makes it the cheap basic tier.

> **The entity-native alternative (even less bespoke — same selection rule):** model signaling as **ephemeral
> entities in a rendezvous namespace** carried by Mode-F relay + SUBSCRIPTION, sharded by `rendezvous_key`,
> TTL-reaped — the system's own coordination substrate instead of a dedicated handler. Either backing uses the
> §3.1 rendezvous-hash selection. **Open item (§10):** ship the dedicated `system/signaling` handler, the
> namespace shape, or both. *Leaned: the dedicated handler for v1 (simplest to reason about + conformance-test);
> the namespace shape as the scale-out option.*

### 2.2 The rendezvous key — one derivation, four modes `[the cross-peer seam — MUST pin]`

The whole discovery surface is *one* keyed rendezvous; the "modes" are only **how the two peers agree on the key
`K`**. One general, mode-tagged derivation — so inputs of different modes can never collide, and adding a mode is
picking an input, not adding a mechanism:

```
payload        = "entity:rdv:v1" ‖ SEP ‖ mode ‖ SEP ‖ canonical(mode_input)
rendezvous_key = varint(0x00) ‖ SHA-256( ecf_for_hash( "system/nat/rendezvous-key", cbor_bstr(payload) ) )
```

Both peers MUST produce byte-identical hash input **and** byte-identical key bytes — divergence here means the two
peers derive different keys and **silently never meet** (the load-bearing cross-peer-seam failure shape; stated
once, generally). That failure is invisible to a same-impl test, so **every free variable is pinned here** and
exercised by the differential tests in §2.2.1:

- **`H` is the substrate content-hash primitive** (V7 §1.2) — which hashes ECF-encoded `{data, type}`, **not** a
  bare byte string. So its two inputs are pinned: `type` = the fixed string **`system/nat/rendezvous-key`**, and
  `data` = **`payload` wrapped as a single CBOR byte string** (`bstr`, minimal-length head). A `tstr` wrapping, a
  multi-element array, or a bare unwrapped concatenation each yield a different digest — pick one, and this is it.
- **The digest format is pinned to the SHA-256 floor (`content_hash_format` `0x00`, V7 §8.2) — *not* the deriving
  peer's home format.** This is the one that would have bitten hardest: the content hash is self-describing and
  format-carrying, so two conformant peers running different home formats (SHA-256 vs. SHA-384) would derive
  different keys for the same agreed input and never meet, with nothing failing loudly. A rendezvous key is a
  **lookup token that two independent parties must reproduce**, not authored content, so it does not follow the
  authoring peer's format. It stays in the self-describing wire encoding (`varint(format) ‖ digest`, 33 bytes at
  the floor) so a future format migration is expressible; it is compared **byte-wise** by peers and by the node.
- **`SEP` = `0x1F`** (ASCII US), and it appears **after the domain string and after the mode tag** — not only
  before the input. `mode` is one of the exact ASCII tags `pair` / `tag` / `secret` / `lobby`.

The node never derives or interprets a key — it is mode-blind (§2.1) and compares opaque bytes — so this pin binds
**peers only**.

| `mode` | `canonical(mode_input)` | Who meets | Secrecy of the input |
|---|---|---|---|
| `pair` | the two peer-ids **byte-wise sorted ascending**, then joined **with `SEP` between them** (`lo ‖ SEP ‖ hi`) | exactly those two peers | public (peer-ids are public) — identity *is* the key |
| `tag` | a **byte-exact UTF-8** label string (e.g. `chess`, `my-family`) | anyone who knows the tag | **public label** — a discovery convenience, *not* access control |
| `secret` | a **byte-exact UTF-8** agreed string, treated as high-entropy | anyone who knows the secret | **confidential** — knowing the secret *is* a lightweight admission gate |
| `lobby` | a fixed well-known constant — **`lobby:default`** unless the pool advertises another | anyone on that service, right now | public — "just connect me to anyone here" |

**Two sub-pins the table carries (both are silent-never-meet bugs otherwise):** (a) `pair` joins the sorted
peer-ids **with a separator** — bare concatenation is ambiguous, since `sorted("ab","c")` and `sorted("a","bc")`
both yield `abc`, so two *different* pairs would share a bucket; (b) `lobby` names an actual default constant —
"per deployment" alone means two peers pointed at the same open node still never meet. A pool MAY advertise a
different lobby constant via `advertise` (§2.1); absent that, `lobby:default` is the value.

**String inputs are byte-exact — no Unicode normalization, no case-folding.** This is *not* a new rule: it is the
core's **byte-preservation** discipline (§1.8) applied to a key that two peers derive *independently* from an
agreed string. It only shows up here because this is the first surface where two parties must produce the *same*
hash from a string each supplied on its own (everywhere else a string is authored once and its bytes are preserved
and dedup'd). So the protocol stores no Unicode database and runs no normalization step; instead **charset policy
is the app's job**: an app that lets a human type a `tag` constrains its input (recommended: lowercase ASCII
letters, digits, and `-`), and a `secret` is a generated high-entropy ASCII string exchanged verbatim. Both peers
simply hold the identical bytes.

**`tag`, `secret`, and `lobby` are the same mechanism** — `H(string)` over an agreed string; they differ only in
the **entropy and secrecy** of that string, and therefore in what property you get. This is the operator's point:
a shared secret is *just an agreed term both peers use at the same time on the same service pool* — it "fits in"
as a `secret`-mode key with no new machinery. Three consequences that MUST be stated because they are
security-load-bearing:

- **A `secret` is only as strong as its entropy.** A low-entropy `secret` (a short word) is enumerable — anyone
  who guesses the string lands in the same rendezvous bucket. A `secret` used as a gate MUST be high-entropy; a
  human-memorable phrase is really a `tag` (convenience), not a boundary. The **capability-gated admission mode
  (§2.3)** is the real access control; a `secret` key is a *zero-infrastructure* gate layered on top, not a
  replacement.
- **The key introduces; it does not authorize.** Reaching the same `K` only gets two peers *rendezvoused* — the
  coordination entities are still signed end-to-end (§3), and the resulting connection still runs the ordinary
  handshake + capability flow (§8). A shared key gets you to the meeting point; it never authorizes the session.
- **Presence is concurrent, on one pool.** Buckets are TTL-reaped (§2.1), so a `tag`/`secret`/`lobby` meeting
  needs both peers present in the same window on the **same provider pool** — the same-provider rule as `pair`
  (§2.3), since every mode still rendezvous-hashes `K` into the configured pool.

#### 2.2.1 Implementer diagnostics (not conformance vectors)

Authored expected keys are **not** published here on purpose: which bytes are correct is settled by the impls
meeting or failing to meet in a live cross-impl run, not by one author's oracle. What *is* useful up front is the
ability to bisect a mismatch to a stage, and to know which properties are differential (an impl that only checks
"my key equals my key" is still silently non-interoperable).

**Stage bisect.** For `mode = tag`, input `chess`, each stage of the pinned derivation:

```
payload            : "entity:rdv:v1" 1F "tag" 1F "chess"
                     656e746974793a7264763a76311f7461671f6368657373
cbor bstr(payload) : 57 656e746974793a7264763a76311f7461671f6368657373
ecf_for_hash       : a2 64 64617461 <bstr…> 64 74797065 7819 73797374656d2f6e61742f72656e64657a766f75732d6b6579
key                : 0x00 ‖ SHA-256(ecf_for_hash)
```

Two impls that disagree compare these four lines and know immediately whether they differ on the concatenation,
the CBOR framing, the ECF envelope, or the digest.

**Properties that require a differential test** — each is a way to be self-consistent and still never meet:

| Property | Test | A failure means |
|---|---|---|
| order independence | `pair(A,B)` **==** `pair(B,A)` | the sort is missing or not byte-wise |
| pair disambiguation | `pair("ab","c")` **!=** `pair("a","bc")` | the `SEP` join is missing — two different pairs share a bucket |
| case sensitivity | `tag("chess")` **!=** `tag("Chess")` | the impl case-folds |
| no normalization | `tag(NFC "café")` **!=** `tag(NFD "café")` | the impl Unicode-normalizes (the likeliest accidental import — many string libs do it by default) |
| mode separation | `tag("x")` **!=** `secret("x")` | the mode tag is missing from the payload |
| **fixed format code** | every key is **33 bytes**, first byte `0x00` | the impl used its **home hash format** — a SHA-384-home peer never meets a SHA-256-home peer, and passes every other check above |

The last row is the one that catches a real bug in an otherwise-correct implementation.

### 2.3 The standalone connection node → **promoted to `PROPOSAL-CONNECTION-NODE`**

The node-side half of this proposal — the `offer`/`collect`/`advertise` core verbs plus `reflect` on the
unwrapped listener (`PROPOSAL-CONNECTION-NODE` §1.4), the two admission
modes (open/rate-limited vs. capability-gated), the two deployment lenses (private device mesh vs. shared
infrastructure), and the same-provider-per-handshake rule — now lives in its own ratifiable DRAFT,
**`PROPOSAL-CONNECTION-NODE`** (2026-07-28). It was split out because the node is the piece that gets built and
deployed first, on its own schedule.

**This proposal retains the peer-side half:** the rendezvous key derivation (§2.2), the coordination messages
(§3), and the punch (§5). The one rule worth restating here because the peer-side depends on it: **both peers of a
handshake must meet at the same provider** — every mode rendezvous-hashes `K` into a shared pool, so provider
selection is invariant across modes and a mismatch is a silent never-meet just like a key-derivation mismatch.

## 3. The coordination messages (opaque, signed, carrier-agnostic)

The DCUtR-analog "we both dial at T" dance — small, three message types, ordinary signed V7 entities that consume
unit #1's candidates. Carried by any §2 carrier; the relay/rendezvous sees only the 33-byte hash.

```
type: "system/nat/connect-request"        ; A → B (via a §2 carrier)
data: {
  initiator:   <peer_id>,
  candidates:  [ {type, substrate, address, priority}, ... ],   ; A's candidates (unit #1 §4)
  nonce:       <bytes>,                    ; correlates the exchange; freshness
}

type: "system/nat/connect-response"       ; B → A
data: { responder: <peer_id>, candidates: [ ... ], nonce: <echo A's> }

type: "system/nat/punch-sync"             ; either direction — align the firing
data: { nonce: <echo>, fire_at: <uint ms — DELAY FROM RECEIPT, never a timestamp (§4.1)> }
```

`candidate.type` ∈ {`host`, `srflx`, `relay`} and `candidate.substrate` ∈ {`tcp`, `quic`, `webrtc`} (unit #1 §4;
`quic`/`webrtc` are declared-but-unbuilt in v1 — §5). Naming (`system/nat/*` vs `system/signaling/*` vs
`system/connectivity/*`) is an open item (§10); kept as `system/nat/*` matching the source DRAFT.

> **`fire_at` is a delay, not an instant — corrected 2026-07-29.** This table previously read
> `<carrier-RTT-derived instant>`, which is where an implementer builds the wire shape from, and it said the
> wrong noun. **`entity-core-rust` built it from this line as a signed `i64` of milliseconds-since-epoch** — it
> round-tripped green, because the failure "passes every same-host test" (both peers read the same clock). §4.1
> pins the domain and §9 carries it as conformance MUST #5; this table now agrees with both. Recorded rather
> than silently patched, because the existence proof is the argument for §4.1 being a MUST at all.

### 3.1 The blob framing — what `offer` actually carries `[cross-peer seam — MUST pin]`

The node stores an opaque byte string and never decodes it (`PROPOSAL-CONNECTION-NODE` §1). **What those bytes
are is therefore a peer-side pin — and it was missing.** The Rust Stage-1 build had to choose one in order to
ship, which is the signal that the choice belongs here instead.

> **Pin:** the blob is the **canonical V7 entity wire encoding** of the coordination entity —
> `{type, data, content_hash}` — with `data` embedded **verbatim**.

Both halves are load-bearing:

- **The type must travel with the blob.** A bucket is a mixed set and the reader has nothing else to dispatch on.
  `connect-request` and `connect-response` differ by a single field *name* (`initiator` vs `responder`), so a
  reader handed bare `data` is reduced to sniffing map keys to guess what it is holding.
- **`data` is embedded raw, never decoded-and-re-encoded** (§1.8 byte preservation; re-encoding on
  receive/forward is *the* interop hazard). This is not hygiene here — the messages are signed (§3.3), so a
  re-encode round trip through the carrier silently invalidates the signature.

A blob that fails to decode is **not an error** — see §3.2.

### 3.2 Reading a bucket — the filters, all MUST `[cross-peer seam]`

`collect` is non-destructive (`PROPOSAL-CONNECTION-NODE` §1.1 pin 1) and `tag`/`lobby` buckets are shared, so
**every poll returns a mixture**: your own offer (you always re-read what you wrote), the answer you want, other
pairs' traffic, and — MUST-ignore — message types a newer impl introduced. This is where a naive implementation
breaks, so the reading rules are pinned rather than left to each impl:

- **A peer MUST skip its own messages** (`initiator`/`responder` equal to self). Otherwise a peer answers its own
  `connect-request` and "succeeds" at meeting itself — a failure that is miserable to diagnose because every
  individual step reports success.
- **A peer MUST correlate a response by the nonce echo.** In a shared bucket it is the only thing distinguishing
  *your* answer from someone else's, or a fresh answer from one already processed two polls ago.
- **An undecodable or unrecognized blob MUST be skipped, never treated as an error.** Anyone may write to a shared
  bucket, and a newer impl's message type is a MUST-ignore, not a fault.
- **Bucket order is deposit order, oldest first** (`PROPOSAL-CONNECTION-NODE` §1.1 pin 4). *Which* of several
  requests to answer in a `lobby` bucket is **peer policy** and deliberately not pinned.

The nonce is a **correlator, not a secret** — anyone who can `collect` the bucket can read it and echo it. That is
what §3.3 is for.

### 3.3 The signature — how it travels, and the two things it does not buy `[security — MUST pin]`

This section's heading has said "signed" since it was written. What was never pinned is **how the signature
reaches the verifier**, and the ecosystem's normal answer is *structurally unavailable* here.

Everywhere else, a signature binds at the V7 invariant pointer
`/{signer_peer_id}/system/signature/{target_hash_hex}` (`EXTENSION-IDENTITY` §6.2, `EXTENSION-ATTESTATION` §4.0)
and reaches a verifier by cross-peer sync or `envelope.included` ingestion. **At rendezvous time neither exists** —
the whole purpose of the carrier is that the two peers have no connection yet.

> **Pin:** the coordination blob is **self-contained** — the entity plus a **detached signature carrying the
> signer's `public_key`**. A verifier (a) recomputes `Base58(0x01 ‖ 0x01 ‖ SHA-256(public_key))` and checks it
> equals the claimed `initiator`/`responder` peer-id, then (b) verifies the signature over the entity's content
> hash. **A message failing either check MUST be skipped exactly as an undecodable one is** (§3.2) — never acted
> on, never an error.

This verifies **for a stranger**, with no key lookup and no prior contact, because a peer-id *is* a commitment to
the public key (`ENTITY-SYSTEM-REFERENCE`: `PeerID = Base58(0x01 ‖ 0x01 ‖ SHA-256(public_key))`). It is the same
self-contained shape `system/protocol/connect/authenticate` already uses at the handshake.

**Stated as an exception on purpose.** This is not a parallel signature-storage convention of the kind
`EXTENSION-IDENTITY` §6.2 forbids — the invariant-pointer convention is not being *replaced*, it is *unreachable*:
there is no session to sync over and no tree to bind into. Recording it as an exception is what keeps it from
being read as the extension-private-sibling-path defect class.

**What the signature does not buy** — both are easy to over-read:

1. **It authenticates the author, not the address.** An attacker signs *its own* candidate list containing a third
   party's address perfectly well. A candidate is evidence of nothing beyond "someone holding this key suggested
   this address."
2. **So the punch needs anti-amplification hygiene independently of it.** A peer MUST cap unsolicited traffic to a
   candidate at a small fixed probe budget and MUST NOT send more until that address answers. Absent this, an open
   `lobby` bucket is a traffic amplifier aimed by whoever fills it. (The node-side half is
   `PROPOSAL-CONNECTION-NODE` open item 2.)

What bounds the whole section: **the key introduces, it never authorizes** (§2.2). The punched connection still
runs the ordinary handshake and capability flow, so the worst case here is a wasted dial, not a compromised
session.

**Staging.** Stage 1 does not punch, so an unsigned coordination message is a **known Stage-1 gap, not a Stage-1
blocker** — the Rust carrier shipped without signing and does not need rework to be correct for what it does.
Signing and the probe budget MUST both land **before any punch is attempted from a `tag`/`lobby` bucket**, which
puts them in Stage 2 alongside the socket work.

## 4. The flow (concrete)

1. **Both gather candidates** (unit #1 §4): each learns its `srflx` mapping from a reflector (unit #1 §2) and
   lists `host` / `srflx` / `relay`.
2. **Exchange over the rendezvous carrier:** A `:offer`s `connect-request` at `rendezvous_key`; B `:collect`s it,
   `:offer`s `connect-response`; A `:collect`s. Each now holds the other's candidate list. (Both reached the same
   server via the §3.1 rendezvous-hash.)
3. **Measure the carrier round-trip; schedule the simultaneous open.** A measures RTT-through-the-carrier and sends
   `punch-sync` with `fire_at` so **both start sending to each other's `srflx` at ≈ the same instant** (A fires
   after sending sync; B fires on receiving it, ~½ RTT later → they cross in the middle). Getting `fire_at` right
   is what makes the holes line up.
4. **Simultaneous open:** each side sends to the other's `srflx`. Each side's *outbound* packet punches its own
   hole; the other's packet, arriving after the hole is open, gets through. After the crossfire both NAT mappings
   exist → a direct path is open.
5. **Upgrade & drop the carrier:** the direct connection becomes the live transport (§5); the peers stop using the
   signaling carrier (keeping it only if the direct link drops).
6. **If the punch fails** (no direct path within a timeout — symmetric/CGNAT): **fall back to the `relay`
   candidate** — the opt-in Mode-C/Mode-S tier (§7). Correctness preserved; only the direct-path optimization is
   lost.

### 4.1 The `fire_at` derivation `[cross-peer seam — MUST pin]` (2026-07-28)

Two peers must fire at ≈ the same instant, each computing that instant **independently** — the same shape as §2.2
and `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` §3.1.1, and it gets the same treatment: **pin the shape here, let
the live gate settle the numbers.** (Closes the §10 open item, which had the direction right and no pin.)

**Clock domain — relative, never absolute (MUST).** `fire_at` is a **delay from the receiving peer's moment of
receipt of the `punch-sync` entity**, encoded as **unsigned integer milliseconds**. It is *not* a timestamp. V7
assumes no synchronized clocks, and two peers' wall clocks can differ by far more than the entire punch window —
so a wall-clock instant on the wire is a silent cross-peer failure. It is also **invisible to a same-host test**,
where both peers read the same clock: exactly the masking shape §2.2 exists to warn about.

**Who measures, who fires:**

- **A measures `rtt`** — the round trip *through the carrier* (its own `offer` → the `collect` that returns the
  response). That is the only latency estimate either peer has, since by construction neither can yet reach the
  other directly.
- **A fires at `d` after *sending*** the `punch-sync`; **B fires at `d` after *receiving*** it. The sync traverses
  roughly half the carrier round trip, so the two firings cross near the middle.
- **`d` MUST be ≥ the observed one-way carrier latency (`rtt/2`)** — otherwise B's fire time has already elapsed
  when the message lands, and the two sides never overlap.

**Defaults — tunable, `[K]` industry-typical, not measured here:** `d = max(rtt, 250 ms)`; up to 3 attempts, each
with a fresh nonce; then abandon to the relay fallback (step 6). These are the part the live gate settles.

**What stays local and MAY diverge:** how a peer measures `rtt` (sample count, smoothing), per-candidate probe
pacing, socket options, the retry counts and timeouts above. §7's conformance line says as much — but read alone
it says `fire_at` *tuning* is local, which must not be misread as licensing a different **encoding or clock
domain**. Those two are the interoperable surface and are pinned above.

## 5. The substrate — TCP-simopen v1, our-QUIC the sequenced upgrade; the punched connection is an ordinary transport

**The load-bearing reframe (unit #1 §1, now acted on): a punched connection is not a new transport type.** Once a
candidate pair connects, the result is a **live full-duplex transport**, indistinguishable to the entity layer
from a dialed `tcp` connection — it slots into the §10 full-duplex-listener class and carries EXECUTE/TREE_GET/etc.
identically. **Two transport-layer-only differences from a normal dial:**

1. **Establishment** is the synchronized simultaneous-open (§4), not `connect()`/`accept()`.
2. **Liveness requires keepalives (MUST).** A punched NAT mapping expires after seconds-to-minutes of silence,
   closing the hole. NETWORK §5 keepalive already exists; **a punched connection MUST use it** (an idle punched
   connection silently dies). A peer MAY record `punched: true` as a connection attribute purely to drive this
   discipline — **not** a new entity-layer transport type.

| Substrate | `candidate.substrate` | v1 | Notes |
|---|---|---|---|
| **TCP simultaneous-open** | `tcp` | **✅ buildable now** | Reuses the existing `tcp` profile — both sides SYN at `fire_at`. The punched result *is* a `tcp` live transport. **NAT caveat `[K]`:** many NATs handle simultaneous *TCP* open worse than UDP (fine on cone, flakier elsewhere). Chosen first because it reuses what ships — **but see the port-reuse correction below; "reuses what ships" was undercounted.** |
| **our UDP/QUIC** | `quic` | declared, **unbuilt** | Punches more reliably; the modern default. **Gated on building our own QUIC transport first** (`quic` is aspirational in NETWORK §6.5.2, L797). Study Iroh's QUIC+traversal design as prior art; **build our own** (we build, we don't depend). Sequenced *after* TCP-simopen proves the coordination protocol. |
| **WebRTC data channel** | `webrtc` | unit #3 | The browser's only P2P transport; the browser's own ICE stack punches, driven by our signaling. `PROPOSAL-EXTENSION-WEBRTC-TRANSPORT`. |

The state/resource silhouette (reflect / signal / punch / relay-fallback) is **substrate-independent** — identical
for `tcp`, our-`quic`, or `webrtc` — so the coordination + infra design carries across whichever substrate we build.

**Port-reuse correction — TCP-simopen is not free** (2026-07-29, `entity-core-rust`)

*(Deliberately not numbered `§5.1`: across the connectivity proposals a bare "§5.1" already means
`PROPOSAL-CONNECTION-NODE` §5.1, the unwrapped protocol. Cite this as "signaling §5, port-reuse correction.")*

`REACHABILITY-FACTS` §4.2 (a `srflx` candidate MUST be the mapping of the socket the peer punches from), §2.1(b)
(the mapping is observed on an established connection), and this section's TCP-simopen choice **jointly** require
punching **from the reflector connection's own local port**:

| Substrate | What §4.2 costs there |
|---|---|
| **UDP / our-QUIC** | Trivial — one socket, reused for the reflector exchange and the punch. |
| **TCP simopen** | `SO_REUSEADDR`/`SO_REUSEPORT` **plus explicit bind** on both the reflector dial and the punch dial. **Not satisfiable by discipline** — no amount of careful code substitutes for the socket options. |

**What this does and does not change.** §4.2 is substrate-independent and stands. The G1 rationale does not:
"chosen first because it reuses what ships" counted the `tcp` transport profile and did not count the socket
work, so the gap between the two substrates on this axis is **smaller than G1 assumed, and in QUIC's favour.**
The other half of G1's rationale — that our-QUIC does not exist yet and TCP does — is untouched and is still
the larger term. **G1 is an operator call and is not re-decided here**; this records the corrected cost so the
call is re-made on real numbers if it is re-made at all. Logged in §10 as an open item.

**Also corrected, same round:** adopting a punched connection needs **no new transport seam** — Rust withdrew a
wider earlier claim after checking, `handle_connection` being public and documented as accepting any transport's
`Connection`. **Only the dial side is missing.** The punch is narrower than the previous packet implied.

## 6. Dispatch composition (how it slots onto §10)

A peer reaching target D, with traversal, is an **extension of the existing §10 dispatch loop**, not a parallel
system:

1. §10 runs as today: active connection? held cap (§6.6)? durable transport profiles in `(priority asc, profile-id
   lex)` order?
2. **If D has no dialable durable profile** (D is NAT'd — its published profiles are unreachable), the dispatcher
   escalates to **traversal**: gather candidates (unit #1 §4), run the §4 dance over the rendezvous carrier, try
   `host` → `srflx` → `relay`.
3. **First candidate pair that connects wins.** `host`/`srflx` → a direct live transport (§5); `relay` → the
   opt-in fallback (§7).
4. The result is **a live transport the rest of §10 uses unchanged.**

This is the live-punch rung the **§10.2 seam** reserved (L1334: "later, the live-connection punch … deliver now") —
filling a reserved rung, not re-architecting.

## 7. The opt-in heavier tier (dedicated peers, not basic coordination)

Per §0, everything past the cheap coordination primitive is **opt-in capacity a dedicated peer volunteers**,
advertised via `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` (`data_relay` / `inbox_relay`, with a `policy` budget
lever), **never** part of the always-on infra:

- **RELAY Mode-C — live relayed circuit** (the real-time fallback when a punch fails *and* the use-case needs
  liveness, e.g. a call between two symmetric-NAT peers). Entity types named for forward-compat
  (`system/relay/circuit-reservation` / `-dial`, RELAY §3.4/§11.1); **normative text deferred** to a RELAY
  follow-on, gated on a real driver.
- **RELAY Mode-S — async inbox** (offline/store-and-forward delivery). Already v1 in RELAY; the opt-in `inbox_relay`.
- **relay-forward (Mode-F) as an opt-in *signaling* carrier** — for a deployment that prefers to route signaling
  through its RELAY rather than run a dedicated `system/signaling` endpoint. Same message schema (§3), carrier
  opacity per RELAY §9.

A deployment that runs **only** the basic tier (reflector + rendezvous) still connects the ~70–90% `[K]` of peer
pairs that punch successfully; symmetric-NAT pairs (~10–20% `[K]`) need someone to opt into a relay, or fall to
async.

## 8. Security (baseline audit — not deferred)

Inherits unit #1's reflection/dial-back rules (dial-back-to-observed-source-only, multi-reflector agreement) and
adds the coordination surface:

| Surface | Threat | Mitigation |
|---|---|---|
| **Candidate exchange (§3)** | Forged candidates → trick a peer into dialing an attacker/victim (connection-laundering, DDoS) | Messages are **signed entities** (V7 §5.2); a peer only punches toward candidates from the **authenticated, expected** counterparty. The connectivity check confirms the peer answering is the expected peer (**nonce + identity binding**) — a forged address that doesn't lead to the real peer simply fails the check. |
| **Signaling carrier (§2)** | Carrier reads/tampers with the exchange | Messages are **opaque to the carrier and signed end-to-end** (mirrors RELAY §9); the carrier can drop/delay (DoS) but cannot read or forge. Choose a carrier you trust as a *carrier*. |
| **Rendezvous server (§2.1)** | A malicious rendezvous server maps peers / floods a key | Only holds opaque blobs at a derived key; rate-limit per requester per key; TTL-reap. It learns *that* two peer-ids are pairing (metadata), never the content. For `tag`/`secret`/`lobby` keys it sees the derived `K`, never the underlying string. |
| **Shared-key rendezvous (§2.2)** | A guessed or low-entropy `tag`/`secret` lets an uninvited peer land in the same bucket | The key **introduces, never authorizes**: coordination entities are signed (§3) and the connection runs the full handshake + capability flow (bottom row). For actual access control use a **high-entropy `secret`** or the **capability-gated node (§2.3)**; a memorable label is discovery convenience, not a boundary. |
| **Punched transport (§5)** | Same as any live transport | Ordinary handshake/capability flow authorizes the connection once up (V7 §4.6). |

**Deliberately NOT re-derived (research, not invented):** the exact connectivity-check / consent-freshness
handshake (RFC 8445 §7 + RFC 7675) and symmetric-NAT port prediction. Mine the RFCs at build time. Flagged §10.

## 9. Conformance & v1 posture

- **v1 floor unchanged; this is roadmap, not v1-blocking** (the release shape — content sites, CDN, offline
  delivery — needs none of it). This lands as a coherent v1.x once unit #1's facts ship.
- **No wire-format change to the entity model.** New surface is a thin handler (`system/signaling`), three ordinary
  signed entity types, and a connection attribute (`punched`). No V7 bump; no new error code beyond
  capability-denied (403) / timeout reuse.
- **The cross-impl-observable surface to pin (MUST):**
  1. **Both peers derive `rendezvous_key` identically** via the mode-tagged derivation (§2.2:
     `H("entity:rdv:v1" ‖ mode ‖ SEP ‖ canonical(mode_input))`) and select the pool server by rendezvous-hash —
     divergence in the mode tag, the domain separator, the peer-id sort, or the **string bytes** = the pair
     never meets. **MUST** (shared with service-ad §3.1). String-mode inputs (`tag`/`secret`) are **byte-exact
     UTF-8** — no normalization, no case-folding (§2.2, the core §1.8 byte-preservation rule); both peers MUST hold
     the identical bytes.
  2. **The signaling carrier MUST NOT decode the coordination entity** — carrier opacity + byte-preservation
     (RELAY §9); a re-encoding carrier breaks the signature. **MUST.**
  3. **A punched connection MUST run keepalive** (§5) — else it silently dies; a cross-impl "works then drops"
     bug. **MUST.**
  4. **Connectivity-check identity binding** (nonce + expected peer-id, §8) before treating a punched path as the
     peer — divergence is a security hole. **MUST.**
  5. **`fire_at`'s clock domain and encoding** (§4.1) — relative-milliseconds-from-receipt, never a wall-clock
     instant. Divergence is a silent never-meet that passes every same-host test. **MUST.**

  Substrate specifics (TCP-simopen `fire_at` **tuning** — the `d` value, retry counts, timeouts) are local and MAY
  diverge. Read that as tuning only: the encoding and clock domain it is tuning *within* are pinned by §4.1.
- **The CDN-corridor meta-rule (the real gate):** none of this is validated until a **cross-impl conformance run
  exercises an actual punch between two conformant peers** — two impls, real NATs, a `connect-request`/`-response`
  over a real rendezvous, a `fire_at` simultaneous-open that actually connects, and a keepalive that actually holds
  the mapping. That run — and a real entity-chat exchange riding it — is the best integration test the whole stack
  has. Prose review does not catch these.

## 10. Open items

- **G1 re-check — TCP-simopen's port-reuse cost (§5 port-reuse correction, NEW 2026-07-29).** `REACHABILITY-FACTS` §4.2 makes
  TCP-simopen require `SO_REUSEADDR`/`SO_REUSEPORT` + explicit bind, which G1 did not count when it chose TCP
  "because it reuses what ships." **Operator call; nothing here is blocked on it** — the coordination protocol is
  substrate-agnostic (§5) and §4.2 holds either way. *Arch lean: G1 stands* — the socket options are bounded,
  known work, and "our-QUIC does not exist yet" is still the dominant term. Re-open only if the operator wants
  the substrate order re-weighed.
- **Message namespace** — `system/nat/*` (source DRAFT) vs `system/signaling/*` vs `system/connectivity/*`. The
  reframe says connectivity isn't NAT-specific; *leaned: keep `system/nat/*` for the punch-coordination types,
  `system/signaling` for the carrier handler.* Cohort call at fold.
- **Dedicated handler vs entity-native namespace** for the rendezvous carrier (§2.1) — ship one or both. *Leaned:
  dedicated handler v1, namespace as scale-out.*
- **QR/out-of-band carrier** — the operator flagged it for review (§0). Confirm its message-framing (a
  `connect-request`/`-response` pair encoded into a QR/short-code) and whether it is v1 or a fast-follow.
- **Consent-freshness / connectivity-check handshake + symmetric-NAT port prediction** — research (RFC 8445 §7,
  RFC 7675); do not invent. Gates the production-correct check sequence.
- ~~**`fire_at` derivation** — the carrier-RTT measurement + clock model (relative offset, not wall-clock —
  respects the V7 no-synced-clock non-goal). Pin the relative-instant convention at fold.~~ **CLOSED 2026-07-28 —
  pinned in §4.1** (relative ms from receipt, uint; A fires on send, B on receipt; `d ≥ rtt/2`; the `d` value and
  retry counts stay local tunables). Raised by `entity-core-rust` as a punch-path blocker: correct direction, no
  pin, and it could not be built from a lean.
- **String-key encoding (§2.2) — DECIDED (2026-07-24): byte-exact UTF-8, no normalization / no case-folding**
  (the core §1.8 byte-preservation discipline applied to an independently-derived key; charset policy is the app's
  — recommend lowercase-ASCII + digits + `-` for human tags, generated high-entropy ASCII for secrets). Not a core
  change. ~~The only residual fold pin is fixing the `mode` tag spellings and the `SEP` domain-separator bytes as
  constants — a trivial cohort call at fold.~~ **CLOSED 2026-07-28:** §2.2 as revised pins `SEP = 0x1F` *and* its
  two positions, and pins the exact ASCII tags `pair`/`tag`/`secret`/`lobby`; §2.2.1's stage bisect publishes the
  bytes. **No residual, and no cohort call at fold** — this bullet outlived its subject and was telling an
  implementer who read §10 before §2.2 that two pinned variables were still open.

## 11. References

- **Brought forward from (read-only):**
  `PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY.md` (the internal legacy corpus, read-only)
  §4/§5/§6/§7 (coordination, separable services, fallback, dispatch); `…/PROPOSAL-EXTENSION-WEBRTC-TRANSPORT.md`
  (signaling gap — unit #3).
- **Reconciled under:** `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md` Parts B/D, Part F steps 2–3,
  Part H unit #2; `ANALYSIS-INFRA-HORIZONTAL-SCALING-AND-STATE-COORDINATION.md` Part E (rendezvous-hash).
- **Depends on:** `PROPOSAL-NETWORK-REACHABILITY-FACTS.md` (unit #1 — reflection, dial-back, candidates).
- **Complementary:** `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT.md` (advertises `signaling` pool + optional
  `data_relay`/`inbox_relay`; §3.1 rendezvous-hash selection).
- **Current spec anchors:** `EXTENSION-NETWORK.md` §5 (keepalive), §6.5 (transport profiles / `quic` aspirational
  L797), §10 + §10.2 (dispatch + the reserved live-punch seam, L1296–1304 / L1334); `EXTENSION-RELAY.md` §3.1
  (Mode-F), §3.4/§11.1 (Mode-C deferred), §6.2.1 (Mode-S held-outbound), §9 (opacity / byte-preservation).
- **Reference systems (decomposition we follow, not depend on):** libp2p DCUtR / Circuit-Relay-v2; WebRTC ICE
  (RFC 8445), STUN (RFC 5389), TURN (RFC 8656), consent (RFC 7675); Iroh (QUIC + traversal, prior art for our-QUIC).
  `[K]` industry-typical, not spec-pinned.
