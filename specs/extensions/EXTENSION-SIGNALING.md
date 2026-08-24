# Signaling Extension — Normative Specification

**Version**: 1.0

**Status**: Draft
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.9+), EXTENSION-NETWORK.md (v1.6+ — §6.7 reachability facts, §10.3 live-establishment seam)
**Optional**: EXTENSION-REGISTRY.md (v1.2+) — service advertisement for reflector/signaling pools; EXTENSION-RELAY.md (v1.2+) — Mode-F as an alternate carrier, Mode-C/S as fallbacks
**Encoding**: ENTITY-CBOR-ENCODING.md (ECF)

---

> **Cross-reference notation.** A bare `§N` in this document refers to **this** specification. References to another spec always carry its filename (`EXTENSION-NETWORK.md §10.3`) — several of this document's section numbers collide with NETWORK's, so the qualifier is load-bearing, not decoration.

> **Cohort review absorbed 2026-07-31 (same day as v1.0), fixed in place — no rev bump.** Two implementations built the §7 punch against the committed text and reported back. Three additions, each closing something the text left open rather than changing something it said: **§7.4.1** pins the **handshake role** — after a punch *both* sides dialed, so nothing in the socket pair says who sends HELLO, and two implementations resolving it differently hang a connection that punched perfectly (both had independently chosen initiator-as-client, and both asked for it to be normative); **§7.2** pins how the punch's attempt budget composes with `EXTENSION-NETWORK.md` §4.1's reconnect backoff, because nested retry loops multiply onto **third-party** carrier and reflector peers; **§7.3.1** states the substrate matrix and the native↔browser model, which the §7.3 table implied but never said. Per `AGENTS.md`, cohort findings on a landed spec fix it in place. Two coordination drifts (`priority` on candidates; `advertise` limits shape and the `ttl_seconds`-vs-`bucket_ttl_ms` unit) were reported against §6.1/§4.5 and needed **no edit** — the committed text was already right and both traced to a pre-v1.0 brief; see `docs/status/ROUTING-2026-07-31-punch-rulings-to-cohort.md`.

> **Second review round, same day — one defect found in the text above, in the text this section had just added.** Building the new §7.2 composition rule, an implementation mapped "exactly one attempt" onto its **crossing** retry counter instead of its **exchange** counter and broke a working punch. The word *attempt* was unqualified, and a punch has two nested retry layers — one that costs only the two peers, one that costs third-party reflectors and carriers. **New §7.2.1** names both layers, pins the MUST to the exchange layer, and adds a MUST NOT against starving the crossing. *A rule folded to prevent load on third parties nearly caused a different implementation to break its own traversal — which is the argument for stating the **rationale** in normative text, since that is what let the finding be diagnosed once the counter was in the wrong place.*

> **Path notation.** Paths use peer-relative notation (without leading `/{peer_id}/`), resolving to the local peer's namespace. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model.

## 1. Overview

Two peers both behind NAT cannot dial each other, and there is no public party holding a socket to either. This extension provides the one thing that fixes it: a **mutually reachable third party that carries an opaque exchange** between them long enough to coordinate a direct connection.

It introduces; it never carries data. The service holds an opaque blob at an opaque key for a few seconds. It never decodes an entity payload and never learns who met whom beyond a hash.

### 1.1 Scope

This extension covers:

- **The rendezvous mailbox** — `offer` / `collect` / `advertise` (§4), and the bucket semantics that make a retry safe (§5).
- **The rendezvous key** — one mode-tagged derivation, four modes (§3).
- **The coordination messages** — candidate exchange and punch synchronization, signed and carrier-opaque (§6).
- **The punch** — the simultaneous-open dance and its timing (§7), registered behind `EXTENSION-NETWORK.md` §10.3.
- **The unwrapped surface** — the non-entity protocol a public deployment speaks (§9).

It does **not** cover:

- **Reachability facts** — observed-address reflection, dial-back, and candidate *gathering* are `EXTENSION-NETWORK.md` §6.7. Gathering a candidate is a local fact; **exchanging** one is this extension's protocol. Keeping them apart is the clean seam.
- **Transport establishment semantics** — that a punched connection is an ordinary transport and MUST run keepalive is `EXTENSION-NETWORK.md` §10.3.
- **The browser leg** — WebRTC's SDP/ICE schema over this carrier is a separate spec; browsers reach the carrier through the wrapped surface (§2.2).
- **Relayed data paths** — RELAY Mode-C (live circuit) and Mode-S (async inbox) are the opt-in fallbacks when a punch fails (§10).

### 1.2 Design Principles

**The key introduces; it never authorizes.** Reaching the same rendezvous key gets two peers *met*. The coordination messages are still signed end-to-end (§6.3), and the resulting connection still runs the ordinary handshake and capability flow. A shared key gets you to the meeting point; it never authorizes the session.

**The service is mode-blind and content-blind.** It compares opaque bytes and stores opaque bytes. All four key modes fall out of one keyed mailbox — *one code path exercised with four derived keys, not four features.*

**Every default is the one that makes a retry safe.** A rendezvous that fails silently is the failure mode this whole surface exists to avoid, so where a choice existed, §5 takes the option that survives a retry.

**Losing the service loses nothing that mattered.** State is per-key TTL-reaped buckets and nothing else: no bulk storage, no cross-node coordination, no presence map, no durable data. In-flight handshakes drop and peers retry. This is what makes it cheap enough to be a universal fallback and trivial to scale horizontally.

## 2. Roles and Surfaces

### 2.1 Roles `[departs from the RELAY precedent — deliberate]`

`EXTENSION-RELAY.md` §10.1 sets the ecosystem pattern: *a conformant implementation MUST implement the service role; a deployment MAY choose to enable it.* **This spec departs from that:**

> **The server role is OPTIONAL for a conformant implementation. The client role is the conformance surface.**

RELAY is a **peer capability** — any peer may relay for a neighbor, so every implementation needs it. Signaling is **deployed infrastructure** with a deliberately non-entity hot path; nobody will run a Python rendezvous box. Requiring three server implementations is work with no consumer.

**The cost, stated:** a single-implementation server means **underspecification stays invisible** — one implementation cannot disagree with itself. Two mitigations, both load-bearing: §5 pins the bucket semantics *in advance*, and the **clients are independently written in three languages**, hitting the server from outside. That is a real convergence signal on the semantics even without a second server. A further question of the §5 kind surfacing during a build comes back here as a spec fix; it is not settled in one implementation's code.

### 2.2 Two surfaces, one core

The three verbs are offered on either or both of two surfaces:

| Surface | Transport | Admission |
|---|---|---|
| **Wrapped** | ordinary entity dispatch (EXECUTE to `system/signaling`) | the capability model (§8) |
| **Unwrapped** | the non-entity protocol of §9 | none — the rate limiter is the entire admission story (§8.2) |

> **MUST.** The three verbs are **semantically identical on both surfaces.** A verb completable on one and not the other is non-conformant. The core is implemented once; a surface is a front end over it, never a redefinition of it.

Browser peers use the **wrapped** surface (they cannot open raw TCP); native peers avoiding entity overhead use the unwrapped one. A deployment MAY offer only one.

## 3. The Rendezvous Key

The whole discovery surface is *one* keyed rendezvous. The "modes" are only **how two peers agree on the key** — so adding a mode is picking an input, not adding a mechanism.

### 3.1 Derivation `[cross-peer seam — MUST]`

```
payload        = "entity:rdv:v1" ‖ SEP ‖ mode ‖ SEP ‖ canonical(mode_input)
rendezvous_key = varint(0x00) ‖ SHA-256( ecf_for_hash( "system/signaling/rendezvous-key", cbor_bstr(payload) ) )
```

> **MUST.** Both peers produce a byte-identical hash input **and** byte-identical key bytes.

Divergence here means the two peers derive different keys and **silently never meet** — and that failure is invisible to a same-implementation test, so every free variable is pinned:

- **The hash is the substrate content-hash primitive** (ENTITY-CORE-PROTOCOL.md §1.2), which hashes ECF-encoded `{data, type}`, **not** a bare byte string. Its two inputs are pinned: `type` is the fixed string **`system/signaling/rendezvous-key`**, and `data` is **`payload` wrapped as a single CBOR byte string** (`bstr`, minimal-length head). A `tstr` wrapping, a multi-element array, or a bare unwrapped concatenation each yield a different digest.
- **The digest format is pinned to the SHA-256 floor** (`content_hash_format` `0x00`, ENTITY-CORE-PROTOCOL.md §8.2) — **not the deriving peer's home format.** A content hash is self-describing and format-carrying, so two conformant peers running different home formats would derive different keys for the same agreed input and never meet, with nothing failing loudly. **A rendezvous key is a lookup token two independent parties must reproduce, not authored content**, so it does not follow the authoring peer's format. It stays in the self-describing wire encoding (`varint(format) ‖ digest`, 33 bytes at the floor) so a future format migration remains expressible, and it is compared **byte-wise**.
- **`SEP` is `0x1F`** (ASCII US), appearing **after the domain string and after the mode tag** — not only before the input.
- **`mode`** is one of the exact ASCII tags `pair` / `tag` / `secret` / `lobby`.

### 3.2 The four modes

| `mode` | `canonical(mode_input)` | Who meets | Secrecy of the input |
|---|---|---|---|
| `pair` | the two peer-ids **byte-wise sorted ascending**, joined **with `SEP` between them** (`lo ‖ SEP ‖ hi`) | exactly those two peers | public — identity *is* the key |
| `tag` | a **byte-exact UTF-8** label (e.g. `chess`, `my-family`) | anyone who knows the tag | **public label** — discovery convenience, *not* access control |
| `secret` | a **byte-exact UTF-8** agreed string, treated as high-entropy | anyone who knows the secret | **confidential** — a lightweight admission gate |
| `lobby` | a fixed well-known constant — **`lobby:default`** unless the pool advertises another | anyone on that service, right now | public — "connect me to anyone here" |

**Three sub-pins the table carries, all silent-never-meet bugs otherwise:**

- **`pair` joins the sorted peer-ids with a separator.** Bare concatenation is ambiguous: `sorted("ab","c")` and `sorted("a","bc")` both yield `abc`, so two *different* pairs would share a bucket.
- **`lobby` names an actual default constant.** "Per deployment" alone means two peers pointed at the same open service still never meet. A pool MAY advertise a different constant via `advertise` (§4.5); absent that, the value is `lobby:default`.
- **`pair` ids are canonical `system/peer-id`s — one namespace `[cross-peer seam — MUST]`.** Each of the two inputs is that peer's canonical `system/peer-id` (the Base58 form §6.3 pins), byte-for-byte — **never** an alternate encoding of the same identity (an identity-entity `content_hash`, a different digest base). This mode's `canonical(mode_input)` **and** the §6.5 glare/offerer sort consume these ids **byte-exact** (§3.3) and by construction cannot detect a namespace mismatch, so mixing encodings fails *twice*, both silently: the derived keys differ (`pair_key(A_pid, B_hash) ≠ pair_key(B_pid, A_hash)` — never meet) **and** every Base58 id sorts below every `ecfv1-…` hash, collapsing glare to *both-Impolite / both-offer* (§6.5, offerer determination). A caller that holds a non-canonical identifier — e.g. it resolved a target URI to the counterpart's identity-entity hash — **MUST** canonicalize to the peer-id **before** deriving the key or computing the role. The guard belongs at the caller / `establish_live` seam, where the mode is known — **not** inside `pair_key`/glare, which stay mode-generic and byte-exact because `tag`/`secret` legitimately carry arbitrary labels (§3.3). *(Found live: `entity-browser-rust`'s two-browser rig stalled at `included_count=0` with no error anywhere; `entity-core-rust` proof-tabled it with real keypairs and routed it — see `ROUTING-2026-08-03-id-encoding-contract-to-arch`. Making the implicit contract explicit is what turns a silent never-meet into a findable one.)*

### 3.3 String inputs are byte-exact (MUST)

> **MUST.** No Unicode normalization. No case-folding. Both peers hold the identical bytes.

This is not a new rule — it is the core's **byte-preservation** discipline (ENTITY-CORE-PROTOCOL.md §1.8) applied to a key two peers derive *independently* from an agreed string. It surfaces here because this is the first surface where two parties must produce the *same* hash from a string each supplied on its own; everywhere else a string is authored once and its bytes are preserved.

