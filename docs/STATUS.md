# entity-system-architecture — status

_Updated: 2026-08-24 · public: **v0.8.0** (`master`) · working: **0.8.2 cut in the headers, not
tagged**._

_**The published surface was four documents short and nobody had looked.** `CANONICAL-DOCS.toml`
is a **keep-list** — everything not declared in it is dropped at publish — and it had not gained a
document since **2026-06-30**. Two months of landed work was silently unpublished, including
`EXTENSION-SIGNALING` v1.1, a spec this repo's own CHANGELOG was already counting (26 claimed, 25
published; applications 4 claimed, 3 published). **All four now publish** by operator ruling; disk
and keep-list reconcile **76 = 76**. Surfaced by `entity-core-papers`' render report — see
`COHORT-OPEN-ITEMS.md` §0c and `ROUTING-2026-08-24-a`._

_**Also cleared: `egui` is gone from the specs and guides** — 48 lines, 15 files. It named a real,
unrelated third-party Rust GUI crate this project used briefly and fully removed, so a reader met a
false claim about what the system depends on. Now `entity-browser-rust`; the four places where it
genuinely means the library are untouched. And two published specs stopped citing repositories that
do not publish._

_**Their one ask was already built.** `entity-core-papers` reported that `tools/spec/` did not
survive the repo split and their topology pin could not be regenerated. It is
`entity-system-arch-tools/spec-tool/`, live and extended — **this team owns three repos, and the
tooling is the third.** `spec topology` is the command._

_**Release-blocking set is exactly two, and neither is arch's text.** (1) `entity-core-keystone`'s
`AGENTS-STANDARD.md` is canonical and cites a commit that resolves in no repo. (2)
`entity-browser-rust`'s release workflow pins `CORE_RUST_REF` to a `dev`-only ref on four checkout
legs — it must be the **tag** `v0.8.2`, and a push alone does not fix it. Both are routed. Everything
else found this cycle is hygiene, UX, or deliberately deferred._

_Arch gates: `check` **exit 0** · `ledger` **0** · `sdksync` **0 err** (47 unpinned, held as
backlog) · `pins` reader-mode. The spec corpus lives in `entity-core-protocol`; `spec corpus` is
not applicable in this tree and reports so rather than passing over an empty set._

_History: this log carried twelve dated entries and a 145-line July narrative. They are moved
**verbatim** to `docs/status/STATUS-2026-08-23-f-the-rolling-log-history-through-2026-08-15.md`,
which does not publish. See "Where we left off" for why._

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

**The release is a sequencing problem now, not a specification problem.** Nothing on the arch board
blocks a cut, and the two blockers that exist are build-surface defects in other seats' trees, both
routed 2026-08-23.

**What the last cycle established, in order:**

- **A build input is not only a dependency manifest.** The fear going in was that commit pins in
  build files would break when the release boundary re-authors history. Swept every manifest in the
  ecosystem: **no build file names a commit** — zero submodules, zero `git`/`rev` Cargo deps, zero
  `git+` lock entries, and all Go pseudo-versions are third-party. **The real coupling is layout:**
  `entity-browser-rust` carries 31 path deps on `../entity-core-rust/…` and `entity-workbench-go`
  `replace`s to `../../entity-core-go/…` in all 18 modules.
- **And then it was verified by building, not by resolving manifests.** From clean clones with
  siblings present, both coupled pairs build end to end — browser-rust produces its wasm bundle,
  workbench-go produces its five binaries. The documented sibling workflow is accurate; the open
  item is the error a user gets when they *skip* the sibling, which is UX.
- **Release order is a DAG, not a cycle.** `entity-core-rust` tags first; browser-rust's
  `CORE_RUST_REF` can carry `v0.8.2` **today**, before that tag exists, because a tag name is
  knowable in advance where a SHA is not. That is the whole reason the pin must be a tag.
- **CI ships basic and iterates on a branch.** `.github/` does not exist on browser-rust's public
  `master`, so tagging today triggers nothing. A minimal workflow has to land on `master` at release
  for `workflow_dispatch` to be dispatchable at all; the full six-leg matrix is iterated on a side
  branch and merged when green. Their existing dry-run gating makes that loop safe by construction.

**The rolling log itself moved this cycle, and that is a release item.** Per [ADR-0031] as corrected,
the dated snapshots under `docs/status/` are working memory and publish nothing, while the single
rolling log publishes — and it now lives at `docs/STATUS.md`, so the two kinds separate by **path**
rather than by remembering a filename. Arch's copy was the fleet's worst instance of the problem the
ADR names: **61 of 75 unique commit citations sat in accumulated history**, all `dev` SHAs that by
[ADR-0027] have never resolved for a public reader. Moving the history out is the fix; sweeping the
citations would have regenerated the problem the following week.

**The residual pin backlog is arch-owed, finite, and now honest.** What remains after the move is
concentrated in durable documents — `AGENTS.md` (the charter, newly canonical by operator ruling),
`GUIDE-CONFORMANCE`, `EXTENSION-NETWORK` — where a citation sweep is a real once-per-repo job.
`entity-core-protocol` has already completed its equivalent and sits at **0**, which is what proves
it finishable.

**The active design frontier is unchanged** and is the **network / resolution family** (Stage B in
`ROADMAP-EXTENSIONS.md`): NETWORK / SIGNALING / RELAY / ROUTE / SUBSTITUTE / REGISTRY / DISCOVERY /
ENCRYPTION are landed as specs and still converging across implementations, distinct from the frozen
v1 set. The unsolved root under all of it remains **consumer↔consumer connectivity**. Build state for
that family is peer-reported and dated — read the cohort's latest packets and
`docs/DOCTRINE-COHORT-STATE-TRACKING.md`, not this paragraph.

