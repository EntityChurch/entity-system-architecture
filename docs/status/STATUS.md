# entity-system-architecture — status

_Updated: 2026-07-31 · public: v0.8.0 (master) · 0.8.1 amendment cycle: bucket-B ratified spec-side · STANDING-MODEL §3+§4 CLOSED (v1.21, nothing owed) · connectivity fold closed — see `HANDOFF-2026-07-31-connectivity-fold-closeout.md`_

## Where it is

This repo is the **conceptual architecture + specification** for the optional capability
layer **above** the Entity Core Protocol — everything that is *not* the irreducible protocol
floor. Concretely, the published surface is:

- **Extensions** — `specs/extensions/EXTENSION-*.md`, the capability library (26 landed specs, incl.
  `EXTENSION-SIGNALING` v1.0 — new 2026-07-31).
- **SDK conventions** — `specs/sdk/SDK-*.md`, the cross-impl binding layer over the type surface.
- **Applications (L5)** — `specs/applications/` content-format conventions (Embed, Semantic
  Content Site) so web / Godot / terminal front-ends render the same bytes.
- **Domains** — `specs/domains/DOMAIN-LOCAL-FILES.md`, the handler-domain pattern.
- **System model** — `specs/SYSTEM-ARCHITECTURE.md`, `specs/SYSTEM-COMPOSITION.md`,
  `specs/SYSTEM-IDENTITY-COMPOSITION.md`, `specs/ENTITY-SYSTEM-REFERENCE.md`,
  `specs/ARCHITECTURE-IDENTITY-INFRASTRUCTURE.md`.
- **Guides** — `guides/GUIDE-*.md`, developer how-to + discipline docs (incl.
  `guides/GUIDE-CONFORMANCE.md`, the conformance methodology + vector index, and
  `guides/GUIDE-EXTENSION-DEVELOPMENT.md`).
- **Authoring standards** — `specs/SPECIFICATION-FORMAT.md`, `specs/STYLE-NAMING-CONVENTIONS.md`
  (normative) — plus the three living roadmaps below.

The model is **"agree with us → you interoperate"**: each extension is an opt-in contract; the
core floor never requires any of them (except TREE's get/put, which lives in core). Divergence
is allowed; convergence is offered.

**Maturity model.** Every artifact sits on a cumulative ladder — **M0 Proposed → M1 Ratified →
M2 Landed → M3 Implemented → M4 Peer-converged (3-way Go/Rust/Python) → M5 Validated (conformance
vectors) → M6 Keystone-converged** — plus a trajectory flag (🟢 stable / 🟡 active / 🔴 volatile).
A spec's own `Status:` header is **not** a reliable maturity signal (many converged extensions
still read "Draft"); the M-level in `ROADMAP-EXTENSIONS.md` is authoritative.

**Convergence floor today.** The `--profile full` surface is 3-way converged across the Go, Rust,
and Python reference implementations at **0-FAIL** and exercised by the keystone cohort (15
generated peers all `--profile core` 0-FAIL). Maturity: **public research preview, v0.8.0** — the
v1 extension set is mature; the network/resolution family is still converging.

## Where we left off

The **0.8.1 amendment cycle** is landing. Three big spec obligations are discharged: the compute
corpus is locked three-way (330/330), the continuation/bounds fold is **validated three-way GREEN**,
and the core-window **bucket-B** (F40 / RT-6 / RT-13a-b / RT-14) is **ratified spec-side** — validated
by the `fceb61f` cohort census, `6/45` honest baseline (see
`docs/status/CLOSEOUT-2026-07-28-0.8.1-bucketB-ratify.md`). Spec-authoring is now largely discharged;
the center of gravity has moved to **cohort build capacity**. The continuation/network substrate is further
along than the 07-28 tracker claimed: a wire-pinned audit (2026-07-28, see `docs/DOCTRINE-COHORT-STATE-TRACKING.md`
for why it was needed) established the ground truth —

- **NETWORK reactive/liveness half — BUILT + green three-way** (go/rust/py), landed 2026-07-15…07-17, re-proven
  live 2026-07-28 (liveness 4/4, network 5/5, continuations 61/61, bounds 3/3). The 07-28 "unbuilt" claim was a
  **stale re-citation** of the 2026-07-14 proven-negative and is retracted. Residuals are pin-and-fold cleanups
  (§7.2 remote re-subscribe; STANDING-vs-one-shot backoff), not a build.