So the protocol stores no Unicode database and runs no normalization step. **Charset policy is the application's job:** an application that lets a human type a `tag` constrains its input (recommended: lowercase ASCII letters, digits, and `-`), and a `secret` is a generated high-entropy ASCII string exchanged verbatim.

### 3.4 What a shared key does and does not buy

`tag`, `secret`, and `lobby` are **the same mechanism** — a hash over an agreed string. They differ only in the entropy and secrecy of that string, and therefore in what property you get. Three consequences, all security-load-bearing:

- **A `secret` is only as strong as its entropy.** A short word is enumerable — anyone who guesses it lands in the same bucket. A `secret` used as a gate MUST be high-entropy; a human-memorable phrase is really a `tag`. The **capability-gated wrapped surface (§8.1)** is the real access control; a `secret` key is a zero-infrastructure gate layered on top, not a replacement.
- **The key introduces; it does not authorize** (§1.2).
- **Presence is concurrent, on one pool.** Buckets are TTL-reaped, so a `tag` / `secret` / `lobby` meeting needs both peers present in the same window on the **same provider pool**.

> **MUST — same provider.** Both peers of a handshake meet at the same provider. Every mode rendezvous-hashes the key into the configured pool (`EXTENSION-REGISTRY.md` service advertisement), so provider selection is invariant across modes. **A provider mismatch is a silent never-meet exactly like a key-derivation mismatch.**

### 3.5 Implementer diagnostics (informative)

Authored expected keys are deliberately **not** published here: which bytes are correct is settled by implementations meeting or failing to meet in a live cross-implementation run, not by one author's oracle. What is useful up front is the ability to bisect a mismatch to a stage.

**Stage bisect** — `mode = tag`, input `chess`:

```
payload            : "entity:rdv:v1" 1F "tag" 1F "chess"
                     656e746974793a7264763a76311f7461671f6368657373
cbor bstr(payload) : 57 656e746974793a7264763a76311f7461671f6368657373
ecf_for_hash       : a2 64 64617461 <bstr…> 64 74797065 7819 73797374656d2f6e61742f72656e64657a766f75732d6b6579
key                : 0x00 ‖ SHA-256(ecf_for_hash)
```

Two implementations that disagree compare these four lines and know immediately whether they differ on the concatenation, the CBOR framing, the ECF envelope, or the digest.

**Properties requiring a differential test** — each is a way to be self-consistent and still never meet:

| Property | Test | A failure means |
|---|---|---|
| order independence | `pair(A,B)` == `pair(B,A)` | the sort is missing or not byte-wise |
| pair disambiguation | `pair("ab","c")` != `pair("a","bc")` | the `SEP` join is missing — two different pairs share a bucket |
| case sensitivity | `tag("chess")` != `tag("Chess")` | the implementation case-folds |
| id canonical form | both sides feed the Base58 `system/peer-id` (§6.3), never an identity-entity hash | a peer fed its counterpart's `content_hash` — mixed namespaces never meet *and* both offer (§3.2 third sub-pin) |
| no normalization | `tag(NFC "café")` != `tag(NFD "café")` | the implementation Unicode-normalizes — the likeliest accidental import, since many string libraries do it by default |
| mode separation | `tag("x")` != `secret("x")` | the mode tag is missing from the payload |
| **fixed format code** | every key is **33 bytes**, first byte `0x00` | the implementation used its **home hash format** — a SHA-384-home peer never meets a SHA-256-home peer, and passes every other check above |

The last row catches a real bug in an otherwise-correct implementation.

## 4. Handler and Operations

### 4.0 Handler manifest

```
system/handler := {
  pattern:    "system/signaling"
  name:       "signaling"
  operations: {
    offer: {
      input_type:  "system/signaling/offer-request"
      output_type: "system/signaling/offer-result"
    }
    collect: {
      input_type:  "system/signaling/collect-request"
      output_type: "system/signaling/collect-result"
    }
    advertise: {
      input_type:  null
      output_type: "system/signaling/advertise-result"
    }
  }
  internal_scope: [
    {handlers: {include: ["system/tree"]}, resources: {include: ["system/signaling/*"]}, operations: {include: ["get", "put"]}}
  ]
}
```

Operation type names follow the canonical `{handler-path}/{op-name}-request` / `-result` convention (ENTITY-CORE-PROTOCOL.md §3.7).

**Service-owning declaration.** A peer offering the **unwrapped** surface (§9) owns a listener the entity system does not mediate, and therefore **MUST** publish a service declaration per `SDK-OPERATIONS.md` §11.6.9 — `kind: "network-listener"`, `exposure: "unmediated-public"`, with the bind address and port in `descriptor`. A peer offering only the wrapped surface declares `exposure: "entity-mediated"`, or nothing at all if it owns no listener. This is the motivating case for that contract: an operator must be able to read *what is listening on this peer and which handler owns it* off the tree, without reading the implementation.

### 4.1 Type definitions

```
system/signaling/offer-request := {
  fields: {
    rendezvous_key: {type_ref: "primitive/bytes"}   ; exactly 33 bytes, opaque (§3.1)
    message:        {type_ref: "primitive/bytes"}   ; opaque blob (§6.2)
  }
}

system/signaling/offer-result := {
  fields: {
    ok: {type_ref: "primitive/bool"}
  }
}

system/signaling/collect-request := {
  fields: {
    rendezvous_key: {type_ref: "primitive/bytes"}   ; exactly 33 bytes, opaque
  }
}

system/signaling/collect-result := {
  fields: {
    messages: {array_of: {type_ref: "primitive/bytes"}}
              ; deposit order, oldest first (§5 pin 4); empty list when absent
  }
}
```

### 4.2 Operation summary

```
system/signaling:offer(rendezvous_key, message)  → { ok }
system/signaling:collect(rendezvous_key)         → { messages: [<blob>, ...] }
system/signaling:advertise()                     → { endpoint, limits }
```

### 4.3 Offer

Deposits an opaque blob at a key. `rendezvous_key` is **33 opaque bytes compared byte-wise** — the service derives nothing and knows nothing about the modes of §3. `message` is an opaque byte string whose framing is pinned peer-side (§6.2); the service never decodes it and never needs to.

Errors: `message_too_large` (§5 pin 5), `bucket_full` (§5 pin 5), `bad_request` (malformed input or wrong key length), `rate_limited` (§8.2).

### 4.4 Collect

Returns the blobs at a key, **oldest first** (§5 pin 4). Non-destructive (§5 pin 1). A key with no bucket returns an **empty list, not an error**.

> **`collect` returns the blobs themselves, never their hashes.** A hash reply would require a fetch surface the service structurally does not have (§1.2: no bulk storage), and an unwrapped client holds no entity machinery with which to resolve a hash to bytes. It would also make one verb completable on one surface only — the exact divergence §2.2 forbids.

### 4.5 Advertise

Announces the endpoint, its rate and TTL limits, and — if it overrides the default — its `lobby` constant, for pool membership (`EXTENSION-REGISTRY.md` service advertisement).

```
system/signaling/advertise-result := {
  fields: {
    endpoint: {type_ref: "primitive/string"}
    limits:   {type_ref: "system/signaling/limits"}
  }
}

system/signaling/limits := {
  fields: {
    max_blob_bytes:   {type_ref: "primitive/uint"}   ; default 8192  (§5 pin 5)
    max_bucket_blobs: {type_ref: "primitive/uint"}   ; default 32    (§5 pin 5)
    ttl_seconds:      {type_ref: "primitive/uint"}   ; default 60    (§5 pin 6)
    lobby_constant:   {type_ref: "primitive/bytes", optional: true}
                      ; present only if the deployment overrides `lobby:default`
  }
}
```

**Clients read the limits rather than assuming them.** A limit the client does not know is a cross-implementation reject boundary: one peer offers 64 KiB at a service that stops at 4 KiB, and the failure presents as a rendezvous miss.

## 5. Bucket Semantics `[single-implementation server — MUST pin]`

Six questions a lone server implementer answers by fiat and never re-asks. Because the server role is single-implementation (§2.1), nothing downstream surfaces a wrong answer — so they are pinned here, before code.

| # | Question | Pin | Why |
|---|---|---|---|
| 1 | Does `collect` **drain** the bucket? | **Non-destructive.** It returns what is at the key and removes nothing; TTL is the only reaper. Absent key ⇒ **empty list, not an error**. | A handshake has both peers polling, and a `pair` bucket may be collected by both sides and re-read on retry. A draining read makes a retry lose the peer's offer — a silent handshake failure indistinguishable from absence. |
| 2 | Does `offer` at an existing key **append or replace**? | **Append.** A key holds a set of deposited blobs, deduplicated by content hash. | `lobby` and `tag` are inherently multi-party; replace would make the last writer erase everyone. Dedup by hash makes a retry idempotent rather than an accumulation. |
| 3 | Is the advertised TTL **binding on peers**? | **Advisory to peers, binding on the service.** Peers MUST NOT assume a blob is still there and MUST be prepared to re-`offer`. | Peers cannot enforce a remote service's reaping, and a peer that treats TTL as a guarantee hangs instead of retrying. |
| 4 | In what **order** does `collect` return blobs? | **Deposit order, oldest first.** | A reader scans a bucket for the first message it can act on (§6.4). Two implementations scanning in different orders answer *different peers* out of one shared `lobby` bucket — a cross-peer divergence wearing the costume of a preference. |
| 5 | What are the **size bounds**? | **8 KiB per blob; 32 blobs per bucket.** An over-size `offer` is **refused with `message_too_large`**, never truncated; a full bucket refuses with **`bucket_full`** rather than evicting. | A handshake is ~1 KB, so 8 KiB is generous by 8×. **Refuse, don't evict** — eviction reproduces exactly the silent-never-meet shape this surface exists to avoid, while a refusal tells the offerer it failed and can retry. Filling a bucket to deny service is a rate-limit concern (§8.2), not a bucket-semantics one. |
| 6 | What **TTL** does a service default to? | **60 s**, configurable, published in `advertise`. | Bound by the lifetime of what the blob *describes*, not by service memory: a `srflx` candidate expires with the NAT binding that produced it (commonly 30–120 s), so a longer TTL only serves candidates that are already unpunchable. |

## 6. Coordination Messages

Three entity types carry the exchange. They are ordinary signed entities, opaque to the carrier, which sees only the 33-byte key.

### 6.1 Type definitions

```
system/signaling/connect-request := {    ; A → B
  fields: {
    initiator:  {type_ref: "system/peer-id"}
    candidates: {array_of: {type_ref: "system/network/candidate"}}   ; EXTENSION-NETWORK §6.7.3
    nonce:      {type_ref: "primitive/bytes"}    ; correlates the exchange; freshness
  }
}

system/signaling/connect-response := {   ; B → A
  fields: {
    responder:  {type_ref: "system/peer-id"}
    candidates: {array_of: {type_ref: "system/network/candidate"}}
    nonce:      {type_ref: "primitive/bytes"}    ; echoes the initiator's
  }
}

system/signaling/punch-sync := {         ; either direction
  fields: {
    nonce:   {type_ref: "primitive/bytes"}       ; echoes
    fire_at: {type_ref: "primitive/uint"}
             ; DELAY FROM RECEIPT in milliseconds — never a timestamp (§7.2)
  }
}
```

Candidates are `EXTENSION-NETWORK.md` §6.7.3's type: `host` / `srflx` / `relay`, with substrate `tcp` / `quic` / `webrtc`. `quic` and `webrtc` are declared-but-unbuilt in v1 (§7.3).

### 6.2 Blob framing `[cross-peer seam — MUST]`

The service stores an opaque byte string and never decodes it, so **what those bytes are is a peer-side pin.**

> **MUST.** The blob is the **canonical entity wire encoding** of the coordination entity — `{type, data, content_hash}` — with `data` embedded **verbatim**.

Both halves are load-bearing:

