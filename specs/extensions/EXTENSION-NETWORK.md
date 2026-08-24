# Network Extension — Normative Specification

**Version**: 1.8

> **Amendment 12 — partial fold: the §A1/§5.4 join (new §5.4a; §5.4 pseudocode corrected; §12.1 bullet; two new vectors) `[2026-08-12]`.** Amendment 12 remains **ratified but not folded** as a whole; this folds the one part a cohort implementation proved was load-bearing, ahead of the rest. `entity-core-go` found that the §A1 transport-error eviction **destroyed the keepalive loop that owes the §5.4 `suspect → disconnected` escalation, at the moment it became owed** — so the peer stayed `suspect` forever, §4.1 reconnect never fired, and the §A3 consumer latency contract was silently unmet on every transport-error-first path (the *common* path — a transport error is how a dead peer is usually noticed first). **Root cause is a fold gap made worse by this spec's own pseudocode:** §A1 lives only in the proposal, §5.4 lives here, nothing owned the composition — and §5.4's reference pseudocode put the escalation *inside* the ping loop, so the defect was a faithful implementation of what this section said. **The pseudocode is corrected, not merely annotated** (the §8.3 containment lesson: when pseudocode and prose disagree, implementations follow the pseudocode). §5.4a states the join as a MUST, pins the `suspect`-guard scope so §10.2 fallback and RELAY terminal-hop evictions still MUST NOT demote, orders the grace `sleep` before the status read, and **rules `reason` preservation** — the escalation carries the episode's *originating* reason (`transport-error` on the seam path, `keepalive-miss` on the idle path), because re-stamping asserts pings that were never sent and destroys the only signal distinguishing the path that was broken. Both halves vectored; the negative half is required, since escalating on any *unbound* peer rather than any *`suspect`* peer passes the positive vector and breaks the §A1 seam scope. **Second gap of this exact shape in two cycles** (after the §5.5a granter frame): a reachable state all impls agree on by construction that **no vector visits**, so conformance-green said nothing about it — found by an implementation, not by prose review, both times. *(Observed: `entity-core-go`, source-read at `b55101f`, 2026-08-12.)*
>
> **Amendment 12 — second partial fold: §A6.6, the `chain_id` spelling (§2.4 field declaration; §4.1 pseudocode) `[2026-08-13]`.** §4.1 minted `chain_id = "network/maintain/" + session_id` — **three path segments** — and that is not a style question. `chain_id` is **interpolated as a path segment** of `EXTENSION-CONTINUATION` §3.10.1's marker path `.../lost/{chain_id}/{step_index}/{reason}/{marker_hash}`, so a value containing `/` forks the marker tree into extra levels and makes the scheme unwalkable at a fixed depth: given `lost/network/maintain/abc/req-1/connection_failed/<hash>` nothing can tell where `chain_id` ends and `step_index` begins. **The old form was therefore internally inconsistent with CONTINUATION's own landed path scheme, not merely stale** — this spec and that one could not both be satisfied. Corrected to `"network-maintain-" + session_id`; the single-segment rule itself belongs to and is routed to `ENTITY-CORE-PROTOCOL` §3.11, where `chain_id` is declared. **No wire change, no renumber:** `chain_id` is opaque (§3.11 gives it no format, nothing parses one), so this is three impls emitting a different generated string. **Found by an audit of arch's own fold debt, and the sharpest evidence is that the reference conformance oracle already rejected this spec:** `entity-core-go`'s `validate-peer` FAILs any peer whose `maintain-result.chain_id` contains `/` (`cmd/internal/validate/network.go:125`, source-read at `a02ab5e`), so **a peer implementing §4.1 as written failed the conformance run** — while go, rust and python had all three independently converged on the single-segment form and recorded in their own source comments that the spec's version was non-conformant (go `ext/network/session.go:118`, rust `extensions/network/src/lib.rs:726`, py `entity_handlers/network.py:373`). Three-way convergence against stale pseudocode is the strongest signal this method produces; the failure was that nothing carried it back here for four weeks. *(Confirmed by `entity-core-go`, `ROUTING-2026-08-13-k`, with the CONTINUATION-inconsistency argument above supplied by them and adopted in preference to arch's weaker "three impls disagree" framing.)*
>
> **Amendment 12 — third partial fold: §A2 / §A6.2 / §A6.3 / §A6.4 / §A6.5, the retry lifecycle (new §2.2.1, new §2.11; §2.2 gains two OPTIONAL bounds) `[2026-08-13]`.** `system/peer/status` carried a bare three-state enum, so no consumer could tell *why* a peer left and none could choose a recovery — a transient drop, an auth rejection and a deliberate shutdown were one value. §2.11 lands the `reason` vocabulary and its recovery mapping; §2.2 gains `max_attempts`/`max_elapsed_ms` (**both default unset ⇒ retry forever, which §A6.2 now states normatively** so no implementation invents its own cap); exhausting a bound is a `reason` (`retry-exhausted`), **not** a fourth status. §2.2.1 pins the retry schedule as a **pure function of `(failing_since, cfg, now)`**, with `attempt`/`next_attempt_at` **derived and explicitly never stored** — storing them is a tree write per attempt, the exact fan-out §6.6 rejected — and with the three divergence points (1-indexed `k`, `attempt` counts retries *fired*, `elapsed_to(0)=0`) pinned, because an unpinned derivation relocates a divergence rather than removing it. **The fields themselves are declared upstream at `ENTITY-CORE-PROTOCOL` §3.13 (`9829c6f`), not here** — this spec owns the lifecycle semantics only; two specs declaring fields on one entity is how implementations end up with two shapes. **Additive: all OPTIONAL, no wire change, no renumber; today's behavior is preserved exactly when the bounds are unset.** *(Adopted from convergence, not legislated ahead of it: `failing_since` was **measured on the wire 3-of-3** — go, rust `1152d35`, py `ad0ef98`, `network_reconnect_anchor` PASS 5/5 with 0 skips per seat — reported by `entity-core-go` at `a02ab5e`, whose own conformance check had been holding this at WARN on a stale build-state comment that survived because **a WARN is invisible in a green run.** Three implementations interoperating on a field no landed spec carried: the same shape as the `chain_id` defect corrected the same day, caught one step earlier.)*

> **Amendment 14 — the live-establishment seam (new §10.3; §10 step 3b; §10.2 correction).** §10 step 3 resolves *durable* transport profiles, so a NAT'd peer — whose published endpoints are unreachable from outside — falls straight to the store-and-forward terminal even when it is reachable *right now* by traversal. Amendment 14 names `establish_live(peer_id) → connection | null`, consulted at **step 3b**: after profile resolution fails, **before** the §10.2 delivery fallback. **It returns a connection, not a result, and that is why it is a separate seam** — the ladder re-enters ordinary dispatch on success, so the connection is pooled and reused by every later dispatch. Forcing traversal through §10.2's `dispatch_fallback` (which returns a delivered result) would punch a fresh hole per message and hide the connection from §10 step 1. **This corrects §10.2's forward-looking claim** that the punch and store-and-forward would "escalate from the same step-4 site": that paragraph predates the punch's design, and its own "tries live first and store-and-forward last" is unachievable from a single site consulted once. Ordering is now a **MUST** — live first, store-and-forward last. Two obligations on the returned connection, both MUST: it is an **ordinary transport** (never a new transport type, never published as a durable `system/peer/transport/*` profile — the mapping is session-scoped, §6.7.3), and it **MUST run keepalive** (§5), because a punched NAT mapping expires on silence and an idle punched connection dies in a way no same-host test reproduces. **Additive:** `null` when no traversal extension is installed ⇒ byte-identical to the pre-seam ladder; no V7 change, no wire change, no new capability or error code. The v1 policy behind the seam is `EXTENSION-SIGNALING.md` §6. **Cohort review absorbed the same day it landed (2026-07-31), and the seam survives with four additions** — the four open items `PROPOSAL-NETWORK-LIVE-ESTABLISHMENT-SEAM` §6 flagged at fold are now closed by two independent implementations that built it (Go and Rust): the signature gains **`ctx`** (a seconds-long seam with no cancel is a hang — both raised it independently); **one call with strict ordering is confirmed** as right rather than merely simple (both rejected racing it against §10.2 as a layering regression); the **`connection` type and the handshake boundary stay unpinned** as impl-idiomatic — the two builds factor the handshake differently on opposite sides of the seam and both interop, because the seam is internal to one peer — while the **identity check and the retry composition become MUSTs** (obligations 3 and 4), because those two *are* cross-peer observable. Per `AGENTS.md`, cohort findings on a just-landed spec fix it **in place**: no rev bump, v1.6 stands, and the proposal's open items now carry their answers. **Build state (peer-reported, observed 2026-07-31, post-review):** `entity-core-go` has built the seam (`core/peer.tryEstablishLive`, `ext/signaling/peerwiring`) and `entity-core-rust` has built its `LiveEstablish` counterpart; Python has not. *(The pre-review draft of this note read "not present in any implementation" — asserted the morning of the day Go's build landed and Rust reported theirs. See `docs/DOCTRINE-COHORT-STATE-TRACKING.md`: build state is peer-reported and dated, and a spec header is a poor place to carry it.)* The gate remains two NAT'd peers establishing a direct transport that survives idle — **not yet run**; both builds are loopback/in-process, which proves the choreography and not NAT traversal.

> **Amendment 13 — reachability facts (new §6.7: observed-address reflection, dial-back, candidate gathering; two new capabilities; two new operations).** A peer dispatches by reachability class (§10) but has had no protocol way to learn its **own** reachability facts — its public NAT mapping, whether it is publicly dialable, what addresses it might be reached at. §6.7 lands the three facts and stops there: **gathering is a local fact, exchanging is a protocol**, and the punch-coordination protocol that acts on them is not in this spec. Adds `observe-address` → `system/network/observe-address-result` (§6.7.1) and `check-reachability` → `system/network/check-reachability-result` (§6.7.2), gated by new `system/capability/network-reflect` (broad default grant reasonable — a mirror) and `system/capability/network-dialback` (restricted — it causes the responder to emit traffic at an address), plus the `system/network/candidate` type (§6.7.3). **Additive and not v1-blocking:** the section is OPTIONAL as a whole (§12.3) — the asymmetric NAT case already works via §10's `held_connection_client`, and this changes no §10 pseudocode, no wire format, and adds no error code. **Every rule inside it is a MUST when offered** (§12.1), because each is a cross-peer seam that prose review does not catch: reflection returns the transport source and never a body echo; the observed address is **never** persisted to `system/connection.address` or a transport profile (it is a *responder-side* fact and every durable address field in this spec is *dialer-side dialable-endpoint* state — the cheap fix corrupts §10 dispatch for every other reader); dial-back targets **only** the observed source (a body-supplied target makes every dial-back peer a DDoS reflector); candidates are never durable profiles; and a `srflx` candidate is the mapping of the socket the peer punches from. Folds `PROPOSAL-NETWORK-REACHABILITY-FACTS` §2–§5 in full, including the 2026-07-29 ownership ruling (the HELLO-handshake mechanism is `entity-core-protocol`'s to ratify, not this repo's — it routes upstream and gates nothing here, so the op is the v1 path) and the §4.2 candidate-to-socket MUST. **Build state (peer-reported, observed 2026-07-31, corrected):** `entity-core-go` **has built the §6.7.1 `observe-address` responder** — dispatched, rate-limited, and live-validated under its `reachability` validator category. **Corrected 2026-08-07 (`entity-core-go`, full-coverage run):** the **client-side srflx gatherer** (dial a reflector → call `observe-address` → produce the `srflx` candidate the punch fires from) is **built in Go and in Rust** — Go via `--reflector` on `cmd/signaling-punch` (`punchwire.ObserveSRFLXFrom` + `signaling.DialReflector`), Rust at the same rung on its `signaling-punch`. The prior "absent in every tree" reading is retracted; it was already stale against this repo's own 07-31 status entry. `check-reachability` remains unbuilt. *(The pre-correction draft of this note claimed both operations were absent everywhere on the strength of an arch-side name search; the peer that had built one reported it the same day. Build state is peer-reported — `docs/DOCTRINE-COHORT-STATE-TRACKING.md` D1.)* The §6.7.5 gate is **PARTIALLY DISCHARGED, not closed `[corrected 2026-08-07]`**: its **reflect half has now run cross-impl, 4/4** — a Go reflector serving both drivers, `srflx_source: "reflector"` on both seats. That half had never been exercised in any implementation before (every V3 crossing the cohort published reported `srflx_source: "bind"` — built, never driven). **The dial-back half across a real NAT is still un-run**, and is one of the three gates blocked on the same missing cohort infrastructure (§6.7.5 dial-back · `EXTENSION-SIGNALING.md` §11.5.1 S5 · the §10.3 seam gate): hosts behind genuinely different NATs. **That is infrastructure, not implementation debt against any repo.** *(Numbered 13, not 12: Amendment 12 — the NETWORK liveness reactive buildout — is ratified but not folded, and holds that number. **Partially folded 2026-08-12** — the §A1/§5.4 join landed as §5.4a; see the Amendment 12 banner above. The rest is still owed.)*

> **Amendment 11 — dispatch-fallback seam at §10 step 4 (store-and-forward escalation; new §10.2).** The §10 ladder's step-4 terminal (queue/502) cannot originate delivery to a peer with no live or recurring session that is offline/NAT'd right now. Amendment 11 names a `dispatch_fallback(peer_id, execute) → {ok, result} | null` seam consulted once at step 4 **before** the terminal; the store-and-forward policy lives in RELAY (§6.2.1), never in NETWORK — same layering boundary as the relay→routing `resolve_next_hop` seam. **Additive / v1.x:** `null` (no RELAY installed) ⇒ byte-identical to the pre-seam terminal; non-RELAY v1 floor unchanged. Two cohort-convergence corrections fold in with it: **(a)** the step-4 terminal is restated as impl-variable — `queue_pending` (§8 outbox) is an OPTIONAL rung an impl MAY interleave only if it implements §8 (Rust + Python ship no §8 outbox; their terminal is a bare error), so "byte-identical when unset" means "behaves as this impl's terminal does today"; **(b)** a normative insertion-site MUST — the seam is consulted at the caller holding both `peer_id` and the `execute` envelope, never inside connection-resolution (3-impl independent convergence). Conformance gates the outcome (offline-target delivery lands at the inbox; target polls + verifies signature as direct), not the policy's internal rung choices. No V7 change, no wire change, no new cap/error code. Cohort review converged 3-way before fold (Go build-tested `INBOX-RELAY-FALLBACK-1` PASS; Rust + Python confirmed seam-vs-inline fit against their dispatch ladders — Rust as a hard crate-DAG constraint).

> **Amendment 10 — `serve_scope` MUST cover the trie-node closure when `signed_pointer` is advertised (§6.5.6 `published-set` bullet).** Cross-impl-run absorption: three impls picked three publisher serving-scope shapes (Go = whole-store; Rust = namespace-only; Python = closure-of-signed-root). Only Python's shape lets the PEER-MANIFEST §1.1 walk-from-signed-root complete: CHAMP trie nodes are hash-linked, not path-bound (V7 §1.7), so the §6.5.6 `published-set` path-binding check 404s on interior nodes — and the consumer's `CONTENT_GET` walk from `root_hash` halts. **Resolution:** when a publisher advertises a `signed_pointer` in the same `http-poll` profile, the `serve_scope` content-face MUST also resolve the transitive trie-node closure reachable from `published-root.root_hash` (root + interior + leaf-bound content + `published-root` entity + signature entity). Publishers serving `published-set` without `signed_pointer` (content-only mirror) retain path-bound-only shape — unchanged. `whole-store` trivially covers this case. Cohort impact: Rust either implements closure-scope when keeping `signed_pointer` advertised, or drops `signed_pointer` from the advertised profile to stay namespace-only conformant.

> **Amendment 9 — reserved-word table extensibility hook + length-floor rule (§6.5.6 / G4).** L5 application conventions (sites / repos / spaces and future L5 patterns) need to register URL projection prefixes at the §6.5.6 demux layer so the legacy web can address them. The v0.4 review's first instinct — enumerate the L5 entries inside NETWORK — was a layer violation: NETWORK is the entity system's *networking* extension; it owns the URL routing mechanism and its own reservations, but it has no business cataloguing L5 applications. Amendment 9 makes the layer separation explicit: **(a)** the reserved-word table is **explicitly extensible** — other specs MAY register additional first-segment literals, and the §6.5.6 demux consults the union; **(b)** any registered reserved word MUST satisfy the **length-floor rule** (strictly shorter than the Ed25519 peer-id minimum encoded length) so the literal-then-parse-by-string demux algorithm stays correct regardless of how many words are registered; **(c)** NETWORK does NOT enumerate the registered extensions — each owner spec lists its own. The change is to §6.5.6 G4 (the reserved-words bullet); no new wire surface, no demux algorithm change, no behavior change for existing impls. Companion: `APP-CONVENTION-SEMANTIC-CONTENT-SITE v0.5` registers `sites` as the SITE convention's URL projection prefix in the L5/applications domain.

> **Amendment 8 — session state entity (R6) + reachability-class outbound dispatch (R5); folds the §8 round (Q1 `priority`, Q6 `advertised_at`→optional, Q2 profile-id=final-segment) and drops the R2 inbound-reuse SHOULD.** R6 makes §6.1 literal: per-peer session state (held capability + handshake bookkeeping) becomes a tree entity at `system/peer/session/{remote_peer_id}` (new §6.6), so the held capability survives reconnect and is inspectable, instead of living in per-connection memory. R5 rewrites §10 around the reachability-class table: the dispatcher reads `system/peer/session/{peer}:held_capability` to skip the handshake, resolves the target's transport profiles by `(priority asc, profile-id lex)`, and branches push / queue by the target's reachability class. Q1 adds an optional `priority: uint` to the live profile entities (lower = preferred; default 100; reserved `primary` unset → 0). Q6 relaxes `advertised_at` from MUST to OPTIONAL (it was advisory-only — §6.5.1a D3 — so requiring it was ceremony and a cross-impl footgun). Q2 pins `profile-id` = the final path segment. R2's "SHOULD reuse the accepted inbound socket" prose is dropped to pure impl-detail (post-R6 it carries no auth content). **Landed behind a 3-way-green gate: R6 validated 7/7; Q1 field round-trip + the §10 priority-selection-ordering (Q5) gate all 3-way green (Go/Rust/Py).** Amendment 8 covers PROPOSAL-TRANSPORT-FAMILY-LIVE-REACHABILITY-AND-SESSION-LIFECYCLE §8.9–§10 plus the staged R5–R6 landing. NOT in scope: R4 poll-fallback `delivery_mode` (Phase-2; deferred), REGISTRY, WS/WebRTC build, the cap-axis capability-model ruling (separate track).

> **Amendment 7 — live-transport reachability framing; the §6.5.2c half-duplex caveat corrected (§6.5.1b new + §6.5.2c).** Impl review of HTTP-live bidirectional delivery (entity-core-go `24de569`) found the §6.5.2c parenthetical *"subscribe needs a duplex transport: tcp/websocket"* to be **false**, not merely ambiguous. Subscription/async/peer-initiated delivery is a *fresh EXECUTE from the source to the target's inbox* (EXTENSION-INBOX §2) — the source dials the target — so the constraint is the **target's reachability**, never per-connection duplex. An HTTP-listener target receives deliveries over an ordinary half-duplex source→target POST. WebSocket's real distinction is narrower: it also pushes to a **non-listening** target (browser). Rewrites the §6.5.2c parenthetical to the reachability framing and adds informative §6.5.1b (duplex taxonomy + "active connection" is direction-agnostic for full-duplex transports ⇒ inbound sockets are reusable for outbound dispatch). **Wording / convergence-intent only — no wire change; no behavioral change for tcp / http-listener / ws-server targets (impls already behave this way).** Two items are explicitly NOT in this amendment and travel as a proposal + cross-impl feedback round: (1) capability held per-peer-session vs per-connection (§6.1 leans per-peer; impls lean per-connection); (2) the poll-fallback delivery mode for targets that can neither listen nor hold a duplex socket (Phase-2; new-extension-vs-application-logic open). R1+R2 are this amendment; R3 cap-lifecycle + R4 poll-fallback are proposed/deferred.

> **Amendment 6 — TREE_GET leaf is a hash pointer, not the dereferenced entity (§6.5.3 / §6.5.3.1 / §6.5.6).** Impl feedback on Amendment 5 surfaced an internal contradiction: §6.5.3 step 5 (two-hop) vs §6.5.3.1 leaf bullet (one-hop, dereferenced entity). **Resolved two-hop.** The `http-poll` `TREE_GET` leaf route (`{peer_id}/{path}{tree_leaf_suffix}`) returns the **bound content hash** as a `system/hash` value (bare 2-key `ECF({type:"system/hash", data:H})`), NOT the dereferenced wire entity. The consumer reads `H` and fetches the bytes with a second hop `CONTENT_GET /content/{hex(H)}`. This is exactly **`tree:get mode:"hash"` (V7 §1.7) over HTTP** — no new type, no V7 bump. One-hop is non-conformant: it materializes a separate entity copy at every tree path bound to `H`, defeating the V7 §1.7 content-store dedup invariant (same content at N paths ⇒ one copy) that a static CDN cannot recover. Serving mode (§6.5.6) returns the same pointer (uniform consumer code path live + static). Also grounds `serve_scope` in V7 §1.4's *Local view* / *Authority layers* model (pointer, not restatement). Cohort impact: one function each (live peers' `serveTreeEntity` → return hash; `validate-peer serving_mode` → assert pointer + second-hop deref; workbench-go → wrap its hash payload in the ECF envelope, at whatever width its format byte implies).

> **Amendment 5 — HTTP pathing standardization (§6.5.3 / §6.5.3.1 / §6.5.6).** Read-surface pathing: listings are **named objects** (`{path}{tree_listing_suffix}`, default `.list`, distinct from `tree_leaf_suffix`) — **no trailing slash** (it doesn't survive static CDNs); append-one/strip-one bijection; `MANIFEST_GET` body+status defined; three-prefix endpoint (+`manifest_url_prefix`); first-segment **literal-or-peer-id-parse** demux with reserved words `{content, manifest, peers}`; canonical status table (200/400/404/405, +414 MAY; no 501; no 3xx); `serve_scope` is a **capability token** (one ACL machinery, evaluated by the live-surface cap evaluator with the published cap as the effective context); listing scope-gating (filtered `count`, empty-in-scope ⇒ 200, post-scope offset); pagination via optional `next_page` on `system/tree/listing` (V7 §3.9). Landed at Rev e after two-round cohort convergence (rust/py/entity-browser-rust/go). CORS + MIME are deployment config (`RUNBOOK-CDN-BROWSER-DEPLOYMENT`), not impl code.
**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.9+), EXTENSION-INBOX.md (v5.0+), EXTENSION-CONTINUATION.md (v1.2+), EXTENSION-SUBSCRIPTION.md (v3.3+)
**Optional**: EXTENSION-REVISION.md (v2.0+) — reconnection catch-up via revision pull
**Encoding**: ENTITY-CBOR-ENCODING.md (ECF)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. Overview

The network extension manages persistent peer relationships: maintaining connections, detecting failures, reconnecting, restoring subscriptions, and draining pending deliveries. It builds on the reactive pipeline (inbox + continuation + subscription) to express connection lifecycle as entity-native processes.