- **STANDING-MODEL §3 reactive authority — BUILT three-way** (keys on `reactive_trigger`). core-go landed the
  AT-1..AT-4 oracle vectors (`92b2c4f`) that *formalize* the already-green behavior; siblings run them → apply the
  staged fold edit. Cohort-routine, not an arch gate.
- **STANDING-MODEL §4 join completion — ✅ CLOSED 2026-07-29 (both sides).** All residuals (O4 drop-body, O5
  sweep-all reap, fire-partial) + the §4.2 finite-exhaustion MUST-delete built + green three-way and folded into
  EXTENSION-CONTINUATION v1.21 (§2.3/§3.4/§3.5/§3.5a). Final live three-way: Go `1e62eac` · Rust `aec13b1` · Python
  `a4eb929`, continuations 61/61 + bounds 3/3 byte-identical (in-proc Go ref / Rust 72/72 / Py 2938). **Nothing owed
  by any impl.** Non-goals (not pending): O6 quiescent-no-op abandon (traffic-driven reaping is by design); `round_id`
  = `tick.sequence` (only if a tick-driven W-COMPUTE realtime-join consumer is ever built). Packet:
  core-go `ROUTING-2026-07-29-cycle-close-arch-packet.md`.
- **§A2 ownership** was a declaration-home formality over fields already built and green — closed (core §3.13
  upstream); never a build gate.

Firm foundation comes from folding what is already green, not from a build that is largely done.

