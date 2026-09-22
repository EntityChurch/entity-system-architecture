# entity-system-architecture — status

_Updated: 2026-09-20 · public: **v0.8.0**, the only tag on `master` · the release number for the
next cut is not set here and is not this team's to set._

_**A tag is a release; a push is not ([ADR-0015]) — and that distinction is the whole state of this
cut.** The 0.8.2 content reached public `master` on 2026-08-24 and **no corresponding tag was ever
created**, here or anywhere in the ecosystem. So that content is published-as-content and
unreleased-as-a-version, against a tree that has since moved 349 commits — **91 of the 217
published paths differ**, measured 2026-09-20 by diffing the tree against public `master` rather
than by reading the commit log. The CHANGELOG's `[Unreleased]` section carries that delta and the
written breaking verdict; it read "Nothing yet." until this was measured._

_**The public surface is declared, as of 2026-09-20, and it was not before** — `specs/` and
`guides/`; everything under `docs/` is out, including the proposal and exploration corpus, which
publishes as the design record but whose paths are not promised. `AGENTS.md` carries the line and
the CHANGELOG restates it for a reader. Until it existed, every argument about what a release of
this repo breaks was an opinion._

_**The release is in motion and the specification is described as *current*, never *final*.** The
core protocol is at **`0.8.2.32`** and it will not be the last: revisions are still landing against
text days old, and that is the corpus working rather than the corpus being unready. What a reader
gets at the cut is a **current, reproducible state with its conformance anchored on content
digests** — not a frozen one. Nothing in this log should be read as claiming the design has stopped
moving._

_**This repo's number follows `entity-core-protocol` rather than the implementations** (ruled
2026-08-24; the 2026-08-24 cut went out as 0.8.2 on that basis). The specs are not semantically
versioned as a set — each spec's own header is authoritative for that document — so the repo-level
number tracks the core protocol this corpus layers on, and not any implementation's line. Arch had
never carried a release number before that; the `v0.8.0` tag on this repo was an ecosystem-wide
genesis tag, not an arch release line. **The number for the next cut is settled at the cut and is
not stated in this log** — what this log owes is the measurement and the breaking verdict, both of
which are above._

_**The proposal and exploration corpora publish.** The mechanism is `[[keep_tree]]` in
`CANONICAL-DOCS.toml` — a repo can declare a **directory** as product rather than dev history —
built by devops on arch's ask after the per-file route (117 hand-written entries) was rejected as
not being a mechanism. `ABSORPTION-*` reviews and the dated status snapshots stay internal.
**The corpus grows every week, so this paragraph no longer carries the counts** — it stated
"82 proposals, 35 explorations" for long enough to be wrong by a quarter. Run
`spec register` for the design-document total and `spec ledger` for the per-directory counts, both
of which are gated against the directories they name._

_**Why they publish, and it is structural rather than a preference:** this repo's central authoring
rule forbids rationale in spec text and sends it to the proposal. Shipping the specs without the
proposals would mean stripping derivation out of normative text and then not shipping where it
went — MUSTs with no reachable reason._

_**The published surface needed two different fixes and I graded them backwards the first time.**
`CANONICAL-DOCS.toml` is a **keep-list**, but it governs only `docs/` and loose root documents —
`specs/` and `guides/` are always kept, declared or not (`canon/canon.go:10`). So the four
undeclared specs and guides were **never at risk** — they survive the filter today as undeclared
files, measured — while the five undeclared **root** documents were the ones a cut would delete.
I had both in front of me in one sweep and called the harmless set the emergency. The keep-list
had genuinely gained no document since **2026-06-30**, and the CHANGELOG had been contradicting it
in print (26 extensions claimed, 25 declared; applications 4 and 3) — so all four are declared now
and disk reconciles **76 = 76**. Surfaced by the paper team's render report, severity
corrected by meta. See `COHORT-OPEN-ITEMS.md` §0c and `ROUTING-2026-08-24-a`._