This extension defines:

- Session vs connection distinction
- Keepalive mechanism
- `maintain-peer` operation and its continuation graph
- Reconnection with backoff
- Subscription restoration after reconnection
- Pending delivery queue
- Graceful close with reason codes

### 1.1 Scope

This extension covers:

- **Connection lifecycle** — liveness detection, failure, reconnection
- **Session management** — relationships that survive connection drops
- **Subscription restoration** — re-establishing notification delivery after reconnect
- **Pending delivery** — queuing outbound messages during disconnection

This extension does **not** cover:

- Peer discovery (mDNS / DNS-SD — `EXTENSION-DISCOVERY`, shipped); seed / bootstrap peers — `EXTENSION-REGISTRY` §7 + `system/config/bootstrap` (not a DISCOVERY backend)
- Relay / store-and-forward (`EXTENSION-RELAY`, shipped)
- NAT traversal — the **punch-coordination dance** (a thin protocol over a pluggable signaling carrier).
  See `EXTENSION-SIGNALING.md`, which registers behind the §10.3 live-establishment seam.
  **In scope, as of Amendment 13:** the *reachability facts* the punch consumes — observed-address
  reflection, dial-back, and candidate gathering — are **§6.7 of this spec**. Gathering a candidate is a
  local fact and lives here; exchanging one is a protocol and does not.
  (Note: the asymmetric-NAT case — reaching a NAT'd peer from a public peer — is *already* covered by
  the `held_connection_client` reachability class, §10.)
- Group coordination (EXTENSION-GROUP)
- Transport negotiation (ENTITY-CORE-PROTOCOL.md §4)

### 1.2 Design Principles

**Entity-native lifecycle.** Connection lifecycle is expressed as continuation graphs in the entity tree. Subscriptions watch operational state entities; continuations react to state changes. The lifecycle is inspectable, subscribable, and debuggable through the same mechanisms as any other entity data.

**Wire interoperability.** This extension describes entity-native peer lifecycle using continuations and subscriptions. Imperative implementations (reconnection in connection manager code) interoperate — the wire protocol is the same regardless of internal execution mechanism. Operational state types (v7.9 §3.13) enable cross-implementation observability.

**Session over connection.** An authenticated relationship (session) outlives any individual transport connection. Subscriptions, pending deliveries, and capability grants are tied to sessions, not connections. Connection drops are recoverable; session termination is deliberate.

---

## 2. Type Definitions

### 2.1 Maintain Request

```
system/network/maintain-request := {
  fields: {
    peer_id:     {type_ref: "system/peer-id"}
                 ; Peer to maintain a relationship with
    address:     {type_ref: "primitive/string", optional: true}
                 ; Initial address (e.g., "192.168.1.100:4040")
    reconnect:   {type_ref: "primitive/bool", optional: true}
                 ; Auto-reconnect on disconnect (default: true)
    resubscribe: {type_ref: "primitive/bool", optional: true}
                 ; Auto-restore subscriptions on reconnect (default: true)
    keepalive:   {type_ref: "system/network/keepalive-config", optional: true}
                 ; Override keepalive defaults
    backoff:     {type_ref: "system/network/backoff-config", optional: true}
                 ; Override reconnection backoff defaults
  }
}
```

### 2.2 Backoff Configuration

```
system/network/backoff-config := {
  fields: {
    min_ms:          {type_ref: "primitive/uint", optional: true}
                     ; Minimum delay between reconnection attempts (default: 1000)
    max_ms:          {type_ref: "primitive/uint", optional: true}
                     ; Maximum delay (default: 60000)
    strategy:        {type_ref: "primitive/string", optional: true}
                     ; "exponential" (default), "linear", "constant"
    max_attempts:    {type_ref: "primitive/uint", optional: true}
                     ; Amendment 12 §A6.3. UNSET = retry forever (the default).
    max_elapsed_ms:  {type_ref: "primitive/uint", optional: true}
                     ; Amendment 12 §A6.3. UNSET = retry forever (the default).
  }
}
```

**Retry-forever is the default and it is intended, not an oversight (§A6.2, normative).** A
maintained peer relationship retries **indefinitely**; `release-peer` (§4.2) is the exit. A peer
offline for a week and returning is the P2P norm, and the "give up after N" instinct imports a
client-server assumption that does not hold here. Stated normatively so no implementation invents
its own cap and diverges silently.

`max_attempts` / `max_elapsed_ms` are OPTIONAL bounds for deployments that *do* want give-up
(mobile, constrained devices, short-lived agents). **Both default unset, so today's behavior is
preserved exactly.** Exhausting a configured bound is a **`reason`, not a fourth status**
(§A6.4): the peer is written `status: disconnected` with `reason: retry-exhausted`, which is
terminal — cleared only by an explicit re-`maintain-peer`. The `system/peer/status` enum stays
three-state (`ENTITY-CORE-PROTOCOL` §3.13); `status` carries *where the peer is*, `reason` carries
*why, and what to do about it*.

#### 2.2.1 The retry schedule is a pure function `[MUST — Amendment 12 §A6.5]`

Retry state lives in the tree as **one transition-written field**, `failing_since` (declared at
`ENTITY-CORE-PROTOCOL` §3.13): set on the transition into `suspect`/`disconnected`, cleared on the
transition back to `connected`. **`attempt` and `next_attempt_at` are DERIVED, never stored** —
storing them would mean a tree write per attempt, which is exactly the subscription/revision
fan-out §6.6 rejected and §A4 removed.

The inversion is **pinned normatively**, because an unpinned derivation does not remove a
divergence, it relocates it — three implementations would invert the series three ways:

```
; Retry schedule — a PURE FUNCTION of (failing_since, backoff-config §2.2).

delay(k, cfg)  =                              ; delay BEFORE the k-th retry; k is 1-indexed
    exponential : min(min_ms * 2^(k-1), max_ms)     ; §2.2 default
    linear      : min(min_ms * k,       max_ms)
    constant    : min_ms

elapsed_to(n, cfg) = Σ(k=1..n) delay(k, cfg)  ; elapsed_to(0) = 0
                                              ; ms after failing_since at which retry n fires

attempt(now)    = max { n ≥ 0 : elapsed_to(n, cfg) ≤ (now − failing_since) }
next_attempt_at = failing_since + elapsed_to(attempt(now) + 1, cfg)

retry_exhausted ⟺ (max_attempts   set ∧ attempt(now) ≥ max_attempts)
                ∨ (max_elapsed_ms set ∧ (now − failing_since) ≥ max_elapsed_ms)
```

Pinned explicitly at the three divergence points: **`k` is 1-indexed** (the first retry is
`k=1`); **`attempt` counts retries *fired*** (0 immediately after `failing_since` — the drop
itself is not a retry); **`elapsed_to(0) = 0`**. Worked example (defaults, exponential): delays
`1,2,4,8,16,32,60,60…s` → `elapsed_to = 1,3,7,15,31,63…s`; at `now − failing_since = 10s`,
`attempt = 3` and `next_attempt_at = failing_since + 15s`.

**No jitter in v1** — deterministic, and cross-peer decorrelation comes free from the natural
spread in `failing_since`. Deferred until a deployment demonstrates thundering-herd.

**The timer is host-provided; the schedule is spec-defined.** §5.4's pseudocode already assumes a
host `sleep(interval_ms)`; this is that existing blessed pattern, not a new concession.

**Why this shape pays for itself twice.** Because the schedule is a pure function, its conformance
vector is a **pure-function table** — `(failing_since, cfg, now) → (attempt, next_attempt_at)`, no
peers, no sockets, no timing flake. The most divergence-prone surface in the retry design becomes
the most cheaply converged one. And because `failing_since` is durable, a **restarting peer resumes
at the correct escalation** instead of dropping back to `min_ms` and hammering.

### 2.3 Keepalive Configuration

```
system/network/keepalive-config := {
  fields: {
    interval_ms:  {type_ref: "primitive/uint", optional: true}
                  ; Ping interval (default: 30000)
    timeout_ms:   {type_ref: "primitive/uint", optional: true}
                  ; Pong timeout (default: 10000)
    max_missed:   {type_ref: "primitive/uint", optional: true}
                  ; Missed pongs before failure (default: 3)
  }
}
```

### 2.4 Maintain Result

```
system/network/maintain-result := {
  fields: {
    peer_id:       {type_ref: "system/peer-id"}
    session_id:    {type_ref: "primitive/string"}
                   ; Identifier for the maintenance session
    subscriptions: {array_of: {type_ref: "primitive/string"}, optional: true}
                   ; Subscription IDs created for lifecycle monitoring
    chain_id:      {type_ref: "primitive/string"}
                   ; Process chain_id for the lifecycle continuation graph.
                   ; MUST be a single path segment (no "/") — ENTITY-CORE-PROTOCOL
                   ; §3.11; see §4.1 and Amendment 12 §A6.6 for why.
  }
}
```

### 2.5 Release Request

```
system/network/release-request := {
  fields: {
    peer_id: {type_ref: "system/peer-id"}
             ; Peer to release
    reason:  {type_ref: "primitive/string", optional: true}
             ; "shutdown", "idle", "migration" (default: "shutdown")
  }
}
```

### 2.6 Release Result

```
system/network/release-result := {
  fields: {
    peer_id:    {type_ref: "system/peer-id"}
    cleaned_up: {array_of: {type_ref: "system/tree/path"}}
                ; Tree paths removed during cleanup
  }
}
```

### 2.7 Status

```
system/network/status := {
  fields: {
    maintained_peers: {array_of: {type_ref: "system/network/peer-summary"}}
    pending_count:    {type_ref: "primitive/uint"}
                      ; Total queued outbound messages across all peers
  }
}
```

### 2.8 Peer Summary

```
system/network/peer-summary := {
  fields: {
    peer_id:        {type_ref: "system/peer-id"}
    session_id:     {type_ref: "primitive/string"}
    status:         {type_ref: "primitive/string"}
                    ; "connected", "disconnected", "reconnecting"
    pending_count:  {type_ref: "primitive/uint"}
                    ; Queued outbound messages for this peer
    subscriptions:  {type_ref: "primitive/uint"}
                    ; Active subscription count on this peer
  }
}
```

### 2.9 Close Request

```
system/network/close-request := {
  fields: {
    peer_id: {type_ref: "system/peer-id"}
    reason:  {type_ref: "primitive/string"}
             ; "shutdown", "idle", "error", "migration"
  }
}
```

### 2.10 Pending Delivery

```
system/network/pending-delivery := {
  fields: {
    peer_id:    {type_ref: "system/peer-id"}
    sequence:   {type_ref: "primitive/uint"}
                ; Monotonic sequence for ordering
    execute:    {type_ref: "system/protocol/execute"}
                ; The EXECUTE to deliver
    created_at: {type_ref: "primitive/uint"}
                ; ms since epoch
    expires_at: {type_ref: "primitive/uint", optional: true}
                ; ms since epoch; delivery abandoned after expiry
  }
}
```

Pending deliveries are stored at `system/outbound/{peer_id}/{sequence}`.

### 2.11 Peer-status transition reasons `[Amendment 12 §A2/§A6.4]`

`system/peer/status` carries a three-state lifecycle enum, which tells a consumer *where* a peer
is but not *why it left* — so neither a consumer nor the reconnect continuation can choose a
recovery. A transient drop wants backoff-reconnect; an auth rejection wants re-handshake; a
deliberate peer shutdown wants *stop trying*. §10 already forks on this for **dispatch**
(403 → handshake); before Amendment 12 the **liveness** path collapsed every cause to
"disconnected."

**The fields (`reason`, `last_error`, `failing_since`) are declared once, upstream, at
`ENTITY-CORE-PROTOCOL` §3.13.** This section owns their *lifecycle semantics* — the vocabulary
and its recovery mapping — and does not re-declare the shape. NETWORK's put-sites (§4.2, §6.2,
§5.4a, the §A1 seam) are **minimal writes, not exhaustive shapes**: a bare `{peer_id, status}`
write is conformant.

`reason` is an OPTIONAL kebab enum. **A reader MUST treat an unrecognized value as generic and
fall back to backoff** (MUST-ignore-unknowns) — that is what lets the vocabulary grow without a
flag day.

| `reason` | Recovery the consumer / reconnect continuation selects |
|---|---|
| `transport-error` | backoff-reconnect (§4.1) — a single transport error on a connection believed active (§A1) |
| `keepalive-miss` | backoff-reconnect (§4.1) — the idle path (§5.4) |
| `auth-rejected` | re-handshake (§6.3), **then** reconnect — do NOT reuse the held capability |
| `peer-shutdown` | **terminal** — the session ended deliberately (§6.1) |
| `local-release` | **terminal** — this peer called `release-peer` (§4.2) |
| `retry-exhausted` | **terminal** — a configured §2.2 bound was reached; cleared only by an explicit re-`maintain-peer` (§A6.4) |
| `peer-idle` | preserve subscriptions; expect resume (§9.1) |
| `peer-migration` | preserve subscriptions; expect resume (§9.1) |

`last_error` is coded/opaque detail for humans and logs. **It is never parsed** — nothing may
fork on its content.

**`reason` is preserved across an escalation, not re-stamped** (§5.4a): a `suspect → disconnected`
escalation carries the *originating* reason of the episode, because re-stamping `keepalive-miss`
onto a transport-first episode asserts pings that were never sent and destroys the only signal
distinguishing which path broke. Both values map to `backoff-reconnect`, so nothing downstream
forks on the distinction — it is there for the operator reading the tree.

---

## 3. Handler

### 3.1 Handler Manifest

```
system/handler := {
  pattern:    "system/network"
  name:       "network"
  operations: {
    maintain-peer: {
      input_type:  "system/network/maintain-request"
      output_type: "system/network/maintain-result"
    }
    release-peer: {
      input_type:  "system/network/release-request"
      output_type: "system/network/release-result"
    }
    status: {
      input_type:  null
      output_type: "system/network/status"
    }
    close: {
      input_type:  "system/network/close-request"
      output_type: null
    }
    observe-address: {
      input_type:  null
      output_type: "system/network/observe-address-result"
    }
    check-reachability: {
      input_type:  null
      output_type: "system/network/check-reachability-result"
    }
  }
  internal_scope: [
    {handlers: {include: ["system/tree"]}, resources: {include: ["system/*"]}, operations: {include: ["get", "put"]}},
    {handlers: {include: ["system/subscription"]}, resources: {include: ["system/*"]}, operations: {include: ["subscribe", "unsubscribe"]}},
    {handlers: {include: ["system/protocol/connect"]}, resources: {include: ["*"]}, operations: {include: ["hello", "authenticate"]}}
  ]
}
```

### 3.2 Capability Model

```
system/capability/grant-entry := {
  handlers:   {include: ["system/network"]}
  resources:  {include: ["system/network/*"]}
  operations: {include: ["maintain-peer", "release-peer", "status", "close"]}
}
```

The network handler is a system handler — typically only the local peer admin has grants for it. Remote peers do not call `maintain-peer` on another peer.

**The two reachability-fact operations are the exception (§6.7).** `observe-address` and `check-reachability` are *designed* to be called by remote peers — they answer "how do I look from outside" and "am I dialable," questions only another peer can answer. They carry their own capabilities, deliberately separate from the admin grant above:

```
system/capability/network-reflect     ; gates observe-address     (§6.7.1)
system/capability/network-dialback    ; gates check-reachability  (§6.7.2)
```

A peer granting `network-reflect` is **not** thereby granting `maintain-peer`. Both are rate-limited; posture and rationale are in §6.7.4.

---

## 4. Operations

### 4.1 Maintain Peer

Creates a continuation graph that manages the lifecycle of a peer relationship.

```
EXECUTE system/network  operation: "maintain-peer"
  resource: {targets: ["system/network"]}
  params: {
    type: "system/network/maintain-request"
    data: {
      peer_id: "12D3...",
      address: "192.168.1.100:4040",
      reconnect: true,
      resubscribe: true
    }
  }
```

The handler:

1. Connects to the peer if not already connected (handshake via `system/protocol/connect`)
2. Creates operational state entities (v7.9 §3.13):
   - `system/peer/status/{peer_id}` → `"connected"`
   - `system/connection/{peer_id}` → connection details
3. Creates lifecycle subscriptions:
   - Subscription on `system/peer/status/{peer_id}` → detects disconnect
   - Subscription on `system/connection/{peer_id}` → detects connection state changes
4. Creates lifecycle continuations (when `reconnect: true`):
   - Continuation at `system/inbox/network/{peer_id}/on-disconnect` → dispatches reconnection
   - Continuation at `system/inbox/network/{peer_id}/on-reconnect` → dispatches subscription restoration
5. Starts keepalive (§5)
6. Returns session info

```
handle_maintain_peer(ctx, params):
  peer_id = params.peer_id
  session_id = generate_id()
  ; Amendment 12 §A6.6. A chain_id MUST be a SINGLE PATH SEGMENT (declared at
  ; ENTITY-CORE-PROTOCOL §3.11) — it is interpolated as a path segment of the
  ; EXTENSION-CONTINUATION §3.10.1 marker path
  ; `system/runtime/chain-errors/lost/{chain_id}/{step_index}/{reason}/{marker_hash}`,
  ; so a value containing "/" forks that path into extra levels and the scheme
  ; stops being walkable at a fixed depth. The value is otherwise OPAQUE (§3.11
  ; gives it no format; nothing parses one) — the label exists for a human
  ; reading a marker path.
  chain_id = "network-maintain-" + session_id

  ; 1. Connect if needed
  current_status = ctx.entity_tree.get("system/peer/status/" + peer_id)
  if current_status is null or current_status.data.status != "connected":
    connect_result = ctx.execute("system/protocol/connect", "hello", {
      address: params.address or resolve_address(peer_id)
    })
    if connect_result.status != 200:
      return error(502, "connection_failed")

  ; 2. Create reconnection continuation (if reconnect enabled)
  if params.reconnect != false:
    backoff = params.backoff or {min_ms: 1000, max_ms: 60000, strategy: "exponential"}

    ; Continuation that triggers reconnection on disconnect notification
    reconnect_continuation = {
      type: "system/continuation"
      data: {
        target:               "system/protocol/connect"
        operation:            "hello"
        resource:             {targets: ["system/protocol/connect"]}
        params:               {address: params.address or resolve_address(peer_id)}
        result_field:         null        ; trigger mode
        remaining_executions: null        ; standing
        on_error: {
          uri: "system/inbox/network/" + peer_id + "/on-reconnect-backoff"
        }
        dispatch_capability:  ctx.handler_grant.content_hash
      }
    }
    ctx.entity_tree.put(
      "system/inbox/network/" + peer_id + "/on-disconnect",
      reconnect_continuation
    )

    ; Backoff continuation for retry — STANDING, and it handles its own failure
    backoff_continuation = {
      type: "system/continuation"
      data: {
        target:               "system/network"
        operation:            "maintain-peer"    ; retry the whole flow
        resource:             {targets: ["system/network"]}
        params:               params             ; same config
        result_field:         null
        remaining_executions: null               ; STANDING — see below [MUST]
        on_error:             {                  ; the loop re-arms itself [MUST]
          target:    "system/network"
          operation: "maintain-peer"
          params:    params
        }
        dispatch_capability:  ctx.handler_grant.content_hash
      }
    }
    ctx.entity_tree.put(
      "system/network/peers/" + peer_id + "/on-reconnect-backoff",
      backoff_continuation
    )

  ; 3. Create subscription restoration continuation (if resubscribe enabled)
  if params.resubscribe != false:
    resubscribe_continuation = {
      type: "system/continuation"
      data: {
        target:               "system/network"
        operation:            "restore-subscriptions"    ; internal operation
        resource:             {targets: ["system/network"]}
        params:               {peer_id: peer_id}
        result_field:         null
        remaining_executions: null
        dispatch_capability:  ctx.handler_grant.content_hash
      }
    }
    ctx.entity_tree.put(
      "system/inbox/network/" + peer_id + "/on-reconnect",
      resubscribe_continuation
    )

  ; 4. Subscribe to peer status changes
  disconnect_sub = ctx.execute("system/subscription", "subscribe", {
    resource: {targets: ["system/peer/status/" + peer_id]},
    params: {
      events: ["updated"],
      deliver_to: {uri: "system/inbox/network/" + peer_id + "/on-disconnect"},
      deliver_token: create_deliver_token("system/inbox/network/" + peer_id + "/on-disconnect")
    }
  })

  ; 5. Start keepalive
  start_keepalive(peer_id, params.keepalive or defaults)

  return {
    status: 200,
    result: {
      peer_id:       peer_id,
      session_id:    session_id,
      subscriptions: [disconnect_sub.subscription_id],
      chain_id:      chain_id
    }
  }
```

#### 4.1.1 The backoff continuation is STANDING, lives in the managed namespace, and handles its own failure `[MUST]`

**The backoff continuation MUST be standing (`remaining_executions: null`).** A one-shot cannot work
here, and the reason is **ordering, not timing**: the re-install lands *inside* the dispatch, while
the advance's consume runs *after* the dispatch returns — and deletes the path the dispatch just
wrote.

> **An operation MUST NOT re-arm its own trigger through a continuation that consumes itself around
> the dispatch.** The one-shot always loses the race.

The observable cost of getting this wrong is that **an offline peer is never recovered**: the loop
manages exactly two dials against a dead peer and then stops silently — no marker, no error, a `200`.
**This generalizes beyond this operation: any *re-arm by re-dispatch* pattern MUST use a standing
continuation.**

**The continuation MUST carry an `on_error` routing back to the retry re-arm.** A reconnect failure is
neither *silent* — `system/peer/status` broadcasts it, which is the whole point of the liveness
surface — nor *unhandled*: the loop re-arms, and **that re-arm is the handler**. Binding a
silent-burn `EXTENSION-CONTINUATION` §3.10 lost-error marker for an expected retry failure is
therefore a **category error**, and it is an expensive one: a dead peer's marker tree grows by
roughly 1,440 nodes/day. With `on_error` present the marker is left doing its actual job — catching
the *exceptional* failure, where the `on_error` dispatch itself fails.

> **The `on_error` target rule, corrected.** Earlier guidance said to route `on_error` to a
> `chain-errors` sink and **never** to `system/inbox/*`. That is too broad, and it left this delta
> with no legal target. **The rule is: inbox-routing advances the continuation bound there — use it
> when the error is *meant* to drive the next step (a retry); use a `chain-errors` sink for *passive
> observation*.** The hazard is unintended advancement, not inbox-routing itself.

**The install path is `system/network/peers/{peer_id}/on-reconnect-backoff`** — the §11 managed
namespace, not `system/inbox/*`. **The continuation graph is handler-managed state, and it belongs in
the namespace the handler's own grant authorizes** — `system/inbox/*` is a delivery surface, and
binding management state there conflates *what advances this* with *where messages arrive*.

### 4.2 Release Peer

Tears down the continuation graph and optionally closes the connection.

