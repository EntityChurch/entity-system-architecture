# docs/research/ — INDEX of the design record

**What this file is for:** answering *"what have we already studied about X, and where is it?"*
**without grepping.** `README.md` beside this file describes the **lifecycle** (how work moves);
this one describes the **inventory** (what exists). Both are needed and only the first existed.

> **Read this before opening a design question.** L7 (*check what exists before building it*) fired
> **five times in three days** and L11 was ratified after three rulings were published on a design
> space whose founding study was never opened. **This index is L7's enforcement point.** If the topic
> you are about to explore has a row here, read that row's document first — and if it does not, add
> the row when you write it.

**`explorations/` — 35**

*(That count is gated: `spec ledger` compares it against the directory it names, the same rung that
holds `docs/proposals/INDEX.md`. Run it after adding or moving a document.)*

> **`reviews/` is internal and is not published.** The `ABSORPTION-*` and `REVIEW-*` documents are
> cross-implementation absorption — a peer's report, read and reconciled — and they are session
> working memory, not part of the published surface ([ADR-0031]). Rows below that cite one are
> recording **where a finding was worked out**, not offering a document you can open here. Their
> count is deliberately not gated in this file, because a count naming a directory the published
> tree does not contain is a finding in every reader's copy.
>
> **The internal families, in full, so this is checkable rather than a blanket excuse:**
> `STATUS-*` · `ROUTING-*` · `HANDOFF-*` · `PLAN-*` · `CHECKPOINT-*` · `AUDIT-*` · `TRIAGE-*` ·
> `SIGNOFF-*` · `MAP-*` · `CENSUS-*` · `WORKSTREAMS*` · `ARCH-RESPONSE-*` · `ABSORPTION-*` ·
> `REVIEW-*` — **plus everything under `docs/status/` and `docs/archive/`, by path.**
> **A citation to anything outside that set is expected to resolve** — if one does not, that is a
> defect and worth reporting.

---

## §1 Network · connectivity · relay