_**Also cleared: `egui` is gone from the specs and guides** — 48 lines, 15 files. It named a real,
unrelated third-party Rust GUI crate this project used briefly and fully removed, so a reader met a
false claim about what the system depends on. Now `entity-browser-rust`; the four places where it
genuinely means the library are untouched. And two published specs stopped citing repositories that
do not publish._

_**Their one ask was already built.** The paper team reported that `tools/spec/` did not
survive the repo split and their topology pin could not be regenerated. It is
`entity-system-arch-tools/spec-tool/`, live and extended — **this team owns three repos, and the
tooling is the third.** `spec topology` is the command._

_**Arch is next in the release order** (operator, 2026-08-24). The one mechanical blocker meta
routed — K1, five public root documents declared nowhere, which a cut would have deleted — **is
fixed and verified by running the publication filter**, not by reading the config. `SECURITY.md`,
`CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `CHANGELOG.md` and `CLAUDE.md` all dropped at `3e1f3b6`
and all survive now._

_**A second mechanical blocker was found on 2026-09-19 and is fixed, and it was found the same
way — by running the filter rather than reading the config.** Exporting the tree, running the real
publication filter over it and diffing the survivors against what is public today returns **22
files that are public and would vanish**, none of them declared. The gate that blocks on exactly
this is fail-closed and would have stopped the cut. **Four are genuine withdrawals** — the
applications charter (folded into the application-development guide), the retired condensed
reference, the domains directory (its one spec moved to `specs/bridge-extensions/`), and the
discipline charter (deliberately made internal on 2026-09-08). **Eighteen are proposals that folded
and moved from `active/` to `implemented/`**, verified present at the new path, 18 of 18. All 22
are now declared with their reasons. **The lesson is the one K1 already taught and it did not
transfer: the declaration file is not evidence about the published tree — only running the filter
is.** Re-run that diff at every cut; it is one command and it took under a minute._

_**The two items this log carried as release-blocking are both stale, and neither was arch's.**
keystone's dead `AGENTS-STANDARD.md` pin is **CLOSED** — they shipped 0.8.2 on 08-24 and `e8524ed`
appears nowhere in their public tree; arch's finding is why it was caught pre-cut. The
`CORE_RUST_REF` pin is **still live and is now meta's B1**: the fix arch asked for was applied, the
value is a tag name — and `entity-core-rust` shipped as **0.9.0** and was never tagged, so the
defect survived its own fix with a new value. It does not gate arch._

_**Re-measured 2026-09-19 across the ecosystem's live checkouts — and the headline is a negative.**
**`v0.8.0` is the only release tag anywhere in it.** Every repository's development branch is a
long way past what it has published: browser-rust **+522** · keystone **+346** · arch **+349** ·
workbench-go **+185** · go **+140** · py **+138** · formalization **+96** · rust **+69** ·
protocol **+66** · arch-tools **+65**. Nothing cut in this release cycle has been tagged in any
repo, which is meta's **B1** and is not arch's to close. Two seats — the generator and the
conformance instrument — have no public remote at all and are not part of this cut._

_**What that changes about reading the numbers below:** a "published version" in this log means the
content that reached public `master`, not a tag a reader can fetch. Where those two differ, the
tag is the one that does not exist yet._

_**How this corpus is kept consistent, and what that does and does not tell you.** The
specifications are checked mechanically on every change — for authoring-standard conformance, for
citations that resolve to a section that exists, for restated rules that have drifted from the
document they name as their authority, for version numbers copied into a roadmap that no longer
match the spec header, and for each spec's declared dependencies. Re-measured for this cut, every
one of those is clean. **Two things they deliberately do not tell you.** A citation resolving is not
a citation being *right* — a reference to a real section that is the wrong real section passes every
check here. And none of it is evidence about an implementation: this repo specifies, and whether a
peer does what a spec says is settled by the conformance suites in the implementation
repositories, never from here._

_**Where the corpus carries acknowledged debt, stated rather than smoothed over:** the
requirement inventories that make an individual conformance obligation citable by a stable id exist
in **2 of 28** specs — the rest state their obligations in prose, which is checkable by a reader and
not by a machine. `EXTENSION-ROLE` has no conformance section at all. These are held as a floor
that only rises, so the debt cannot grow, and paying it down is authoring work rather than
formatting._

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
- **Bridge extensions** — `specs/bridge-extensions/`, extensions that reach out onto a foreign
  technology rather than extending the entity system itself. One member today
  (`DOMAIN-LOCAL-FILES.md`, the host filesystem); the family discipline is
  `guides/GUIDE-BRIDGE-EXTENSION-DEVELOPMENT.md`.
- **System model** — `specs/SYSTEM-ARCHITECTURE.md`, `specs/SYSTEM-COMPOSITION.md`,
  `specs/SYSTEM-IDENTITY-COMPOSITION.md`, `specs/ARCHITECTURE-IDENTITY-INFRASTRUCTURE.md`.
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
and Python reference implementations at **0-FAIL**, and the generated cohort stands at **46 peers,
46 measured, all at one content-pinned check set, 46 of 46 at 0-FAIL**. Maturity: **public research
preview, v0.8.0** — the v1 extension set is mature; the network/resolution family is still
converging.

> ⚠ **Read that cohort number for exactly what it claims.** Those peers share a generation lineage
> and pass one author's vectors at one pinned check set: **cohort-consistent, not independent
> convergence.** They are also measured against a **pinned specification snapshot that is behind the
> current core revision**, deliberately and with the gap tracked in the cohort's own published
> matrix — the check set is what gates the wire, and a snapshot reaching a peer is a regeneration
> question on its own cadence. **A green row is evidence about the wire, not a proof about the
> peer.**

## Where we left off

**Nothing on the arch board blocks a cut, and that has been true for a while — what is still open is
VALIDATION, which is not a specification problem and is not arch's to close.** The components of the
release exist; no cross-seat exercise of the whole chain has been run. **A table of finished
components is not a readiness claim**, and this log has previously read like one.

**The core protocol arc, `0.8.2.4` → `0.8.2.32`.** Twenty-nine revisions since the 0.8.2 cut, and
the shape of the work is consistent enough to name: **almost none of it invented a rule.** It found
the same rule stated in several places at several strengths, ruled which home is normative, and made
the others say so. The recurring findings, in the order they cost the most:

- **A rule that EXECUTES in a pseudocode block, while the prose table enumerating its class omits
  it.** A reader consults the *right* home and it answers *wrongly* — which produces a confident
  false absence rather than a visible gap. Landed at `0.8.2.28`, and its mirror (the homes dropping a
  rule's bound rather than the rule) at `0.8.2.29`.
- **A conformance-floor row that restates a rule and then does not follow it.** `0.8.2.31` found
  §9.1 still publishing, positively, the authority discriminator `0.8.2.22` had corrected — for eight
  revisions, in the MUST-implement list an implementer builds from. §9.1 now carries a standing
  `[MUST]` that **a restating row names its normative home**, and `0.8.2.32` swept the other 26 rows
  onto it. **A floor row is a bullet in a list, not a paragraph arguing a rule**, so neither a search
  for the words nor a search by subject reaches it.
- **A code slot that is divergent while a token-scoped sweep reports it closed** (`0.8.2.6`
  through `0.8.2.9`). The unit is the **slot**, never a spelling.
- **A scope sentinel that is fail-closed in one position and fail-OPEN in the other**
  (`0.8.2.21`), and **a shared parameter whose meaning changed under readers listed nowhere**
  (`0.8.2.19`). Both were live over-acceptances in shipped code, and both were found by
  implementations building the previous revision.

**The changelog now carries all of it.** `entity-core-protocol`'s `CHANGELOG.md` is declared
canonical and had an entry for ten of the thirty-two `0.8.2.x` revisions, stopping at `.23`. All
twenty-four missing entries are written, each saying what a conformant peer must do about the
revision — including the ones that move nobody, and including the two that record a rule a later
revision superseded, so a reader going top-down does not learn a withdrawn rule as current.

**Instruments built this cycle, each for a defect that had no enforcement point.** `spec pointers`
(does a declared restatement still say what its authority says — plus `--floor`, for the
conformance-floor rows where an *undeclared* restatement is cheap to find) · `spec arms` (what a
pseudocode block refuses, beside the prose enumerating that class) · `spec sections` (does a section
number name exactly one section — three live duplicates, all created by inserting a section out of
numeric order onto a number already taken) · `spec deps` (the declared dependency graph, which
nothing read — one declared **cycle**, which makes the freeze sequencing derived from that graph
undecidable for two nodes) · `spec expiry` (is a tracker row still asserting OPEN on evidence from a
tree that has moved).

> ⚠ **The transferable half of that list is not the tools.** Seven of them shipped with a scope or a
> resolver root that had been set once, for a layout that later changed, and **a wrong scope produces
> confident findings rather than an error.** Every one was caught by validating the new gate against
> the incident that motivated it, in both directions, before publishing its first number — and
> several of the worst defects were found by *writing the assertions* or by *reading the section the
> worklist was for*, never by running the tool.

**The routing channel was the other half of the cycle, and the finding is worth stating plainly: the
inbox gate's scope was two assumptions, and the seat with the most open asks tripped both.** Packets
were discovered by globbing one filename pattern in one directory. **Fifteen documents across three
seats address this corpus in their own FILENAME**, are not routing packets, and were cited nowhere on
the ledger — and naming a recipient in the filename is a *stronger* signal than the addressee field
the gate was built to parse. Discovery is now by name across a peer's whole tree, with a **flow
window** (`--since`) as the run to start a session with, because a lifetime count over a channel
carrying ~30 packets a day answers a question nobody can act on. **Reciprocity is reported in both
directions**: six seats keep a standing index aimed at this corpus, this corpus kept four, and the
intersection was one.

**What the earlier release-mechanics cycle established, still current:**

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
- **Release order is a DAG, not a cycle**, and a tag name is knowable in advance where a commit
  hash is not — which is why a cross-repository build pin must name a tag. **The fix was applied
  and the defect survived it**: the pin names a tag now, and the tag it names was never created in
  the repository it points at. Correct reasoning, correct application, same failure with a new
  value. **A reference is only a pin if it resolves in the history and the layout the consumer
  actually fetches** — one command checks that, and it was checked at none of the three values that
  line has carried.
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

**The residual pin backlog is arch-owed and is larger than this log said, because the published
surface grew and nothing re-read the rule that governs it.** Re-measured 2026-09-17: **705 of 711
short-SHA citations are unreachable to a reader of `master`** *(694 of 699 on 2026-09-08 — an
unswept backlog grows with the corpus, so the delta is the cost of not having done it)*, and they
are **not** concentrated in
the durable documents this paragraph used to name — they are overwhelmingly in the proposal
corpus, which was declared publishable as a whole directory. **The guidance in force at the time
said proposals were internal and SHAs could be cited freely there**, which was true when written
and silently false from the day the directory was declared. The declaration is right and stays; the
rule has been corrected, and the sweep is the work. The core-protocol corpus has completed its
equivalent and sits at **0**, which is what proves it finishable.

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

- ~~**`entity-core-keystone`** — the dead citation in their canonical `AGENTS-STANDARD.md`~~
  **CLOSED.** They shipped 0.8.2 on 08-24 and the dead commit appears nowhere in their public tree;
  arch's finding is why it was caught pre-cut. *(This entry contradicted the header paragraph above,
  which had recorded it closed, for two weeks — a **build-state claim expires like one**, and the
  place it goes stale is the section nobody re-reads when they update the section that changed.)*
- **A downstream release pin in another repository.** A build in a sibling repository fetches this
  ecosystem's Rust implementation by a reference that does not resolve in the tree it fetches, so
  that build fails at checkout before anything compiles. Both ends are owned and tracked elsewhere;
  it does not gate this corpus, and it is listed only because the shape has now repeated three
  times on one line. **A reference is only a pin if it resolves in the history and the layout the
  consumer actually gets** — an internal commit hash does not survive a release boundary, and a tag
  name does not resolve until somebody cuts the tag. Both failure modes were reached by correcting
  the previous one.

  _This entry used to name the reference, the key it is set on and the exact tag string. That is a
  tag which does not exist, written in a file addressed to a reader outside this ecosystem — a
  broken example and an instruction are the same three characters to anyone who copies the line
  before reading the sentence around it. The operational detail belongs in this repo's internal
  cohort ledger, and that is where it now lives._
- **The implementation cohort, as always.** Converging network/resolution specs are not validated
  until exercised by a cross-impl (Go / Rust / Python) conformance run; prose review does not catch
  route/path/dedup defects. REGISTRY/DISCOVERY cohort impl and the ENCRYPTION end-to-end block are
  the open items.
- **The tag point itself, which is not this team's call.** This corpus has reached public `master`
  as content without a corresponding release tag, and no release tag for this cycle exists anywhere
  in the ecosystem yet. When one is cut is decided outside this repo.

## Done recently

- **A core fold now discloses the conformance cells it crosses (2026-09-17).**
  `GUIDE-CONFORMANCE` **§5.3a**, binding from `ENTITY-CORE-PROTOCOL` `0.8.2.32` forward: a proposal
  folding a normative change into the core protocol names the cells of the conformance scope table
  its deltas touch and, for each, whether a check has been driven against it. **An undriven cell does
  not block the fold — an undisclosed one does.**
  The stronger form of this rule, *cells driven green before a fold may land*, was adopted earlier
  and is unmeetable while most cells carry no vector: it forbids every fold, so folds landed under it
  and nothing noticed. **A condition that cannot be met is not a strict gate; it is an unenforced
  one, and the two are indistinguishable from outside.** What the disclosure buys is not coverage —
  it is that a change landing in a region no check can see leaves a record saying so.
  It is gated, and the gate checks the shape only: whether a disclosed drive state is *true* is
  verifiable by the party that runs the checks, which is not the party that writes the specification.
  **The earlier revisions are not reconstructed** — a reconstructed disclosure is a table nobody
  measured.
- **The published core-protocol changelog reaches the current revision again (2026-09-17).** It
  carried an entry for ten of the thirty-two `0.8.2.x` revisions and stopped at `.23`; all
  twenty-four missing entries are written from the folds themselves. **It is a declared canonical
  document**, so the gap was a public reader being handed a record that stops eight revisions short
  of the specification beside it.
- **Four authoring axes swept to their floor, re-measured 2026-09-17.** Every canonical spec now has
  a guide **and** a design record (`spec coverage`: **0 and 0**, where it was 3 and 0). Every
  extension spec carries a **complete** seven-field dependency header (`spec declare`: **26 of 26**,
  where the first honest measurement was **1 of 26** — the two specs then credited carried six of
  seven). Every design document is indexed by a register row (`spec register`: **253 of 253**,
  gating). Every roster row agrees with the spec header it copies (`spec roster`: **64 rows, 0
  findings**, across five roster documents — the fourth and worst of which was found two weeks after
  the gate shipped, because pointing an existing gate at a new document is not the same as that gate
  *matching* anything there).
- **The inbox stopped being a filename convention (2026-09-17).** Packet discovery is by **name
  across a peer's whole tree**, not by one glob in one directory; a **flow window** replaced a
  lifetime count as the run to open a session with; and **tracker reciprocity** is reported in both
  directions, which is the one blind spot no check starting from a file this corpus wrote can reach.
  **0 owed** on the routing channel at the current measurement, in-window and overall — with fifteen
  name-addressed documents newly visible and their reconciliation still owed.
- **The corpus learned to answer "does this question already have an answer" (2026-09-07/08).**
  Every settled design conclusion now has a row in a single register, and a gate — `spec register`
  — asserts that **every** design document in the workspace is either cited by a row or explicitly
  marked as carrying no conclusion. **203 of 203, enforcing.** It exists because the same
  conclusions were being re-derived from scratch, each time with a weaker argument than the
  original: the process side of this repo had ratcheted for months and the *design* side had
  nothing. **A row is a pointer and never an authority** — it names the document that owns the
  answer, and that document is what you cite.
- **The install seam and the walk contract landed, and both were the same class of gap** — a
  normative claim that no check could drive. The tree-walk completeness contract now says what a
  consumer does when an origin serves every node but one, which is the difference between *silently
  hidden* and *visibly incomplete*; the registry-side consumer of that rule shipped with it.
- **Two authoring gates shipped for the extension family (2026-09-08).** A conformance requirement
  is now an **identified, citable row** rather than a section reference — a conformance section
  routinely carries a dozen independently failable obligations, so *"which requirements does no
  check drive"* could not previously be asked at all. And an extension spec now declares, in its
  header, **what installing it touches**: namespaces, kinds, handler operations, and the extension
  points it exposes and consumes. Both are ratchets: existing debt is held, new debt fails.
- **A retired condensed reference, and the reasoning matters more than the document.** A summary
  document had accumulated 49 citations, one of them from a normative `MUST`. **A document that was
  never a citation target does not acquire authority by being cited** — 49 citations to a
  non-authority are 49 wrong citations, and the fix for each is to cite the source. Archived with a
  section→source map.
- **The error-code arc closed, three-way green (2026-09-06).** An eight-cycle run that started as
  *"what is the default code at status 400"* and ended at `ENTITY-CORE-PROTOCOL` **0.8.2.11**. Its
  last leg was `system/tree:put`: the spec mandated a code when a submitted entity *"does not
  decode"* and never defined decoding, and the definition turned out to be already in the corpus —
  `put-request.entity` is a `core/entity`, three fields, none optional. All six `put` error rows are
  now driven green on all three reference peers. **Five error-code tables exist where there was
  one** (`TREE`, `TYPE`, `REGISTRY`, `DISCOVERY`, and now `CONTENT` + `LOCAL-FILES` makes seven).
- **The defect the arc existed to find:** no rust or py SDK could `put` to a go peer at all. At both
  seats a lenient peer and a hash-stripping SDK sat **in the same tree**, so each round-trip was
  perfect and neither seat's suite could see it — *a compensating pair inside one repo is invisible
  to that repo's entire test suite, by construction*, and it surfaced only when one seat drove
  another. Fixed at both, verified by source read at named commits.
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

**The mode has changed and this list reflects it.** The work is no longer *rule the next question* —
it is *get the corpus into a state a stranger can pull, read and act on*, and hold the backlog for
after the cut. New rulings are produced only for a party genuinely blocked now.

1. **Keep the published surface honest and current.** The changelog reaches `0.8.2.32`; the
   remaining surface obligation is the **pin sweep** — 705 of 711 short-SHA citations unreachable to
   a reader of `master`, overwhelmingly in the proposal corpus that was declared publishable as a
   whole directory. Anchor on content digests or `master`-reachable refs, then run
   `spec pins --gate`. The core-protocol corpus sits at **0**, which is what proves it finishable.
2. **Decide whether the process tier publishes at all.** `spec standards --scope
   published-narrative` is red at **38 errors** and it is a **classification** question, not a prose
   one: the rule's prescribed fix is *move this material to the internal directory*, which is
   unavailable to a proposal, because a proposal is the ratifiable unit and has to live where
   proposals live. Three of the four red files are documents whose *subject* is this project's own
   process. **The baseline must not be widened to get green** — lowering it is the one move the
   ratchet exists to prevent.
3. **Reconcile the fifteen name-addressed documents** now visible on the routing channel, and the
   standing asks behind them. This is the largest single block of unworked inbound and it is
   **backlog, not release**.
4. **Continue network/resolution family convergence** — route REGISTRY/DISCOVERY/ENCRYPTION changes
   through a cohort conformance run before or at fold, and update `ROADMAP-EXTENSIONS.md` M-levels
   as they land. **This is the work that continues past the cut**, and the release language says
   *current*, never *final*, because of it.
5. **Pay down the `sdksync` unpinned backlog** (47 of 51 restated SDK blocks), re-reading each block
   against its source before pinning — a pin is a claim that someone looked.
6. **Run the header-label normalization pass** so published spec `Status:` headers match the
   maturity ladder.
