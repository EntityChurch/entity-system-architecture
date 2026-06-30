# entity-system-architecture — status

_Updated: 2026-06-30 · public: v0.8.0 (master)_

## Where it is

This repo is the **conceptual architecture + specification** for the optional capability
layer **above** the Entity Core Protocol — everything that is *not* the irreducible protocol
floor. Concretely, the published surface is:

- **Extensions** — `specs/extensions/EXTENSION-*.md`, the capability library (23 landed specs).
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

Stable at the v0.8.0 public research-preview line; no code or spec changes are in flight.
The active design frontier — and the next substantive work — is the **network / resolution
family** (Stage B in `ROADMAP-EXTENSIONS.md`): NETWORK / RELAY / ROUTE / SUBSTITUTE /
REGISTRY / DISCOVERY / ENCRYPTION are landed as specs but still converging across
implementations, distinct from the frozen v1 set.

## Backlog

Richest area, organized by stage of the extension roadmap and its companion roadmaps.

### Network / resolution family convergence (release-critical, M2–M4, 🟡)

The peer-to-peer surface: transport → forwarding → name resolution → confidentiality. Phase
ordering is NETWORK transport → RELAY/ROUTE forwarding → REGISTRY/DISCOVERY resolution →
ENCRYPTION confidentiality. Per-extension state (track M-levels in `ROADMAP-EXTENSIONS.md`):

| extension | ver | maturity | open work |
|---|---|---|---|
| `EXTENSION-NETWORK` | 1.4 | M4 | transport family; v1 publish/relay gate 3-way green. Base wire framing lives in core. |
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

1. Continue **network/resolution family convergence** — route REGISTRY/DISCOVERY/ENCRYPTION changes
   through a cohort conformance run before/at fold, and update `ROADMAP-EXTENSIONS.md` M-levels as
   they land.
2. Run the **header-label normalization** pass so published spec `Status:` headers match the
   maturity ladder.