```
handle_release_peer(ctx, params):
  peer_id = params.peer_id
  reason = params.reason or "shutdown"

  cleaned_up = []

  ; Stop keepalive
  stop_keepalive(peer_id)

  ; Remove lifecycle continuations
  for path in list_paths("system/inbox/network/" + peer_id + "/"):
    ctx.entity_tree.put(path, null)
    cleaned_up.append(path)

  ; Remove lifecycle subscriptions
  for sub in find_subscriptions_by_prefix("system/peer/status/" + peer_id):
    ctx.execute("system/subscription", "unsubscribe", {
      params: {subscription_id: sub.subscription_id}
    })

  ; Drain or discard pending deliveries based on reason
  if reason == "shutdown":
    discard_pending(peer_id)
  else:
    ; Leave pending for potential future reconnection
    pass

  ; Close connection if reason is shutdown
  if reason == "shutdown":
    close_connection(peer_id, reason)
    ctx.entity_tree.put("system/peer/status/" + peer_id, {
      type: "system/peer/status",
      data: {peer_id: peer_id, status: "disconnected"}
    })

  return {
    status: 200,
    result: {peer_id: peer_id, cleaned_up: cleaned_up}
  }
```

### 4.3 Status

Returns a summary of all maintained peer relationships.

```
handle_status(ctx):
  peers = []
  for peer_path in list_paths("system/peer/status/"):
    peer_status = ctx.entity_tree.get(peer_path)
    peer_id = peer_status.data.peer_id

    pending = count_paths("system/outbound/" + peer_id + "/")
    subs = count_subscriptions_for_peer(peer_id)

    peers.append({
      peer_id:       peer_id,
      session_id:    lookup_session_id(peer_id),
      status:        peer_status.data.status,
      pending_count: pending,
      subscriptions: subs
    })

  total_pending = sum(p.pending_count for p in peers)

  return {
    status: 200,
    result: {maintained_peers: peers, pending_count: total_pending}
  }
```

### 4.4 Close

Initiates a graceful connection close with a reason code.

```
handle_close(ctx, params):
  peer_id = params.peer_id
  reason = params.reason

  ; Send close notification to remote peer (best-effort)
  ctx.execute_remote(peer_id, "system/protocol/connect", "close", {
    reason: reason
  })

  ; Update local state
  ctx.entity_tree.put("system/connection/" + peer_id, {
    type: "system/connection",
    data: {
      peer_id:        peer_id,
      transport:      "",
      address:        "",
      status:         "closed",
      established_at: 0
    }
  })

  ; Subscription preservation depends on reason
  if reason == "shutdown":
    ; Remote peer should delete subscriptions from us
    pass
  elif reason == "idle" or reason == "migration":
    ; Subscriptions preserved — connection may resume
    pass

  return {status: 200, result: null}
```

---

## 5. Keepalive

Application-level keepalive detects connection liveness independent of transport-level mechanisms.

### 5.1 Mechanism

The keepalive is an EXECUTE/EXECUTE_RESPONSE exchange on the connection handler:

```
EXECUTE system/protocol/connect  operation: "ping"
  params: {
    type: "system/network/ping"
    data: {timestamp: 1709740800000, sequence: 42}
  }

EXECUTE_RESPONSE
  result: {
    type: "system/network/pong"
    data: {timestamp: 1709740800000, sequence: 42, server_time: 1709740800050}
  }
```

*Implementation note:* WebSocket (RFC 6455 §5.5.2-3) has its own ping/pong mechanism at the transport level. Some WebSocket libraries send transport pings automatically. Transport-level pings detect TCP connectivity; application-level pings (this section) detect protocol-level liveness — a peer may be TCP-alive but protocol-unresponsive (e.g., handler deadlock, event loop stall). Implementations MUST NOT disable application-level keepalive because WebSocket transport pings are active. The two mechanisms are complementary.

### 5.2 Ping Type

```
system/network/ping := {
  fields: {
    timestamp: {type_ref: "primitive/uint"}
               ; Sender's clock, ms since epoch
    sequence:  {type_ref: "primitive/uint"}
               ; Monotonic sequence number
  }
}
```

### 5.3 Pong Type

```
system/network/pong := {
  fields: {
    timestamp:   {type_ref: "primitive/uint"}
                 ; Echoed from ping
    sequence:    {type_ref: "primitive/uint"}
                 ; Echoed from ping
    server_time: {type_ref: "primitive/uint"}
                 ; Responder's clock, ms since epoch
  }
}
```

### 5.4 Failure Detection

```
keepalive_loop(peer_id, config):
  interval_ms = config.interval_ms or 30000
  timeout_ms  = config.timeout_ms or 10000
  max_missed  = config.max_missed or 3
  missed = 0
  sequence = 0

  loop:
    sleep(interval_ms)

    ; The connection this loop was watching may have been evicted by the
    ; Amendment 12 §A1 transport-error seam. Losing the connection does NOT
    ; end the failure episode — the escalation is still owed. See §5.4a.
    if not bound(peer_id):
      return escalate_after_grace(peer_id, timeout_ms)

    ; Skip ping during active message exchange
    if recent_activity(peer_id, interval_ms):
      missed = 0
      continue

    sequence += 1
    result = execute_with_timeout(peer_id, "system/protocol/connect", "ping", {
      timestamp: now(), sequence: sequence
    }, timeout_ms)

    if result is timeout or result is error:
      missed += 1
      if missed >= max_missed:
        ; Connection failed
        update_peer_status(peer_id, "suspect", reason: "keepalive-miss")
        return escalate_after_grace(peer_id, timeout_ms)
    else:
      missed = 0
      ; Implementation-INTERNAL bookkeeping, NOT a tree write (§5.4.1).
      ; It already exists for adaptive suppression below.
      note_last_seen_internal(peer_id, now())


; Owed by the failure episode, not by the connection (§5.4a). Reachable from
; BOTH entry points: an idle keepalive miss, and an §A1 transport-error
; demotion whose eviction ends the loop before any ping is ever sent.
escalate_after_grace(peer_id, timeout_ms):
  sleep(timeout_ms)                          ; the grace period
  if reconnected(peer_id):
    return                                   ; recovered — nothing owed
  if status_of(peer_id) != "suspect":
    return                                   ; no failure episode — see §5.4a scope pin
  update_peer_status(peer_id, "disconnected",
                     reason: reason_of(peer_id))   ; the episode's originating reason
  ; This tree write fires the disconnect subscription
  ; → continuation handles reconnection (§4.1)
```

**Adaptive pinging.** When the connection is actively exchanging messages, keepalive pings are suppressed. Any successful message exchange resets the missed counter. Keepalive only fires during idle periods.

#### 5.4.1 `system/peer/status` is transition-written; per-tick freshness is implementation-internal `[MUST]`

**`system/peer/status/{peer}` MUST be written only on a transition** — a change of `status` or of
`reason`. A successful keepalive tick MUST NOT produce a tree write.

**This section's own rationale convicts the alternative.** `system/peer/status` is *the subscribed
lifecycle signal*, so a per-tick write is write amplification on exactly the entity every lifecycle
consumer is subscribed to: **two non-transition events per minute per peer per side**, delivered to
every subscriber — each of whom must then diff for a transition that did not happen — and roughly
**2,880 superseded status entities per peer per day** accreting in the content store, which nothing
collects. §6.6 rejected a `last_active` field on the session entity *for this precise reason*, and
then §5.4 put the same amplification on a more heavily subscribed entity.

- **`last_seen` stays OPTIONAL** (`ENTITY-CORE-PROTOCOL` §3.13). At a transition write its meaning is
  *"last heard as of this transition"* — on a demotion write, that is the demotion's own evidence.
- **Superseded operational-state entities are GC-eligible.** §3.13 self-description is explicitly
  non-durable; the collection mechanism is implementation-defined.
- **Consumers already have a freshness contract and it is not a heartbeat**: `status == connected`
  plus the §5.4/§A3 demotion envelope (`interval_ms × max_missed + timeout_ms`). **A cadence-visible
  heartbeat surface is deliberately deferred** until a consumer demonstrates it needs one — adding it
  speculatively re-introduces the amplification this rule exists to remove.

### 5.4a The escalation is owed by the failure episode, not by the connection `[MUST]` `[RULED 2026-08-12]`

**The rule.** A peer demoted to `suspect` that has not reconnected when the grace period expires MUST be written `disconnected`, **whether the episode began at an idle keepalive miss (§5.4) or at the transport seam (Amendment 12 §A1)**. The §A1 eviction MUST NOT cancel the pending §5.4 escalation. An implementation MUST NOT scope the escalation's lifetime to the connection binding, the pooled socket, or the keepalive loop's own liveness.

**Why this needed saying.** §A1 owns the eviction. §5.4 owns the escalation. **Nothing owned the composition** — and §5.4's own reference pseudocode, above, previously put the escalation *inside* the ping loop, so the natural reading (the loop watches a connection; no connection, no loop) destroyed the escalator at exactly the moment the escalation became owed. That is a faithful implementation of what this section used to say, not a careless one. It is the same defect class as §8.3's containment pseudocode, which likewise prescribed the bug it was meant to prevent: **when the pseudocode and the prose disagree, implementations follow the pseudocode.** The pseudocode is therefore corrected above, not merely annotated.

**What it cost.** In the peer that found it, the demoted peer stayed `suspect` forever: the disconnect subscription never fired, so **§4.1 reconnect never triggered**, and the §A3 consumer latency contract — *"a consumer of any NETWORK-conformant peer MAY rely on an idle-dead connection demoting within the keepalive envelope"* — was silently unmet on **every transport-error-first path**. Since a transport error is how a dead peer is usually noticed first, that is the common case, not the exotic one. Left unpinned, the §A3 contract holds or fails **by race**, which is worse than either answer. *(Observed: `entity-core-go`, source-read at `b55101f`, 2026-08-12; deterministic in-process reproduction, fixed there in the same commit.)*

**Scope pin (the guard is load-bearing).** The escalation is owed only where a **failure episode** is open — i.e. the peer's status is still `suspect`. The §10.2 dispatch-fallback path and the RELAY terminal-hop forward **evict without demoting** per §A1's seam scope; those bindings are still `connected`, so a teardown on those paths MUST NOT manufacture a demotion. An implementation that escalates on any unbound peer rather than on any `suspect` peer converts this rule into a new defect.

**Ordering (the second race, worth naming).** The grace `sleep` MUST precede the status read. Eviction and the `suspect` write are not necessarily one atomic event — an implementation that reads status at eviction time can observe `connected` and wrongly conclude nothing is owed.

**`reason` on the escalation — RULED: preserve the episode's originating reason `[MUST]`.** The escalating write carries the `reason` written by whichever demotion **opened** the episode: `keepalive-miss` when it began at an idle miss, `transport-error` when it began at the §A1 seam. It is not re-stamped to `keepalive-miss` on the transport path.

Three reasons, in order of weight. (a) **`keepalive-miss` on that path asserts an event that provably did not happen** — the connection was gone before the loop could send a ping, so a consumer reading it looks for missed pings that do not exist. (b) `reason` answers *why this peer left* (§A2), not *which timer fired*; the transport error is the cause, and the grace expiry is only its confirmation. (c) Re-stamping **destroys the one signal that distinguishes the previously-broken path** — preserving it makes a transport-first episode legible in the tree, where it is exactly the case no suite visited. The §A2 recovery mapping is `backoff-reconnect` for both values, so nothing downstream forks on this and the existing idle-path vector's `reason` assertion stays green unchanged.

*Pinned as a MUST rather than left to converge because `reason` is written to a subscribable tree entity and read across the peer boundary: two conformant implementations writing different values for the same event is a cross-impl-observable divergence, which is the case `AGENTS-STANDARD` says to lean MUST on. `reason` remains OPTIONAL to emit (§A2); this rule governs its value when emitted.*

**Conformance — `NET-LIVENESS-ESCALATE-AFTER-EVICTION-1` (new).** Establish → drop the remote → one dispatch (assert `suspect` + `reason: transport-error`) → **stay idle** → within the keepalive envelope, assert `disconnected` with `reason: transport-error` still carried.

**Conformance — `NET-LIVENESS-NO-ESCALATION-WITHOUT-EPISODE-1` (new, the §2.4a negative half).** A §10.2 fallback or RELAY terminal-hop eviction on a `connected` peer MUST NOT produce a `disconnected` write. **Both halves are required**: the positive vector alone is satisfied by an implementation that escalates every unbound peer, which is the scope-pin defect above.

**How the negative half is satisfied `[corrected 2026-08-12]`.** As first written this section required both halves as cross-impl vectors without checking that the second was constructible. **It is not, for a peer without RELAY.** The state it describes — *unbound yet still `connected`* — is reachable only from the two paths §A1's scope deliberately excludes from demoting, and both need a store-and-forward deployment (in the peer that raised this, the §10.2 seam is nil-gated on the fallback hook being installed at all). A conformance client cannot construct it. The satisfaction mode is therefore stated here rather than left to be discovered:

- **Where an evict-without-demote path is installed** (RELAY deployed, or any equivalent seam), the negative half **MUST be driven at the wire** like any other vector.
- **Otherwise** it is satisfied **in-process**, and the implementation **MUST record a declared exclusion** naming the mutation it was verified against — removing the `suspect` guard MUST fail the test. Per `GUIDE-CONFORMANCE` §5.2b, the gap is then *stated* rather than *absent*; an in-process result reported as a cross-impl vector pass is a false conformance claim.

> **The obvious wire proxy is rejected, and the reasoning is the point.** Substituting a live idle counterpart makes the case reachable — but a live counterpart stays **bound**, so the proxy exercises a timer-driven escalation and never the scope-pin defect it exists to catch. **It would read as coverage while missing the case**, which is strictly worse than a declared exclusion: §5.2b's whole failure mode is a scoreboard reading *covered* over an unreached surface. A proxy that cannot fail the way the real case fails does not stand in for it.

*(Authoring lesson, recorded because it is ours: this section invoked the `GUIDE-EXTENSION-DEVELOPMENT` meta-rule — a claim about what a peer writes is not validated until a cross-impl vector exercises it — and then pinned a vector without checking such a vector could exist. **Before a vector is pinned MUST, the state it requires MUST be shown constructible by a conformance client**; where it is not, the rule states its satisfaction mode at the point of the MUST. Now generalized at `GUIDE-CONFORMANCE` §5.2b.)*