| Document | What it establishes |
|---|---|
| `EXPLORATION-NAMING-LANDSCAPE-AND-THE-DNS-MAPPING` | **Start here for any naming, registry or resolution scope question — and note that until 2026-08-20 there was nothing to start with.** REGISTRY's founding studies are in **legacy** (the eight-point continuum plan + the six-system survey: DNS/DID:web · libp2p+IPNS · Nostr · ActivityPub · ATProto · ENS), named by path in §0 and carried forward as substance. Establishes: the **DNS record-type map** (12 of 15 answered, NS + NSEC deferred with the direction pinned to DNS-zone/TUF, one missing); the three divergences that are *toward* the stronger property (signature-anchored not transport-trusted · query privacy at the **substrate** where the founding plan deferred it per-backend · **no CNAME, correctly** — aliasing is a symptom of names bound to locations, ours bind to keys); **Zooko's triangle as the frame the corpus never named, which the three binding kinds pass by exposing all three corners under one contract**; and **the one real hole — peer-id → endpoint has no answer**, while every id-first system surveyed has one (libp2p: it *is* the DHT's primary content · ATProto: the DID doc's endpoint · Nostr: NIP-65). DNS is no guide there because DNS is name-first by construction and we are id-first. Also ranks the four paper backends: **`well-known-url` first** — it covers DID:web + NIP-05 + WebFinger in one — and **`dns-txt` is lower-value than it looks**; and argues **browse stays a `SHOULD`** because DNS ran that experiment with AXFR and reversed it |
| `EXPLORATION-RELAY-LANDSCAPE-AND-PRIOR-ART` | **Start here for any relay scope question.** The prior-art survey (SMTP · NNTP · Nostr incl. NIP-17/44/59 · ATProto · ActivityPub · Matrix · libp2p circuit-relay-v2 · Tor), why the four modes are a **design space and not a taxonomy to prune**, the operator boundary (**relay = addressed messaging; gossip/CDN = data expansion**), and the retraction of the aggregate-leaves-RELAY disposition. **Part 2 is done — see the row below** |
| `reviews/REVIEW-2026-08-17-the-relay-prior-art-read-and-the-mode-set-disposition` | **The legacy prior art, read, by path** — the `L11` obligation discharged. The four modes are clusters in a **five-dimensional** space drawn from a six-system survey, not a taxonomy; **no surveyed system is single-mode**; the cross-mode unification is named by the study as the extension's contribution versus every surveyed system. Mode A's blocker is **void against `EXTENSION-SUBSCRIPTION` §6, which is cross-impl verified**. The operator boundary separates Mode A from **gossip**, not from relay. **Disposition 2** (narrow §1's wording, keep Mode A). Also: `PLAN-EXTENSION-LANDSCAPE` is the identity stack, **not** the network family — two handoffs mis-read it from its name |
| `EXPLORATION-THE-UNIVERSAL-NAMESPACE-AS-THE-MULTI-PEER-DATA-MODEL` | V7 §1.4's cached-remote-namespace model as the multi-peer data model: facts-*about*-a-peer vs copies-of-a-peer's-data; **mirrors preserve the publisher's path**; aggregation is a `type_filter` query with no peer filter, so **type tags must converge and paths must not**; SUBSCRIPTION §8.2's missing path obligation. §4.3 is **retracted** |
| `EXPLORATION-THE-NETWORK-FAMILY-CONSOLIDATED-DESIGN-RECORD` | **The design record the V8 split lost, carried across as substance rather than as files.** Twenty legacy documents' findings: the three primitives and the V0.5→V2.0→V7 lineage (**V2.0 shipped `routes`/`relay`/`gossip`/`replicate` as four Layer-5 extensions; `routes` and `gossip` did not survive — ROUTE is now recovered, GOSSIP is not**), the five-dimensional mode space, the two load-bearing relay rulings, the unified delivery model + reachability classes, the nine-layer subsystem map, and **ten open items each tagged by constraint regime**. Carries the tracking rule: **we keep what is load-bearing for work under active refinement** — and a landed spec citing a support doc is a **§11.3 leak to clean up, not a demand signal to import against** |
| `EXPLORATION-THE-THREE-CONSTRAINT-REGIMES` | **Why the core/extension/network boundary reads as blurry, and the fix.** A second axis orthogonal to `SYSTEM-ARCHITECTURE` §13.1's tiers: **I substrate** (mathematics governs; simplifying = being wrong) · **II design space** (coherence governs; the only regime where complexity is ours to remove) · **III accumulated technology** (NAT, ICE, the browser sandbox — real, not ours, and only *placeable*). The blur's actual cause: **an extension has a regime of structure and a regime of motivation and they differ** — RELAY's internals are II, its existence is III. Load-bearing rule: **Regime III must never become Regime I or II structure**, and most of the network family's apparent fussiness is that quarantine working |
| `EXPLORATION-P2P-CONNECTIVITY-AND-OVERLAY` | The consolidated P2P connectivity substrate + overlay fast-follow |
| `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE` | NAT traversal + WebRTC as one architecture — how the pieces fit |
| `ANALYSIS-CONNECTIVITY-LIBP2P-COMPARATIVE-AUDIT-AND-LANDSCAPE` | Comparative audit against libp2p + a landscape survey |
| `ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT` | Connection-establishment flows and the minimum infrastructure a deployment needs. **Its Part D envelope was tightened 2026-08-21 — see the row below** |
| `reviews/REVIEW-2026-08-21-THE-ACQUISITION-PATH-AND-WHAT-THE-TWO-DEVICE-RUN-GENERALIZES-TO` | **Start here for any question about how a peer *becomes* reachable, as opposed to how a message travels once it is.** Absorbs `entity-browser-rust`'s two-device run — two browsers chatted and moved a file over WebRTC through a desktop rendezvous, **zero reflectors, zero relays, nothing typed**. Establishes: **acquisition is a fourth concern the corpus never modelled** — an intermediary is learned from **four** sources (operator handle · stored row · build default · co-located backend) and only one is durable state, so *the node in force* is the property and the precedence is impl policy; **the fixture-blindness class**, distinct from `SIGNALING` §11.5.1's substrate blindness **and with the opposite remedy** (*stop supplying the input*, not *upgrade the substrate*); that **`GUIDE-NETWORKING-MODEL` §3's LAN row is false on every clause for a browser**, which falls between the LAN row and the roadmap row; that the run **tightens Part D's envelope** — a LAN needs *no* infrastructure when a participant runs the desktop app; that browser-rust's "well-known conversation id" **is** `content_hash(genesis)` and the two chat models are one, sound **for `policy: closed` only** because a derived `creator` hands admin to whoever sorts first; and **the workbench-go recipe — a native peer does not need WebRTC to talk to a browser** (§6.5.2b `websocket`; native WebRTC is absent from all 22 go modules and is a project, not a gap) |

> **The boundary map is not here — it is `guides/GUIDE-NETWORKING-MODEL.md`** (six concerns kept
> separate · deployment-topology table · v1-vs-roadmap status board). **Read it before drawing any
> information-travel diagram.** It was found only after a second one was started.

## §2 Compute

| Document | What it establishes |
|---|---|
| `EXPLORATION-COMPUTE-PROGRAM-RUNTIME-CONTRACT` | Hostable compute programs + the runtime interface contract |
| `EXPLORATION-COMPUTE-EXECUTION-BUDGET-AND-BOUNDARY-LOWERING` | **The budget is a policy, not a law**; lowering across the boundary |
| `EXPLORATION-COMPUTE-WHOLE-STATE-TICK-COST` | The whole-state O(N)-per-tick floor — workbench-go **lever 2**'s home |
| `EXPLORATION-GENERIC-HOST-AND-DESCRIPTOR` | The generic host + program descriptor — what makes programs transferable |
| `EXPLORATION-BRIDGE-HOST-AND-ANY-NATIVE-COMPUTE` | The bridge host, byte streams, FFI, any-native compute |
| `ANALYSIS-COMPUTE-CONSTRUCTION-AND-EXECUTION-READINESS` | DSL readiness + Axis-1 guidance |
| `ANALYSIS-SUBSTRATE-CONVERGENCE-COMPUTE-DEVICE-SCHEDULER` | Do generic host / compute / scheduler / device align (+ the GPU question) |

## §3 Device · execution substrate · infrastructure

| Document | What it establishes |
|---|---|
| `EXPLORATION-EXECUTION-SUBSTRATE-ORCHESTRATION` | Where device sits, and the full orchestration scope |
| `EXPLORATION-RUN-ENVIRONMENTS-AND-WORKLOAD-ACTUATION` | WASM-first run-environments; what it implies for device |
| `EXPLORATION-SUBSTRATE-INTROSPECTION-DEVICE-AND-STORAGE` | Device + storage capability introspection |
| `EXPLORATION-MACHINE-BOUNDARY-AND-FULL-STACK-TRAJECTORY` | The machine boundary and the full-stack trajectory |
| `ANALYSIS-DEVICE-STANDARDS-ALIGNMENT-REDFISH-OTEL` | Alignment to OpenTelemetry + Redfish |
| `ANALYSIS-DEVICE-STORAGE-SHAPE-OPERATIONAL-STATE` | Is "storage stats" even an extension? |
| `ANALYSIS-INFRA-HORIZONTAL-SCALING-AND-STATE-COORDINATION` | Horizontal scaling + multi-server state coordination |

## §4 Supply chain

| Document | What it establishes |
|---|---|
| `EXPLORATION-CONTENT-ADDRESSED-SUPPLY-CHAIN` | The content-addressed supply chain — the capstone unifying the stack |
| `EXPLORATION-SUPPLY-CHAIN-EXECUTION-AND-REPO-ARCHITECTURE` | How we actually get to building/running it |
| `ANALYSIS-SUPPLY-CHAIN-LANDSCAPE-ALIGNMENT` | Nix / OCI / SLSA / TUF / Sigstore / Bazel / Unison + our legacy |

## §5 Identity · freshness · rotation

| Document | What it establishes |
|---|---|
| `EXPLORATION-ROTATION-AUTHORITY-AND-KEY-LIFECYCLE` | Rotation authority, key lifecycle, and where each belongs |
| `EXPLORATION-NON-INTERACTIVE-FRESHNESS-AND-ANTI-REPLAY` | The freshness/anti-replay trade-off surface + the three knobs |
| `ANALYSIS-NON-INTERACTIVE-FRESHNESS-CRITICALITY` | Is strong non-interactive freshness critical for v1? |

## §6 Application tier · L5 · corpus shape

| Document | What it establishes |
|---|---|
| `EXPLORATION-L5-APP-HOSTING-UNIFICATION` | The entity app standard beyond JS iframes |
| `EXPLORATION-FULL-STACK-DISCOVERY-INFRA-AND-ENTITY-CHAT` | discovery → managed infrastructure → entity-chat as one stack |
| `ANALYSIS-LANDSCAPE-COMPLETENESS-AND-ENTITY-NATIVE-TRANSLATION` | Landscape completeness + the entity-native translation principle |
| `ANALYSIS-CORE-CATALOGUE-DUPLICATION-AND-DRIFT` | ~~38 duplicated types, 13 drifted~~ — **struck through in its own title; read before citing** |
| `ANALYSIS-CORE-NAMESPACE-CLAIMS-IMPACT` | Types inside core-owned namespaces: what is actually wrong |

## §6a External landscape — adjacent projects and name collisions

| Document | What it establishes |
|---|---|
| `ANALYSIS-THE-ENTITY-CORE-NAME-COLLISION-AND-THE-EIOS-INITIATIVE` | **Read before anyone concludes that `github.com/entity-core` / `entitycore.org` is related to us — it is not, and the doc carries the pins.** Valto Loikkanen's **EIOS** (*Entity Information Operating System*) is an organizational-knowledge-management framework — prose only, no code, 0 stars, last touched 2026-07-10. **Independent convergence, established rather than assumed:** their §37 table derives "Entity Core" from PIOS's `Core Self` by a documented fork rename, their §37.1 publishes a collision check naming WHO's EIOS and EntityOS **and not us**, and an exhaustive grep of their **entire** published corpus (8 files, 23,938 words, tree listed via API) returns **zero** hits for `entitychurch`, `peer`, `hash`, `CBOR`, `merkle`, `conformance`, `opcode`, `cryptograph`, `content-address`. We predate them ~7 months on naming, 3 weeks on the GitHub org. **The overlap is two words of English at two altitudes** — their "operating system" is the sense in which a company runs on one; structurally an Entity Keel is an L5 application, so the two stack rather than compete. **Three real costs, none arch's:** `entitycore.org` was keystone's intended pub.dev verified-publisher and `org.entitycore` Java reverse-DNS namespace and is now unobtainable (keystone/devops) · the `entity-core` org slug is gone (operator) · and **the published prose is statically hosted and unblocked but has no crawl path** — all five apexes serve the Entity Browser SPA with the real content projected beneath at `/sites/<peer-id>/<site>/`, and the apex HTML contains **zero** references to `/sites/`, no `<noscript>`, no `sitemap.xml`, so a crawler stops at 24 words (devops/content). **§4.3 carries a correction:** its first version claimed the sites "expose no crawlable text," which was a negative measured from `/` plus four guessed paths — false, and the shape `AGENTS-STANDARD`'s *prove a negative* rule exists for. **An absence found at the front door is not an absence** |

## §7 Reviews — cohort absorption and arch responses

**`ABSORPTION-*`** — a peer's findings absorbed: compute (Asteroids actor probe · Axis-1 results ·
first cross-impl corpus run · Life/Snake POC), keystone (corpus gaps F29–F30 · findings F31–F46),
workbench-go (continuation error model).