- **The type must travel with the blob.** A bucket is a mixed set and the reader has nothing else to dispatch on. `connect-request` and `connect-response` differ by a single field *name*, so a reader handed bare `data` is reduced to sniffing map keys to guess what it holds.
- **`data` is embedded raw, never decoded-and-re-encoded** (ENTITY-CORE-PROTOCOL.md §1.8 byte preservation; re-encoding on receive or forward is *the* interop hazard). This is not hygiene here — the messages are signed (§6.3), so a re-encode round trip through the carrier **silently invalidates the signature**.

> **MUST.** A carrier MUST NOT decode the coordination entity. Carrier opacity plus byte preservation (mirrors `EXTENSION-RELAY.md` §9).

What a bucket actually holds is the §6.3 **`signed-blob`** envelope (type `system/signaling/signed-blob`), which carries the coordination entity byte-for-byte in its `entity` field: the **outer** type dispatches (and distinguishes a signed deposit from any bare one), the **inner** coordination entity rides **verbatim**, and the byte-preservation MUST above protects the signed inner bytes through the container. The two bullets apply to that inner entity.

### 6.3 The signature and its envelope `[security — MUST; folded 2026-08-04]`

Everywhere else in this ecosystem a signature binds at the invariant pointer `/{signer_peer_id}/system/signature/{target_hash_hex}` and reaches a verifier by cross-peer sync or `envelope.included` ingestion. **At rendezvous time neither exists** — the entire purpose of the carrier is that the two peers have no connection yet. So a coordination deposit is **self-contained**: the entity, a **detached signature**, and the signer's **`public_key`**, packaged in one envelope a stranger can verify with no key lookup and no prior contact (a peer-id *is* a commitment to its public key — the same self-contained shape `system/protocol/connect/authenticate` uses at the handshake).

**The envelope** (cross-verified both ways by the cohort, `signed-blob` container):

```
system/signaling/signed-blob := {          ; what a bucket holds; itself an ordinary entity, so the carrier frames it under §6.2
  fields: {                                ; ECF-sorted field order: entity, public_key, signature, signer
    entity:     {type_ref: "primitive/bytes"}   ; the §6.2 canonical encoding of the coordination entity, embedded VERBATIM (never decoded+re-encoded)
    public_key: {type_ref: "primitive/bytes"}   ; the raw public key the signer commits to
    signature:  {type_ref: "primitive/bytes"}   ; detached signature over signing_input (below)
    signer:     {type_ref: "system/peer-id"}    ; the signer's canonical peer-id — self-describing per §1.5 multikey (key_type ‖ hash_type ‖ digest)
  }
}
```

> **MUST — what is signed (bucket-bound).** The signature is over a domain-separated input that **covers the rendezvous key**, so a valid blob cannot be replayed into another bucket:
>
> ```
> signing_input = "entity:sigblob:v1" ‖ 0x1F ‖ rendezvous_key(33) ‖ inner_content_hash(variable, last)
> ```
>
> `inner_content_hash` is the `content_hash` (`content_hash_format ‖ digest`) of the entity in `entity`. **It is authored content, so it follows the signing peer's home format and its length follows its own leading format byte** — 33 bytes under ECFv1-SHA-256, 49 under ECFv1-SHA-384, never a fixed width (`SPECIFICATION-FORMAT.md` §8.4.5). `rendezvous_key` is the key of the bucket the blob was collected from — it is **NOT** a field of the envelope and does not travel; the verifier supplies the key it passed to `collect`. (A carried key would be forgeable and then checked against the real one anyway — the `signer` trap — so it is bound, not carried.) **The rendezvous key stays pinned at 33 bytes, and that is not an exception to §8.4.5:** §3.1 pins its *digest format* to the SHA-256 floor rather than the deriving peer's home format — because a rendezvous key is a lookup token two independent parties must reproduce, not authored content — and the width follows from that pin.
>
> **The concatenation needs no length prefixes because the one variable-length component is LAST**, and everything before it is fixed: tag (17) ‖ SEP (1) ‖ rendezvous key (33) ‖ content hash (the remainder). The domain tag is versioned, so a future change to what is covered is a **new domain**, not a silent reinterpretation.
>
> > **Corrected 2026-08-10.** This block previously required `inner_content_hash(33)` and asserted "both components are fixed-length." That is false the moment a peer exercises V7 §1.2 — observed as `signaling.signaling_punch`: *`content_hash is 49 bytes, want the 33-byte wire form`* (core-go `62712fd`, SHA-384 run). **The neighbouring field was already right for exactly the reason this one was wrong** — §3.1 reasons explicitly about authored-vs-reproduced and pins only the key. The fixed-33 requirement is dropped on `inner_content_hash` only; §3.1 is untouched.

> **MUST — verification.** A verifier, on a collected `signed-blob`, with the `rendezvous_key` it collected from:
>
> 1. **Parse `signer`** as §1.5 multikey → `(key_type, hash_type, digest)`. `key_type` selects the algorithm; **`hash_type` is not an input** (it is covered by step 2). No hardcoded `0x01`.
> 2. **Bind key to id.** Check `public_key`'s length matches `key_type`. Compute `digest′ = Hash_{canonical_hash_type(key_type)}(public_key)` — the §1.5 mapping fixed per key type (Ed25519 → identity-multihash `0x00`; Ed448 → SHA-256 `0x01`). Require `signer` to equal `varint(key_type) ‖ varint(canonical_hash_type(key_type)) ‖ digest′` **in full**. Any mismatch — **including a well-formed but non-canonical `hash_type`** — → **`unusable_key`**. *(The wire `signer` is forgeable; it is derived-and-compared, never trusted as given — else one key presents two peer-ids and a peer chooses its own glare role / bucket. This is check (a), generalized: the id is derived-and-bound, not compared to a separate claim.)*
> 3. **Where the inner entity carries an `initiator`/`responder`** (§6.1): the verified `signer` MUST equal it → else **`signer_mismatch`** (reserved for exactly this §6.1 claim comparison). **Where it carries none** (all of §6.5): `signer` **is** the identity; step 2 is the whole of the check.
> 4. **Verify `signature`** over `signing_input`, dispatching the algorithm on `key_type`. Fail → **`bad_signature`**. (A blob replayed into a different bucket is checked against a different `rendezvous_key` and fails here.) **Ed25519 (`0x01`) is the MUST-implement floor; a well-formed but *unsupported* `key_type` is not a fault — it is skipped as MUST-ignore (ADR-0002), never rejected as `signer_mismatch`** (the hardcode that locked out an Ed448 identity).

> **MUST — disposition and the anti-downgrade rule `[security]`.** Only a **verified** `signed-blob` is actionable; everything else is skipped (§6.4). But two kinds of skip MUST be kept distinct, because conflating them reintroduces a downgrade:
> - **Not a `signed-blob`** — it does not parse as the container (a bare/legacy entity, or an unrecognized type). Skipped as §6.4's unrecognized case. *"Never was a container."*
> - **A `signed-blob` that failed step 2/3/4** (`unusable_key` / `signer_mismatch` / `bad_signature`). Skipped — and it **MUST NOT** fall back to being read as its inner `entity`. *"Is a container that did not verify."*
>
> These are **opposite dispositions**: if a verify-failure could fall back to the bare inner entity, an attacker flips one signature byte and a signed offer is accepted as an **unsigned** one. An implementation therefore needs the parse/container outcome as a **fourth** classification distinct from the three verify-failure names — four dispositions, not three, is what makes the anti-downgrade rule implementable. On the wire every skip is silent (§6.4); the four-way outcome is for diagnostics + conformance vectors, and the skip **SHOULD** be locally observable (the disposition label + the offending `key_type`) — wire-silence is required, but implementation-silence is how the Ed448 hardcode survived until a vector crossed it.

> **Stated as an exception on purpose.** This is not a parallel signature-storage convention of the kind `EXTENSION-IDENTITY.md` §6.2 forbids. The invariant-pointer convention is not being *replaced*, it is **unreachable**: there is no session to sync over and no tree to bind into. Recording it as an exception is what keeps it from being read as the extension-private-sibling-path defect class.

**What the signature does not buy** — both are easy to over-read:

1. **It authenticates the author, not the address.** An attacker signs *its own* candidate list containing a third party's address perfectly well. A candidate is evidence of nothing beyond "someone holding this key suggested this address."
2. **So the punch needs anti-amplification hygiene independently of it.** See §8.3.

### 6.4 Reading a bucket `[cross-peer seam — all MUST]`

`collect` is non-destructive and `tag` / `lobby` buckets are shared, so **every poll returns a mixture**: your own offer, the answer you want, other pairs' traffic, and message types a newer implementation introduced. This is where a naive implementation breaks.

- **A peer MUST skip its own messages** (`initiator` / `responder` equal to self). Otherwise a peer answers its own `connect-request` and "succeeds" at meeting itself — miserable to diagnose, because every individual step reports success.
- **A peer MUST correlate a response by the nonce echo.** In a shared bucket it is the only thing distinguishing *your* answer from someone else's, or a fresh answer from one already processed two polls ago.
- **An undecodable or unrecognized blob MUST be skipped, never treated as an error.** Anyone may write to a shared bucket, and a newer implementation's message type is a MUST-ignore, not a fault.
- **Bucket order is deposit order, oldest first** (§5 pin 4). *Which* of several requests to answer in a `lobby` bucket is **peer policy** and deliberately not pinned.

The nonce is a **correlator, not a secret** — anyone who can `collect` the bucket can read it and echo it. That is what §6.3 is for.

## 6.5 WebRTC substrate coordination `[the browser leg — folded 2026-08-02]`

The `webrtc` substrate (§7.3) is the browser's only peer-to-peer transport; the browser's own ICE stack punches, driven by this carrier. Its coordination is an **SDP/ICE exchange**, which the native `connect-request` / `connect-response` shapes (§6.1) cannot carry — those hold a `candidates` array and no SDP — so the substrate defines three of its own signed coordination entities under `system/signaling/webrtc/*`. They ride the carrier under §6.2 (blob framing), §6.3 (each **wrapped in a `signed-blob`** envelope, signature bucket-bound), and §6.4 (bucket read) **unchanged**; only the inner payload differs. This schema is versioned **`webrtc-sdp-ice/1`** — the identifier a `system/peer/transport/webrtc` profile pins in its `signaling_schema` field (`EXTENSION-NETWORK.md` §6.5.2d).

```
system/signaling/webrtc/offer := {       ; offerer decided by perfect negotiation (lo wins a glare) — see "Offerer determination"
  fields: {
    session_id: {type_ref: "primitive/bytes"}    ; correlates this pairing within one rendezvous key (MUST random ≥16B — below)
    sdp:        {type_ref: "primitive/string"}    ; RFC 8866 SDP offer — opaque, fed VERBATIM to setRemoteDescription
  }
}
system/signaling/webrtc/answer := {      ; responder's SDP answer
  fields: {
    session_id: {type_ref: "primitive/bytes"}
    sdp:        {type_ref: "primitive/string"}    ; opaque, fed VERBATIM to setRemoteDescription
  }
}
system/signaling/webrtc/candidate := {   ; a trickled ICE candidate (RFC 8838), either direction, post-offer/answer
  fields: {                              ; STRUCTURED — RTCPeerConnection.addIceCandidate() rejects a bare line
    session_id:        {type_ref: "primitive/bytes"}
    candidate:         {type_ref: "primitive/string"}   ; the RFC 8445 candidate line
    sdp_mid:           {type_ref: "primitive/string"}   ; REQUIRED — addIceCandidate() sdpMid
    sdp_mline_index:   {type_ref: "primitive/uint"}     ; REQUIRED — addIceCandidate() sdpMLineIndex
    username_fragment: {type_ref: "primitive/string"}   ; OPTIONAL — addIceCandidate() usernameFragment
  }
}
```