> **Why these vectors did not exist.** The suite had `liveness_suspect_on_transport_error` (terminates at `suspect`) and `liveness_disconnected_on_keepalive_miss` (starts from a **fresh** counterpart, specifically so the first probe's eviction cannot interfere). **The composition of the two is a state no check occupied** — and it is the state a real consumer lands in constantly. This is the second gap of this exact shape in two cycles, after the §5.5a self-delegated-child granter frame: **a reachable state that every implementation agrees on by construction, that no vector visits, so conformance-green says nothing about it.** Both were found by an implementation, not by prose review. Per the `GUIDE-EXTENSION-DEVELOPMENT` meta-rule, a claim about what a peer writes is not validated until a cross-impl vector exercises it.

---

## 6. Session Management

### 6.1 Session vs Connection

A **connection** is a transport-level construct: a TCP socket, a QUIC stream, a Bluetooth channel. It has remote address, buffer state, and protocol-level sequence numbers.

A **session** is a protocol-level relationship: an authenticated peer identity, held capability tokens, active subscriptions, and pending deliveries. Sessions are identified by `peer_id` and persist in the entity tree. The session's authentication state — the capability this peer holds to dispatch to the remote, and the handshake cap it minted in return — is recorded at **`system/peer/session/{remote_peer_id}`** (§6.6, Amendment 8). That entity, not per-connection memory, is the durable answer to "do I already hold a valid capability to talk to this peer, or must I re-handshake?"

Sessions outlive connections. Sessions are identified by `peer_id`, not by transport or address. On reconnection, a peer MAY use a different transport than the original connection (e.g., reconnecting via WebSocket after a TCP connection drop). Transport selection uses the remote peer's `system/transport/*` entities. The `system/connection` entity (v7.13 §3.13) records the current transport for the active connection.

When a connection drops:

- The connection entity status updates to `"closed"` (v7.13 §3.13)
- The peer status updates to `"disconnected"`
- Subscriptions remain in the tree (they're entities, not connection state)
- Pending deliveries accumulate at `system/outbound/{peer_id}/`
- Capability tokens remain valid (subject to TTL) — the `system/peer/session/{peer_id}` entity (§6.6) is **NOT** deleted on disconnect; that persistence is the point

When a new connection is established with the same `peer_id`:

- The session resumes implicitly — same peer identity means same session
- Subscription restoration re-activates notification delivery (§7)
- Pending deliveries drain to the new connection (§8)

### 6.2 Session Resumption

Session resumption is implicit, based on `peer_id` matching during handshake:

```
on_connection_established(peer_id, connection):
  ; Check for existing session state
  existing_status = entity_tree.get("system/peer/status/" + peer_id)

  if existing_status is not null and existing_status.data.status == "disconnected":
    ; Session resumption — restore state
    entity_tree.put("system/peer/status/" + peer_id, {
      type: "system/peer/status",
      data: {
        peer_id:      peer_id,
        status:       "connected",
        connected_at: now(),
        last_seen:    now(),
        connection:   "system/connection/" + peer_id
      }
    })
    ; This tree write fires the reconnect subscription
    ; → continuation handles subscription restoration + pending drain
  else:
    ; New session
    entity_tree.put("system/peer/status/" + peer_id, {
      type: "system/peer/status",
      data: {
        peer_id:      peer_id,
        status:       "connected",
        connected_at: now(),
        last_seen:    now(),
        connection:   "system/connection/" + peer_id
      }
    })
```

*Implementation note:* Reconnection continuations SHOULD resolve the remote peer's current transport and address at reconnection time rather than using a cached address from the original connection. The remote peer's transport entities at `system/peer/transport/{peer_id}/*` reflect currently available transports and may have changed during the disconnection period.

### 6.3 Self-Authenticating Messages

EXECUTE envelopes with valid capability chains are self-authenticating (ENTITY-CORE-PROTOCOL.md §4.6). On reconnection, a peer MAY send an EXECUTE with a previously-held capability token instead of performing a full handshake. If the receiver validates the signature and capability chain, the connection is authenticated:

```
Zero-RTT reconnection:
1. Open connection
2. Send EXECUTE with previously-held capability token
3. Receiver validates: signature valid, capability chain roots at local_peer_id
4. If valid: connection authenticated, process request
5. If invalid: respond with 403, sender falls back to full handshake
```

The "previously-held capability token" in step 2 is read from `system/peer/session/{remote_peer_id}.held_capability` (§6.6) — the cap the remote granted this peer at the original handshake. Because it lives in the tree, it survives the connection drop and (for a persisted identity) a process restart, so zero-RTT reconnection is available without re-deriving the cap from connection state.

This is an optimization, not a requirement. Implementations MAY always perform full handshake on reconnection.

### 6.4 Composition with Identity Rotation

*(Informative.)*

Sessions in this extension are keyed by `peer_id` (§6.1). The identity-rotation surfaces defined in EXTENSION-IDENTITY interact with this layer as follows:

- **Op rotation** (EXTENSION-IDENTITY §5.6, `rotate_operator`): invisible to NETWORK. The deployment's runtime peers retain their `peer_id`s; sessions are unaffected.
- **Public_X rotation** (handoff or compromise-recovery; identity-recognition territory): invisible to NETWORK. Sessions are unaffected.
- **Runtime peer keypair rotation** (rare; rotation typically retires a peer rather than re-keying): produces a new `peer_id`. The rotated peer is structurally a new peer to NETWORK; the old session terminates and a new session is required, with its own `maintain-peer` continuation.
- **Runtime peer retirement** (typical flow; EXTENSION-IDENTITY §5.9, `revoke_peer`): terminates that peer's sessions. Contacts opening new sessions to the deployment connect to a different runtime peer per the registry's resolved endpoints; NETWORK requires no special handling beyond normal session establishment to the new endpoint.

NETWORK does not subscribe to identity rotation events; the layering is one-directional. Identity manages rotation and attestation; NETWORK observes the resulting peer_id population through normal session lifecycle.

### 6.5 Transport Profiles

*Amendment 1.* §6.1, §6.2 reference `system/peer/transport/{peer_id}/*` as the namespace for "what transports does this peer speak." This section defines the entity-type and lands two initial profiles (`quic` and `http-poll`) that v1 needs. Sibling profiles for SMTP/IMAP, Git-over-HTTP, Nix substituter, Tor onion services, etc., are expected from forthcoming bridge specs.

A transport profile entity advertises one way that this peer can be reached. The dispatcher in §10 consults the set during peer dispatch — active connection → held-capability authentication (§6.6) → transport-profile resolution by reachability class (`(priority asc, profile-id lex)`) → queue (§8) → (post-corridor) REGISTRY lookup.

**A transport MAY be store-and-forward.** Nothing in this spec assumes a transport binding requires simultaneous liveness of both peers. Sessions outlive connections (§6.1); pending delivery (§8) handles async traversal; signed mutable pointers and signature verification cover the static / asynchronous transport family. The `freshness` field on each profile names the model.

#### 6.5.1 Profile Entity-Type Definition

```
type: "system/peer/transport/<profile-name>"     ; profile-name is the transport family
data: {
  peer_id:            system/peer-id, ; the peer this profile is for (Base58 id per V7 §2.8 — matches the {peer_id} path segment below, NOT a content hash)
  transport_type:     string,         ; matches the profile-name suffix
  endpoint:           transport-specific (object),  ; e.g. {host, port, alpn} for quic; {base_url} for http-poll
  supported_ops:      string[],       ; closed enum {EXECUTE, TREE_GET, CONTENT_GET, MANIFEST_GET} + reserved SUBSCRIBE (D-13; PROPOSAL-EXTENSION-NETWORK-TRANSPORT-FAMILY §7). Descriptive (what the wire physically carries), never a grant.
  freshness:          "live" | "async" | "static-immutable+signed-pointer",
  nonce_required:     bool,           ; true for live handshake; false for store-and-forward
  cap_flow:           "egress" | "ingress" | "both",   ; with respect to the peer this profile describes
  poll_interval_ms?:  int,            ; for async/static; informative cadence for pollers
  signed_pointer?:    Path,           ; for static; where to fetch the publisher's signed mutable root pointer
  priority?:          uint,           ; lower = preferred; default 100; reserved id `primary` unset → 0 (Q1, §6.5.1a)
  advertised_at?:     time            ; OPTIONAL (Amendment 8, Q6); when this profile was last asserted by the peer
}
```

Profile entities live at `system/peer/transport/{peer_id}/{profile-id}` where `{profile-id}` is a per-peer-unique identifier (e.g. `primary`, `cdn-mirror`, `backup-relay`). A peer MAY have multiple profiles of the same type (e.g. two HTTP mirror URLs for redundancy).

The MUST list:

- `peer_id`, `transport_type`, `endpoint`, `supported_ops`, `freshness`, `nonce_required`, `cap_flow` — all required.
- `poll_interval_ms`, `signed_pointer` — required when `freshness` is `async` or `static-immutable+signed-pointer`; absent when `freshness` is `live`.
- `advertised_at` — **OPTIONAL** (Amendment 8, Q6). It is advisory-only (§6.5.1a D3); requiring an advisory field was ceremony and a cross-impl footgun. `omitempty`; consumers treat absence as "no advisory info" and proceed.
- `priority` — **OPTIONAL** (Amendment 8, Q1); `uint`, lower = preferred, default 100. See §6.5.1a selection.

#### 6.5.1a Selection, self-publication, and field rules

*Amendment 2 (cohort Chunk-C cross-impl rollup).*

**Profile selection (D1).** When a peer publishes more than one profile of the wanted `transport_type`, a consumer builds a **deterministic ordered candidate list** and attempts each in order until one connects. The list is sorted by **`(priority asc, profile-id lex)`** (Amendment 8, Q1):

- **`priority`** — lower value = more preferred (DNS-SRV semantics). Default `100` when unset. The reserved profile-id **`primary`**, when its `priority` is unset, has an implicit `priority` of `0` (preserving the prior "primary first" convention verbatim). An explicit `priority` is always authoritative.
- **`profile-id`** — the **final path segment** of `system/peer/transport/{peer_id}/{profile-id}` (Amendment 8, Q2), independent of whether the location-index layer presents relative or absolute paths. Selection operates on the segment, not on a prefixed form. Used as the lexicographic tie-break among equal-priority profiles, stable across every location-index backend (SQLite collation, in-memory map, dict).

**`advertised_at` MUST NOT be used as a selection key** (it is wall-clock and skew-prone — D3 below). Redundant profiles (mirrors) are ordered fallback via the sort, not a single deterministic pick. (Weight-based load-balance among equal-priority mirrors — SRV `weight` — is NOT v1; equal-priority profiles stay an ordered, lex-tie-broken fallback list. Noted as future.)

Back-compatibility: existing single-`primary` deployments and lex-fallback deployments behave identically — `primary`-unset → 0 sorts first as before; all-unset → all 100 → pure lex order as before. The earlier "name the profile so it lex-sorts after TCP" hack (G1) is no longer needed: set `priority`, name freely.

**Self-publication (D1).** A peer **SHOULD** publish a profile entity for each transport it accepts on, at `system/peer/transport/{peer_id}/{profile-id}`. RECOMMENDED, not REQUIRED — a peer may be reachable only via REGISTRY / manifest / out-of-band (e.g. a browser peer that cannot self-host its tree). **Consumers MUST NOT assume the self-published path exists**; absence falls through to other discovery.

**`transport_type` consistency (D5).** The `transport_type` field MUST match the entity-type suffix (`system/peer/transport/<X>` ⇒ `transport_type: "<X>"`). Decoders MUST reject a mismatch (fail closed). The field is retained (self-describing for tooling that holds the data without the tree path), but the entity-type is authoritative.

**`advertised_at` clock (D3).** Wall-clock epoch ms, **informational only**, judged against the consumer's local clock for advisory staleness. Not a selection key; no logical clock in v1 (deliberate — staleness is advisory, not a correctness input).

**Live-transport `endpoint` shape (D4).** Every **live** transport profile (`tcp`, `websocket`, `http`) carries `endpoint: { url: "<scheme>://..." }` — a single scheme-prefixed `url` field (`tcp://host:port`, `wss://host/path`, `https://host/path`). Per-profile variant shapes (`{base_url, route}`, `{scheme, host, port, path}`) are NOT permitted. `http-poll` is the exception: it advertises lookup *prefixes* (`tree_url_prefix`/`content_url_prefix`), not a single dial endpoint.

#### 6.5.1b Duplex, reachability, and connection reuse (informative)

*Amendment 7.* This note pins cross-impl convergence intent for peer-initiated outbound dispatch. It introduces no new wire format or required behavior; it makes explicit what §6.1 (sessions outlive connections), §6.3 (held-cap reconnection), and the §10 dispatch fallthrough already imply.

**Peer-initiated dispatch is general.** Subscription delivery is one instance of "peer B sends an EXECUTE to peer A": continuation chain-advancement, async inbox delivery, pending-delivery drain, and a long-running handler on B choosing to EXECUTE on A all resolve the same way. The source consults the **target's** transport profiles (§6.5.1a D1 order) and either reuses an active connection or dials one. Because each direction consults the *target's* profiles independently, the two directions need not share a transport ("B→A over `http`, A→B over `tcp`" is ordinary).

**Duplex taxonomy.** Whether one connection serves both directions is a property of the transport:

| Transport | One connection | Server→client push on it | Bidirectional dispatch needs |
|---|---|---|---|
| `tcp` | full-duplex | yes | ONE connection, reused both ways |
| `websocket` | full-duplex | yes — *incl. to a non-listening client (browser)* | ONE connection, reused both ways |
| `webrtc` | full-duplex | yes — *NAT-traversing* | ONE connection, reused both ways *(negotiated, not dialed — §6.5.2d)* |
| `http` | half-duplex | no | each side runs a listener (two connections), OR poll-fallback |
| `http-poll` | one-way fetch | no (no session) | consumer polls; publisher never dials |

**Inbound-connection reuse is an impl-detail optimization** *(Amendment 8 — was a SHOULD in Amendment 7; dropped to impl-detail per proposal R2/§7.1 #4).* For full-duplex transports (`tcp`/`websocket`/`webrtc`) an established connection is bidirectional whether dialed or accepted, so an impl MAY reuse an accepted inbound connection for outbound dispatch rather than dialing a second one. Post-R6 this carries no authentication content (the outbound EXECUTE is signed with the `held_capability` from §6.6, independent of which socket carries the bytes), so it is left to the implementation — not a normative SHOULD and not conformance-gated. For `http` the inbound (server) and outbound (client) sides are necessarily distinct.

**Half-duplex targets that cannot listen** (a CLI/browser client with no listener and no duplex socket) cannot be dialed at all. Delivery to them requires the **poll-fallback**: the source queues to a source-hosted path the target drains via `TREE_GET`. This inverts the standard subscriber-hosted-inbox model and is **Phase-2** — not specified or conformance-gated in v1. See the exploration referenced in Amendment 7.

**Capability lifecycle across connections is per-peer-session** (Amendment 8, R6): the cap is held by the `system/peer/session/{peer_id}` entity (§6.6), not by the connection, so it survives drops and a second connection (any transport) reuses it. This resolves the cap-direction question — the held cap is the dispatcher's auth, independent of which socket carries the bytes.

#### 6.5.2 Profile: `system/peer/transport/quic` (aspirational)

An **optional / future** live transport — **no reference impl ships QUIC yet**. The real default live transport is `tcp` (§6.5.2a). Profile shape retained for when an impl adds QUIC.

```
type: "system/peer/transport/quic"
data: {
  peer_id:        <peer_id>,
  transport_type: "quic",
  endpoint: {
    host:  "peer.example.com",
    port:  4433,
    alpn:  "entity-core/1"
  },
  supported_ops:  ["EXECUTE"],          ; full duplex; all V7 message types
  freshness:      "live",
  nonce_required: true,                  ; V7 §4.6 handshake nonce required
  cap_flow:       "both",
  priority?:      100,                   ; OPTIONAL; default 100 (Amendment 8, Q1)
  advertised_at?: <time>                 ; OPTIONAL (Amendment 8, Q6)
}
```

Test-impl correspondence: a conforming impl SHOULD publish a `quic` profile for its peer-id **if and when** it accepts QUIC connections.

#### 6.5.2a Profile: `system/peer/transport/tcp` (the real default live transport)

TCP is the **actual default** live transport across all three reference impls (Go, Rust, Py). Builds on the existing `"tcp"` transport-value + framing (ENTITY-CORE-PROTOCOL.md §3.13/§1.6); this profile makes it discoverable. Impls SHOULD publish a `tcp` profile for their peer-id.

```
type: "system/peer/transport/tcp"
data: {
  peer_id:        <peer_id>,
  transport_type: "tcp",
  endpoint:       { url: "tcp://host:port" },
  supported_ops:  ["EXECUTE"],           ; full duplex; carries server-push (subscribe)
  freshness:      "live",
  nonce_required: true,
  cap_flow:       "both",
  priority?:      100,                   ; OPTIONAL; default 100 (Amendment 8, Q1)
  advertised_at?: <time>                 ; OPTIONAL (Amendment 8, Q6)
}
```

#### 6.5.2b Profile: `system/peer/transport/websocket`

The **browser-capable full-duplex** live transport — the only browser transport that carries server→client push. Real in Rust (native + WASM); built on the existing `"websocket"` transport-value + framing (ENTITY-CORE-PROTOCOL.md §3.13/§1.6, v7.13).

```
type: "system/peer/transport/websocket"
data: {
  peer_id:        <peer_id>,
  transport_type: "websocket",
  endpoint:       { url: "wss://host/ws" },
  supported_ops:  ["EXECUTE"],           ; full duplex; carries server-push
  freshness:      "live",
  nonce_required: true,
  cap_flow:       "both",
  priority?:      100,                   ; OPTIONAL; default 100 (Amendment 8, Q1)
  advertised_at?: <time>                 ; OPTIONAL (Amendment 8, Q6)
}
```

#### 6.5.2c Profile: `system/peer/transport/http` (live — EXECUTE over POST)

The normal protocol over HTTP request/response: POST an EXECUTE envelope; the response body is the EXECUTE-RESPONSE. **POST-only** (nothing in the protocol is idempotent — no GET sub-mode). **Half-duplex per connection** — a single HTTP connection is connector-driven (client POSTs, server responds) and carries no server→client push. This does **not** block subscription / async / peer-initiated delivery: such delivery is a *fresh* EXECUTE from the source to the **target's** inbox (EXTENSION-INBOX.md §2 — the source dials the target), so what it requires is that the *target be reachable* — publish a live profile (`tcp`, `http`, or `websocket` listener) the source can dial — not that any connection be duplex. A target running an HTTP listener receives deliveries over an ordinary half-duplex source→target POST; two peers that each run an HTTP listener form a full-duplex *pair* across two independent connections. Reachability is per-peer and asymmetric: a delivery B→A consults A's profiles, A→B consults B's, so "B reaches A over HTTP while A reaches B over TCP" is the ordinary case, not a special one. `websocket` (§6.5.2b) is distinguished only in that it *additionally* pushes to a **non-listening** target (e.g. a browser page) down that target's own outbound socket. A target that can neither listen nor hold a duplex socket falls to the poll-fallback (Phase-2 — source-hosted queue the target drains via `TREE_GET`; see §6.5.1b). A **wrapper, NOT BRIDGE-HTTP** — the bytes on the wire ARE entity envelopes (Mechanism A), not foreign content. The browser linchpin (a browser can POST but has no raw socket).

**Body framing (MUST).** The HTTP request and response body is the **bare ECF-encoded envelope** (ENTITY-CORE-PROTOCOL.md §5.3). HTTP message framing — `Content-Length` or `Transfer-Encoding: chunked` — delimits it. The ENTITY-CORE-PROTOCOL.md §1.6 **TCP length prefix MUST NOT be applied**: that 4-byte prefix is *stream-transport framing* (TCP delimiting a medium with no message boundaries), and ENTITY-CORE-PROTOCOL.md §1.6 states framing is per-transport — "other transports define their own framing." HTTP is message-oriented and frames the body natively, so an inner prefix is redundant and creates two conflicting length authorities. This keeps the body a clean CBOR document that composes with `fetch`/`curl`/CDNs/proxies — the browser-and-ecosystem reachability that is the `http` profile's entire reason to exist. (Contrast `websocket` §6.5.2b, which *does* reuse the ENTITY-CORE-PROTOCOL.md §1.6 prefix per its explicit V7 v7.13 blessing — that reuse is stated, not a default, and does not extend to HTTP.) A bad body decode is the substrate-level `400` (entity-protocol errors instead travel *inside* the response envelope under a `200`).

#### 6.5.2d Profile: `system/peer/transport/webrtc` (live — negotiated data channel)

The browser's peer-to-peer transport, filling the §6.5.1b reserved slot (folded 2026-08-02). It is a durable **advertisement** that this peer accepts a WebRTC data channel — and, unlike every other live profile, it carries **no dial `endpoint`**: a WebRTC channel is never dialed, it is *negotiated* through signaling (`EXTENSION-SIGNALING.md` §6.5), so the profile declares the capability + negotiation parameters, not an address.

```
type: "system/peer/transport/webrtc"
data: {
  peer_id:        <peer_id>,
  transport_type: "webrtc",
  supported_ops:  ["EXECUTE"],           ; full duplex; carries server-push
  freshness:      "live",
  nonce_required: true,
  cap_flow:       "both",
  negotiation: {
    signaling_schema: "webrtc-sdp-ice/1",  ; the EXTENSION-SIGNALING §6.5 schema version this peer speaks
    ice_policy?:      "all",               ; OPTIONAL; "relay" forces TURN (no host-candidate leak); default "all"
    dtls_role?:       "auto"               ; OPTIONAL; "auto" = the SDP offer/answer decides
  },
  priority?:      100,                   ; OPTIONAL; default 100 (§6.5.1a, Amendment 8 Q1)
  advertised_at?: <time>                 ; OPTIONAL
  ; NO endpoint — a WebRTC channel is negotiated, not dialed (contrast §6.5.1a D4)
}
```

- **Profile vs channel `[do not conflate]`.** The published profile is a **durable advertisement** ("a direct WebRTC path to me exists"). The **established data channel** is a **session-scoped §10.3 connection** — never itself published as a durable `system/peer/transport/*` profile (per Amendment 14 the §10.3 result is an ordinary transport, session-scoped, and MUST run keepalive). The profile advertises that a punch is *possible*; the channel is its result.
- **Reachability class.** A peer publishing a `webrtc` profile is reachable as a **punch-substrate class** at the §10.3 `establish_live` seam — not a `full_duplex_listener` (nothing is listening). Resolving such a peer, the §10 dispatcher escalates to §10.3 and drives the `EXTENSION-SIGNALING.md` §6.5 negotiation instead of dialing. No new dispatch branch and **no new reachability class** (the Amendment-14 seam).
- **Once open, it is ordinary.** The channel carries **bare ECF envelopes** like any live transport; framing is the data channel's own message boundaries (message-oriented, like `http` — it does **NOT** apply the ENTITY-CORE-PROTOCOL.md §1.6 TCP length prefix). The §10 dispatcher, held-capability model (§6.6), and session state (`system/peer/session/*`) apply unchanged.
- **Who publishes it.** A **native** peer MAY publish a `webrtc` profile to be punchable by browser peers (§14 / `EXTENSION-SIGNALING.md` §7.3.1); until a native WebRTC terminator exists that MAY is latent. Whether a **browser** peer publishes one — versus only ever initiating, its reachability being rendezvous/session-scoped — is open (`PROPOSAL-EXTENSION-WEBRTC-TRANSPORT` open item #1) and gates the later browser↔native-direct milestone, not the browser↔browser S5 gate.

The framing *mechanism* (Content-Length/chunked, no length-prefix) is shared by `http` and `http-poll`; the body *payload* differs by route: the `http` EXECUTE route carries a MaterializedEnvelope (`{root, included}`); the `http-poll` `CONTENT_GET` route carries a single bare-hashable entity `ECF({type, data})` per §6.5.3, the `TREE_GET` **leaf** route carries a `system/hash` pointer `ECF({type: "system/hash", data: H})` (the bound hash, two-hop — §6.5.3.1, Amendment 6; NOT the dereferenced entity), and the `TREE_GET` **listing** route carries a `system/tree/listing` wire entity `ECF({type, data, content_hash})`.

```
type: "system/peer/transport/http"
data: {
  peer_id:        <peer_id>,
  transport_type: "http",
  endpoint:       { url: "https://host/entity" },
  supported_ops:  ["EXECUTE"],           ; half-duplex per connection; delivery TO this peer needs it reachable (HTTP listener), not duplex — §6.5.2c / §6.5.1b
  freshness:      "live",
  nonce_required: true,
  cap_flow:       "both",
  priority?:      100,                   ; OPTIONAL; default 100 (Amendment 8, Q1)
  advertised_at?: <time>                 ; OPTIONAL (Amendment 8, Q6)
}
```

##### 6.5.2c.1 HTTP-substrate conventions (Amendment 4)

A live `http` peer threads V7's session identity across stateless POSTs and follows these conventions:

- **Session header `X-Entity-Session`** (MUST agree cross-impl). Server-allocates on the first POST that lacks it; returns it in the response header; client echoes it thereafter. The session ID is **server-allocated and opaque to the client** (the client never parses it) — its *format* is impl-defined and is NOT a cross-impl constant; it SHOULD carry ≥128 bits of unguessable entropy.
- **Status discipline (SHOULD).** `2xx` ⇔ a response envelope is present, *even if it carries an entity-protocol error* (errors travel inside the envelope). Non-`2xx` ⇔ no envelope: decode failure → `400`, dispatcher failure → `502`, wrong method → `405` (`Allow: POST`; `Allow: GET` on the `http-poll` routes), unknown path → `404`.
- **Default path `/entity`** (conventional, not normative — operators advertise the actual path in `endpoint.url`). **Content-Type `application/cbor`** (SHOULD set).
- **Idle-session eviction is an operator knob** (conservative default; NOT a cross-impl constant — two peers needn't share a TTL, since a client re-handshakes on eviction).
- **Hex strictness (MUST).** Content-hash hex on the `http-poll` routes is the **full wire form — format-code byte included** (§3.5; §6.5.3), never the digest-only form. **The expected length is the one the leading format byte implies** (66 chars under ECFv1-SHA-256 `00`, 98 under ECFv1-SHA-384 `01`) and MUST NOT be treated as a constant — `SPECIFICATION-FORMAT.md` §8.4.5. A request whose hex length disagrees with its own format byte → `400`; that rejects the 64-char digest-only form (no back-compat shim — no deployed legacy data; clean-break) **and** a 98-char string claiming `00`. A hash whose algorithm byte the peer cannot verify → `400` (fail-closed).

#### 6.5.3 Profile: `system/peer/transport/http-poll`

The static-HTTP transport binding for **Mechanism A** (HTTP-as-storage-transport, per `GUIDE-EXTENSION-DEVELOPMENT.md §3.7`). A publisher peer using this profile has published its tree + content to a static HTTP origin (typically a CDN-backed bucket); consumers poll and fetch the bytes **directly** — inline HTTP GET + content-hash verification, NO `EXTENSION-BRIDGE-HTTP` involvement. The bytes-on-wire ARE entity-encoded; the substrate's only role is byte storage. (BRIDGE-HTTP is Mechanism B — foreign-substrate fetch of HTML/JSON/arbitrary content to wrap as `system/bridge/http/fetched`; it is NOT on this path. See §6.5.5 for the two consumer modes and EXTENSION-BRIDGE-HTTP.md §1 (planned) for the scope boundary.)

**Three-prefix URL space (Amendment 5).** The profile carries THREE independently-configured URL prefixes (per the Amendment-5 pathing standardization):

- `tree_url_prefix` — tree-path-keyed URLs. **Every form is a concrete object key (no trailing slash):** `{tree_url_prefix}/{peer_id}/{path}{tree_leaf_suffix}` serves the entity at that path; `{tree_url_prefix}/{peer_id}/{path}{tree_listing_suffix}` serves the listing; `{tree_url_prefix}/{peer_id}{tree_listing_suffix}` the peer root; `{tree_url_prefix}/peers{tree_listing_suffix}` the all-peers (universal-tree-root) listing. See §6.5.3.1.
- `content_url_prefix` — content-hash-keyed URLs: `{content_url_prefix}/{layout-path}/{hash}` serves the bytes for that hash.
- `manifest_url_prefix` — the singular signed manifest: `{manifest_url_prefix}` (terminal; no suffix, no trailing slash). **This slot is reserved for the `system/peer/published-root` entity and nothing else (MUST).** Serving any other entity here — in particular a `system/peer/transport/*` profile — is non-conformant, however plausible it looks: the profile is *how a consumer reached this origin*, and the signed root is *what the origin is asserting*. See §6.5.4 for where a static publisher's own transport profile lives.

**Publish-side closure obligation (MUST when `signed_pointer` is advertised).** §6.5.6's Amendment-10 closure MUST is written for a **live** peer and phrased as *"MUST resolve"* — a static origin resolves nothing, it serves files that were uploaded. The obligation is the same and the form is not: a publisher advertising `signed_pointer` on an `http-poll` profile **MUST upload**, under `content_url_prefix`, the transitive hash-linked closure of `published-root.root_hash` — the trie root node, all interior nodes, and all leaf-bound content hashes — plus the `published-root` entity itself and its `system/signature` entity at the `ENTITY-CORE-PROTOCOL.md` §5.2 invariant-pointer path.

  Without it the failure mode is the one prose review does not catch and a smoke test passes: **every pointer resolves and a pinned consumer still gets nothing**, because `CONTENT_GET` 404s on an interior trie node and the walk from `root_hash` cannot complete. A publisher that does **not** advertise `signed_pointer` is a content-only mirror and carries no closure obligation (§6.5.6's carve-out, unchanged). Mechanism is impl choice — project the closure at publish time, or verify it after upload; the contract is *"MUST upload."* This is §6.5.6's obligation in the form a bucket can satisfy, and neither section is the only home: see §6.5.6 *Signed-root closure* for the serving-side statement.

These three prefixes MAY be **co-located** on one origin (demuxed by first path segment per §6.5.6 — reserved literals `content`/`manifest`/`peers`, else a parseable peer-id ⇒ tree; in co-located form `tree` has no segment word) OR be **entirely separate origins** (tree on one CDN, content on a dedup bucket, etc.). Decoupling supports multi-peer-shared-domain deployments (peer-scoped prefixes) and cross-peer content-store deduplication (hash-keyed, peer-agnostic).

```
type: "system/peer/transport/http-poll"
data: {
  peer_id:        <publisher-peer-id>,
  transport_type: "http-poll",
  endpoint: {
    tree_url_prefix:    "https://shared.example.com/peers/<peer_id>",
                                       ; URLs: {tree_url_prefix}/{tree-path}
    content_url_prefix: "https://shared.example.com/peers/<peer_id>/content",
                                       ; OR shared-dedup: "https://shared.example.com/content"
                                       ; URLs: {content_url_prefix}/{layout-path}/{hash}
    content_layout:     "flat" | "sharded-2-flat" | "sharded-2-4" | "sharded-2-2"
                                       ; pure hash-layout; NO peer-namespacing concern
                                       ;   flat:           /{hash}
                                       ;   sharded-2-flat: /{hash[0:2]}/{hash}
                                       ;   sharded-2-4:    /{hash[0:2]}/{hash[2:4]}/{hash}
                                       ;   sharded-2-2:    /{hash[0:2]}/{hash[2:4]}/{hash}  (alias for sharded-2-4)
    tree_leaf_suffix:   ".bin"         ; default ".bin"; consumers append for ENTITY (leaf) URLs
                                       ;   (operator-overridable; append-one/strip-one per §6.5.3.1)
    tree_listing_suffix: ".list"       ; default ".list"; consumers append for LISTING URLs
                                       ;   MUST differ from tree_leaf_suffix (Amendment 5)
    manifest_url_prefix: "https://shared.example.com/peers/<peer_id>/manifest"
                                       ; singular signed manifest; terminal, NO suffix, NO trailing slash
  },
  supported_ops:  ["TREE_GET","CONTENT_GET","MANIFEST_GET"],  ; D-13 closed enum; partial publisher MAY advertise any non-empty subset (content-only mirror = ["CONTENT_GET"]); distinct anchors: TREE_GET→hash-chain-from-root, CONTENT_GET→hash, MANIFEST_GET→signature. No EXECUTE; no writes; no subscribe (consumer polls).
  freshness:      "static-immutable+signed-pointer",
  nonce_required: false,                 ; static has no session; signatures self-authenticate
  cap_flow:       "egress",              ; with respect to the publisher: they push to the static store (egress); consumers fetch
  poll_interval_ms: 60000,               ; informative; consumers may poll less or more often
  signed_pointer: "system/peer/published-root",   ; TREE PATH naming what manifest_url_prefix serves.
                                                  ;   NOT a URL and NOT a suffix — see below.
  advertised_at?: <time>                 ; OPTIONAL (Amendment 8, Q6)
}
```

**`signed_pointer` is a tree path; `manifest_url_prefix` is where you fetch. A consumer MUST NOT join `signed_pointer` onto an origin `[MUST, v1.8]`.** The two answer different questions: `signed_pointer` says *what entity type the origin is asserting* (the `EXTENSION-TREE` §3.3a path, so a consumer knows a signed root exists and what to verify), and `manifest_url_prefix` says *where to GET it* — terminal, no suffix (§6.5.3.1). **The manifest's location is DISCOVERED from the profile, never derived by convention from the tree path.** A publisher is free to serve at any `manifest_url_prefix` it advertises; `{origin}/manifest` and `{origin}/{peer_id}/system/peer/published-root` are equally conformant, and only the advertised one is findable.

> **Why this needs saying `[found by cross-impl publish/consume]`.** A consumer that treats `signed_pointer` as a path fragment to append to the origin reads a **tree** path as a **transport** path. It then fails on any publisher whose `manifest_url_prefix` differs — which is every publisher that does not happen to share its convention — and the failure is at **hop 0**, so nothing downstream is exercised and the alignment that does exist stays invisible. The tree path additionally may not be a servable file at all: it names a binding in a trie, not a byte range at an origin. **A profile field that exists to be read is not a default to be assumed.**

**Multi-peer-shared-domain example (the load-bearing case).** When a single domain owner hosts content from multiple peers, each peer's profile embeds its peer-ID literally in the prefix strings:

```
For peer A:
  endpoint.tree_url_prefix:    "https://shared.example.com/peers/<A>"
  endpoint.content_url_prefix: "https://shared.example.com/peers/<A>/content"   ; OR shared: "https://shared.example.com/content"

For peer B:
  endpoint.tree_url_prefix:    "https://shared.example.com/peers/<B>"
  endpoint.content_url_prefix: "https://shared.example.com/peers/<B>/content"   ; OR shared: "https://shared.example.com/content"
```

No template-substitution at consume time — the peer_id is part of the literal prefix string the publisher author writes into their transport profile entity. Consumer uses the prefix as-is.

**Deployment scenarios** (see audit doc §2 for the full stress-test):
- S1 — single peer dedicated domain: `https://peer-a.example.com` + `/content`
- S2 — single peer on CDN with path prefix: `https://cdn.example.com/my-peer` + `/content`
- S3 — multi-peer shared domain (per-peer content): `https://shared.example.com/peers/<peer_id>` + `/content`
- S4 — multi-peer shared with deduplicated content: `https://shared.example.com/peers/<peer_id>` (tree) + `https://shared.example.com/content` (shared)
- S5 — peer publishing to multiple domains: multiple profiles per peer
- S6 — subdomain-per-peer: `https://peer-a.cdn.example.com` + `/content`

All work cleanly under the two-prefix model.

The consumer-side flow:

1. Consumer needs a binding/content from a publisher peer P.
2. Local content-store / tree miss; dispatcher (§10) falls through.
3. Dispatcher resolves `system/peer/transport/{P}/*` → finds `http-poll` profile with `tree_url_prefix` + `content_url_prefix` + `content_layout` + `signed_pointer`.
4. For hash-keyed lookups: dispatcher builds URL = `{content_url_prefix}/{layout-path}/{hash}`, performs an **inline HTTP GET** of the bytes, computes the content hash over the body, verifies it equals `{hash}` (MUST; mismatch → discard, do not ingest), then ingests via `system/content:ingest` into `target_namespace`. No `system/bridge/http:get`; no `system/capability/bridge-http-fetch` cap — the trust anchor is the content hash.
5. For tree-path lookups (less common; typically consumers walk refs from known root): dispatcher builds URL = `{tree_url_prefix}/{tree-path}{tree_leaf_suffix}` and fetches the **bound hash** — the body is a `system/hash` pointer (`ECF({type:"system/hash", data:H})`, §6.5.3.1), NOT the dereferenced entity; reads `H` from `data`; falls through to step 4 (`CONTENT_GET`) for the actual content. **Two hops** — the tree route resolves `path → hash`, the content route resolves `hash → bytes` (ENTITY-CORE-PROTOCOL.md §1.7); collapsing them would duplicate entity bytes per-path on a static CDN.
6. For mutable-binding lookups, consumer fetches the signed pointer first; verifies signature; then fetches the resolved target via step 4.

**This is Mechanism A, not BRIDGE-HTTP.** The profile names the endpoint and op-set; the consumer (dispatcher-driven in Mode A1, operator-tool in Mode A2 per §6.5.5) performs the fetch + hash-verify **inline**. There is no BRIDGE-HTTP handler and no bridge cap surface on this path — the bytes are already entity-encoded and the content hash is the sole trust anchor. BRIDGE-HTTP (Mechanism B) is a structurally-distinct surface for foreign content (HTML/JSON) and is not used here. Empirically validated by workbench-go's `entity-fetch` (stdlib `net/http` + hash-verify, no BRIDGE-HTTP import).

**Why this lands in NETWORK.** `system/peer/transport/*` is the general transport-pluggability namespace; the `http-poll` profile is one transport among many (alongside `quic`, future libp2p/etc.). Landing per-profile entities here keeps the transport slot consistent and independent of any bridge extension.

##### 6.5.3.1 Route response semantics (Amendment 4)

The `http-poll` routes (`CONTENT_GET`, `TREE_GET` entity + listing, `MANIFEST_GET`) are content-addressed fetch over HTTP — the consumer trusts the math, not the host. Validated three-way (Go/Rust/Py via `validate-peer -category serving_mode`). Per the Amendment-5 pathing standardization.

- **`CONTENT_GET` body (MUST).** A `GET` for content hash `H` returns the **bare hashable form `ECF({type, data})`** — the **2-key** CBOR map of the entity whose `content_hash` is `H` — `Content-Type: application/cbor`. This is **NOT** the 3-key wire entity carrying `content_hash` (that value hashes `{type, data}` and so cannot appear in its own preimage). The body **MUST satisfy pure-body rehash**: a consumer computes `0x00 ‖ SHA-256(body)` and accepts only if it equals `H` (Mechanism A — verifiable by a CDN/browser with no ECF decoder). Serving the 3-key entity, the inner `data`, or a chunk's raw payload is non-conformant. Raw-bytes/rendering delivery (reassembled blob + MIME) is a consumer-side concern (CONTENT §5.3 descriptor), not this route.
- **`TREE_GET` leaf (hash pointer) vs listing — named objects, no trailing slash (MUST; Amendment 5; leaf-body per Amendment 6).** Two configurable suffixes (§6.5.3), REQUIRED distinct: `tree_leaf_suffix` (default `.bin`) and `tree_listing_suffix` (default `.list`). There is **no trailing-slash listing form** (it does not survive static CDNs — index-document convention + per-CDN slash normalization; and a normalizing `foo/`→`foo` would 404 a listing by the no-redirect rule below). Every read URL is a concrete object key:
  - `{tree_prefix}/{peer_id}/{path}{tree_leaf_suffix}` ⇒ the **bound content hash** — a `system/hash` value in the **bare 2-key form `ECF({type: "system/hash", data: <H>})`**, `Content-Type: application/cbor` (the bound hash `H` verbatim — **its length follows its own format byte and MUST NOT be fixed**, `SPECIFICATION-FORMAT.md` §8.4.5). The tree is `path → hash` (ENTITY-CORE-PROTOCOL.md §1.7; EXTENSION-TREE.md §1); this route returns the hash, **not** the dereferenced entity. It is exactly `tree:get mode:"hash"` (ENTITY-CORE-PROTOCOL.md §1.7) exposed over HTTP. The consumer reads `data` (the bound hash `H`) and fetches the entity bytes with a second hop `CONTENT_GET {content_prefix}/{hex(H)}` (§6.5.3 step 5). **Two-hop, normative (Amendment 6).** Returning the dereferenced wire entity `ECF({type, data, content_hash})` here is **non-conformant** — it materializes a separate copy of the entity at every tree path bound to `H`, defeating the content-store dedup invariant (ENTITY-CORE-PROTOCOL.md §1.7: same content at N paths ⇒ one copy) that a static CDN cannot recover. Body is the bare 2-key form (one hash, no ambiguity), not 3-key: a path-addressed pointer has no useful self-`content_hash`, and a 3-key body would carry two hashes (the bound `data` and the pointer's own `content_hash`), forcing the consumer to disambiguate. **`tree_leaf_suffix` is still `.bin` by default** — the suffix names "the leaf at this path," and the leaf *is* its bound hash.
  - `{tree_prefix}/{peer_id}/{path}{tree_listing_suffix}` ⇒ the **listing** (below). `{tree_prefix}/{peer_id}{tree_listing_suffix}` ⇒ the peer-root listing; `{tree_prefix}/peers{tree_listing_suffix}` ⇒ the all-peers (universal-tree-root) listing.
  - **Append-one / strip-one (MUST):** append exactly one suffix; the server strips exactly one recognized suffix; the suffix *identity* selects entity vs listing. Total bijection for any name (entity `foo`⇒`foo.bin`, listing `foo`⇒`foo.list`, entity `foo.bin`⇒`foo.bin.bin`, listing `foo.bin`⇒`foo.bin.list`) — no publish-time check.
  - A bare path with no recognized suffix ⇒ `404`; `{peer_id}{tree_leaf_suffix}` (entity at a peer-id root) ⇒ `404` (roots are directories, ENTITY-CORE-PROTOCOL.md §1.4); bare `peers` (no listing suffix) ⇒ `404`. **No redirects** — no URL form depends on a trailing slash, so there is nothing to canonicalize.
  - A path that is both a bound entity and a parent of children answers the leaf-suffix form with the entity and the listing-suffix form with the listing (mirrors TREE §2.2; MUST NOT impose a file-or-directory exclusivity).
- **Listing body (MUST).** The **existing `system/tree/listing` entity** (ENTITY-CORE-PROTOCOL.md §3.9) — wire entity in ECF, `Content-Type: application/cbor`, shape `{path, entries: {name → {hash?, has_children}}, count, offset, next_page?}`. **Mutable view:** carries its own `content_hash` but MUST NOT be `immutable`-cached (re-rendered on subtree change). `count` is the **in-scope filtered** total (never the raw subtree total — §6.5.6 scope-gating + TREE §1176). **Pagination** is the optional `next_page` content hash: if more entries remain, the head listing carries `next_page` = the hash of the next `system/tree/listing` page, fetched via `CONTENT_GET /content/{hex(H)}` (immutable, content-addressed). The publish pipeline MUST bind each chain page into the served content namespace (EXTENSION-CONTENT.md §6.4.2) or it is unreachable. Live peers MAY also honor `?offset=&limit=` over the **scope-filtered** child set (independent of the `next_page` chain — not a seek into it). No JSON form.
- **`MANIFEST_GET` body (MUST).** `GET {manifest_prefix}` returns the publisher's signed manifest as a **wire entity `ECF({type, data, content_hash})`** (`system/peer/published-root`, defined normatively at `EXTENSION-TREE.md` **§3.3a**), `Content-Type: application/cbor`, carrying `content_hash` + signature ref. **The body MUST carry `prefix`** — a consumer cannot reconstruct a path or determine the published extent without it (§3.3a). Singular/terminal: no suffix, no trailing slash; `GET {manifest_prefix}/` ⇒ `404`; none published ⇒ `404`. The manifest is **mutable** (revocation lives here) ⇒ MUST NOT be `immutable`-cached. **Freshness bound (MUST — state it, do not infer it).** `seq` monotonicity is a **rollback defense**: a consumer that has seen `seq = N` MUST reject a served root with `seq < N`. It is **not a freshness guarantee.** No landed mechanism obliges an origin to serve the *newest* signed root, and **a consumer cannot distinguish a withholding origin from a publisher that simply has not republished** — the two are byte-identical at the consumer. A consumer therefore MUST NOT treat a successfully verified root as evidence that it is current; what it verifies is *authenticity and non-rollback as of the root it received*. The freshness ceiling that does exist is §6.5.6's 30 s bounded-convergence MUST, and it binds the **publisher's republish cadence**, not the origin's serving behavior — an origin that withholds is outside its reach.
- **Hash hex includes the format-code byte, and its length follows that byte (MUST).** The content hash in the URL (`/content/{hex(H)}`), the EXTENSION-CONTENT.md §6.4.2 tree binding key, and the `ETag` all use the ENTITY-CORE-PROTOCOL.md §3.5 convention: **lowercase hex of the full content hash *including the format-code byte***, NOT the 64-char digest-only hex. **The length is implied by the leading format byte and is never assumed** — 66 chars beginning `00` under ECFv1-SHA-256, 98 beginning `01` under ECFv1-SHA-384, and so on for any allocated code (`SPECIFICATION-FORMAT.md` §8.4.5; ENTITY-CORE-PROTOCOL.md §1.2). This keeps URL ⇄ binding parity (`in_scope(H)` = `tree:get({ns}/{same-hex})`, no boundary conversion) and preserves the algorithm discriminator for crypto-agility. (See EXTENSION-CONTENT.md §6.4.2.)

  > **Corrected 2026-08-10 — this bullet pinned 66 and reserved a partition it forbade anyone from entering.** The width was stated as the requirement while the very next bullet justified the format byte as *auto-diversifying the instant a non-SHA-256 hash ships*; nothing could ship into that second bucket. Measured cohort split at go `62712fd` / rust `0480712` / py `e60c822`: a `CONTENT_GET` for an in-scope 98-hex SHA-384 hash returned **`400` on go and rust** (the letter — a length gate at 66) and **`200` on python** (the stated intent — decode the format byte, let it imply the length). **Python's reading is the ruled one.** Neither implementation misread anything; the rule said both. Nothing caught it because no run in this project's history had produced a hash that was not 66 chars — a configuration axis with no coverage, not a missing test.


- **Sharded `content_layout` (`flat` / `sharded-2-flat` / `sharded-2-4` / `sharded-2-2`) slices the same `{hash}` hex defined above.** Slices are positions in that string, **counted from the left so they are independent of its total length**: `[0:2]` = the format-code byte (`00` for ECFv1-SHA-256, `01` for ECFv1-SHA-384) = the **algorithm partition** (one bucket per format actually authored; it diversifies the instant a non-SHA-256 hash ships — and a longer digest changes only how much tail follows the shard prefix, never the prefix itself); deeper slices shard the digest. `sharded-2-4` → `/00/{d0}/{wire}`. The shard prefix is a slice of the leaf id (git-style CAS); characters repeating between dir path and leaf are by construction. One definition of `{hash}` everywhere — no separate digest-sliced layout family.
- **Cache-Control (MUST).** `CONTENT_GET` content is **immutable** (the hash addresses fixed bytes forever) → `Cache-Control: immutable, max-age=<long>`. `TREE_GET` bindings are **mutable** (a path may be rebound to different content) → impls **MUST NOT** emit `immutable` on `TREE_GET` (it would make caches serve stale bindings after a rebind); omit it or use a revalidation-friendly directive. `ETag: "{hex(H)}"` on both, where **`H` is the addressed content hash**: for `CONTENT_GET` the URL hash; for the `TREE_GET` leaf the **bound hash carried in the pointer's `data`** (the hash the path currently resolves to — Amendment 6), NOT the pointer entity's own content hash; for a listing the listing entity's `content_hash`. A leaf `ETag` thus changes exactly when the path is rebound, which is the correct mutable-binding cache key. *(Rust surfaced the immutable rule; Go/Python drop `immutable` on tree-get.)*
- **Status codes (MUST — all `http-poll` read routes; Amendment 5).**
  - `200` — leaf hash-pointer (`system/hash`, §6.5.3.1) · listing (incl. empty in-scope, `entries={}` `count=0`) · content bytes · manifest found.
  - `400` — malformed: 64-char digest-only hash, unverifiable algorithm byte (§6.5.2c.1), suffix/slash on a hash, malformed percent-encoding, **an encoded slash (`%2F`) inside a path component** (path components are `/`-delimited; reject rather than recover as a literal).
  - `404` — not found AND no recognized suffix · out-of-scope (**identical** to not-held — T4 presence-oracle) · `{manifest_prefix}/` (terminal) · bare unknown first segment · `{peer_id}{leaf_suffix}` (root is a directory) · bare `peers`.
  - `405` — wrong method: `Allow: GET` on read routes; `Allow: POST` on the EXECUTE route.
  - `414` — **MAY**, request URL exceeds the operator-configured cap (RECOMMENDED 8 KB per RFC 7230 §3.1.1; tree paths are uncapped per ENTITY-CORE-PROTOCOL.md §1.4, so a length bound is the parser-DoS guard).
  - `501` is **not** a conformant steady-state response on any route in a shipped peer. **No `3xx`** — no URL form depends on a trailing slash.

#### 6.5.4 Discovery and freshness

How a consumer learns of a publisher's transport profiles is out of scope of this section. For v1: out-of-band (well-known URL, hard-coded, manually configured). Post-corridor: the EXTENSION-REGISTRY.md extension will define peer-ID → endpoint-set resolution.

**A static publisher's own transport profile is out-of-band too, and specifically is NOT served at `{manifest_url_prefix}` (MUST).** That slot is reserved for the signed `system/peer/published-root` (§6.5.3, §6.5.3.1). The collision is worth naming because the mistake is a *reasonable* one: a static publisher has no live surface on which to answer "what are your transports," `manifest_url_prefix` is the one singular terminal slot it does serve, and a transport profile is plausibly "the manifest." It is not — a consumer that follows `signed_pointer` to that URL and finds a profile entity has no signed root, and the `signed_pointer` advertisement is then false. Ship the profile beside the site (a well-known URL, a deployment descriptor, a pinned config), or do not advertise `signed_pointer` at all.

Freshness of a profile entity is the publisher's concern. Stale profile entries (publisher advertised then dropped a transport) are handled by the consumer's poll cycle on the signed pointer + retry semantics on the bridge call (`network_error` if endpoint unreachable; consumer attempts other profiles).

#### 6.5.5 Consumer modes (A1 dispatcher-driven, A2 operator-driven)

A Mechanism A consumer (per `GUIDE-EXTENSION-DEVELOPMENT.md §3.7` — HTTP-as-storage-transport, NOT HTTP-as-foreign-substrate) MAY operate in either of two modes. Both are conformant. The conformance contract is the same: hash-verify on every fetched byte stream + signature-verify on the manifest (when present per `EXTENSION-SUBSTITUTE.md` §7.2, which owns the manifest signature MUST and the `seq` freshness rule together with their error codes `manifest_signature_invalid` / `manifest_stale_seq`).

**Mode A1 — Dispatcher-driven (peer-resident).** Consumer is a full peer participating in dispatch (§10). On local content-store miss the dispatcher resolves `system/peer/transport/{publisher_peer_id}/*` → finds the `http-poll` profile entity (§6.5.3) with `tree_url_prefix`, `content_url_prefix`, `content_layout`, `tree_leaf_suffix`, and (optionally) `signed_pointer`; builds the URL; performs GET + hash-verify (no BRIDGE-HTTP import — Mechanism A); ingests via `system/content:ingest` into `target_namespace`.

**Mode A2 — Operator-driven (verifying client).** Consumer is a standalone tool (CLI fetcher, mirror utility, debugging client); NOT a peer; no dispatch participation. Receives URL prefix + publisher peer-id + path directly (CLI flags / config / env); builds the URL by hand using the same convention (`{tree_url_prefix}/{path}{tree_leaf_suffix}` for tree-leaves; `{content_url_prefix}/{layout-shape}/{hash}` for content); performs GET + hash-verify by hand; output is the entity bytes (inspection / mirroring / debugging). No `target_namespace`; no ingest.

The trust anchor in both modes is the content-hash + (when present) the manifest signature. The rest of §6.5's profile-entity / dispatcher / ingest apparatus is REQUIRED for A1 and OPTIONAL for A2.

**Conformance.** An impl SHOULD provide Mode A1. It MAY additionally provide Mode A2 (operator tool) for deployment / debugging / mirroring. Implementations MUST NOT publish manifests inconsistent between modes — the URLs A1 builds via dispatch MUST match what A2 receives via CLI for the same `(publisher_peer_id, path)` pair, since §6.5.3's URL-construction rules are common to both.

> **Scope note.** §6.5.5 names the two consumer *shapes*; it does NOT specify the substrate transport-composition mechanism (registration / selection / translation / fallback loop with registry re-resolve / mix-and-mash / extensibility). That mechanism is a separate forthcoming exploration. A1 vs A2 is about who holds the URL, not how transports compose.

#### 6.5.6 Serving mode — a live peer exposing `http-poll` routes (Amendment 4)

A live peer MAY expose `http-poll` content/tree/manifest routes from its own store on the **same HTTP listener** as its `http` EXECUTE route, composing both profiles per §6.5 ¶721 (validated three-way). This collapses the "live peer + separate CDN" two-process story into one box (dev loop, embedded deployments, CDN origin).

- **Route demux (G2) — literal-or-peer-id-parse (MUST; Amendment 5).** One listener. Demux the first path segment **in order** (this is NOT a length threshold — there is a 9–45-char gap between the reserved words and a peer-id; the *check* is literal-then-parse): (1) literal `content` → `GET {content-prefix}/{hex(H)}` content-by-hash; (2) literal `manifest` → `MANIFEST_GET`; (3) literal `peers{tree_listing_suffix}` (`peers.list`) → all-peers (universal-tree-root) listing — bare `peers` ⇒ `404`; (4) else **parse as a peer-id** — success → `TREE_GET {tree-prefix}/{peer_id}/{path}{suffix}` (leaf **hash-pointer** or listing by suffix, §6.5.3.1 — the leaf-suffix route returns the bound `system/hash`, NOT the dereferenced entity, identically to the static surface so consumers have one code path; Amendment 6); (5) `POST {http-path}` → EXECUTE; (6) else `404`. **Tree has no reserved word** — a parseable-peer-id first segment *is* the tree signal (co-located drops the `tree/` segment). The reserved literals `{content, manifest, peers}` can't collide with a peer-id (a peer-id parse of those short strings fails). No separate HTTP server is required.
- **`serve_scope` exposes the local view (grounding — ENTITY-CORE-PROTOCOL.md §1.4).** A serving peer publishes a scoped projection of its **local view of the universal tree** (ENTITY-CORE-PROTOCOL.md §1.4 *Local view* + *Authority layers*): paths under `/{any_pid}/...` it holds — its own authoritative namespace and any cached/mirrored remote namespaces. `serve_scope` declares *which slice of that local view* this listener answers for; it is not a claim of cross-peer authority (the keyholder for `{pid}` remains canonical — ENTITY-CORE-PROTOCOL.md §1.4 layer 2). A `peers.list` enumerating multiple peer-ids is the **normal universal-tree-root view** (every peer's root is a set of peer-ids it holds), not a multi-tenant feature. The cross-layer discipline that every path-taking layer be peer-id-agnostic — and its conformance guard — lives in `GUIDE-SERVING-MODE.md` (the model itself is ENTITY-CORE-PROTOCOL.md §1.4, not restated here).
- **Served-set scope (`serve_scope`) — a capability token (Amendment 5).** The read routes carry **no request auth** (the client may not speak the protocol) — hash-knowledge (content) / path-presence (tree) is the read authority. The security lever is **which entities the routes answer for**, and `serve_scope` is a literal `system/capability` token the publisher generates, evaluated by **the same cap evaluator the live-EXECUTE surface uses** (`check_permission` / `check_path_permission`). Because the request is unauthenticated, the **published `serve_scope.cap` IS the effective cap** passed to the evaluator: where the live surface asks "does the connection's cap-set permit `get(path)`?", the serving surface asks "does `serve_scope.cap` permit `get(path)`?" — one ACL machinery, no drift. Impls MUST pass the published cap as the effective cap (MUST NOT reach into capability internals or synthesize a connection context). The published-set / whole-store distinction below is expressible as cap shapes, in EXTENSION-CONTENT.md §6.4.1 topology terms:
  - **`published-set` (SHOULD default)** = EXTENSION-CONTENT.md §6.4.1 namespace-scoped topology. A `CONTENT_GET(H)` resolves iff EXTENSION-CONTENT.md §6.4.2 Hash Tree Presence holds (`tree:get({namespace}/{hex(H)}) ≠ null`). Populate by binding shareable hashes under a content namespace (`system/content/{ns}/{hex(H)}`) and/or the closure of a published subtree.
    - **Signed-root closure (MUST when `signed_pointer` advertised; Amendment 10).** A publisher that advertises a `signed_pointer` (i.e. participates in the §6.5.3 / `EXTENSION-TREE.md` §3.3a walk-from-signed-root threat model) MUST also resolve, under the same `serve_scope`, the **transitive trie-node closure reachable by hash-link from `published-root.root_hash`** — the CHAMP trie root node + all interior nodes + all leaf-bound content hashes + the `published-root` entity + its `system/signature` entity at the ENTITY-CORE-PROTOCOL.md §5.2 invariant-pointer path. Trie nodes are hash-linked, not path-bound (ENTITY-CORE-PROTOCOL.md §1.7), so the bare path-binding check above does not reach them; without the closure, `CONTENT_GET(H)` 404s on interior trie nodes and the hash-chain walk from `root_hash` cannot complete — the `signed_pointer` machinery is non-functional in its premised security model. Mechanism is impl choice (precompute closure at publish time and stamp into the served-set, or compute lazily via a `trie_reachable_from(root_hash, H)` check); the contract is "MUST resolve." A publisher who serves `published-set` WITHOUT advertising `signed_pointer` (content-only mirror, no signed-root walk) retains the path-bound-only shape — unchanged. `whole-store` trivially covers this case.
      - **Evaluation timing — the closure tracks the current signed root, it is not a startup snapshot `[MUST; ruling 2026-08-07, from `entity-core-go`'s first full-coverage cross-impl run]`.** The served closure is the closure of **`published-root.root_hash` as it stands now**. When the publisher republishes the signed root, the served set is the **new** root's closure, and entities bound since the previous publish become servable **at that republish**. Two readings were live across the cohort — *recompute-on-root-change* vs *evaluate-once-per-publish/startup* — and **recompute-on-root-change is the ruling**. The spec already forces it and the proof is one line up: `whole-store` is stated to "trivially cover this case," and `whole-store` serves any *stored* hash **including one stored after startup** — which a startup-frozen closure would not. A live serving peer whose served set could never grow without a restart would also contradict §6.5.6's own premise (a *live* peer collapsing the "live peer + separate CDN" story into one box). **The trigger is the root republishing, not the bare `tree:put`.** An entity written into the local trie but not yet carried into a re-signed `published-root` is outside the closure under **both** readings — so a conformance check that binds and then probes **without** an intervening republish is asserting something this rule does not grant, whichever way the ruling went. (No timing rule is needed for the non-`signed_pointer` path-bound shape: path-binding is checked per request and is live by construction.)
      - **The closure's extent is prefix-scoped `[MUST; added 2026-08-08]`.** What a signed-root closure must cover is determined by the published root's **`prefix`** (`EXTENSION-TREE.md` §3.3a) — a publisher at `prefix: "system/"` commits to a strictly smaller set than one at `prefix: "/"`. **Two conformant publishers therefore legitimately serve different extents**, and a consumer MUST read the extent from `prefix` rather than inferring it. *(Observed live: three impls publishing 401 / 415 / 490 bindings, which was three different published subtrees — not a defect in any of them.)*
      - **Bounded-convergence republish `[MUST; added 2026-08-08]`.** A publisher advertising a `signed_pointer` **MUST** republish its signed root after a change to the tracked prefix, such that the published root **converges** to the tracked root within a **maximum convergence delay of 30 s**. A publisher MAY advertise a **tighter** value and consumers MAY rely on the advertised one; it MUST NOT advertise a looser one. **Coalescing / debouncing is explicitly permitted** — a signature per `tree:put` is write-amplifying and is not the intent; the requirement is convergence, not immediacy (a cascade SHOULD produce one republish, not one per binding). `seq` MUST increase monotonically across republishes and `predecessor` MUST carry the prior `published-root` content hash once one exists. A publisher that does **not** advertise a `signed_pointer` has no republish obligation (consistent with the content-only-mirror carve-out above).
        - **Why this is a MUST and not left to the publisher.** Without it, a publisher that mints once at startup and never republishes is *conformant*, and its served closure is frozen forever — which is functionally the evaluate-once reading this section rejects, reached by a route the timing rule alone does not close. The rule above governs what the closure *tracks*; this one governs whether the thing it tracks ever *moves*.
        - **Why the ceiling is normative rather than purely advertised.** A purely publisher-advertised bound is **not cross-impl testable**: a peer advertising 24 h would be conformant and unexercisable in any conformance run, making the MUST unfalsifiable at exactly the cross-peer boundary that motivates it — and it would force a consumer to fetch publisher-specific configuration before it could know whether what it holds is stale, the same out-of-band-parameter defect §3.3a removes from the prefix. The asymmetric form (normative ceiling + optional tighter advertisement) keeps one number cross-impl observable and leaves the other an optimization. *(Argued by `entity-core-go` as lead implementation, 2026-08-08; arch's initial lean toward a purely advertised bound is withdrawn.)*
        - **Provenance of the value, and how to change it.** 30 s is **derived, not authored**: `entity-core-go`'s `seed_republished` observes convergence within a **10 s** window on all three implementations today, so 10 s is the measured-achievable figure; 30 s takes 3× headroom for debounce and load while keeping a single conformance check bounded well under a minute so CI can exercise it every run. **It is falsifiable on measured evidence** — if a conformant publisher under realistic load cannot meet it, that is a finding that fixes this number in place, not a reason to weaken the rule's shape.
        - **Vocabulary is pinned: "maximum convergence delay"** — not "floor", not "bound". *(Requested by `entity-core-go` after `entity-core-rust` spent part of a cycle on a genuine floor/ceiling collision in which two readings shared one key.)*
  - **`whole-store` (explicit opt-in)** = EXTENSION-CONTENT.md §6.4.1 single-trust-domain topology; serves any stored hash. MUST be explicit, documented, default-off (EXTENSION-CONTENT.md §6.4.1 marks multi-party use of this "security-defective").
  - A serving impl SHOULD be able to bound its served set to a published-set. **Out-of-scope and not-held MUST return an identical `404`** (T4 presence-oracle mitigation).
- **Two faces of the published set.** `serve_scope` has a **tree-face** (which tree paths `TREE_GET` resolves) and a **content-face** (which hashes `CONTENT_GET` resolves via EXTENSION-CONTENT.md §6.4.2). `TREE_GET` over poll is **published-scope-gated, not unsupported** — it resolves iff the path is within the served subtree; `501` is never the normative shape. A peer publishes a subtree; both its tree paths and the content reachable from it are served under the same `serve_scope`.
- **Listings are scope-gated (MUST; Amendment 5).** A listing enumerates only children within `serve_scope`; `count` MUST be the **in-scope filtered** total (never the raw subtree total — a discrepancy leaks hidden-path existence, TREE §1176). An out-of-scope or non-existent prefix returns `404` **identical** to not-held (T4). An **in-scope** prefix with no children returns `200` + `entries={}` + `count=0` (in-scope-ness is the access boundary; an empty published directory is legitimately observable). Live `?offset=&limit=` MUST run **post-scope** (offset numbering over raw children leaks an offset oracle).
- **G4 — reserved words (normative; Amendment 5; extensibility hook added Amendment 9).** The co-located demux reserves the literal first-segment words **`{content, manifest, peers}`** (§6.5.6 demux); operators MUST pick a live EXECUTE `POST` path that is none of the three and not a parseable peer-id. The reserved-word table is **explicitly extensible**: other specs MAY register additional first-segment literals for their own purposes (e.g. an L5 application convention reserving a URL projection prefix). The §6.5.6 demux consults the union of NETWORK's reservations and registered extensions. **Length-floor rule (normative; Amendment 9).** Any registered reserved word MUST be strictly shorter than the Ed25519 peer-id minimum encoded length (which sets the floor of the "9–45-char gap" observation above) so that a parse-as-peer-id can never succeed on a reserved word — preserving the literal-then-parse-by-string demux algorithm unchanged regardless of how many words are registered. NETWORK does NOT enumerate the registered extensions; each owner spec lists its own.

Request-side *who*-restriction (where a deployment needs it) is **foreign auth** — OAuth / HTTP Basic / mTLS / API gateway via a reverse proxy — a deployment concern outside this spec (a future `bridge-oauth` only on a driver). Serving mode is for mostly-public sharing by design. Operator guidance (closure computation, the `whole-store` caveat + the T2/T3 confidentiality audit, foreign-auth context, content-store hygiene) lives in `GUIDE-SERVING-MODE.md`.

### 6.6 Session State Entity (`system/peer/session/{peer_id}`)

*Amendment 8.* §6.1 states that a session's held capability tokens "persist in the entity tree." This section makes that literal: per-peer authentication state is a tree entity, not per-connection memory. The entity is the **durable per-peer AUTH record** — it answers exactly one question for §10 dispatch: *do I already hold a valid capability to dispatch to this peer, or must I re-handshake?* It is **not** the liveness or reachability record; those are the other three entities of the per-peer set (§6.1, ENTITY-CORE-PROTOCOL.md §3.13, §6.5):

| Dispatch question | Answered by |
|---|---|
| Do I hold a valid cap (skip handshake)? | **`system/peer/session/{peer}`** — `held_capability` + `expires_at` |
| Is there a live connection to reuse? | `system/connection/{peer}` (ENTITY-CORE-PROTOCOL.md §3.13) |
| What transport / poll-fallback? | `system/peer/transport/{peer}/*` profiles + reachability classes (§6.5.1b) |
| Connection lifecycle? | `system/peer/status/{peer}` (§6.2, ENTITY-CORE-PROTOCOL.md §3.13) |

**Schema:**
```
type: "system/peer/session"
path: {local_peer}/system/peer/session/{remote_peer_id}
data: {
  remote_peer_id:       text,                    ; the peer this session authenticates
  remote_identity_hash: bstr(33),                ; system/hash of the remote identity entity
  remote_public_key?:   bytes,                    ; OPTIONAL denorm — see note below
  held_capability:      { hash: bstr(33), chain: [bstr(33), ...] },   ; cap I wield to dispatch to remote
  minted_capability?:   { hash: bstr(33), chain: [bstr(33), ...] },   ; cap I issued to remote (granter bookkeeping)
  granted_at:           uint,                     ; epoch ms — = last handshake
  expires_at?:          uint                      ; epoch ms, omitempty — cap validity window
}
```

**Field semantics:**

- **`held_capability`** (REQUIRED) — the capability the *remote* granted *this peer* at handshake. §10 dispatch reads this to authenticate an outbound EXECUTE and skip re-handshake. `chain` is an array of `system/hash` pointers, **leaf→root, length ≥ 1** (full-wire-form hashes — format byte ‖ digest, length per that byte, §8.4.5 — per the ENTITY-CORE-PROTOCOL.md §3.5 wire form), resolved through the content store at reuse — entities are referenced by hash, never inlined (ENTITY-CORE-PROTOCOL.md §1.7 dedup model). `chain` carries **only** `system/capability/token` content-hashes — no signature/identity/granter padding (a verification bundle does not belong here; the verifier resolves the chain from the content store).
- **`minted_capability`** (OPTIONAL) — the *connection-handshake* cap *this peer* issued *to the remote*, recorded for R3a idempotency (a re-dial returns the same cap rather than minting a churny new one) and for revocation. **It is NOT a reverse-delivery cap.** In a bidirectional pair, A's `minted_capability` for B is the *same cap entity* as B's `held_capability` from A — one cap, recorded from both ends. Back-direction **delivery** authorization is the per-delivery `deliver_token` (granted at subscribe/continuation time, EXTENSION-INBOX / EXTENSION-SUBSCRIPTION), unchanged by this amendment. The session entity moves ONLY the handshake cap; it MUST NOT be built to solve generic back-direction dispatch. **Carve-out — symmetric establishment.** This one-directional mint is the *asymmetric* case (dial-by-address). A **§6.5 (b) symmetric rendezvous** establishment (`EXTENSION-SIGNALING.md` §6.5) adds one reciprocal grant — the *dialer* mints, for the acceptor, the grant it would issue that peer **as an inbound dialer** (`EXTENSION-SIGNALING.md` §6.5 (b) Contents: the assembled floor ∪ policy, advertisement-filtered — a symmetric *construction*, not a byte-copy of this handshake cap) — yielding bidirectional authority *by design*, which is not the generic back-direction dispatch this MUST NOT forbids but a match to the establishment's own symmetry (back-direction-authority taxonomy, `guides/GUIDE-CAPABILITIES.md` §4a). That reciprocal grant is **connection-scoped** and is deliberately **not** recorded here (it is establishment-scoped, drop-on-disconnect; a stale grant reused on reconnect without re-meeting at the key would authorize outside its establishment). Reaching a profile-less peer by reusing a connection it opened (the V7 §6.11 reentry seam surfacing in §10 dispatch; `GUIDE-CONFORMANCE.md` §7a.2a) is **resolution, not authority**: a dispatch over it still needs a row-1/2/3 cap, and a *generic* dispatch to a profile-less peer over an asymmetric connection is authorized by its trigger's `deliver_token` where one exists and **fails closed** otherwise — the prohibition holds for every asymmetric connection.
- **`granted_at`** — handshake time. There is deliberately **no** `last_active` field: per-message liveness is `system/peer/status.last_seen`'s job (**snapshot at transition; per-tick freshness is implementation-internal — §5.4.1**, ENTITY-CORE-PROTOCOL.md §3.13); putting it here would force a tree write per message (write amplification → subscription/revision/history fan-out) on the auth record. *(§5.4.1 applies the same reasoning to the status entity itself, which is where it had leaked back in.)*
- There is deliberately **no** `status` field: connection lifecycle is `system/peer/status`'s job, and cap validity is derivable from `expires_at` (+ revocation check). The auth record needs no separate status.
- **`remote_public_key`** (OPTIONAL) — a denormalization. `peer_id` is a *hash* of the key, not the key, so the pubkey is not trivially derivable without the identity entity; 32 bytes for signature-verify-without-fetch is cheap. Peers MAY omit it and dereference `remote_identity_hash`.

**Lifecycle:**

- **Persistence (MUST).** The entity persists across `disconnected` status — **a disconnection MUST NOT delete it.** Surviving reconnect (and, for a persisted identity, process restart / WASM page-reload) is the entire purpose. (An ephemeral-keypair browser peer gets a new `peer_id` on reload by the identity model and therefore inherits no session — that is correct, not a gap; see ENTITY-CORE-PROTOCOL.md §1.5 / EXTENSION-IDENTITY.)
- **Grants change → mint fresh + overwrite in place.** One entity per peer, mutable. A runtime widen/narrow of the grant rejects the cached cap, re-mints, and updates `held`/`minted` hashes in place — preserving R3a's "same grants ⇒ same cap."
- **No self-session.** A peer never writes `system/peer/session/{local_peer_id}`; local dispatch short-circuits (the in-memory degenerate row, §10).
- **Writes fire ordinary subscription events.** The entity is a real tree write; it hits revision/history and fires events like any write. Subscriptions observing the session subtree MUST scope their patterns accordingly.

### 6.7 Reachability Facts

*Amendment 13.* §10 dispatches by **reachability class** (§10 decision table) — but a peer has no protocol way to learn its **own** reachability facts. It cannot learn the public-facing `IP:port` its NAT allocated (only the router knows, and the router reveals it only implicitly, on outbound packets); it cannot learn whether it is publicly dialable; it therefore cannot honestly populate or order its own transport profiles for a peer on the far side of a NAT.

This section defines the three facts and stops there:

| §6.7.1 | **Observed address** — how this peer looks from outside (the STUN binding, in our protocol) |
| §6.7.2 | **Dial-back** — whether this peer is publicly dialable |
| §6.7.3 | **Candidates** — the typed set of addresses this peer might be reached at |

**These are pure transport facts, which is why they belong here** — below the model, in the extension that owns connections. The entity layer never sees an `IP:port` (ENTITY-CORE-PROTOCOL.md §1.4, local-view authority); an application extension is the wrong altitude for all three.

**They are useful the moment they land, independent of any hole-punch.** NAT-type detection falls out for free (§6.7.1); a peer that knows it is NAT'd stops advertising unreachable direct profiles and leans on its held outbound socket, cutting failed dials. The **punch protocol that exchanges and acts on these facts is deliberately not here** — gathering is a local fact, exchanging is a protocol, and keeping them apart is the clean seam. §6.7.3 gathers and types candidates; nothing in this spec sends one to another peer.

**What is already handled, so it is not re-solved here.** The asymmetric case works today: a non-listening NAT'd peer that holds an **outbound** duplex socket to a public peer receives pushes down that socket (§10 `held_connection_client`) — **NAT'd peer ↔ public peer works in both directions with no traversal at all.** The one genuinely missing case is *two* peers both behind NAT wanting a direct connection, and that is the punch's problem, not this section's.

#### 6.7.1 Observed-Address Reflection (`observe-address`)

When peer A connects to peer R, R can see the source `IP:port` its transport reported for that connection. That observed source **is** A's public NAT mapping. R telling A what it saw is the whole mechanism.

```
observe-address() → system/network/observe-address-result

system/network/observe-address-result := {
  fields: {
    observed_address: {type_ref: "primitive/string"}
                      ; the source IP:port the responder observed on THIS connection,
                      ; e.g. "203.0.113.7:51820"
  }
}
```

**The three MUSTs.** Each is a cross-peer seam where two conformant readings diverge — none is a style preference:

1. **`observed_address` is the transport-layer source of the connection the request arrived on, and never a value echoed from the request body.** A body-supplied address makes the responder a laundering service for an attacker's chosen address, and it is the same amplification seam §6.7.2 closes on the dial-back side.

2. **`observed_address` MUST NOT be persisted to any durable per-peer address field** — not `system/connection.address` (ENTITY-CORE-PROTOCOL.md §3.13), not a `system/peer/transport/*` profile (§6.5.1), not `system/peer/status`. It is read from the live connection and returned; it is not connection-state, not a transport profile, and not a peer attribute.

   > **Why this needs saying.** An observed source address is a **responder-side** fact, and every existing place this spec writes an address is **dialer-side dialable-endpoint** state. `system/connection/{peer_id}` is the record an implementer reaches for first — it is keyed by peer, it has an `address` field, and it is already reachable from the handler. It is wrong twice: its `address` means *the endpoint I dial to reach this peer* and it is written dialer-side (the responder holds no dialable address for the remote and correctly records nothing). An ephemeral source port written there is a routable-**looking** value that routes nowhere, and §10 and `system/peer/status` both consume that field as dialable. The cheap fix corrupts dispatch for every other reader. It is also insufficient on its own terms: §6.7.3's mapping is **per socket**, and one record per peer cannot express it.

3. **The mapping belongs to a socket, and the peer MUST punch from that socket** — stated in full at §6.7.3, because it is a property of the candidate rather than of this operation.

**What it costs to implement, stated plainly.** The handler answering `observe-address` needs the source address of the connection the request arrived on. A dispatch seam that extracts only the remote peer identity does not carry it, and this spec deliberately does **not** widen the general handler context to fix that — a narrow, NETWORK-scoped accept-side path from the connection to this operation is the intended shape. Where it sits in a given implementation's layering is that implementation's call; that it is a real addition, and not free, is not in dispute.

**NAT-type detection falls out for free.** A peer collects `observed_address` from **several** reflectors. Agreement ⇒ a stable, endpoint-independent mapping (punchable). Disagreement ⇒ the mapping differs per destination ⇒ symmetric NAT ⇒ a punch will likely fail ⇒ prefer relay. No extra mechanism, and it is why the security posture below can afford to be permissive.

**A single reflector is advisory, never trusted.** A lying reflector feeds a peer a wrong mapping — wasted punches, or steering toward an attacker. **No security decision rests on one observed address**; the agreement-across-reflectors discipline above is what makes the fact usable.

**Native and browser do not share this mechanism (do not conflate).** A browser's own ICE agent gathers `srflx` candidates by speaking **STUN/UDP to a standard STUN server**, a wire protocol our peers do not speak. **A peer offering `observe-address` is not a STUN reflector for a browser.** Browser reachability uses standard STUN/TURN infrastructure; what this ecosystem contributes on the browser leg is signaling carriage, not reflection. Collapsing the two is a cross-peer-seam error.

#### 6.7.2 Dial-Back (`check-reachability`)

"Am I publicly dialable, or behind NAT?" A peer learns this by asking another peer to dial it back and reporting whether the dial arrived.

```
check-reachability() → system/network/check-reachability-result

system/network/check-reachability-result := {
  fields: {
    reachable:      {type_ref: "primitive/bool"}
                    ; did the dial-back to address_tested succeed
    address_tested: {type_ref: "primitive/string"}
                    ; the observed source address the responder dialed back —
                    ; the same value observe-address would return on this connection
  }
}
```

**The load-bearing security rule (MUST).** This is not optional to get right: a body-supplied target turns every dial-back peer into a DDoS reflector.

> **MUST.** The dial-back targets the **requesting peer's own observed source address** — the address the asked peer *itself observed* on the request connection — and **never an address supplied in the request body.** The asked peer **MUST** rate-limit dial-backs per requester and keep the dial-back payload small and fixed-size, so there is no amplification factor.

This is a **MUST and not a SHOULD** for the reason this spec pins anything: two conformant readings of "dial back the requester" — one using the observed source, one honoring a body field — diverge into a security hole **at the peer boundary**, and prose review does not catch it. (It mirrors the STUN binding rule and libp2p AutoNAT's dial-back-to-observed-address-only.)

`address_tested` is reported back so the requester can confirm *which* address was proved, rather than inferring it — the value is the responder's, and echoing it closes the loop without ever accepting one.

#### 6.7.3 Candidate Gathering and Typing

To be reachable, a peer knows all the addresses it might be reached at — its **candidates** — typed and ordered. **This section defines gathering and typing only.** The exchange of candidates between two peers is the punch-coordination protocol and is not in this spec.

```
system/network/candidate := {
  fields: {
    address:   {type_ref: "primitive/string"}
               ; IP:port
    type:      {type_ref: "primitive/string"}
               ; "host" | "srflx" | "relay"
    substrate: {type_ref: "primitive/string"}
               ; "tcp" | "quic" | "webrtc" — which transport this candidate is punchable on
  }
}
```

| Type | Origin | Priority |
|---|---|---|
| `host` | a local/LAN address — works when peers share a network | highest (cheapest) |
| `srflx` | server-reflexive: the mapping observed via §6.7.1 — the hole-punch target | middle |
| `relay` | a public relay address — the always-works fallback | lowest |

**Ordering: `host` → `srflx` → `relay`; the first pair that completes a connectivity check wins.** This is the **session-scoped** extension of §10's existing "try profiles in `(priority asc, profile-id lex)` order" — the same try-in-order idea applied to ephemeral candidates instead of durable profiles.

**Candidates are session-scoped and ephemeral — NOT durable transport profiles (MUST).**

> **MUST NOT** model a candidate as a durable `system/peer/transport/{peer}/{profile-id}` profile entity (§6.5.1). §6.5 profiles are **stable published endpoints** — a TCP listener URL, an `http-poll` CDN prefix. Candidates change per session and per NAT mapping and exist for one connection attempt. **A candidate written as a durable profile goes stale instantly and mis-routes every later dispatch that reads it.**

Candidates therefore travel inside coordination messages, **never as published tree state.** This is the durable-vs-ephemeral seam that, uncaught, produces the classic "worked for the issuer, stale for everyone else" cross-peer failure.

**A `srflx` candidate MUST be the mapping of the socket the peer will punch from (MUST).**

A NAT allocates a mapping **per local socket**. An observed address is therefore meaningful only *for the socket that produced it*.

> **MUST.** A peer publishing a `srflx` candidate **MUST** punch from the **same local endpoint whose mapping was observed** — binding the reflector connection and the punch socket to the same local port using the platform's address/port-reuse options. A `srflx` gathered on one ephemeral socket and punched from another **is not the peer's address**: it describes a hole that will never open.

**Why this is normative rather than an implementation detail.** The socket options are the implementation's business; the **binding between the candidate and the socket** is not. A peer that gets it wrong sends its counterparty an address that is a **lie**, and the counterparty punches at a mapping that does not exist. It never appears on the wire, yet it is cross-peer observable in its effect.

> **This MUST is unverifiable on loopback — do not write a loopback test for it `[demonstrated 2026-08-02]`.** A peer that advertises one endpoint and punches from another **passes a loopback harness**: with no NAT in path its dial still lands on the counterpart's listener, so every observable exchange reads green. Under two NATs the same peer fails with an outbound dial and no path. This was shown with a purpose-built violating peer on both substrates, not argued. The requirement is **mapping-dependent**, and mapping-dependent requirements are structurally invisible without a NAT — see `EXTENSION-SIGNALING.md` §11.5.1 for the class and the substrate table. Emulated dual-NAT is the cheapest substrate that exercises it and is CI-able.

> **Debugging note (informative, and the reason this is pinned in advance).** This failure **wears another failure's costume.** The symptom is "the punch didn't land" — indistinguishable from a mistimed simultaneous-open, whose delay is a sanctioned local tunable. An implementer can therefore spend an entire debugging budget inside the one knob guaranteed not to be the problem. **On a first cross-implementation punch failure, bisect against this rule before touching the timing.**

Its cost is substrate-dependent: on UDP/QUIC it is one socket reused and effectively free; on TCP it requires `SO_REUSEADDR`/`SO_REUSEPORT` plus an explicit bind on both dials, and **is not satisfiable by discipline.**

#### 6.7.4 Capabilities and Rate Limiting

| Capability | Gates | Posture |
|---|---|---|
| `system/capability/network-reflect` | `observe-address` (§6.7.1) | **A broad default grant is reasonable** — the operation only echoes the source address the peer itself observed, so it leaks nothing the requester does not already imply by connecting, and the response is a single small address (no amplification factor). Still **rate-limited**. |
| `system/capability/network-dialback` | `check-reachability` (§6.7.2) | **Restricted.** A peer **SHOULD** grant it to peers it is actively connecting with, so setup-time reachability checks work, and **SHOULD NOT** make it an open grant to arbitrary peers. Always **rate-limited** per requester (§6.7.2 MUST). |

The asymmetry is deliberate and is the whole security story: reflection is a mirror (the requester learns about itself), while dial-back **causes the responder to emit traffic at an address**, which is why it is capability-restricted, rate-limited, fixed-size, and pinned to the observed source.

Denial uses the ordinary capability path — 403, no new error code.

#### 6.7.5 Composition with §10 Dispatch

The facts feed the existing dispatch ladder; they do not fork it.

- A peer that learns it is **not** publicly dialable (§6.7.2) **SHOULD NOT** advertise direct listener profiles that cannot be reached, and should rely on its held outbound socket (§10 `held_connection_client`) or a relay. This is the immediate, punch-independent payoff: fewer failed dials against addresses that were never reachable.
- A peer that finds its mapping **disagrees across reflectors** (§6.7.1, symmetric NAT) **SHOULD** prefer relay over any future punch attempt — it knows before trying.
- The candidate ladder (§6.7.3) is **session-scoped** and does not enter `resolve_profiles` (§10 step 3), which resolves durable profiles only. Nothing in §6.7 changes the §10 pseudocode or the reachability-class table.

**Conformance posture: additive, and not v1-blocking.** A NAT'd peer already receives via store-and-forward and talks to public peers via its held outbound socket. §6.7 adds facts that *improve* dispatch and *enable* a later punch. No V7 change, no wire-format change, no new error code.

> **The validation gate (the meta-rule applies).** None of these facts is *validated* until a cross-implementation conformance run exercises them — two conformant peers where one reflects the other's real observed address, and a dial-back that correctly reports reachable/not across a real NAT. **That run is the gate; prose review is not.** Two items are deliberately **not** re-derived here and are to be read from the references at build time rather than invented: the connectivity-check / consent-freshness handshake (RFC 8445 §7, RFC 7675) and symmetric-NAT port prediction. Both are detailed, well-studied, and security-sensitive, and both gate the punch rather than these facts.

---

## 7. Subscription Restoration

After reconnection, subscriptions that were active before the disconnect need their notification delivery re-activated.

### 7.1 Subscription Persistence

Subscription entities at `system/subscription/*` are tree entities (EXTENSION-SUBSCRIPTION.md §2.1). They survive connection drops because they are stored in the tree, not in connection state. However, the notification delivery path may be broken — the deliver token's target peer was unreachable.

### 7.2 Restoration Algorithm

```
restore_subscriptions(peer_id):
  ; Find subscriptions with delivery targets on the reconnected peer
  for sub_path in list_paths("system/subscription/"):
    sub = entity_tree.get(sub_path)
    if sub.data.deliver_uri starts with "entity://" + peer_id:
      ; Re-validate deliver token
      token = content_store.get(sub.data.deliver_token)
      if token is null or is_expired(token):
        ; Token expired during disconnect — subscription is dead
        entity_tree.put(sub_path, null)
        continue

      ; Re-register in notification index
      notification_index.register(sub)

  ; Find subscriptions we hold on the remote peer
  ; These need to be re-validated with the remote
  for sub in local_remote_subscriptions(peer_id):
    result = execute_remote(peer_id, "system/subscription", "subscribe", {
      resource: {targets: [sub.pattern]},
      params: {
        events:        sub.events,
        deliver_to:    sub.deliver_to,
        deliver_token: sub.deliver_token    ; Re-use existing token if valid
      }
    })
    if result.status != 200:
      ; Remote rejected — subscription lost, subscriber must re-create
      log_warning("subscription_restoration_failed", sub.subscription_id)
```

### 7.3 Gap Detection

Between disconnect and reconnect, tree changes may have occurred that produced no notifications. After restoration, the subscriber SHOULD reconcile:

1. For each subscription path pattern, GET the current entity hash
2. Compare with the last known hash from the most recent notification
3. If different, an intermediate change was missed — process accordingly

For revision-enabled prefixes, this gap is automatically handled by revision pull (EXTENSION-REVISION.md §7.1).

---

## 8. Pending Delivery

When a handler emits an EXECUTE targeting a remote peer that is currently disconnected, the message is queued for later delivery.

### 8.1 Queue Storage

Pending deliveries are stored as entities in the tree:

```
system/outbound/{peer_id}/{sequence}  → system/network/pending-delivery
```

The sequence number is monotonically increasing per peer, providing delivery ordering.

### 8.2 Queueing

```
on_outbound_dispatch(peer_id, execute):
  connection = entity_tree.get("system/connection/" + peer_id)

  if connection is null or connection.data.status != "active":
    ; Queue for later delivery
    sequence = next_sequence(peer_id)
    pending = {
      type: "system/network/pending-delivery",
      data: {
        peer_id:    peer_id,
        sequence:   sequence,
        execute:    execute,
        created_at: now(),
        expires_at: execute.bounds.ttl_absolute or (now() + 3600000)    ; 1h default
      }
    }
    entity_tree.put("system/outbound/" + peer_id + "/" + sequence, pending)
    return QUEUED

  ; Connection active — send directly
  send(connection, execute)
  return SENT
```

### 8.3 Drain

On reconnection, queued messages are delivered in sequence order:

```
drain_pending(peer_id):
  for path in sorted(list_paths("system/outbound/" + peer_id + "/")):
    pending = entity_tree.get(path)

    ; Check expiry
    if pending.data.expires_at is not null and now() > pending.data.expires_at:
      entity_tree.put(path, null)    ; Expired — discard
      continue

    ; Deliver
    result = send(connection, pending.data.execute)
    if result is error:
      ; Connection failed again during drain — stop, will retry on next reconnect
      return

    ; Delivered — remove from queue
    entity_tree.put(path, null)
```

### 8.4 Queue Limits

Implementations SHOULD impose limits on pending delivery queues:

| Limit | Recommended | Behavior on exceed |
|-------|-------------|-------------------|
| Max messages per peer | 1000 | Reject new queuing, return error to sender |
| Max total bytes per peer | 10MB | Reject new queuing |
| Max age | 1 hour | Expire on drain (§8.3) |

---

## 9. Graceful Close

### 9.1 Reason Codes

| Reason | Meaning | Subscription behavior |
|--------|---------|----------------------|
| `shutdown` | Peer is shutting down | Delete subscriptions from this peer |
| `idle` | Connection idle, may resume | Preserve subscriptions with timeout |
| `error` | Connection error occurred | Preserve subscriptions with timeout |
| `migration` | Moving to different address | Preserve subscriptions, expect reconnect |

### 9.2 Close Flow

```
Initiator                           Responder
    |                                    |
    +-- EXECUTE connect/close ---------->|
    |   params: {reason: "idle"}         |
    |<---- EXECUTE_RESPONSE (200) -------+
    |                                    |
    [transport close]                    |
```

The close operation is best-effort — if the connection is already broken, the close message may not arrive. The responder detects connection loss via keepalive failure (§5) and applies default behavior (preserve subscriptions with timeout).

*Implementation note:* When closing a WebSocket connection, the entity protocol close (this section) SHOULD complete before the WebSocket close frame (RFC 6455 §5.5.1) is sent. Sending the WebSocket close frame transitions the connection to a closing state in which application messages may not be delivered. Implementations SHOULD: (1) send the EXECUTE connect/close message, (2) await the EXECUTE_RESPONSE, (3) then send the WebSocket close frame.

---

## 10. Outbound Dispatch

When a handler constructs an EXECUTE targeting a remote peer, the dispatch layer routes it by the target's **reachability class** (Amendment 8, R5). The dispatcher composes the four per-peer entities (§6.6 session, ENTITY-CORE-PROTOCOL.md §3.13 connection, §6.5 transport profiles, §6.2 status): it reuses a live connection if one exists, else authenticates with the held capability and resolves the target's profiles, else queues.

```
dispatch_remote(peer_id, execute):
  ; 0. Local short-circuit — never route self through the table.
  if peer_id == local_peer_id:
    return dispatch_local(execute)

  ; 1. Active connection wins — most-live. For full-duplex transports this MAY
  ;    be an accepted (inbound) connection reused for outbound (§6.5.1b, impl-detail).
  connection = entity_tree.get("system/connection/" + peer_id)
  if connection is not null and connection.data.status == "active":
    return send(connection, execute)

  ; 2. Held capability → authenticate without re-handshake (§6.3, §6.6).
  session = entity_tree.get("system/peer/session/" + peer_id)
  if session is not null and session.data.held_capability is not null
     and not expired(session.data.expires_at):
    execute = sign_with(execute, session.data.held_capability)   ; zero-RTT eligible

  ; 3. Resolve the target's reachability class from its profiles (§6.5.1a D1:
  ;    sort (priority asc, profile-id lex)) and dispatch by class:
  profiles = resolve_profiles(peer_id)            ; ordered candidate list
  for p in profiles:
    switch reachability_class(p):
      full_duplex_listener, half_duplex_listener:
        r = connect(peer_id, p.endpoint); if r.ok: return send(r.connection, execute)
      held_connection_client:                     ; non-listening peer holding a duplex socket to us
        if held_socket(peer_id): return push(held_socket(peer_id), execute)
      pollable_client:                            ; non-listening, no held socket → poll-fallback
        ; R4 / delivery_mode poll path — Phase-2; not specified or gated in v1.
        ; Until R4 lands, fall through to queue (step 4).
        continue
      static_publisher:
        continue                                  ; consumer-fetch target, never a dispatch target

  ; 3b. No durable profile connected, but the target may still be reachable LIVE
  ;     via traversal (NAT hole punch / WebRTC). Consult the live-establishment
  ;     seam (§10.3). Null if none registered (no traversal extension installed)
  ;     ⇒ falls straight through to step 4, byte-identical to the pre-seam ladder.
  ;     This seam returns a CONNECTION, not a result: on success the ladder
  ;     re-enters ordinary dispatch, so the connection is pooled and reused by
  ;     every later dispatch (§6.5.1b). It is NOT a delivery fallback — see §10.3.
  ;     ctx carries the deadline: traversal takes seconds, unlike every other rung.
  conn = establish_live(ctx, peer_id)             ; §10.3 seam — null if unregistered
  if conn is not null:
    return send(conn, execute)

  ; 4. No live path at all. Consult the registered store-and-forward
  ;    fallback seam (§10.2) BEFORE the step-4 terminal. Null if none registered
  ;    (e.g. no RELAY installed) ⇒ byte-identical to the pre-seam terminal below.
  ;    Consulted HERE — at the dispatch caller holding BOTH peer_id and the execute
  ;    envelope — never inside connection-resolution (which holds only peer_id).
  fb = dispatch_fallback(peer_id, execute)        ; §10.2 seam — null if unregistered
  if fb is not null and fb.ok:
    return fb.result                              ; e.g. {status: queued-fallback, stored_at}

  ; 4t. Step-4 terminal (impl-variable). The §8 outbound queue is an OPTIONAL rung
  ;     an impl MAY interleave here iff it implements the §8 outbox; an impl with no
  ;     §8 outbox falls straight to the unreachable error. "Byte-identical to v1 when
  ;     the seam is unset" means: behaves exactly as THIS terminal does today,
  ;     whatever the terminal is in this impl.
  if entity_tree.get("system/peer/status/" + peer_id) is not null:
    return queue_pending(peer_id, execute)        ; §8 outbound queue — §8-conditional
  return error(502, "peer_unreachable")
```

**Reachability classes (the decision table):**

| Target class | Dispatch |
|---|---|
| full-duplex listener (`tcp`/`websocket`/`webrtc`) | reuse active connection (incl. accepted inbound — §6.5.1b); else dial + push |
| half-duplex listener (`http`) | dial its HTTP listener; push (each side runs a listener — §6.5.2c) |
| held-connection client (non-listening, holds a duplex socket to us) | push down its held socket |
| pollable client (non-listening, no held socket) | poll-fallback — source-hosted queue the target drains (R4 / `delivery_mode: poll`; **Phase-2**, not in v1; until then → queue, §8) |
| static publisher (`http-poll`) | n/a — a consumer-fetch target, not a dispatch target |

**Precedence — most-live wins.** When a peer matches multiple classes (e.g. it is both a full-duplex listener and pollable), the order is: active connection > listener-dial > held-socket > **store-and-forward fallback (§10.2)** > outbox-queue. The pseudocode encodes this; the table is the per-class action once a class is selected.

**Held-capability authentication (§6.6, §6.3).** Step 2 reads `system/peer/session/{peer_id}.held_capability` and signs the outbound EXECUTE with it, so the dispatcher skips the handshake when it already holds a valid cap. If the receiver rejects the cap (403), the sender falls back to a full handshake (§6.3). The cap lives in the tree (§6.6), so this works across reconnects and restart.

**Address resolution** (step 3 `resolve_profiles`) consults, in order:
1. `system/peer/alias/{name}` → `peer_id` (if a name was given)
2. `system/peer/transport/{peer_id}/*` → the profile set, sorted per §6.5.1a D1
3. Static configuration at `system/config/bootstrap`
4. *(post-corridor)* REGISTRY lookup — out of scope until the REGISTRY extension lands.

### 10.2 The dispatch-fallback seam (store-and-forward escalation)

The §10 ladder's step-4 terminal (`queue_pending`/`502`) only helps a peer that has, or will re-establish, a session with the target it drains into. It does **nothing** for the case that matters on the open internet: **delivering to a peer you have no live or recurring session with, that is offline or NAT'd right now.** NETWORK names a seam for that escalation; the store-and-forward *policy* lives in an extension (RELAY), never in NETWORK.

**Why a seam, not an inline branch.** NETWORK is **below** RELAY in the layering (RELAY is a network-peer-extension built on NETWORK); NETWORK §10 therefore cannot call into RELAY. This is the identical boundary already used for routing — the substrate exposes the slot, the extension plugs the algorithm (mirror of the relay→routing `resolve_next_hop` seam, `EXTENSION-ROUTE.md` §4). In the strictest impl this is not a preference but a compile-time constraint (an inline "if RELAY present" guard would invert the crate dependency graph).

**Signature.**

```
dispatch_fallback(peer_id, execute) → { ok: bool, result } | null
```

- Consulted **once**, at §10 step 4, after direct dispatch is exhausted and **before** the step-4 terminal (queue/502).
- Returns `null` when no fallback is registered (a peer without RELAY installed). `null` ⇒ the existing terminal runs unchanged — the **additive, no-regression** property. A peer needs no RELAY to *receive* (EXTENSION-RELAY.md §3.1.1) and symmetrically needs none to *dispatch directly*; it gains store-and-forward *send* only by installing RELAY.
- **Insertion-site discipline (normative MUST).** The seam is consulted at the dispatch caller that holds **both** the destination `peer_id` **and** the `execute` envelope it must store/forward — never inside the connection-resolution function (which holds only `peer_id`). All three reference impls converged on this independently; a reference impl that installs the seam inside connection-resolution would have to thread the envelope through and is non-conformant to this rule.

**v1 posture.** The seam is **v1.x**, additive. For non-RELAY peers the v1 floor is unchanged (queue/502, per each impl's terminal — see step 4t). For RELAY peers the new MUST is small: §10 consults the registered `dispatch_fallback` before the terminal; RELAY registers the inbox-relay-resolution behavior behind it (EXTENSION-RELAY.md §6.2.1). Conformance gates the **outcome** (offline-target delivery lands at the target's inbox; the target polls and verifies the sender signature exactly as on a direct delivery), not the rung choices inside the registered policy. No V7 change, no wire-format change, no new capability or error code (reuses `no_inbox_relay`/502, `capability_denied`/403).

**The forward-looking win.** Naming the seam gives the asynchronous floor a home: store-and-forward, *deliver later* — this seam's RELAY policy.

> **Corrected by Amendment 14.** This paragraph originally claimed the live-connection punch would escalate from **the same step-4 site**, giving "both deferred axes a single home." **That was written before the punch was designed and it does not hold.** Two reasons, and either alone is decisive: (a) `dispatch_fallback` returns a **delivered result**, while a punch returns a **connection** the ladder must then re-enter and reuse — forcing a punch through this signature means punching per message and hiding the connection from the pool; (b) the paragraph's own "a mature dispatcher tries live first and store-and-forward last" is unachievable from a seam consulted **once**, at a single site, after profile resolution. Live establishment is therefore its own seam at **step 3b (§10.3)**, ordered before this one. The two are complementary, not alternatives: §10.3 answers *can I reach this peer now*, §10.2 answers *how do I deliver when I cannot*.

### 10.3 The live-establishment seam (traversal escalation)

*Amendment 14.* §10 step 3 resolves **durable** transport profiles. A peer behind NAT has no dialable durable profile — its published endpoints are unreachable from outside — yet it may still be reachable **right now** by traversal (a NAT hole punch, or WebRTC on the browser leg). NETWORK names the seam for that escalation; the traversal *protocol* lives in an extension, never in NETWORK.

**Signature.**

```
establish_live(ctx, peer_id) → connection | null
```

- Consulted **once**, at **§10 step 3b** — after durable-profile resolution has failed to connect and **before** the §10.2 delivery fallback.
- **`ctx` carries a deadline and is cancellable (MUST).** A traversal attempt takes *seconds* — candidate gathering, a carrier round trip, a scheduled simultaneous open, retries — which is one to two orders of magnitude longer than any other rung of the §10 ladder. A seam with no cancellation makes an unreachable peer indistinguishable from a hung dispatcher, and it makes the whole ladder uninterruptible. Both native implementations raised this independently on first build.
- Returns `null` when no traversal policy is registered (a peer with no traversal extension installed) ⇒ the ladder falls straight through to step 4, **byte-identical to the pre-seam behavior**. This is the additive, no-regression property.
- **Returns a connection, not a result — and that is the whole reason it is a separate seam.** On success the ladder re-enters ordinary dispatch (`send(conn, execute)`), so the connection enters the pool and **every subsequent dispatch to that peer reuses it** (§6.5.1b). A seam that returned a delivered result would force a fresh traversal per message and leave the connection invisible to §10 step 1.

**Ordering is normative (MUST).** `establish_live` is consulted **before** `dispatch_fallback` (§10.2). A dispatcher that consults them in the other order, or that consults only one, will store-and-forward to a peer it could have reached directly — correct in outcome, wrong in cost, and it silently defeats the entire traversal path. *Live first, store-and-forward last.*

**What the seam guarantees, and what it does not.** NETWORK guarantees only that the returned connection is treated as an ordinary live transport. It does **not** specify how the connection was obtained — candidate exchange, simultaneous-open timing, carrier selection, and connectivity checks all belong to the registered policy.

**`connection` is deliberately undefined, and the handshake boundary is deliberately unpinned.** `connection` is whatever the host implementation's live-transport handle already is; this spec does not define it, and that is correct layering rather than under-specification — confirmed by two independent implementations that built the seam and reported the type fell out of their existing peer layer with nothing new invented. **The seam MAY return a connection either before or after the `system/protocol/connect` handshake has run on it** — the two reference builds split on exactly this point (one runs the handshake inside the registered policy and returns an established connection; the other returns a pre-handshake connection and runs the handshake in the shared adopt-path used by ordinary dials). **Both are conformant.** The seam is a boundary *inside* one peer, invisible across the wire, and forcing one factoring makes one implementation restructure for zero interop benefit. What is pinned instead is *which* peer runs the client half of the handshake — see obligation 3 — and that the identity check has happened before the connection is used.

**Five obligations on the returned connection (MUST).**

1. **It is an ordinary transport, with stream semantics.** A punched connection is a live full-duplex transport, **indistinguishable to the entity layer** from a dialed `tcp` connection. It slots into the §10 full-duplex-listener class and carries EXECUTE / TREE_GET / CONTENT_GET identically. It is **not** a new transport type, and it MUST NOT be published as a durable `system/peer/transport/*` profile — the mapping behind it is session-scoped (§6.7.3). **It MUST present reliable, ordered, stream semantics** — the registered policy owns reliability *below* the seam and the entity layer above it MUST NOT be asked to re-derive framing. A traversal that punches a raw datagram path therefore owes reliability and ordering before returning; handing an unordered datagram path up through this seam corrupts the length-prefixed framing every implementation layers on a stream. Not yet biting — both v1 substrates (`tcp`, and WebRTC data channels in reliable-ordered mode) are streams — but load-bearing the moment a QUIC or raw-UDP substrate lands.
2. **It MUST run keepalive (§5).** A punched NAT mapping expires after seconds-to-minutes of silence and the hole closes. An idle punched connection therefore **dies silently**, presenting as a "worked, then dropped" bug that a same-host test never reproduces. A peer MAY record a `punched: true` connection attribute purely to drive this discipline — an implementation detail, not an entity-layer type.

   > **A substrate that refreshes the mapping itself discharges this obligation — and the claim MUST say so.** The obligation is on the **property**: the mapping stays alive across idle. Where the substrate keeps it open on its own — a WebRTC data channel, whose ICE agent runs **RFC 7675 consent freshness** on the selected pair every few seconds — the mapping is refreshed whether or not the peer implements §5, and the obligation is met. **Two consequences, both MUST:**
   >
   > 1. **An implementation MUST know and declare which mechanism holds its mapping open.** One that is *silently* relying on the substrate has chosen nothing, and carries that assumption onto a substrate that supplies nothing.
   > 2. **A green survives-idle run on a self-refreshing substrate is NOT evidence for one that is not.** A native punch on the **same** `EXTENSION-SIGNALING.md` §11.5.1 row still owes §5 in full. Such a run may claim the §10.3 **outcome** ("carries ordinary operations and survives idle") on that substrate, and **MUST NOT** be recorded as closing this obligation generally.
   >
   > This is §11.5.1's substrate-scoping rule applied to an **obligation** rather than to a property, and it resolves the tension a reader will otherwise find between this MUST on the *mechanism* and the v1-posture paragraph below gating the *outcome*.
   >
   > **The WebRTC discharge above is reasoned, not measured, and is marked so deliberately.** Consent freshness is unconditional under RFC 7675, so the conclusion follows — but the first peer to answer consequence 1 reported that **consent freshness was not what held its mapping open**: an unrelated application poll was, at roughly two orders of magnitude more traffic, and the peer ran no §5 keepalive at all. **The discharge has therefore never been isolated**, and a rig cannot isolate it while any application traffic flows. **Read this as a substrate-discharge that is sound in principle and unexercised in fact**, and do not cite a green run on such a peer as having demonstrated it. *(This is consequence 1 doing its job on first contact, in the unanticipated direction: it was written expecting peers to discover the substrate was holding the mapping open, and the first answer was that it was not.)*
   >
   > *(A candidate rule for survives-idle gates generally — **measure the silence, do not assume it** — follows from the same finding and is **not folded here**: it belongs in `EXTENSION-SIGNALING.md` §11.5's gate text, and it rests on one implementation's evidence. It is tracked rather than landed.)*

3. **The peer identity MUST be verified before the connection is exposed to the ladder.** Whatever factoring an implementation chooses for the handshake boundary (above), the registered policy's connectivity check — `EXTENSION-SIGNALING.md` §7.4's nonce-plus-identity binding — MUST have completed successfully before this seam's return value enters the connection pool or carries an operation. A traversal returns a path to *an address*; only the check makes it a path to *the peer*. The freedom granted above is over **where** the verification runs, never over **whether** it has run by the time the ladder re-enters ordinary dispatch. The reverse ordering — pool first, verify later — is a valid-looking factoring that hands an attacker who forged a candidate address one dispatch of authenticated traffic.

4. **Exactly one retry authority (§4.1 composition).** When this seam is consulted from a `maintain-peer` reconnection continuation (§4.1), the registered policy MUST make **exactly one attempt at whatever step of its traversal contacts a third party** — for `EXTENSION-SIGNALING.md` §7 that is one coordination exchange (§7.2.1) — and §4.1's reconnect backoff owns re-scheduling. The policy's own budget (`EXTENSION-SIGNALING.md` §7.2 — up to 3 exchanges) applies **only** to a standalone consultation, i.e. one driven directly by a §10 dispatch. **This constrains only the third-party-facing step.** A traversal policy's internal retries against the *target peer's own* socket are local reliability and MUST NOT be reduced to satisfy this obligation — see `EXTENSION-SIGNALING.md` §7.2.1, where cutting the wrong counter broke a working punch. Two nested retry loops multiply: a reconnect backoff re-entering the ladder, each entry spending a 3-attempt punch budget, yields *reconnect-retries × 3* attempts against the shared carrier and reflector for a peer that is simply behind a symmetric NAT. That load lands on **third-party** infrastructure peers, which is why the composition is a MUST and not a tuning note — an implementation that nests the two is invisible to its own tests and visible to everyone else's provider. A peer MAY additionally record a local prefer-relay memo for a peer whose traversal has failed, to skip the seam on subsequent reconnects; like the observed address (§6.7.1) it is **session-scoped local state and MUST NOT be published** as a durable profile or written to `system/connection.address`.

5. **Single-flight establishment per peer (MUST; the general invariant obligation 4 instantiates).** At most **one** third-party-facing `establish_live` is in flight per target peer at a time. Concurrent or repeated triggers to the same peer — every §10-step-1 pool-miss, every dispatch that finds no live connection — MUST **coalesce onto that one in-flight establishment and await its result**, never each spawn a fresh negotiation. This is the **fan-in** face of the same invariant obligation 4 bounds on the **retry** face — *bounded third-party load per peer* — and like obligation 4 it is a MUST because the excess lands on **shared carrier/reflector** infrastructure, invisible to the offending peer's own tests and visible to every provider. Obligation 4 alone does **not** cover this: it bounds retries per attempt and explicitly grants each *standalone* §10 dispatch its own budget, so N concurrent dispatches are N individually-conformant establishments whose sum is exactly the multiplicative load the discipline exists to prevent. The failure it forecloses: N pool-misses each spawn a fresh `establish_live` → fresh negotiation (for WebRTC, a fresh `RTCPeerConnection` / `session_id` / offer) → N deposits pile into the rendezvous bucket, and a channel opens only when two happen to overlap — **brute force**, which also passes a volume-blind gate green (`EXTENSION-SIGNALING.md` §11.5, now asserting the bound). The **observable** requirement is the bounded third-party deposit count; single-flight coalescing at the connection pool is the mechanism, and *how* it coalesces (a per-peer `dialing` state, a shared future, a mutex) is impl-idiomatic. The correct shape is **deposit-once-and-poll-to-completion**: the rendezvous bucket is non-destructive and persists 60 s (`EXTENSION-SIGNALING.md` §5 pins 1 / 6), so one sustained negotiation polled to completion — not N thrown-away-and-re-deposited — is what the model intends. **A native TCP-simultaneous-open punch does not fan in this way**, so this gap is invisible until a WebRTC/poll path exercises it — `EXTENSION-SIGNALING.md` §11.5.1's loopback-blind class-adjacent, single-impl-invisible, and surfaced only at the frontier.

6. **A bound on consultation frequency (MUST; the sequential face of obligations 4 and 5).** A caller **MUST NOT** consult this seam for the same peer at an unbounded rate. After a small number of consecutive consultations that produce no pooled connection, it **MUST** space subsequent consultations for that peer by a non-decreasing interval up to a cap; **any connection to that peer reaching the pool resets the sequence.** Obligations 4 and 5 bound the **retry** and **fan-in** faces of *bounded third-party load per peer*; without this, the **sequential series is unbounded** — and every member of it is individually conformant, so **`EXTENSION-SIGNALING.md` §11.5 cannot see it: §11.5 bounds deposits per establishment, and the series is a count *of establishments*.** Measured on a live path, an application poll driving a pool-miss at 5 Hz produced **~5 offer deposits per second into one rendezvous bucket, indefinitely, per open conversation** — every deposit conformant, the sum a denial of service on shared infrastructure.

   > **Charge the consultation when it is *started*, not when it returns `[MUST]`.** A consultation abandoned by its caller returns nothing to charge, while **the negotiation it started has already reached the carrier.** A bound charged at the outcome therefore exempts precisely the attempts a loaded caller makes most of — measured at **57 negotiations where the same schedule permits ~34**, closing to **28** when the charge moved to the start.

   > **Rate zero is bounded — this obligation sets a ceiling, never a floor.** A caller that stops consulting entirely — via obligation 4's prefer-relay memo, or by falling to a lower rung of `EXTENSION-REGISTRY.md` §3b.4's ladder — satisfies this trivially. **The requirement that a cap stay small enough to notice a counterpart that becomes reachable binds only a caller that *keeps* consulting**: if you back off, do not back off past usefulness. **What this obligation forbids is the unbounded middle** — consulting forever at full rate. It does not reverse obligation 4's MAY.

   > **The stop condition is the caller's alone, and needs no seam change.** No new `NoPath` variant and no wire signal: separating *unreachable* from *not-yet* is required only to **abandon** a peer, and **a backoff never abandons** — it spaces, and a counterpart that becomes reachable still connects, at most one cooldown late. The conflation costs bounded latency, not a lost peer. **The cap's value, the growth curve, and the grace-window size are implementation-defined**; the observable requirement is bounded third-party consultation load per peer, exactly as obligation 5 leaves its coalescing mechanism idiomatic. Two callers with different caps still interoperate — the cost is externalized onto a provider, which is why this is a MUST and not a tuning note.

**Layering.** Identical to §10.2's: NETWORK is **below** the traversal extension and cannot call into it, so the substrate exposes the slot and the extension plugs the algorithm — the same boundary as the relay→routing `resolve_next_hop` seam (`EXTENSION-ROUTE.md` §4). In the strictest implementation this is a compile-time constraint, not a preference.

**v1 posture.** Additive and **v1.x**. For a peer with no traversal extension the v1 floor is unchanged. Conformance gates the **outcome** — two NAT'd peers that could not previously reach each other establish a direct live transport, and that transport carries ordinary operations and survives idle — not the rung choices inside the registered policy. No V7 change, no wire-format change, no new capability or error code.

**The registered policy for v1 is `EXTENSION-SIGNALING.md`** (§6), which registers the rendezvous-coordinated hole punch behind this seam.

---

## 11. Write Authorization

All network handler tree writes are to the handler's managed namespace (`system/network/*`). The handler's own grant authorizes all writes.

| Operation | Write target | Authorization | Capability recorded |
|-----------|-------------|---------------|-------------------|
| maintain-peer | `system/network/peers/{id}/*` | Handler grant | Handler grant |
| release | `system/network/peers/{id}/*` (cleanup) | Handler grant | Handler grant |

Network operations manage connection infrastructure. All writes are handler-authorized.

See ENTITY-CORE-PROTOCOL.md §6.8 for the general write authorization model.

---

## 12. Conformance

### 12.1 MUST Implement

- Handler at `system/network` with `maintain-peer`, `release-peer`, `status`, `close` operations (§3, §4)
- Keepalive ping/pong exchange (§5)
- **`suspect → disconnected` escalation survives connection eviction (§5.4a)** — the escalation is owed by the failure episode, not the connection; it fires whether the episode opened at an idle keepalive miss or at the Amendment 12 §A1 transport seam, and carries the episode's originating `reason`. Both halves are vectored (`NET-LIVENESS-ESCALATE-AFTER-EVICTION-1` + `NET-LIVENESS-NO-ESCALATION-WITHOUT-EPISODE-1`); the negative half is not optional, because escalating on *any* unbound peer rather than any `suspect` peer satisfies the positive vector and breaks the §A1 seam scope. **The negative half is wire-driven where an evict-without-demote path is installed and in-process with a declared exclusion otherwise — see §5.4a**; what is never acceptable is reporting an in-process result as a cross-impl vector pass.
- Peer status tracking via `system/peer/status/*` entities (v7.9 §3.13)
- Connection state tracking via `system/connection/*` entities (v7.9 §3.13)
- Graceful close with reason codes (§9)
- Pending delivery queueing when remote peer is disconnected (§8)
- Pending delivery drain on reconnection (§8.3)
- Session state entity at `system/peer/session/{peer_id}` carrying `held_capability`, persisted across `disconnected` (MUST NOT delete on disconnect) (§6.6, R6)
- Reachability-class outbound dispatch: resolve the target's profiles by `(priority asc, profile-id lex)` and dispatch by class; authenticate via `system/peer/session/{peer}.held_capability` when held (§10, R5)
- **Reachability facts, when §6.7 is implemented at all (Amendment 13)** — the section is OPTIONAL as a whole (§12.3), but every rule inside it is a MUST for a peer that offers it, because each is a cross-peer seam:
  - `observe-address` returns the **transport-layer source** of the request connection, never a body-supplied value (§6.7.1)
  - `observed_address` is **never persisted** to `system/connection.address`, a `system/peer/transport/*` profile, or any durable per-peer address field (§6.7.1)
  - `check-reachability` dials back **only** the requester's observed source address, rate-limited per requester, fixed-size payload (§6.7.2)
  - candidates are **never** written as durable `system/peer/transport/*` profiles (§6.7.3)
  - a published `srflx` candidate is the mapping of the **socket the peer punches from** (§6.7.3)
  - both operations are capability-gated (`network-reflect` / `network-dialback`) and rate-limited (§6.7.4)

#### 12.1.1 The liveness slice is the required floor and is independently conformant `[MUST]`

§4.1 `maintain-peer` bundles connect + subscription + reconnect-continuation + keepalive +
pending-drain into one operation. That is the **full** build. **This names the minimal composable
slice every consumer actually blocks on, so it can be shipped first and independently.**

> **The liveness slice.** Write `system/peer/status/{peer}` on connection state change — on establish
> (`connected`, §6.2), on transport error (`suspect`), on keepalive miss (`suspect → disconnected`,
> §5.4/§5.4a) — **and nothing more.** These are ordinary tree entities, therefore subscribable via
> `system/subscription`, therefore consumers react without polling. The slice requires **none** of
> `maintain-peer`, the continuation graph, or the outbox; those compose on top of it.

- **Keepalive is inside the floor.** The slice's three writes include the keepalive-miss demotion, so
  the floor is the status writes **plus** the §5 keepalive loop. §12.1 already makes keepalive a MUST;
  **the status writes alone are a build stage, not a claimable conformance point.**
- **Consumer latency contract.** A consumer of any NETWORK-conformant peer MAY rely on an idle-dead
  connection demoting within the keepalive envelope (`interval_ms × max_missed + timeout_ms`;
  defaults ≈ 100 s, values implementation-defined per §12.4).
- **`system/connection/{peer}` is NOT in the floor.** It remains MUST at full NETWORK conformance
  (above), written on transition per `ENTITY-CORE-PROTOCOL` §3.13. **The floor's observable contract
  is `system/peer/status` only, and a consumer of a floor-only peer MUST NOT assume the connection
  entity exists.** The division is deliberate: `status` answers *is the peer here* (subscribe);
  `connection` answers *how am I attached right now* (read-on-demand diagnostics).

**The §4.1 automation SHOULD be built on this slice rather than beside it.** This changes no
mechanism — it names a shipping boundary, and it exists because consumers were blocking on the whole
of §4.1 to get a signal the three status writes already provide.

### 12.2 SHOULD Implement

- Subscription restoration after reconnection (§7)
- Adaptive keepalive suppression during active exchange (§5.4)
- Exponential backoff for reconnection (§2.2)
- Self-authenticating zero-RTT reconnection via `held_capability` from the session entity (§6.3, §6.6)
- Pending delivery queue limits (§8.4)
- Pending delivery expiry (§8.3)

### 12.3 MAY Implement

- Linear or constant backoff strategies (§2.2)
- Custom keepalive intervals (§2.3)
- Subscription gap detection and reconciliation (§7.3)
- One-shot connection for unknown peers (§10)
- **Reachability facts (§6.7, Amendment 13)** — a peer MAY offer `observe-address`, `check-reachability`, both, or neither. Offering neither is fully conformant: a requester that gets a 403 or an unimplemented response proceeds to another reflector, exactly as it does when a reflector is unreachable. Whether a peer offers them is a deployment choice (a public peer running as a `reflector` service will; a NAT'd leaf peer has nothing useful to reflect). **What is not optional is getting them right when offered — see §12.1.**
- **An OPTIONAL section's types are owed only if the section is offered `[MUST; ruling 2026-08-07 — routed independently by `entity-core-go` and `entity-core-rust` the same day]`.** Stated generally, because this is not about §6.7: **a type is owed by the surface that uses it.** A peer that declines an OPTIONAL section installs none of that section's operations and therefore does **not** owe that section's types in its `--profile full` type publication — declining §6.7 declines `system/network/observe-address-result` (§6.7.1), `system/network/check-reachability-result` (§6.7.2), and `system/network/candidate` (§6.7.3) with it. This *follows* the core contract rather than departing from it: `ENTITY-CORE-PROTOCOL.md` §9.5 already reads *"publication is what matters; absence of extension handlers under §9.0 means absence of their types under the contract"* — this is the same rule one level down, at the section. The competing reading (types owed regardless, since publication is free and the registry is the machine-readable spec) is coherent, but it rests on §6.13(a)'s *publication is necessary but not sufficient*, which governs a **mandatory** handler where publication sits beneath a required behavior; §6.7 has no required behavior to sit beneath, and charging a peer a conformance penalty for exercising an option this section explicitly grants would make "offering neither is fully conformant" false. **Consumer side — the seam this protects:** a consumer MUST NOT infer support from the type registry. The discovery signal for an optional section is the **operation's response** (the 403-or-unimplemented → try another reflector path above), never the presence or absence of its types.

### 12.4 Implementation-Defined

- Keepalive default intervals
- Backoff default parameters
- Pending delivery queue size limits
- Subscription preservation timeout on `idle`/`error` close
- Address resolution strategy
- Reflection/dial-back rate-limit constants; how many reflectors a peer consults before trusting agreement; re-check cadence for a drifted mapping; where the accept-side path carrying the observed source sits in the implementation's layering (§6.7)
- Connection pooling
- Concurrent reconnection handling

---

## 13. Types Installed

> **Known deviation — op-type naming (logged 2026-07-31, layering audit; NOT fixed here).** ENTITY-CORE-PROTOCOL.md §3.7 makes the canonical op-type name a **MUST** — `{handler-path}/{op-name}-request` / `-result` — and names `network` among the extensions that inherit it. Five of this spec's type names predate that and deviate:
>
> | Op | Current | Canonical per core §3.7 |
> |---|---|---|
> | `maintain-peer` | `system/network/maintain-request` / `-result` | `…/maintain-peer-request` / `-result` |
> | `release-peer` | `system/network/release-request` / `-result` | `…/release-peer-request` / `-result` |
> | `status` | `system/network/status` | `…/status-result` |
>
> **Deliberately not renamed.** These are landed, 3-way-green, conformance-exercised type names; renaming them breaks every implementation's type registry and every vector that references them. That is a cohort decision with a migration, not a spec edit — and the alternative resolution (core grandfathers existing names, or relaxes the MUST to a SHOULD) is an `entity-core-protocol` change, upstream of here. **Routed, not decided.** The Amendment 13 types added in 2026-07-29 *do* follow §3.7 (`observe-address-result`, `check-reachability-result`) — the drift was not propagated.

| Type | Description |
|------|-------------|
| `system/peer/session` | Per-peer session auth record: held/minted capability + handshake bookkeeping (§6.6, R6) |
| `system/peer/transport/tcp` | Live TCP transport profile (§6.5.2a) |
| `system/peer/transport/websocket` | Live WebSocket transport profile — browser-capable (§6.5.2b) |
| `system/peer/transport/webrtc` | Live WebRTC data-channel transport profile — negotiated, no dial endpoint (§6.5.2d) |
| `system/peer/transport/http` | Live HTTP transport profile — EXECUTE over POST (§6.5.2c) |
| `system/peer/transport/http-poll` | Static HTTP transport profile — CDN/static hosting (§6.5.3) |
| `system/peer/transport/quic` | Live QUIC transport profile — aspirational (§6.5.2) |
| `system/network/maintain-request` | Input for maintain-peer operation |
| `system/network/maintain-result` | Output of maintain-peer operation |
| `system/network/release-request` | Input for release-peer operation |
| `system/network/release-result` | Output of release-peer operation |
| `system/network/status` | Network status summary |
| `system/network/peer-summary` | Per-peer status within summary |
| `system/network/close-request` | Input for close operation |
| `system/network/pending-delivery` | Queued outbound message |
| `system/network/backoff-config` | Reconnection backoff parameters |
| `system/network/keepalive-config` | Keepalive parameters |
| `system/network/ping` | Keepalive ping |
| `system/network/pong` | Keepalive pong |
| `system/network/observe-address-result` | Output of `observe-address` — the responder-observed source address (§6.7.1) |
| `system/network/check-reachability-result` | Output of `check-reachability` — dial-back outcome + address tested (§6.7.2) |
| `system/network/candidate` | A typed reachability candidate: `host`/`srflx`/`relay` (§6.7.3). **Ephemeral — carried in coordination messages, never written to the tree as a transport profile (§6.7.3 MUST).** |

---

## 14. Informative: Browser Peer Considerations

Browser peers (running in WASM environments) differ from native peers in several ways that affect network extension behavior:

| Aspect | Native Peer | Browser Peer |
|--------|-------------|--------------|
| **Inbound connections** | Accepts via listener (TCP, WebSocket) | Cannot accept — browser sandbox prevents listening |
| **Connection initiation** | Can connect to any peer via any transport | Can only connect to peers with WebSocket listeners |
| **Identity persistence** | Filesystem keypair | IndexedDB or localStorage; ephemeral if storage unavailable |
| **Background execution** | Continuous process | Page visibility affects timers; service workers possible but limited |
| **Reconnection** | Any available transport | WebSocket only |

**Initiation asymmetry.** Browser peers are always connection initiators. They cannot accept inbound connections. This means:

- Subscription delivery to a browser peer flows through the browser's outbound connection (which the browser initiated). If the browser disconnects, the native peer cannot reconnect — it queues pending deliveries (§8) and waits for the browser to reconnect.
- The `maintain-peer` operation (§4.1) runs on the browser peer. The browser is responsible for reconnection. The native peer's role is to accept the reconnection and resume the session (§6.2).

**Identity model.** Browser peers SHOULD use persistent keypairs stored in IndexedDB for session continuity across page reloads. Ephemeral keypairs (new identity per session) are acceptable but capability grants do not survive page reload. The peer identity model (ENTITY-CORE-PROTOCOL.md §3.5, §4) is unchanged — the keypair source is an implementation decision.

**Timer reliability.** Browser timers (setTimeout/setInterval) are throttled when the page is not visible. Keepalive intervals (§5) may be delayed in background tabs. Implementations SHOULD account for timer throttling when configuring keepalive `timeout_ms` and `max_missed` — more lenient values prevent false disconnect detection for browser peers. Native peers serving browser clients SHOULD tolerate delayed pong responses.