**`ARCH-RESPONSE-*` / `ARCH-RULINGS-*`** — rulings issued to the cohort: the continuation/clock arc
(core-go's 8 asks) · NETWORK Amendment 12 rungs 1–2 · NETWORK reconnect lifecycle · Problems A/B +
the 502 + the backoff formula · the compute corpus first run · the cycle-close packet · 0.8.1 ratify
preconditions · **connectivity + chat buildability**.

**`REVIEW-*`** — identity pre-rotation first pass · device cohort coverage dispatch · keystone
retrospective + red-team · **gaps in keystone's analysis (red-teaming the red-team)**.

---

## §8 What is NOT here — and this is the part that has bitten us

**The legacy workspace holds 727 documents** (60 explorations · 279 reviews · 199 proposals) at
the internal legacy corpus (read-only).
**We carried 51.** It is **read-only** — another team's tree; bring items forward selectively, never
edit there.

**`spec address` reports 585 dangling citations (mechanical 0 / judgment 585)** — this corpus
routinely cites reasoning it does not contain. Two confirmed instances, both of which caused real
harm this week:

| Cited as present here | Actually |
|---|---|
| `PROPOSAL-EXTENSION-GOSSIP` (`GUIDE-NETWORKING-MODEL` §7) | **Legacy only.** Made the gossip lineage read as *abandoned* rather than *deferred* |
| `PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY` (same) | **Legacy only** — and `README.md` in this directory already said so, while the guide said otherwise |

**The eleven relay documents named by `HANDOFF-2026-08-17-relay-audit` §3a are all legacy.** Five are
read (see `EXPLORATION-RELAY-LANDSCAPE-AND-PRIOR-ART` §7); **six are not**, and part 2 of that audit
is what reads them.

---

## §9 Keeping this index true

- **Add a row in the same commit that adds the document.** An index that lags is worse than none.
- **`spec ledger` gates the counts** against the directories — same rung that already holds
  `docs/proposals/INDEX.md`.
- **A row states what the document *establishes*, not what it is about.** "Relay prior art" is not a
  row; "why the four modes are a design space and not a taxonomy to prune" is.
- **Struck-through or retracted content stays listed, marked** — a reader must be able to find a
  superseded conclusion in order to know it was superseded.