## Backlog

Richest area, organized by stage of the extension roadmap and its companion roadmaps.

### Network / resolution family convergence (release-critical, M2–M4, 🟡)

The peer-to-peer surface: transport → forwarding → name resolution → confidentiality. Phase
ordering is NETWORK transport → RELAY/ROUTE forwarding → REGISTRY/DISCOVERY resolution →
ENCRYPTION confidentiality. Per-extension state (track M-levels in `ROADMAP-EXTENSIONS.md`):

| extension | ver | maturity | open work |
|---|---|---|---|
| `EXTENSION-NETWORK` | 1.6 | M4 | transport family; v1 publish/relay gate 3-way green. Base wire framing lives in core. **§6.7 reachability facts (A13) + §10.3 live-establishment seam (A14) folded 07-29/07-31 — M2. A14 passed cohort review 07-31 (two independent builds) and fixed in place: `ctx`, stream semantics, identity-check-before-pooling, retry composition; return shape left impl-idiomatic. Seam built in go + rust; `observe-address` responder built in go; the srflx gatherer and `check-reachability` are unbuilt.** |
| `EXTENSION-SIGNALING` | 1.0 | **M3→M4** | **new 2026-07-31** — folds CONNECTION-NODE + SIGNALING-AND-PUNCH. Client role is the conformance surface; server role optional. **Carrier, key derivation, coordination and pool selection are built in all three** with a `signalingMeet` cross-impl check in Go's validator. **Cohort review folded in place 07-31, two rounds**: §7.4.1 handshake role (initiator = client — MUST, the top cross-impl hang-preventer), §7.2 retry composition, §7.3.1 substrate matrix + native↔browser model — then **§7.2.1**, after building the composition rule broke a working punch: "attempt" was unqualified and a punch has two nested retry layers (crossing = costs only the two peers; exchange = costs third-party reflector + carrier). MUST pinned to the exchange layer, MUST NOT against starving the crossing. **§7 punch built in go (tcp, in-process) + rust (rungs 0–3, punches — the §11.5 gate's 2nd impl), loopback only; §9 unwrapped surface unbuilt; §11.5's cross-NAT gate has not run anywhere.** **Dual-hole ruling 2026-08-01:** §7.1 step 4 corrected in place — both peers MUST fire outbound (a listen-only side opens no NAT hole); §7.4.1 "serves" ≠ TCP-passive. Routed `ROUTING-2026-08-01-…`. |
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

- **`entity-core-keystone`** — fix the dead citation in their canonical `AGENTS-STANDARD.md` by
  content rather than by a fresher commit. Release-blocking.
- **`entity-browser-rust` + `entity-core-rust`** — repin `CORE_RUST_REF` to the tag `v0.8.2`, and
  tag core-rust first. Release-blocking.
- **The implementation cohort, as always.** Converging network/resolution specs are not validated
  until exercised by a cross-impl (Go / Rust / Python) conformance run; prose review does not catch
  route/path/dedup defects. REGISTRY/DISCOVERY cohort impl and the ENCRYPTION end-to-end block are
  the open items.
- **The operator** — the tag point itself. No `v0.8.2` tag exists in any repo, and cutting one is
  not arch's call.

## Done recently

- **The 0.8.2 release sign-off (2026-08-23).** Corpus fold verified end to end by running the build
  owner's own gates rather than relaying a report; three findings no seat's report carried, all on
  `COHORT-OPEN-ITEMS.md`; the build surface swept, the isolation claim measured by building, and the
  CI plan routed.
- **`spec pins` built and the ecosystem measured** — the [ADR-0027] boundary means a `dev` SHA has
  never resolved for a public reader. 808 unreachable citations ecosystem-wide at first measurement,
  and `entity-core-protocol` at **0** by doing the sweep once.
- **The corpus is de-versioned** — a corpus is its name and its artifact's sha256, never a version
  stamp. `GUIDE-CONFORMANCE` §5.1 replaced and the process rules brought home.
- **0.8.1 amendment cycle (2026-07):** compute corpus locked three-way (330/330); continuation/bounds
  folded and validated three-way green; bucket-B ratified spec-side.
- **Released v0.8.0**, the initial public research-preview release.
- **v1 extension set locked** at 3-way convergence + conformance vectors (M5): Content, Type,
  Revision, Subscription, Continuation, Inbox, History, Query, Compute, Group, Identity,
  Attestation, Quorum, Clock, Role.
- **Network/resolution family landed as specs**, and `EXTENSION-SIGNALING` v1.0 folded the
  connectivity surface. **Crypto-agility** validated classically (Ed25519/Ed448 × SHA-256/SHA-384).

## Next

1. **Clear the two release blockers**, then cut in order: `entity-core-rust` `v0.8.2` → browser-rust
   `CORE_RUST_REF` → dispatch dry run → tag browser-rust.
2. **Sweep the residual commit citations** in the durable published documents (`AGENTS.md`,
   `GUIDE-CONFORMANCE`, `EXTENSION-NETWORK` and the smaller instances), anchoring on content digests
   or `master`-reachable refs. Once the backlog is down, run `spec pins --gate`.
3. **Continue network/resolution family convergence** — route REGISTRY/DISCOVERY/ENCRYPTION changes
   through a cohort conformance run before/at fold, and update `ROADMAP-EXTENSIONS.md` M-levels as
   they land.
4. **Pay down the `sdksync` unpinned backlog** (47 of 51 restated SDK blocks), re-reading each block
   against its source before pinning — a pin is a claim that someone looked.
5. **Run the header-label normalization pass** so published spec `Status:` headers match the
   maturity ladder.