The active design frontier remains the **network / resolution family** (Stage B in
`ROADMAP-EXTENSIONS.md`): NETWORK / RELAY / ROUTE / SUBSTITUTE / REGISTRY / DISCOVERY / ENCRYPTION are
landed as specs but still converging across implementations, distinct from the frozen v1 set. The
consumer↔consumer connectivity stack (the connection node) is in active cohort motion — **Stage 1 is
built in Rust as of 2026-07-28** (core verbs, wrapped surface, key derivation, the §3 coordination
carrier; live over TCP, unmerged), with six wire-and-convergence shapes ruled and folded into the DRAFTs
and the go/py client packet issued
(`ROUTING-2026-07-28-signaling-rulings-and-go-py-packet.md`). Three arch-owned gates stood between Stage 1
and a v1 punch; **two are now closed and one remains.** **Gate 3** (`fire_at` derivation) — closed and built
(pinned §4.1, absorbed in Rust, §3's table corrected 2026-07-29 after it taught that impl the wrong shape).
**Gate 2** (the reflector supplying `srflx`) — **closed 2026-07-29 by ratifying and folding
`PROPOSAL-NETWORK-REACHABILITY-FACTS` into `EXTENSION-NETWORK` v1.5 Amendment 13, new §6.7.1–§6.7.5**:
`system/network:observe-address()` is landed spec, buildable in all three impls, with no core-protocol
dependency (mechanism (a)'s HELLO field is a **core-protocol** change and routes upstream to
`entity-core-protocol`, gating nothing here). **Gate 1** (the service lifecycle) — **closed 2026-07-29**: the
manifest field shape ruled (closed `kind`/`exposure` enums + free-form descriptor) and folded into
`SDK-OPERATIONS` v1.11 §11.6.9. Rust's correction that it was never a *functional* blocker stands — a
`#[tokio::main]` binary can own a listener directly — but bypassing it would forfeit the finding the exercise
exists to produce, so the ruling is **do not bypass**. `PROPOSAL-CONNECTION-NODE` §5.1 (the unwrapped wire
protocol: length-prefixed CBOR over TCP for the mailbox, RFC 5389 STUN over UDP for `reflect`) was specified
the same day, on the cohort's ask. **Arch owes nothing on the native punch path** — reaffirmed 2026-07-31, when the cohort's four consolidated asks
were all ruled and folded the same day (`ROUTING-2026-07-31-punch-rulings-to-cohort.md`). **Arch does owe the
browser leg:** `PROPOSAL-EXTENSION-WEBRTC-TRANSPORT` (DRAFT) is the "separate spec" §7.3 defers the `webrtc`
substrate to, and until it folds there is no direct native↔browser path — only relay, which §7.3.1 now pins as
the correct floor rather than a gap. The native critical path is in the **go and rust trees** (py has not started
the punch), and it is now blocked on **live infrastructure, not specification**: every punch build in existence
is loopback.

**The connectivity spec surface is now complete (2026-07-31).** `EXTENSION-SIGNALING.md` **v1.0** folds both
connectivity proposals into a new extension — the rendezvous carrier, the four-mode key derivation, the
coordination messages, the punch, and the unwrapped protocol — and `EXTENSION-NETWORK` **v1.6 Amendment 14**
adds the **§10.3 live-establishment seam** it registers behind. A14 also corrected §10.2's claim that the punch
would escalate from the same step-4 site: that seam returns a *delivered result*, where traversal must return a
*reusable connection*, so forcing the punch through it would punch a fresh hole per message. **Five surfaces are
**One arch process item is open; the other closed the same day.** `EXTENSION-NETWORK` **Amendment 14** (the §10.3
seam) was **folded before its proposal existed** — a normative change binding all three implementations, which
`AGENTS.md` requires be proposal-first. The design record was written after the fact
(`PROPOSAL-NETWORK-LIVE-ESTABLISHMENT-SEAM`) with the deviation stated. **Cohort review has now happened
(2026-07-31) and the seam survives** — two implementations built it independently and answered all four open
items, so the spec edit does not come out. It folds four additions in place, no rev bump: `ctx` on the signature,
a stream-semantics MUST, an **identity-check-before-pooling** MUST (which review surfaced and the proposal had not
anticipated), and a **retry-composition** MUST resolving how §7.2's punch budget composes with §4.1's reconnect
backoff. The return shape and the `connection` type stay **impl-idiomatic and deliberately unpinned** — the two
builds factor the handshake on opposite sides of the seam and both interop. *The deviation was real and the
review vindicated the design; both halves are worth remembering, since the next fold-before-proposal will feel
equally safe.* Separately,
`PROPOSAL-CORE-TYPE-EXTENSION-TIERING` (DRAFT, **not folded**) records the operator ruling that only durable,
broadly-applicable extensions may put optional fields on core types — signaling-class extensions compose through
seams instead. Its retroactive sweep found **four** such fields, not the three previously listed; all pass.

**SIGNALING v1.0 is largely a fold of already-built work**: the carrier, the key derivation, the coordination
messages and pool selection are implemented in **all three** implementations — rust `extensions/signaling` plus
the `entity-signaling-node` binary, go `ext/signaling` plus `cmd/signaling-meet`, py `entity_handlers/signaling` —
and Go's conformance validator carries a `signalingMeet` cross-impl check.

**Build state (peer-reported, observed 2026-07-31 — corrected twice this day after peer packets contradicted arch
claims; pins are go `204cc7c`).** Beyond the meet: **Go has built the §10.3 seam, the §7 `tcp` punch** end-to-end
in-process, **the §6.7.1 `observe-address` responder**, and — landed after the rulings — **the client-side srflx
gatherer** (`ext/signaling.DialReflector` + `cmd/internal/punchwire.SRFLXGatherer`, tested against a live loopback
responder, with the srflx proven to be the reflector-dial's own local bind — the §6.7.3 same-socket invariant)
and **rung 4**: the seam wired into `EnsureConnected`, so a §4.1 maintain-peer reconnect can punch, with a
session-local never-published prefer-relay memo. **Rust has built its `LiveEstablish` seam counterpart and punch
rungs 0–1.** Python has neither. **Go has now built everything arch left unblocked.** Still absent everywhere:
`check-reachability`, SDK §11.6.9's service declaration, and **SIGNALING §9's** unwrapped surface.

**Both punch builds are loopback / in-process — they prove the socket choreography, not NAT traversal.** The
remaining gate is `EXTENSION-SIGNALING` §11.5: two peers in different languages meeting at a derived key in all
four modes and holding a **direct punched transport** through idle. The meet half is exercised; the punch half
needs live infrastructure, not more specification.

> **Do not re-derive this paragraph from an arch-side name search.** Build state is peer-reported and dated
> (`docs/DOCTRINE-COHORT-STATE-TRACKING.md` D1/D2/D4). The prior version of it claimed the seam, the punch and
> `observe-address` were absent everywhere — asserted the morning of the day the peers reported all three. Read
> the peers' latest packets.

## Backlog

Richest area, organized by stage of the extension roadmap and its companion roadmaps.

### Network / resolution family convergence (release-critical, M2–M4, 🟡)

The peer-to-peer surface: transport → forwarding → name resolution → confidentiality. Phase
ordering is NETWORK transport → RELAY/ROUTE forwarding → REGISTRY/DISCOVERY resolution →
ENCRYPTION confidentiality. Per-extension state (track M-levels in `ROADMAP-EXTENSIONS.md`):

| extension | ver | maturity | open work |
|---|---|---|---|
| `EXTENSION-NETWORK` | 1.6 | M4 | transport family; v1 publish/relay gate 3-way green. Base wire framing lives in core. **§6.7 reachability facts (A13) + §10.3 live-establishment seam (A14) folded 07-29/07-31 — M2. A14 passed cohort review 07-31 (two independent builds) and fixed in place: `ctx`, stream semantics, identity-check-before-pooling, retry composition; return shape left impl-idiomatic. Seam built in go + rust; `observe-address` responder built in go; the srflx gatherer and `check-reachability` are unbuilt.** |
| `EXTENSION-SIGNALING` | 1.0 | **M3→M4** | **new 2026-07-31** — folds CONNECTION-NODE + SIGNALING-AND-PUNCH. Client role is the conformance surface; server role optional. **Carrier, key derivation, coordination and pool selection are built in all three** with a `signalingMeet` cross-impl check in Go's validator. **Cohort review folded in place 07-31, two rounds**: §7.4.1 handshake role (initiator = client — MUST, the top cross-impl hang-preventer), §7.2 retry composition, §7.3.1 substrate matrix + native↔browser model — then **§7.2.1**, after building the composition rule broke a working punch: "attempt" was unqualified and a punch has two nested retry layers (crossing = costs only the two peers; exchange = costs third-party reflector + carrier). MUST pinned to the exchange layer, MUST NOT against starving the crossing. **§7 punch built in go (tcp, in-process) + rust (rungs 0–1), loopback only; §9 unwrapped surface unbuilt; §11.5's cross-NAT gate has not run anywhere.** |
| `EXTENSION-RELAY` | 1.2 | M4 | dispatch-fallback seam folded; raw-frame impl gaps tracked in the cohort. |
| `EXTENSION-ROUTE` | 1.0 | M3 | source-routed multi-hop; one impl build-tested, cohort catching up. |
| `EXTENSION-SUBSTITUTE` | 1.0 | M4 | CDN release v1 (Tier-1), the storage-substrate mechanism. |
| `EXTENSION-REGISTRY` | 1.2 | M2→M3 | substrate + local-name (petname, §6) + peer-issued resolve landed; **v1 NOT complete** (see forward-feature list). |
| `EXTENSION-DISCOVERY` | 1.0 | M2→M3 | mDNS peer-finding (`_entity-core._udp.local.`); impl-ready, cohort impl in flight. |
| `EXTENSION-ENCRYPTION` | 1.0 | M2→M3 | self/peer/group; byte-pin layer green 3-way; **end-to-end validation block still gating v1.0**. |

The unsolved root under this whole family is **consumer↔consumer connectivity** (browser/phone ↔
desktop, both behind NAT). The cheapest proof point is the browser WebRTC transfer demo —
off the critical path, but the demonstration target.

### Forward-feature backlog of LANDED extensions (named, unbuilt; NOT shipping v1)

The "Landed" label hides incomplete sub-features one level down. Tracked so "Landed" is not
misread as "complete":

- **`EXTENSION-REGISTRY` (v1 not finished):** live registration (`open`/`allowlist`/`manual`,
  design folded, build in flight); signed binding-manifest impl (format locked, impl deferred);
  `domain-control` DNS-challenge format (not yet designed); additional backends (did-web /
  dns-txt / dht / consensus-anchored — each its own future proposal); aggregator federation +
  outbound DID/DNS bridge (v1-deferred).
- **`EXTENSION-ENCRYPTION` (base v1.0 landed; close-out gated):** end-to-end validation
  (relay-encrypted send, group re-key, rotation/revocation, storage round-trip, cross-tier
  interop, key separation, sender auth) is dispatched and gating v1.0; sealed-sender / padding /
  hybrid-PQ / Shamir Tier-3 designed but not implemented; an encrypted-session sibling
  (Signal/Noise/MLS-style) is deferred to after ENCRYPTION v1.0 closes.

### Early / parked extensions (M0–M2, 🔴)

`EXTENSION-TRANSACTION` (v0.1, initial design, pre-review) and `EXTENSION-DURABILITY` (v0.1,
exploratory, optional, not active — and explicitly *not* a normative SDK surface).

### Proposed, not landed (design intent only)

Real, load-bearing-but-forthcoming directions cited in landed specs as `(planned)`: HTTP/SMTP
bridges, gossip, WebRTC transport, NAT traversal, static peer-manifest handshake, a name grammar,
universal resolution, a GROUP v1.5 sweep, static-peer-hosting, domain-local IO, and the
applications domain. These are intent, not contract — implementations build against landed specs,
never intent.

### SDK binding layer (`ROADMAP-SDK.md`, M1–M2 working drafts)

`SDK-OPERATIONS` (1.10), `SDK-EXTENSION-OPERATIONS` (0.9), `SDK-IDENTITY-INFRASTRUCTURE` (0.5).
These are *conventions*, not an API mandate — "your language, your idioms, but the boundary bytes
and operation semantics agree." Discipline: the SDK surface **follows** an extension landing, never
leads it. Core SDK surface is done for v1; per-extension alignment tracks each landing; the
identity-stack SDK is converging. **Security non-goal pinned:** identity bundles MUST NOT carry
private key material. The `browse_*` seam (the thin surface SITE/SPACES/REPOS share) is forward
work pending L5 validation.

### L5 applications (`ROADMAP-APPLICATIONS.md`, M0–M2 — mostly plan, not spec)

Be precise about plan-vs-spec: **SITE read = shipping at preview**; the Embed + Semantic Content
Site format spine is a converged paper (locked three ways across the reference front-ends); **Forms
are parked** (paper-frozen, not in spec, open post-release); **SPACES / REPOS are future and
undesigned** (named directions only, zero design today). The preview SITE demo is a read path over
published content with honest "unverified" UI until the registry verification path lands
cohort-wide.

### Substrate hardening (resilience floor)

The non-functional substrate program landed its floor into core and is exercised by the conformance
suite; this layer keeps the "stable / secure / always-there / doesn't-crash" properties honest:
store-safety under concurrent dispatch, graceful degradation (**deliver-or-signal, never silently
drop**), and resource bounds (max payload `413`, max chain depth `400`, connection bound). The
contract is the *outcome* (coded rejection + keep-serving), not specific limit values. A
concurrency conformance gate exercises demux / reentry / sustained-load / connection-churn.

### Crypto-agility

Classical agility is validated end-to-end on both axes — key types Ed25519 + Ed448, hash formats
SHA-256 + SHA-384 — with reserved code points for further algorithms. Post-quantum candidates are
allocated and library-confirmed but cross-impl integration is **deferred post-release**. Standard
compliance baseline is Ed25519 + SHA-256 (the largest shared address space); divergence is
permitted but priced (restricted reach + no cross-format dedup).

### Authoring

- Keep the **load-bearing invariants** and recurring-failure catalogs current in their canonical
  homes as the network family lands (the five invariants live across the core model docs,
  `specs/extensions/EXTENSION-TREE.md` §1, `specs/extensions/EXTENSION-NETWORK.md`, and
  `guides/GUIDE-EXTENSION-DEVELOPMENT.md`).
- Header-label normalization: re-stamp spec `Status:` lines to a controlled vocabulary aligned to
  the maturity ladder so each published artifact's header tells the truth (tracked cleanup, not
  release-blocking).
- Promote ROLE v2.0 from M4→M5 on its pending root-cap convergence round.

## Waiting on

- **Implementation cohort.** Converging network/resolution specs are not validated until exercised
  by a cross-impl (Go / Rust / Python) conformance run; prose review alone does not catch
  route/path/dedup defects (the load-bearing meta-rule). The only external handoff is to that
  cohort for cross-impl review + build. REGISTRY/DISCOVERY cohort impl and the ENCRYPTION
  end-to-end block are the open items.

## Done recently

- **0.8.1 amendment cycle (2026-07):** compute corpus locked three-way (330/330; §11 alt-engine + §4.2
  budget ceiling `IMPLEMENTED`); continuation/bounds folded + validated three-way GREEN; **bucket-B
  ratified spec-side** via the `fceb61f` census (F40 control-row differential, RT-6 six-class ladder,
  RT-13b two-part atomicity, RT-14). Durable methodology gain: `GUIDE-CONFORMANCE §3.1(7)` — dual-anchored
  verdicts, no proxy-green.
- **Released v0.8.0**, the initial public research-preview release.
- Established the spec surface layout (`specs/` core model + extensions + sdk + applications +
  domains, `guides/`, the normative authoring standards, the three living roadmaps).
- **v1 extension set locked** at 3-way convergence + conformance vectors (M5): Content, Type,
  Revision, Subscription, Continuation, Inbox, History, Query, Compute, Group, Identity,
  Attestation, Quorum, Clock, Role.
- **Network/resolution family landed as specs:** REGISTRY consolidated to one spec (petname is §6),
  DISCOVERY landed as its sibling, both dispatched to the cohort; ENCRYPTION base v1.0 landed with
  the byte-pin layer green 3-way.
- **Substrate resilience floor** (store-safety / graceful-degradation / resource-bounds) ratified
  and gated; the concurrency conformance gate landed 3-way green.
- **Crypto-agility** validated classically (Ed25519/Ed448 × SHA-256/SHA-384).

## Next

1. **STANDING-MODEL §3 reactive authority — ✅ FOLDED 2026-07-28.** AT-1..AT-4 converged three-way (core-go
   `b98e08b`); the §3.1/§3.1b/§6.1 edit is applied to `EXTENSION-CONTINUATION.md`; proposal §3 IMPLEMENTED. Done.
1b. **NETWORK liveness buildout** — **BUILT + green three-way** (go/rust/py; landed 07-15…07-17, re-proven live
   07-28). *Not* a build — the fold cleanups are §7.2 remote re-subscribe (unbuilt all three) and the
   STANDING-vs-one-shot backoff pin, plus the routed §3.13/§3.11 core-text delta at fold. **§A2 ownership CLOSED**
   (core §3.13 upstream; was a formality over already-green fields, never a build gate). §A6.1/LOST-ERROR Delta-1
   coupling is one-way (folding LIVENESS *unblocks* Delta 1).
1c. **STANDING-MODEL §4 join completion — ✅ FOLDED 2026-07-29.** The three residuals built + green three-way
   (Go `1ccdd51` reference; Rust `aec13b1` O5 sweep-all; Python `1590d8a` O4 `slot` + fire-partial + O5); live
   re-run clean (continuations 61/61, bounds 3/3; in-process rust 72/72, py 2936+8). Folded to EXTENSION-CONTINUATION
   §2.3 (join fields) + §3.5 (round-identity guard + round_id increment) + new §3.5a (deadline / abandon /
   fire-partial / **sweep-all**-on-touch no-timer / round identity / delivered-error / marker+drop-body). Reap and
   fire-partial are peer-local (no wire category) — convergence is source-reconcile + each in-process suite + a
   no-wire-regression run; round_id guard + O4 drop-body are the wire-exercised cross-peer surface. **Fold audit
   (source-read all three):** marker reasons pinned to source — `join_incomplete`/`join_late`/`join_error_slot`
   (`stale_round` is the drop-body value only; first draft conflated the two, corrected); core-go's routed
   drop-response-shape pin discharged. **One open §4 sub-item:** `round_id`=`tick.sequence` (§4.1 tick-driven source)
   is **UNBUILT three-way** (all self-increment a private counter) → folded as design-forward, **deferred to the
   tick-driven W-COMPUTE realtime join consumer**, validated cross-impl when it lands. **O6** (quiescent-no-op
   abandon) deferred — no impl does it. (O5 took three passes — two ruled on summaries not source; the D7 lesson, on
   the record twice.) **Finite-join exhaustion divergence — RULED + FOLDED 2026-07-29 (§4.2):** it *was* a spec
   issue — the landed `SHOULD clean up` (§3.4/§3.5) let two readings diverge cross-peer (consumed finite continuation
   → absent vs `remaining_executions: 0` husk). Source-read: Go + Rust **delete** at exhaustion, Python is split
   (deletes on fire-partial, retains on ordinary). **Ruled MUST-delete** (converge on Go+Rust, spec's existing lean;
   one rule for forward + join), folded to EXTENSION-CONTINUATION §3.4/§3.5. **✅ Python built it** (`a4eb929`,
   ordinary path → shared delete-if-last helper, +2 tests) — **§4 now CLOSED, nothing owed by any impl.**
2. Continue **network/resolution family convergence** — route REGISTRY/DISCOVERY/ENCRYPTION changes
   through a cohort conformance run before/at fold, and update `ROADMAP-EXTENSIONS.md` M-levels as
   they land.
3. **Cohort remediation tail** for 0.8.1 bucket-B (P0 = the 7 RT-6 replay-accepted peers; P1 = status
   migrations + F40 forth/smalltalk + 18 RT-13b Part-B attestations) — cohort work, tracked not owed.
4. Run the **header-label normalization** pass so published spec `Status:` headers match the
   maturity ladder.