- **The `candidate` entity is a STRUCTURED tuple, NOT a bare line, and NOT `system/network/candidate` `[MUST]`.** `RTCPeerConnection.addIceCandidate()` requires `sdp_mid` and `sdp_mline_index` alongside the candidate line, so the entity carries them as distinct fields (an S4-implementer finding — a bare line is rejected by the browser API). Separately, this is still the **browser's** ICE format, produced and consumed by the browser's ICE stack and carried **verbatim** — it MUST NOT be translated to/from `EXTENSION-NETWORK.md` §6.7.3's `system/network/candidate` (entity-core's own reachability fact), the same reflection non-collapse §9.3 draws (the browser's `srflx` comes from its own STUN/UDP to a standard STUN server, never from `observe-address`). The offer/answer `sdp` stays **opaque** by contrast — it is fed verbatim to `setRemoteDescription`, so structuring it would be wrong; only the trickled candidate needs fields.
- **`session_id` correlates within a key, and MUST be random ≥16 bytes `[MUST]`.** A `lobby` / `tag` key may host several concurrent pairings, so offer/answer/candidate tuples group by `session_id` — the WebRTC analogue of the native `nonce` echo. A weak or colliding `session_id` would **splice two concurrent pairings' offer/answer/candidates together** — a silent cross-handshake — so it MUST be freshly random and at least 16 bytes. It does **not** replace the §6.4 bucket-read MUSTs (skip own, skip undecodable/unrecognized), which apply unchanged.
- **Trickle ICE is the sole candidate path** (RFC 8838): candidates flow as `system/signaling/webrtc/candidate` entities as they are gathered, rather than blocking the offer on full gathering — critical for the seconds-bounded handshake. (An earlier inline-in-the-offer fast-path was dropped at review: it forced an array-of-structured-candidate into the offer for a one-round-trip saving not worth the schema cost.)
- **Channel-identity binding `[security — MUST]`.** The offer/answer SDP carries a DTLS fingerprint. A receiving peer **MUST** (a) verify the offer/answer entity's signature (§6.3), and (b) feed **that entity's SDP verbatim** to `setRemoteDescription` — and **MUST NOT** call `setRemoteDescription` on SDP from an unverified entity. The browser's own stack then binds the negotiated DTLS certificate to the SDP `a=fingerprint` (RFC 8827) during the handshake, so the identity binding is discharged **automatically** — no separate runtime fingerprint compare is required or expected. Stated in this achievable form deliberately (an S4-implementer finding): a signaling MITM that cannot forge the signature cannot substitute a different SDP — hence a different fingerprint, hence its own channel — and "verify the entity, use exactly its SDP" is precisely how a browser enforces that. This is the browser-leg analogue of the native §7.4 connectivity check and discharges the §7.4 obligation for this substrate.
- **Split-peer byte preservation `[security — MUST]`.** A peer MAY be split across an execution boundary — the S4 browser topology runs `RTCPeerConnection` on the **main thread** while the signing identity and carrier client live in a **Worker** (a Worker cannot construct `RTCPeerConnection`). The signed unit then crosses that boundary: on send, each `sdp` / `candidate` is produced on the main thread and signed in the Worker; on receive, the entity is verified in the Worker and its `sdp` / `candidate` is fed to `setRemoteDescription` / `addIceCandidate` on the main thread. **The bytes signed MUST equal the bytes produced, and the bytes consumed MUST equal the bytes verified** — the crossing carries the payload **verbatim** and MUST NOT re-encode, canonicalize, or reconstruct it (ENTITY-CORE-PROTOCOL.md §1.8 byte preservation; re-encoding on a boundary is *the* interop hazard). This is what keeps the channel-identity binding above sound in a split peer: that MUST reads "verify the entity, use exactly its SDP," but *verify* and *use* sit on opposite sides of the crossing, so an interposed re-encode verifies one SDP and hands `setRemoteDescription` another — silently defeating the fingerprint bind. The signed `sdp` is this peer's own **local-description** SDP (`localDescription.sdp` after `setLocalDescription` — the one carrying the DTLS `a=fingerprint` it will actually present), so the fingerprint the receiver binds is the fingerprint the sender negotiates. **`[§11.5.1]`** a same-implementation S5 run marshals identically at both ends and so **cannot** expose a re-encoding crossing — this is single-impl-invisible exactly as `fire_at` (§7.2) and the handshake role (§7.4.1) were, so the S5 gate exercises it deliberately (a mangled crossing MUST yield zero delivered payload) or it goes unvalidated. The worker↔main-thread transport that carries the payload across this boundary (message variants, any version bump) is the implementation's **internal** protocol — out of scope here; this spec pins only the byte invariant across it.
- **The async inbox is not a carrier for it.** SDP/ICE is a live, seconds-bounded handshake, so it rides the live `rendezvous` carrier (or RELAY Mode-F forward, opt-in), never Mode-S async mail — the same rule the native punch pins (§7.3.1), not reopened here.
- **Two establishment triggers — the browser↔browser gate uses the second.** (a) **Profile-driven:** a peer resolving a target whose best profile is `webrtc` escalates to the §10.3 `establish_live` seam (`EXTENSION-NETWORK.md` §6.5.2d) — the *deferred browser↔native-direct* case. (b) **Rendezvous-driven:** two peers that publish **no** `webrtc` profile (both browsers — a browser publishes nothing, confirmed against the S4 implementer's shipping code) agree a rendezvous key **out of band** (§3 `pair` / `tag` / `secret` / `lobby`), meet at it, and drive `establish_live` **off that key, not off §10 profile resolution**. Trigger (b) is the **browser↔browser S5 gate** and the flow an S4 implementer hits first — `pair` mode (the two peer-ids, §3.2) is the natural default for a known counterpart.
- **Offerer determination `[cross-peer seam — MUST]`.** WebRTC is **asymmetric** — one peer sends `webrtc/offer`, the other `webrtc/answer` — and a **glare** (both offer) is *fatal* to the `RTCPeerConnection` state machine, so the offerer MUST be deterministic (native's both-fire symmetry and §7.4.1's *empirical* role resolution do not survive it). The resolution is **W3C perfect negotiation**, keyed to §3.2's existing byte-wise ascending peer-id sort (**not** a new convention — it is the sort the key derivation already uses). The sorted ids are canonical `system/peer-id`s per §3.2's third sub-pin — a content-hash or alternate encoding is not a legal input, and feeding one collapses this rule to both-Impolite (canonicalize at the caller):
  - **Normal flow.** A peer initiates by posting `webrtc/offer`; the peer that collects it, holding no offer of its own, posts `webrtc/answer`. Post-order alone settles it — no pre-agreement needed.
  - **Glare (both offered).** When a peer collects an offer while holding its own un-answered one, **both peer-ids are now known** (§6.3), so it resolves deterministically: the **lower-sorting `peer_id` (`lo`) is *impolite* — its offer wins**; the higher (`hi`) is *polite* — it **rolls back** its own offer and answers `lo`'s. Convergent, and the same rule covers the renegotiation / ICE-restart glare (open item #2).
  - **`pair` mode pre-assigns (optimization).** Both ids are known in advance, so `hi` **SHOULD** suppress its offer and wait for `lo`'s, skipping the glare round-trip. `tag` / `secret` / `lobby` **cannot** pre-assign — a peer does not know the counterpart until it collects the counterpart's first entity — so they rely on the glare resolution above. *(A flat "`lo` always offers" is therefore wrong for the matchmaking modes: `hi` may legitimately offer first, and forcing `lo` to counter-offer would manufacture the very glare the rule prevents.)*
  - **Trickle only under a settled `session_id` `[cross-peer seam — MUST]`.** A peer MUST NOT post `webrtc/candidate` entities under a `session_id` it has **not yet settled on** — meaning it has either answered, or had its own offer accepted. In particular, a `hi` peer that offered under its **own** `session_id`, then on collecting `lo`'s offer draws the rollback above, has **abandoned that `session_id`** and not yet adopted `lo`'s; candidates already trickled under the abandoned id are stranded — the counterpart correlates by `session_id` (§6.5 `session_id` floor) and never reads them — and, like §6.4's other never-meet shapes, **nothing fails loudly.** Concretely the gate is three-term, not two: trickle iff `answered ∨ (offered ∧ ¬rolled-back)` — a two-term `offered ∨ answered` gate leaks candidates across the whole rollback window. *(Cohort finding, `entity-core-go` 2026-08-04, `MayTrickle(TrickleState)` + state-table test: it is **not** live in either tree today — Rust's loop adopts an offer only under `¬offered ∧ ¬answered` so a peer that offered never reaches glare, and `pair` pre-assigns roles — so this bites the moment **glare handling lands** in `tag`/`secret`/`lobby`, i.e. exactly when the driver is built. Pinned now because the gate is being written now, and the failure mode is one where every individual step reports success. `[§11.5.1]` single-impl-invisible, like `fire_at`.)*
  - **This is NOT the retracted `lower-dials/higher-listens` split** (`ROUTING-2026-07-31`; §7.1 step 4). That split was non-traversing *because a peer that only accepts never fires an outbound packet and never opens its NAT hole.* The offer/answer role is a **distinct layer** — the SDP-negotiation direction — and **both** peers' ICE agents send connectivity checks regardless of who offered, so **both still fire outbound and both open their mappings** (WebRTC preserves "both dial" inside ICE). The offerer governs negotiation direction, **never** who dials.
  - **It chains with §7.4.1:** the peer that ends up the offerer is the §7.4.1 *initiator*, so the post-establishment HELLO client role follows it — one role assignment, not two.
  - **Cross-impl caveat `[§11.5.1]`.** This is a cross-peer *convention*; a same-implementation run cannot prove two impls agree on it. The S5 gate's two browser peers are likely the **same** WebRTC implementation, so the convention is only truly exercised once a **second** WebRTC impl exists — `fire_at` (§7.2) and the handshake role (§7.4.1) were both cross-peer conventions found wrong exactly this way. Until then, treat this rule as pinned-but-single-impl-validated.
- **Symmetric origination authority (trigger (b)) `[cross-peer seam — MUST; folded 2026-08-05]`.** Trigger (a) is an *asymmetric* dial: the `EXTENSION-NETWORK.md` §6.6 handshake mints one-directionally (acceptor → dialer) and that is the whole authority. Trigger (b) is *symmetric* — both peers agreed the rendezvous key out of band, which is the mutual-authorization act — so after the channel opens **either** peer may originate, and authority must be bidirectional. The §6.6 handshake already supplies dialer → acceptor; a trigger-(b) establishment adds the **one** missing mirror.
  - **The mint `[MUST]`.** The **dialer** mints a capability for the **acceptor** (`granter = dialer`, `grantee = acceptor`, `grants =` the assembled inbound-dialer grant per **Contents** below — *not* the flat default floor, authored under the connection's active `content_hash_format`) and **signs** it (signer = the dialer's granter identity, target = the cap's content hash — a single-sig root cap is otherwise rejected `missing_signature`, `ENTITY-CORE-PROTOCOL.md` §5.5). The `grantee` is an ordinary capability grantee: it MUST be exactly what the core verify contract resolves and compares to the author — the grantee identity entity's **content hash** — and **not** the §3.2 rendezvous `system/peer-id`; conflating them mints a cap that fails `grantee_mismatch` (the §3.2 id-encoding hazard, one field over). The acceptor MUST reject a grant whose `granter` is not the peer it authenticated on this connection, and **SHOULD verify the granter signature at acceptance** — but MUST check **only the legs the frame carries** (the granter signature), never a full chain walk, which resolves the grantee (the acceptor itself, absent from a dialer-authored grant) and would reject every valid grant.
  - **Contents `[cross-peer seam — MUST; contents ruling 2026-08-05]`.** The reciprocal grant is **the grant the minting peer would issue this counterpart as an inbound dialer** — the core §4.4 handshake union (the default connection grant **floor ∪** the counterpart's `system/capability/policy/{peer}` entry), **advertisement-filtered** (`ENTITY-CORE-PROTOCOL.md` §3 advertisement discipline — a peer MUST NOT grant authority it does not advertise it serves). **The filter's matching rule `[cross-peer seam — MUST]`:** an assembled entry is retained iff the peer's advertised served-scope **covers** it under the **same four-axis `scope_subset` relation** the chain already uses for attenuation (`ENTITY-SYSTEM-REFERENCE.md` capability-verification: handlers ∧ operations ∧ resources ∧ peers, entry ⊆ advertised) — an uncovered entry is **dropped, not narrowed**. Exact-op-match and namespace-prefix-match are both **non-conformant** — they diverge across the seam (one impl drops `system/tree:put` on `foo/bar` against an advertised `foo/*`; another keeps it), which is precisely the latent interop bug this MUST forecloses. The one relation the chain already uses is the only divergence-free choice. This governs the §4.4 inbound-dialer assembly **identically** (the reciprocal grant *is* that grant); the V7 §4.4 core-text pin routes upstream — this extension states it because it is where the reciprocal grant is minted. It is **not** the flat floor. Minting the bare floor while an inbound dialer receives the *assembled* set gives the establishment *whose justification is symmetry* asymmetric authority — the measured defect (`entity-core-go`, 2026-08-05: both impls mint the flat floor, so the reciprocal direction `403`s on anything out-of-floor while the inbound direction grants the assembled set). **The mirror is symmetric *construction*, not identical grant sets:** each peer applies its **own** inbound-grant assembly to the counterpart, so A→B and B→A authority differ exactly as A's and B's policy tables differ — correct, because authority is target-owned. Assembly is the **one** §4.4 union direction (union-at-handshake, subset-at-request); there is **no** separate reciprocal-narrowing pass — an operator wanting the reciprocal direction narrower expresses it as that peer's policy entry, in the one policy table. **Widening** beyond the assembled grant is by **`system/capability:request` in the reciprocal direction** — the *same* §4.4 request op as inbound (subset-validated), so no new surface; validated by a **separate conformance vector** (reusing the §4.4 request fixture), decoupled from the loopback-blind two-browser gate (§11.5), reaching the repo's entity-chat integration test. v1 scope, owned by the capability-handler track.
  - **Carriage + receive-construction `[MUST; carriage corrected 2026-08-05, entity-core-go/rust V3 4/4]`.** The reciprocal grant needs **no new fields and no new frame** — it rides **ordinary cap carriage**, and the locked wire core is untouched. **The two phases, concretely (the shape py implements):** (1) **Delivery — dialer→acceptor, once, post-handshake:** the dialer delivers the minted cap's **content hash** to the acceptor over the open connection (it authors the cap only after the authenticate-response reveals the acceptor's identity, so it cannot ride the handshake `auth_included`); this is what "grant-received" (Delivery + timing) gates on. (2) **Wielding — acceptor→dialer, per origination:** the acceptor originates an **ordinary EXECUTE whose root `capability` field is the cap hash** — that field *is* the reference. No in-band triple, no `included`-set chain: the dialer resolves and verifies the cap from **the minted-and-delivered ledger it authored the cap into** (resolution caution below). **Not §7a.2a in-band params `[corrected]`:** that triple lives in the `system/validate/dispatch-outbound` *conformance handler's* params, where a handler **body** receives a reentry cap **as data** to sign a later outbound EXECUTE — connection authority does not ride there and never needed to. The first fold's §7a.2a-carriage framing gated nothing and cost cohort cycles; the correction (`entity-core-go`/`entity-core-rust`, V3 both directions green built from real trees, not a scratch copy) is that the EXECUTE root already carries the reference. Carriage is only half the construction: the receiving peer MUST **install** the carried grant into its connection-scoped originating-authority slot (below, "Where the grant lives") **and** — the half that silently fails — MUST **serve** the reach-back that wields it (next bullet). Wire-shape + install + serve is the whole path; a peer that carries and installs but does not serve passes every grant-shaped test and reaches no handler.
    - **Resolution caution `[cross-peer seam — MUST; single-impl-invisible; entity-core-rust 2026-08-05]`.** A reciprocal cap is resolved from the **minter's minted-and-delivered ledger, NOT the content store** (the minter authored it; it was never stored). **Both the chain-verify and the revocation walk MUST resolve from that same ledger before either runs** — a verify/revocation walk sourced only from the content store cannot resolve the cap and reads the unresolvable chain as **revoked** → `403 capability_revoked` on a *valid* cap. Fail-closed, so it refuses valid authority and never admits invalid — invisible to any frame that inlines its chain, firing only on the reciprocal shape. *(Found live: Rust's `verify_request_with_ctx` walked the leaf twice — chain-verify read `envelope.included`, `is_revoked` re-walked a store-only resolver — so the cap Rust itself minted was unresolvable on the second walk. The `403 capability_revoked` code located it: that code is only reachable after verification already succeeded.)*
      - **The ledger's resolution key is the cap hash `[cross-peer seam — MUST; ruling 2026-08-07, divergence reported by entity-core-go + entity-core-rust]`.** The ledger MUST be **resolvable by the cap's content hash**, because the hash is what the wire actually hands the verifier — the wielding EXECUTE's root `capability` field *is* the cap hash (Carriage above), and it is the only key guaranteed present at the resolution site. An implementation MAY additionally index by recipient peer-id, but **MUST NOT make recipient peer-id the sole key**: the wielder's peer-id equals the delivery recipient's only while the cap is wielded by exactly the peer it was delivered to, so a recipient-keyed-only ledger resolves nothing the moment a cap is wielded by a delegate and fails **closed** — the same `403` on valid authority as the store-only walk above, arriving by a different route. *(Reported as a benign shape difference — Go keys on cap hash, Rust on recipient peer-id; both catch a wrong wielder today, which is why neither impl filed it. Pinned anyway, and pinned to Go's shape, because the two shapes diverge at a **cross-peer** seam under delegation and `entity-core-py` is about to build this from scratch: three shapes is the outcome this rule exists to prevent. Rust's index is conformant as an index; what changes is that hash-resolution MUST also work.)*
  - **Reach-back serving `[cross-peer seam — MUST; single-impl-invisible; found by entity-core-go 2026-08-05]`.** For the reciprocal grant to reach a handler, the peer that **dialed** the connection MUST **dispatch inbound EXECUTE frames arriving on it** — V7 §6.11(b) dialer-side reentry. A client-side reader that treats every inbound frame as a *response awaiting a local waiter* drops the counterpart's origination as an orphan, silently: the mint succeeds, the grant verifies, and nothing serves it. **Minting authority is the visible half; serving the reach-back is the invisible half** — and it is loopback-invisible in the same way as `fire_at` (`§11.5.1`): every individual step reports success. A peer that serves inbound EXECUTE only on connections it *accepted* (server-side) has built exactly **half** of trigger (b). *(Found live: `entity-core-go` had server-side reentry since always and no dialer-side; the reciprocal-origination vector timed out until the dialer-side reader was taught to dispatch. A from-scratch mint — `entity-core-py` next — MUST build **both** sides.)*
  - **Delivery + timing.** The dialer learns the acceptor's authored identity only in the authenticate-response, so the grant is authored and delivered **after** the handshake, over the open connection, in every mode; its supporting chain (signature + granter identity) is **resolved at the dialer from the minted-and-delivered ledger it authored the cap into** (Carriage above; a reciprocal cap is *not* in the content store) — **not** re-inlined in an `included` set: the handshake `auth_included` cannot carry entities the counterpart authored, and re-carrying them is unnecessary when the verifier *is* their author. Origination **gates on grant-received**, with a **bounded wait**: if the grant does not arrive within the bound the peer **fails closed** (no originating authority) — it never blocks, so a non-adopting counterpart degrades to one-directional, not to a hang. **The bound is impl-local; pin a *floor*, not a value `[ruling 2026-08-05]`.** The reciprocal-origination conformance vector MUST allow **at least a floor** so a slow-but-correct peer is not failed by a fast peer's short bound (**2s** recommended as the floor — `entity-core-go`'s value works). This is a conformance-vector timing floor, not a wire MUST; production timeouts stay implementation-local.
  - **Where the grant lives, and which wins `[MUST]`.** The reciprocal grant is **connection-scoped** originating authority, held with the live connection — **not** written to `system/peer/session/{peer}` (`EXTENSION-NETWORK.md` §6.6 records only the durable handshake cap). The origination path MUST consult connection-scoped grants; where an impl also holds a durable `held_capability` for the peer, **the connection-scoped grant wins** — a durable cap can predate the live establishment and MUST NOT shadow it. The two are distinct slots: durable `held_capability` = dialer-side reconnect-skip authority; reciprocal grant = live-establishment origination authority.
  - **The discriminator is the rendezvous key, locally-derived, never wire-carried `[MUST]`.** The mint fires iff the establishment was reached by **meeting at a §3 rendezvous key** (`pair`/`tag`/`secret`/`lobby`) — a key **both** peers brought independently, since §3.4's rendezvous-hash (§3.2's `pair_key`) means they meet only if both used it. Classify **positively on the key**, not on "rendezvous-driven vs profile-driven" (not exclusive — one event can be both) and not on substrate: a §7 punch derives its `pair_key` from the two peer-ids and therefore **mints**; only a **direct dial to a resolved transport endpoint** (a `tcp`/`http` profile or a raw address, no §3 key) is asymmetric and does not. It MUST NOT be a wire field the counterpart sets (a one-sided field is the §7.4.1 failure shape); it need not be, since meeting proves both brought the same key, so each peer classifies **independently** and they agree by construction. Each peer sets its own local `established_via_rendezvous_key` flag from its own establishment path.
  - **Not generic back-direction dispatch.** This grant is confined to trigger (b). It is **not** the authority for an acceptor to originate to a profile-less dialer on an *asymmetric* connection; that back-direction, where legitimate, is the per-delivery `deliver_token` (`EXTENSION-INBOX` / `EXTENSION-SUBSCRIPTION`), and absent such a trigger it fails closed (`EXTENSION-NETWORK.md` §6.6). See the back-direction-authority taxonomy, `guides/GUIDE-CAPABILITIES.md` §4a.

## 7. The Punch

Registered behind `EXTENSION-NETWORK.md` §10.3's `establish_live(peer_id)` seam. NETWORK owns the seam and the resulting transport's obligations; this section owns the choreography.

### 7.1 The flow

1. **Both gather candidates** (`EXTENSION-NETWORK.md` §6.7.3): each learns its `srflx` mapping from a reflector (`EXTENSION-NETWORK.md` §6.7.1) and lists `host` / `srflx` / `relay`.
2. **Exchange over the carrier.** A offers `connect-request` at the key; B collects it and offers `connect-response`; A collects. Each now holds the other's candidate list. Both reached the same provider by §3.4's rendezvous-hash rule.
3. **Measure and schedule.** A measures the round-trip *through the carrier* and sends `punch-sync` with `fire_at`, so both start sending to each other's `srflx` at approximately the same instant.
4. **Simultaneous open.** Each side sends to the other's `srflx`. Each side's *outbound* packet punches its own hole; the other's packet, arriving after that hole is open, gets through. After the crossfire both NAT mappings exist and a direct path is open.

   > **Both sides fire outbound — neither is listen-only `[cross-peer seam — MUST; correction absorbed 2026-08-01]`.** Each peer **MUST** issue an *outbound* connection attempt at `fire_at`; **listening alone opens no hole**, because only an outbound packet creates the local NAT mapping. A peer that merely accepts (TCP-passive) never opens its own hole, so the counterpart's SYN reaches a closed NAT and the punch **cannot traverse** — even though it succeeds on loopback and in every same-implementation test (no NAT is present to expose it). The `[§7.4.1]` signaling-role split is **orthogonal to this** and MUST NOT be read as designating one side passive: it selects (a) **which** of the racing sockets survives simultaneous-open (a local tie-break, e.g. peer-id compare) and (b) **who runs the HELLO client** on the survivor — *after* both holes are open. "The responder **serves** the handshake" (§7.4.1) is a statement about the post-socket HELLO exchange, **not** about who dials the socket: on the wire **both** dial. **Cohort finding (2026-08-01):** two independent implementations built step 4 as *lower-id dials, higher-id listens* — collapsing the hole layer into the handshake-role layer — and both are non-traversing by construction; this pins the de-conflation the committed text implied but did not make unmissable. **Validation:** exercised as of 2026-08-02 under **emulated dual-NAT**, cross-implementation — the cheapest substrate that can exercise it at all. **Loopback by construction cannot** (§11.5.1): with no NAT in path a listen-only peer's counterpart still reaches it, so the harness reports green for a peer that cannot traverse. Real-NAT diversity remains the terminal gate.
5. **Upgrade and drop the carrier.** The direct connection becomes the live transport and is returned through the `EXTENSION-NETWORK.md` §10.3 seam. The peers stop using the carrier, keeping it only if the direct link drops.
6. **On failure** (no direct path within a timeout — symmetric NAT or CGNAT), **fall back to the `relay` candidate** (§10). Correctness is preserved; only the direct-path optimization is lost.

### 7.2 `fire_at` — a delay, never an instant `[cross-peer seam — MUST]`

> **MUST.** `fire_at` is a **delay from the receiving peer's moment of receipt** of the `punch-sync` entity, encoded as **unsigned integer milliseconds**. It is **not** a timestamp.

The core protocol assumes no synchronized clocks, and two peers' wall clocks can differ by far more than the entire punch window — so a wall-clock instant on the wire is a silent cross-peer failure. **It is also invisible to a same-host test**, where both peers read the same clock.

> **This is not hypothetical.** An implementation built this field as milliseconds-since-epoch from an earlier draft that said "instant," and it **round-tripped green** — because both peers were on one host. The wrong shape passes every same-host test, which is precisely why the clock domain is a MUST rather than a convention.

**Who measures, who fires:**

- **The initiator measures `rtt`** — the round trip through the carrier (its own `offer` to the `collect` that returns the response). That is the only latency estimate either peer has, since by construction neither can yet reach the other directly.
- **The initiator fires at `d` after *sending*** the `punch-sync`; **the responder fires at `d` after *receiving*** it. The sync traverses roughly half the carrier round trip, so the two firings cross near the middle.
- **`d` MUST be ≥ the observed one-way carrier latency (`rtt/2`)** — otherwise the responder's fire time has already elapsed when the message lands, and the two sides never overlap.

**Defaults, tunable and industry-typical rather than measured here:** `d = max(rtt, 250 ms)`; up to 3 **exchange attempts** (see the two layers below), each with a fresh nonce; then abandon to the relay fallback.

#### 7.2.1 Two retry layers — never confuse them `[cross-peer seam — MUST]`

A punch has **two nested retry counters**, and they are not interchangeable. Every implementation grows both, because each solves a different problem:

| Layer | What one attempt is | Who it costs | Governed by |
|---|---|---|---|
| **Crossing retry** | One more simultaneous-open dial at the counterpart's already-known `srflx`, inside the current crossing window | **Nobody but the two peers** — packets go to the target's own socket | Purely local; tune freely |
| **Exchange attempt** | A **fresh nonce**, a fresh candidate gathering against a **reflector**, and a fresh coordination round trip through a **carrier** | **Third parties** — the reflector and signaling provider (§3.4) | The MUST below |

> **MUST.** The 3-attempt default above, and the composition rule below, are about **exchange attempts only**. When the `EXTENSION-NETWORK.md` §10.3 seam is consulted from a `maintain-peer` reconnection continuation (`EXTENSION-NETWORK.md` §4.1), the punch performs **exactly one coordination exchange** per reconnect tick, and §4.1's backoff owns re-scheduling. **Exchange budgets MUST NOT nest.**

> **MUST NOT.** An implementation MUST NOT reduce its **crossing** retry count to satisfy the rule above. The two counters are independent, and starving the crossing is not a conservative reading of this section — it is a different bug.

**Why the second MUST is here, and it is not hypothetical.** An implementation built this section's composition rule by setting its crossing-retry count to 1 — a reasonable reading of "exactly one attempt" when the word is unqualified — and **it broke a punch that had been working**, caught by a race detector rather than by review. The crossing *needs* several dials: the whole mechanism is two peers firing at a scheduled instant across an unsynchronized network, so the first dial landing outside the counterpart's window is the expected case, not the failure case. **Cutting the crossing budget makes the punch miss while consuming exactly as much third-party infrastructure as before** — the opposite of what the rule is for.

**The distinguishing question, when in doubt: does one more attempt send a packet to anyone other than the target peer?** If no, it is a crossing retry and this section does not constrain it. If yes — a reflector, a carrier, a fresh nonce anyone else must store — it is an exchange attempt and the budget applies. *The rule was always about third-party load; that rationale is what an implementer should reason from, and it is what let the finding above be diagnosed correctly once the counter was in the wrong place.*

A reconnection continuation re-enters the §10 ladder, which consults §10.3, which runs a punch that retries three times on its own — so the two budgets **multiply**, not add. A peer behind a symmetric NAT, where the punch is *never* going to succeed, is exactly the peer whose reconnect loop runs forever: *reconnect-attempts × 3* punches, each one a fresh candidate-gathering round against a reflector and a fresh coordination exchange through a carrier.

**Neither of those is the punching peer's own infrastructure** — the reflector and the signaling provider are third-party peers (§3.4). So an implementation that nests the budgets sees nothing wrong locally and multiplies its load on someone else's node; the peer harmed is never the one holding the bug. That is what makes the composition normative rather than a tuning suggestion. **A punched connection dropping is the common case, not the exceptional one**, so this path is hot: it is the steady state for every NAT'd peer with `reconnect: true`.

A peer MAY remember that traversal to a given peer failed and skip the seam on later reconnects (`EXTENSION-NETWORK.md` §10.3 obligation 4). Like the observed address (`EXTENSION-NETWORK.md` §6.7.1), that memo is **local session state and is never published**.

**What stays local and MAY diverge:** how a peer measures `rtt` (sample count, smoothing), per-candidate probe pacing, socket options, retry counts and timeouts. **This licenses different *tuning*; it does not license a different encoding or clock domain.** Those two are the interoperable surface and are pinned above.

### 7.3 Substrate

| Substrate | `candidate.substrate` | v1 | Notes |
|---|---|---|---|
| **TCP simultaneous-open** | `tcp` | **buildable now** | Reuses the existing `tcp` profile — both sides SYN at `fire_at`. The punched result *is* a `tcp` live transport. Many NATs handle simultaneous *TCP* open worse than UDP (fine on cone, flakier elsewhere). |
| **QUIC** | `quic` | declared, **unbuilt** | Punches more reliably; the modern default. Gated on building the QUIC transport first. Sequenced after TCP-simopen proves the coordination protocol. |
| **WebRTC data channel** | `webrtc` | **folded** — coordination schema §6.5, profile `EXTENSION-NETWORK.md` §6.5.2d; **unbuilt (S3)** | The browser's only peer-to-peer transport; the browser's own ICE stack punches, driven by this carrier. |

**The socket requirement is substrate-dependent and is not satisfiable by discipline.** `EXTENSION-NETWORK.md` §6.7.3 requires a peer to punch from the **same local endpoint whose mapping was observed**. On UDP and QUIC that is one socket reused and effectively free. **On TCP it requires `SO_REUSEADDR` / `SO_REUSEPORT` plus an explicit bind on both the reflector dial and the punch dial** — no amount of careful code substitutes for the socket options.

#### 7.3.1 Substrates do not interoperate — the coordination layer does

**Two peers can punch only over a substrate they share.** The coordination layer above — carrier, rendezvous key, candidate exchange, `fire_at`, roles — is **substrate-agnostic and shared by all three**; that is the reusable foundation, and it is the half already cross-exercised (§11.5). The transport underneath is not shared. So "the punch is built" is a claim about a *pair* of peer types, never about the network:

| ↓ dials → | native (`tcp`) | browser (`webrtc`) |
|---|---|---|
| **native** | direct punch | **no shared substrate** — see below |
| **browser** | **no shared substrate** — see below | direct punch (once the WebRTC substrate lands) |

A browser peer has no raw sockets and no `SO_REUSEPORT`; it cannot run the `tcp` simultaneous open at all, and `EXTENSION-NETWORK.md` §14 makes it a permanent connection initiator. Its only peer-to-peer transport is a WebRTC data channel. **This is a platform fact, not a gap to be closed by more careful specification.**

**The v1 model, pinned:**

1. **The relay ladder is the floor across a substrate boundary.** A peer that shares no punch substrate with its target falls to §10's relay fallback — RELAY Mode-C (live circuit) or Mode-S (async). This is **correct, not degraded**: §7.1 step 6 already makes relay the outcome of every failed punch, and a substrate mismatch is simply a punch that fails at candidate selection instead of at the crossfire. **No connectivity is lost; only the direct-path optimization is.** An implementation MUST NOT treat a substrate mismatch as a dispatch error.

   > **Read the matrix as being about *direct* paths only.** A ❌ cell means "no direct punch between these peer types," **never** "these peers cannot reach each other." A browser peer holding an ordinary live transport to a native peer — a WebSocket, per §6.5.1b's duplex taxonomy — is connected by every measure this spec cares about; the punch would only remove an intermediary. **A `webrtc` substrate is therefore an optimization, not an unblock**, and no browser-facing product capability is gated on it. *(Recorded because the matrix was read the other way in cohort routing, which mis-set the priority of the browser leg.)*
2. **A native peer MAY publish a `webrtc` transport profile** to become directly punchable by browser peers. It is a **MAY**, deliberately: requiring every native peer to carry a WebRTC stack would impose a large dependency on implementations whose peers may never meet a browser, against this ecosystem's minimal-dependency posture. It is safe as a MAY because the profile is **published and therefore discoverable** — a browser reads the target's transport set and knows whether a direct path exists, rather than guessing. (Contrast §7.2's `fire_at`, where the ambiguity is *undiscoverable* and must therefore be a MUST.)
3. **Substrate selection is ordinary per-peer profile selection** (`EXTENSION-NETWORK.md` §6.5.1a), not a new mechanism. "Native reaches browser over `webrtc`, browser reaches native over `webrtc`, native reaches native over `tcp`" is the ordinary asymmetric case.

**Ownership of the WebRTC substrate is this repo's, and it is folded (2026-08-02):** the SDP/ICE coordination schema is §6.5 above (`system/signaling/webrtc/*`, riding this carrier), and the `system/peer/transport/webrtc` profile filling `EXTENSION-NETWORK.md` §6.5.1b's reserved slot is `EXTENSION-NETWORK.md` §6.5.2d. It is **not** an alternative to §7's punch — it is this same model with the browser's punch substrate plugged in. It is **unbuilt (S3)** and, per §11.5.1, **unvalidated until the S5 gate** (two browser peers over a real signaling node); the rationale and the S0–S5 order live in `docs/proposals/implemented/PROPOSAL-EXTENSION-WEBRTC-TRANSPORT` (FOLDED). It gates browser-facing connectivity work and nothing else — native↔native punch never waited on it.

### 7.4 Connectivity check and handshake role `[security + cross-peer seam — MUST]`

> **MUST.** Before treating a punched path as the peer, confirm the peer answering is the expected peer — **nonce plus identity binding**. A forged address that does not lead to the real peer fails this check.

> **MUST.** The identity check completes **before** the punched connection is returned through the `EXTENSION-NETWORK.md` §10.3 seam into the connection pool or carries any operation (`EXTENSION-NETWORK.md` §10.3 obligation 3). An implementation is free to run it on either side of its own handshake boundary; it is not free to pool first and verify later.

#### 7.4.1 Who runs the client handshake `[cross-peer seam — MUST]`

> **MUST.** After a successful punch, the **handshake role follows the signaling role**: the peer that offered `connect-request` — **the initiator** — runs the **client** half of `system/protocol/connect` (it sends HELLO); the peer that answered with `connect-response` — **the responder** — **serves** it. This is independent of which side's TCP connection won the simultaneous open, and independent of where in its own layering an implementation runs the handshake (§7.4 above).

> **"Serves" is a handshake role, not a socket role.** The responder serving the HELLO exchange does **not** make it TCP-passive: per §7.1 step 4, **both** peers dial outbound at `fire_at` (each must, to open its own NAT hole). This subsection governs *only* who speaks HELLO first on the already-open socket; it never designates a side as listen-only. Collapsing the two is exactly the non-traversing cohort bug §7.1 step 4 now pins.

**Why this is a MUST and not an implementation detail.** A dial establishes the role for free: exactly one side dialed, so exactly one side is the client. **A punch destroys that signal** — both sides dialed, simultaneously, by construction. Nothing in the resulting socket pair says who speaks first. Two implementations that resolve it differently produce one of two failures on a connection that punched *perfectly*:

- both sides send HELLO — a crossed handshake, or
- neither does — a silent hang.

Either is reported as *"the punch didn't land,"* which sends the investigation to the NAT layer, where nothing is wrong. **And it is invisible to every same-implementation test**, because both ends of a Go↔Go or Rust↔Rust punch make the same choice. This is the §7.2 `fire_at` failure shape again — a cross-peer ambiguity that only a cross-implementation run can see — and it was found the same way: two implementations built the punch, then compared. Both had already chosen initiator-as-client independently; this pins the convention that was going to be assumed either way.

**The signaling role is the only durable ordering available.** It is established in the carrier exchange, before any socket exists, and both peers already agree on it — the initiator is by definition the peer whose `connect-request` the responder collected. Nothing else in the punch is asymmetric enough to derive a role from: candidate ordering is per-peer, `fire_at` is deliberately a delay rather than an instant (§7.2), and peer-id comparison would be a *second* convention to keep aligned. Implementations MAY use a peer-id comparison or similar tie-break for **local** concerns that need one — which of two racing sockets to keep, for instance — but the handshake role is not local, and it is pinned here.

**Deliberately not re-derived here:** the exact connectivity-check and consent-freshness handshake (RFC 8445 §7, RFC 7675) and symmetric-NAT port prediction. These are detailed, well-studied, and security-sensitive; read the references at build time rather than inventing them.

## 8. Security

### 8.1 Admission

| Surface | Admission |
|---|---|
| Wrapped | `system/capability/signaling-use` gates `offer` / `collect`. A deployment serving a private device mesh grants it narrowly; `advertise` is typically operator-only. |
| Unwrapped | **None by construction.** The rate limiter is the entire admission story (§8.2). |

### 8.2 Rate limiting (MUST)

> **MUST.** A deployment offering the unwrapped surface rate-limits **per source and per key**, and refuses excess with `rate_limited` rather than dropping silently.

**This is load-bearing, not hygiene** — on the unwrapped surface it is the only thing standing between the service and abuse. The specific thresholds are deployment policy and are deliberately not pinned; the **error code is** pinned (§9.2), so clients are built to handle it from the first line rather than retrofitted.

### 8.3 Anti-amplification (MUST)

> **MUST.** A peer caps unsolicited traffic to a candidate at a small fixed probe budget and MUST NOT send more until that address answers.

Absent this, an open `lobby` bucket is a traffic amplifier aimed by whoever fills it. This is required **independently of §6.3's signature**, which authenticates the author and not the address.

### 8.4 Threat summary

| Surface | Threat | Mitigation |
|---|---|---|
| Candidate exchange | Forged candidates trick a peer into dialing an attacker or victim | Signed messages (§6.3); a peer punches only toward candidates from the authenticated expected counterparty; the §7.4 check confirms identity; §8.3 caps the probe budget |
| Carrier | Carrier reads or tampers with the exchange | Messages are opaque to the carrier and signed end-to-end (§6.2, §6.3). The carrier can drop or delay (denial of service) but cannot read or forge. Choose a carrier you trust *as a carrier*. |
| The service itself | A malicious service maps peers or floods a key | It holds only opaque blobs at a derived key; rate-limited and TTL-reaped. It learns *that* two peer-ids are pairing (metadata), never the content. For `tag` / `secret` / `lobby` it sees the derived key, never the underlying string. |
| Shared-key rendezvous | A guessed or low-entropy `tag` / `secret` lets an uninvited peer into the bucket | The key **introduces, never authorizes** (§3.4). For real access control use a high-entropy `secret` or the capability-gated wrapped surface. |
| Punched transport | Same as any live transport | The ordinary handshake and capability flow authorize the connection once up. |

### 8.5 Metadata exposure on the unwrapped surface

> The unwrapped surface runs **without TLS** (§9.4). A passive on-path observer sees the **33-byte rendezvous key** in cleartext. Blob contents stay opaque, but in `pair` mode the key is derived from both peer identities (§3.2), so an observer who can guess or enumerate a candidate pair can confirm **that those two peers are meeting**. That is a real linkability leak on **metadata, not content**. It is accepted for v1 — the punch itself is observable on the wire regardless — and it is **why `secret` and `tag` modes exist**, since their keys are not derivable from identities.

## 9. The Unwrapped Protocol

A public deployment is **not an entity peer**. It is a stateless service with its own small protocol, outside the entity system entirely — no core wire framing, no handshake, no capability negotiation, nothing to bypass because none of it is in the path. Clients hold no entity machinery to talk to it.

### 9.1 Two listeners, two protocols

| Surface | Transport | Protocol |
|---|---|---|
| `offer` / `collect` / `advertise` | **TCP** | length-prefixed CBOR (§9.2) |
| reflection | **UDP** | **RFC 5389 STUN Binding, unmodified** (§9.3) |

**The split is forced by the consumer, not chosen for tidiness.** Reflection's consumers are the later QUIC substrate and the **browser leg**, whose ICE agent speaks real STUN over UDP and **cannot be taught anything else**. A STUN-shaped protocol of our own would be useless to a browser by construction and a worse STUN than STUN. The mailbox has the opposite constraint: no external standard fits an opaque keyed bucket, and it must be implementable in three languages **without adding a dependency to any of them**.

Whether the two share a port is an implementation detail; they cannot share a *transport*, so in practice they are two sockets.

### 9.2 The mailbox protocol (TCP)

**Framing.** Each message is a **4-byte big-endian unsigned length** followed by exactly that many bytes of **CBOR**. Requests and responses use the same framing.

- A service **MUST** reject a declared length above **1 MiB** by closing the connection without a response. This is the anti-denial-of-service floor and is deliberately far above the §5 semantic limits, which are the real contract.
- A connection is **strictly alternating request → response**. **No pipelining.** A client MAY reuse a connection; a service MAY close an idle one at any time, and a client MUST treat that as ordinary and reconnect rather than as an error.

**Why CBOR and not HTTP.** All three reference implementations already carry a CBOR codec, so this costs **zero new dependencies** — against an ecosystem convention of stock tools and minimal dependencies, where HTTP would cost at least one implementation a client library. HTTP would also drag in header parsing, chunked encoding, status-code mapping, and keep-alive semantics: a large ambiguity surface for three independently written clients, for a **three-verb** protocol. CBOR here is an *encoder*, not entity machinery — the client still holds no entity code.

*A browser therefore cannot speak the unwrapped mailbox (no raw TCP). That is correct and not a gap:* browsers use the wrapped surface, and §2.2 guarantees both surfaces complete the same verbs.

**Requests.**

```
{ op: "offer",     key: bstr(33), message: bstr }
{ op: "collect",   key: bstr(33) }
{ op: "advertise" }
```

**Responses.**

```
{ ok: true }                                    ; offer
{ ok: true, messages: [bstr, ...] }             ; collect — oldest first (§5 pin 4)
{ ok: true, endpoint: tstr, limits: { ... } }   ; advertise — §4.3
{ ok: false, error: tstr }                      ; any — closed enum below
```

**The key is exactly 33 bytes**, opaque, compared byte-wise. Any other length is `bad_request`.

**Error codes (closed enum).**

| Code | Meaning |
|---|---|
| `message_too_large` | blob exceeds `max_blob_bytes` — **refused, never truncated** |
| `bucket_full` | bucket at `max_bucket_blobs` — **refused, never evicted** |
| `bad_request` | malformed frame or CBOR, unknown `op`, wrong key length, missing field |
| `rate_limited` | admission control refused it (§8.2) |

A client receiving an unrecognized code **MUST** treat the request as failed and **MUST NOT** retry it as if it had succeeded. Services **MUST NOT** invent codes outside this set.

### 9.3 Reflection — RFC 5389 STUN, unmodified (UDP)

The service answers a **STUN Binding Request** with a **Binding Success Response carrying `XOR-MAPPED-ADDRESS`**. That is the entire specification; RFC 5389 is the document and this spec adds nothing to it.

- The service **MUST NOT** require authentication. A public reflector demanding credentials is unusable by a browser ICE agent, which is one of the two consumers.
- The service **MAY** additionally emit the legacy `MAPPED-ADDRESS`; clients **MUST** read `XOR-MAPPED-ADDRESS`.
- A peer **MUST** consult **several** reflectors and require agreement before concluding a NAT type (`EXTENSION-NETWORK.md` §6.7.1) — a single reflector is advisory, never trusted.
- `EXTENSION-NETWORK.md` §6.7.3's socket rule applies unchanged: a `srflx` candidate derived here is the mapping of **the socket that sent the Binding Request**, and the peer MUST punch from that socket.

**Anyone's STUN server works, and that is the point.** A deployment MAY point peers at any public STUN server instead of running reflection at all; nothing in the punch path knows the difference. This is offered as a convenience for operators who would rather run one box than two.

**This is not the v1 reflection path.** `EXTENSION-NETWORK.md` §6.7.1's `observe-address` operation is, and it needs no service at all. §9.3 exists so a deployment can serve the QUIC substrate and the browser leg later, and so nobody builds a private reflection protocol in the meantime.

### 9.4 What is deliberately absent

**No TLS.** Blob confidentiality and authenticity are already peer-side: blobs are opaque to the service and signed end-to-end (§6.2, §6.3). The service is untrusted **by design** — it is a dumb keyed mailbox — so terminating TLS at it would protect the one leg that already needs no protection, while adding certificate lifecycle to a service whose scaling story is "run more instances." The cost is stated in §8.5 and is real.

**No authentication**, by construction (§8.1). §8.2's rate limiting is the entire admission story.

## 10. Fallbacks

When a punch fails — symmetric NAT or CGNAT on either side — correctness is preserved by falling back:

| Fallback | What it is | Posture |
|---|---|---|
| **RELAY Mode-S** (async inbox) | store-and-forward delivery to an offline or unreachable peer | already v1 in RELAY; reached through `EXTENSION-NETWORK.md` §10.2 |
| **RELAY Mode-C** (live circuit) | a live relayed data path for real-time use between two symmetric-NAT peers | named-deferred in RELAY; gated on a real-time driver |
| **RELAY Mode-F** (forward) | signaling routed through a RELAY peer instead of a dedicated endpoint — same message schema (§6), carrier opacity per RELAY §9 | an alternate carrier, not a data fallback |

**These are an opt-in tier**, advertised capacity a dedicated peer volunteers — never part of the always-on infrastructure. A deployment running only the basic tier (reflection plus rendezvous) still connects the large majority of peer pairs; symmetric-NAT pairs need someone to opt into a relay, or fall back to async.

## 11. Conformance

### 11.1 MUST Implement

The **client role** is the conformance surface (§2.1). A conformant implementation:

- Derives `rendezvous_key` by the §3.1 mode-tagged derivation, with the SHA-256 floor format, `SEP = 0x1F` in both positions, and the exact mode tags — and selects the provider by rendezvous-hash (§3.4)
- Treats `tag` / `secret` string inputs as **byte-exact UTF-8** — no normalization, no case-folding (§3.3)
- Frames coordination blobs as canonical entity wire encoding with `data` embedded **verbatim**, never re-encoded (§6.2)
- Signs coordination messages self-contained with the signer's `public_key`, and **skips** any message failing the peer-id or signature check exactly as it skips an undecodable one (§6.3)
- Applies all four §6.4 bucket-read filters: skip own messages, correlate by nonce echo, skip undecodable or unrecognized blobs without error, read oldest-first
- Encodes `fire_at` as **unsigned milliseconds of delay from receipt**, never a timestamp, with `d ≥ rtt/2` (§7.2)
- Performs the §7.4 connectivity check — nonce plus identity binding — before treating a punched path as the peer
- Caps unsolicited probe traffic per §8.3
- Honors the limits published by `advertise` rather than assuming them (§4.5)
- For a **trigger-(b) symmetric establishment** (§6.5 (b)): mints the reciprocal grant as the **assembled inbound-dialer grant** (floor ∪ policy, advertisement-filtered — *not* the flat floor), installs it connection-scoped, **and serves dialer-side reentry** (V7 §6.11(b)) so the counterpart's reach-back reaches a handler. Both the mint and the serve — a peer that mints but serves reentry only server-side has built half the establishment, and it is loopback-invisible (§6.5 (b) Contents / Reach-back serving)

A peer that offers a **server role** additionally implements §5's six bucket semantics exactly, and §8.2 rate limiting if it exposes the unwrapped surface.

### 11.2 SHOULD Implement

- Consulting several reflectors and requiring agreement before concluding a NAT type (§9.3)
- Retry with a fresh nonce, up to a small bounded count, before abandoning to a fallback (§7.2)
- Dropping the carrier once the direct transport is live, retaining it only against link loss (§7.1 step 5)

### 11.3 MAY Implement

- The **server role** in either surface (§2.1) — optional by design
- The unwrapped surface (§9), the wrapped surface, or both (§2.2)
- Reflection (§9.3) — a deployment MAY point peers at any standard STUN server instead

### 11.4 Implementation-Defined

- Rate-limit thresholds (§8.2 pins that limiting happens and its error code, not the numbers)
- How `rtt` is measured (sample count, smoothing), probe pacing, socket options, retry counts, timeouts (§7.2)
- Which of several requests to answer in a shared `lobby` bucket (§6.4)
- TTL and size limits within the published `advertise` contract (§5 pins the defaults)
- The reciprocal-origination **grant-received wait** bound (§6.5 (b) Delivery + timing) — impl-local; the conformance vector pins a **floor** (2s recommended), never an exact value, so a slower-but-correct peer is not failed

### 11.5 The validation gate

> **The gate:** two independently written peers, in different languages, that **meet** at a key derived from the same agreed input in each of the four modes, exchange candidates, and establish a **direct punched transport that survives idle**. Prose review does not catch the failures this spec is mostly about — every cross-peer MUST here describes a way to be **self-consistent and still silently never meet**, which is exactly what a same-implementation test cannot see.
>
> **The gate also asserts single-exchange completion `[MUST; added 2026-08-06]`.** The channel MUST open within a **bounded number of coordination deposits** — O(1), a small constant set by the retry budget (§7.2), **not** scaling with poll or dispatch count. A peer that opens the channel only by depositing many independent negotiations and getting lucky on overlap **FAILS the gate even though a channel opened**: brute-force establishment passes a volume-blind "did a channel open?" gate green while violating `EXTENSION-NETWORK.md` §10.3 **obligation 5** (single-flight per peer) and hammering the shared carrier the §1574 / obligation-4 discipline protects. **The gate counts third-party deposits and fails on O(N).** **Counting semantics, pinned so the gate means the same thing cross-impl `[MUST]`:** count **`offer` deposits at the node vantage, per side, per establishment**. The conformant ceiling is **§7.2's exchange-attempt budget (3)** plus a bounded re-offer allowance for glare-rollback / ICE-restart (§6.5) — a **small single-digit** figure per side (`entity-browser-rust` measured **~4/side** under single-flight; a maintain-peer-driven establishment deposits **one** exchange per reconnect tick, §7.2.1, and opens in a few ticks). Set the gate ceiling as a **fixed O(1) constant** with headroom over that and far below brute force. **16 is a safe ceiling** (browser-rust's harness `SIG_DEPOSIT_BOUND`), but it is **not** a read of §7.2 — §7.2's budget is **3 exchange attempts**, and the gate counts *deposits-over-run*, a larger quantity than *exchanges-per-establishment*; the ceiling MUST be fixed (never scaling with poll / dispatch / tick count), which is the property that fails brute force, not the exact value. **The ceiling is scoped by substrate, and a claim MUST name its substrate's value `[MUST; added 2026-08-07]`:** the **floor** is §7.2's 3 exchange attempts, and the re-offer headroom above it is whatever that substrate's re-offer paths can produce — **WebRTC has glare-rollback and ICE-restart re-offers (§6.5), native TCP simultaneous-open has neither.** So a bound of **3 is correct on the native punch** (`entity-core-go` measures **2/1 per side** against 3) and would be **wrong on WebRTC**, where a single legitimate glare rollback exceeds it; **16 on WebRTC** and **3 on native** are both conformant readings of the same rule, and neither transfers to the other substrate. Cross-impl comparison of raw deposit counts across substrates is meaningless — compare each against its own substrate's bound. *(Found by `entity-browser-rust`, 2026-08-06 — a no-single-flight impl opened the channel with ~470 deposits at ~29 s where single-flight opens with ~4/side at ~1 s; the first real exercise of the WebRTC leg, which native TCP-simultaneous-open never drove.)*
>
#### 11.5.1 What each substrate can and cannot prove `[normative — a gate claim is scoped by its substrate]`

> **MUST.** A conformance claim against §11.5 **MUST name the substrate it was obtained on.** A green result on a substrate that cannot exercise a property **is not evidence about that property**, and reporting it as one is an overclaim (`ADR-0012`).

| Substrate | Proves | **Structurally cannot prove** |
|---|---|---|
| **Loopback / in-process** | choreography, socket dance, message encoding, key derivation, `fire_at` alignment | **anything whose failure mode is "the NAT has no mapping"** |
| **Emulated dual-NAT** (`netns` + `iptables`, one host) | traversal, both-sides-fire (§7.1 step 4), endpoint binding (`EXTENSION-NETWORK.md` §6.7.3), keepalive across a mapping | real-world NAT *diversity* — CGNAT, symmetric, carrier quirks, real RTT/jitter |
| **Two real independent NATs** | the above **plus** NAT diversity — closes this gate | — |

> **The loopback blindness class, stated once and generally `[MUST NOT rely on]`.** **With no NAT in path, every packet reaches the counterpart regardless of which socket sent it or whether the counterpart opened a hole.** So a loopback harness **cannot distinguish a conformant peer from one that violates any mapping-dependent MUST** — it will report green for both. **Three instances have now been demonstrated with real peers, not argued:**
>
> - **§7.1 step 4 (both sides fire outbound).** A listen-only peer passes on loopback; it cannot traverse a NAT.
> - **`EXTENSION-NETWORK.md` §6.7.3 (punch from the socket whose mapping was observed).** A peer that advertises one endpoint and punches from another passes on loopback — its dial still lands on the counterpart's listener — and fails under two NATs with `dialed_outbound: true` and no path. *(acting on the mapping)*
> - **§9.3 / `EXTENSION-NETWORK.md` §6.7.3 (a mapping belongs to the socket that observed it).** A NAT-type detector that gathers from two sockets reads each socket's own bind port and concludes *endpoint-dependent* (symmetric) on a machine with **no NAT at all** — `9731→9731, 9732→9732` renders as "port differs per destination," well-formed observations and a confidently-wrong verdict. Demonstrated on loopback by the cohort's `--nat-type` precheck; the fix (`detect_mapping` pins one socket, so the trap is not expressible at the call site) is a **call-site invariant, not discipline**. *(measuring the mapping)*
>
> **The shape all three share:** the peer is *honest in every observable exchange* and wrong only in a **mapping-dependent value** whose consequence lives in a real NAT the harness does not have. The first two *act on* the mapping (which socket fires, which endpoint is punched); the third *measures* it (the NAT-type verdict) — so the class is **any mapping-dependent value, measured or acted-on**, not the punch specifically. **This is the same failure shape as a cross-peer seam equivalence-collapse** — a distinction that is invisible locally and springs apart at the seam — and it is why loopback-green is a statement about choreography and never about traversal.
>
> **Consequence for implementers:** a mapping-dependent MUST is **unverifiable** on loopback. Do not write a loopback test for one and record it as covering the requirement; the test will pass forever and mean nothing. Emulated dual-NAT is the cheapest substrate that exercises this class, it is deterministic, and it runs in CI.

> **On the racing-socket tie-break (§7.4.1) — not a conformance surface.** An implementation whose `fire_at` handler **cancels the losing path on first success** can never observe two surviving sockets: cancellation closes the listener and aborts the remaining dial within microseconds, so the second connection never completes. The tie-break's race arm is therefore **defensive and undrivable above unit level**, and its absence from a process-level suite is **not a coverage gap** — a unit test is the only place it can be driven, in any implementation. **An implementation MUST NOT weaken cancel-on-first-success in order to make the tie-break observable**: that trades correctness in the punch hot path for testability of a defensive branch. *(Recorded so it is not re-filed as a gap, and so that an implementation which does change cancellation semantics knows it has changed something observable.)*
>
> **And it is not hardware-blocked either `[added 2026-08-10 — this ambiguity cost a cycle]`.** The note above says the race arm is not a conformance surface; it did not say **where the tie-break's coverage does belong**, and both readings of that silence were wrong. It is **not** waiting on real-NAT diversity (G4): two *conformant* peers cannot produce distinct 4-tuples under any topology, because `EXTENSION-NETWORK.md` §6.7.3 requires each to punch **from the socket whose mapping it advertised**. Where mapping is endpoint-independent the two directions share one mapping, so the pair is one 4-tuple and its reverse; where it is endpoint-dependent (symmetric NAT) the punch does not establish at all and falls back to relay (§7.1 step 6). Distinct 4-tuples therefore appear only where the punch was already going to fail. It is equally **not** a harness item: driving it above unit level requires building a peer that advertises one endpoint and punches from another — which `§11.5.1` names as a **known defect**, not a test fixture. **The tie-break's race arm is a unit test in each implementation, permanently — not a process-level gap, not a G4 entry, and not work anyone is waiting on hardware to start.**
>
> *(Corrects arch `a19234e` §5, which put it "on the G4 pile." A G4 entry implies hardware makes it drivable; hardware does not. `core-go` was right that it could be constructed and wrong that constructing it would measure anything, and arch was right that the topology cannot produce it and wrong about why that mattered. Both sides read the same fact off the same note and neither could settle it from the text — which is the defect being fixed here.)*

> **Build state does not live here.** Which implementation has reached which rung, as of when, is peer-reported and dated — read `docs/status/STATUS.md`, `docs/status/WORKSTREAMS.md`, and the peers' own reports. *(This note previously enumerated per-implementation build state and was wrong within hours, twice. `docs/DOCTRINE-COHORT-STATE-TRACKING.md` **D8**.)*

## 12. Types Installed

| Type | Description |
|------|-------------|
| `system/signaling/connect-request` | Initiator's candidate offer (§6.1) |
| `system/signaling/connect-response` | Responder's candidate answer (§6.1) |
| `system/signaling/punch-sync` | Firing alignment; carries `fire_at` as a delay (§6.1, §7.2) |
| `system/signaling/signed-blob` | The self-contained §6.3 envelope a bucket holds — `{entity (verbatim), public_key, signature, signer}`; signature bucket-bound (§6.3) |
| `system/signaling/webrtc/offer` | WebRTC-substrate SDP offer + initial candidates (§6.5) |
| `system/signaling/webrtc/answer` | WebRTC-substrate SDP answer (§6.5) |
| `system/signaling/webrtc/candidate` | Trickled ICE candidate — structured `{candidate, sdp_mid, sdp_mline_index, ?username_fragment}` (§6.5) |
| `system/signaling/rendezvous-key` | The fixed type string in the key derivation's hash input — a derivation constant, never a stored entity (§3.1) |
| `system/signaling/offer-request` / `-result` | Input and output of `offer` (§4.1) |
| `system/signaling/collect-request` / `-result` | Input and output of `collect` (§4.1) |
| `system/signaling/advertise-result` | Output of `advertise` (§4.5) |
| `system/signaling/limits` | Published bucket and TTL limits (§4.5) |

## 13. Open Items

| # | Item | Disposition |
|---|---|---|
| 1 | **Message namespace** — ~~`system/nat/*` versus `system/signaling/*`~~ | **RESOLVED 2026-08-02 → `system/signaling/*`** (`PROPOSAL-NAMESPACE-CLEANUP-AND-BROWSER-LEG` §3.1). The owner-not-problem-domain rule (`SPECIFICATION-FORMAT.md` §8.4.4): a segment names *who defines the type*, not what it is about; `nat` was a problem domain that would strand its siblings the moment a second substrate (WebRTC) arrived. **This is the flag day** — wire-visible because the derivation hashes the literal type string `system/signaling/rendezvous-key` (§3.1), so all impls + any deployed lobby change in one window; confirm with a cross-impl meet before deploy. |
| 2 | **Rate-limit thresholds** (§8.2) | **Needs a deployment call before a public unwrapped surface is exposed.** The mechanism and error code are pinned; the numbers are not. |
| 3 | **TLS / linkability** (§8.5) | Accepted for v1 with the cost stated. Revisit if a deployment's threat model makes rendezvous-key metadata unacceptable. |
| 4 | **Out-of-band carrier** (QR / short-code) | The zero-infrastructure human-present case. Kept as a secondary carrier under review; message framing would be a `connect-request` / `-response` pair encoded into a QR or short code. |
