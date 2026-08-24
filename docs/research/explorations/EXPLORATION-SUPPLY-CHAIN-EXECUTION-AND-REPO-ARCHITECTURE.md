# EXPLORATION — supply-chain execution & repo architecture (how we actually get to building/running packages)

**Status:** Exploration / execution map — 2026-07-24. **NOT a proposal.** Turns the content-addressed supply-chain
vision (`EXPLORATION-CONTENT-ADDRESSED-SUPPLY-CHAIN`) + the fleet operations lineage (W-FLEET) into a **manageable,
outside-in program of work**: the minimal repo set, what each stage adds, and the first concrete steps to actually
build and run a signed package. The governing principle is **fewest new repos, outside-in, each stage shippable.**

---

## §1 The executable target (what "building/running packages" minimally means)

Not the full vision at once — the smallest loop that is *real*:

> Build a **signed `system/package`** from source, put it in the content store, have a box **fetch it by hash,
> verify signature + hash, and run it.**

Reach that, and every later property (content-addressed distribution, manifest admission, reproducibility checks,
entity-native source) is an **upgrade to a running system**, not a prerequisite. That is the whole point of the
outside-in build order (`…-SUPPLY-CHAIN` §3a): the outer loop (package → distribute → run) goes entity-native
first on mostly-existing primitives + git; the source end (signed revisions, entity-repo) is pulled in **last**.

---

## §2 The repo architecture

**The established polyrepo** (the set this session references; *confirm the exact roster with the cohort — stated
from context, not a fresh survey*):

| Repo | Role | Relation |
|---|---|---|
| `entity-core-protocol` | the locked core wire spec | authored/owned from arch; core deltas land here at fold |
| `entity-system-architecture` *(this repo)* | extension/capability + app + supply-chain/fleet **design authority** | specs; **no build** |
| `entity-system-arch-tools` | the stdlib-Python `spec` linter | tooling |
| `entity-core-{go,rust,py}` | three ground-up reference impls | the cohort |
| `entity-core-keystone` | conformance anchor | canonical |
| `entity-browser-rust`, `workbench-go`, Godot consumer | consumers branching off core-rust | downstream |

**New repos the supply chain needs — minimal, appearing per stage (not up front):**

| Repo (proposed) | Appears at | What it is |
|---|---|---|
| **connection-node service** *(off core-rust)* | Stage 0 — **now** | the first managed service; the first package producer *and* consumer (W-FLEET / W-CONNECTIVITY, decided) |
| **package / release service** | Stage 2–4 | holds packages by hash + serves the signed release index (may start life *inside* the node, split out later) |

**The source end is *not* a new system.** Entity-native source is `EXTENSION-REVISION` (**Active** — the
content-addressed version-DAG mechanism) plus a **thin git-analog CLI** ("entity-repo", `GUIDE-SHELL-FRAMING` §9.2)
that is **already partly present in `workbench-go`**. The difference between "entity-repo" and REVISION is CLI
polish, not mechanism. And **git stays first-class** — the package's `source_ref` references *any* hashed subgraph
(git commit hash | REVISION root | …; `…-SUPPLY-CHAIN` §3a), chosen per-repo. So the source end needs **no new
repo** and is available near-term.

The runner/host that *executes* a package is **W-HOSTING** (generic → bridge host), not a new repo — it lands
first in `workbench-go` (phase 1) then as a service. So the **only new repo needed to start is the connection
node**; the package service can be a mode of it at first. Manageable by construction.

---

## §3 The dependency shape (what rests on what)

```
system/package (arch: the one new wire)           content store: CONTENT/TREE/SUBSTITUTE (Active)
        │                                                     │
        ▼                                                     ▼
  make package  (Lane B: cargo/rustc on a digest-pinned podman image)  →  signed package entity
        │                                                                        │
        ▼                                                                        ▼
  connection-node repo (off core-rust)  ──runs on──►  the DO box  ◄──fetch-by-hash──  distribute
        │                                                     ▲
        └── system/device management plane (W-DEVICE) ────────┘  admission via `offered` (W-HOSTING, later stage)

  (source end, pluggable:)  source_ref = { git commit-hash | REVISION root } + IDENTITY signature — any hashed subgraph, per-repo
```

The load-bearing observation: **everything on the left/top already exists except the `system/package` wire.** The
content store, identity/signatures, make+podman, and the reachability/W-FLEET design are all in hand. So the
critical-path new spec artifact is *one small entity type*.

---

## §4 The staged sequence (outside-in; each stage is independently shippable)

| Stage | Adds | Repo(s) | Unlocks | Gate |
|---|---|---|---|---|
| **0 — a service that runs** | node built with `make build`, deployed to the DO box, with a `system/device` management plane | connection-node (new) | a real persistent managed service; operational experience (W-FLEET first milestone) | it runs; managed over the connection |
| **1 — build a signed package** | `make package` → signed `system/package` {git source hash, build_env_digest, manifest, artifact_hash}; deploy = verify sig+hash then run | connection-node + arch (the type) | **"building/running packages," minimally.** Source still git | a package whose sig+hash verify before it runs |
| **2 — distribute by hash** | package lives in the content store; the box **fetches the package by hash**, not a copied binary | node/package-service | content-addressed distribution; dedup'd, mirrorable | box runs a package it pulled by hash |
| **3 — admit + host** | a runner checks the package's **manifest against `offered`** and runs it in an isolation unit | W-HOSTING (workbench-go → service) | entity-native run + admission gating; the sense→actuate seam live | a package admitted on its manifest, hosted |
| **4 — signed index + reproducibility** | a REGISTRY §6a.7-shaped signed release index (name/version→package_hash); `make verify` second-builder check | package/release service | anti-rollback signed distribution + independent verification | a 2nd builder reproduces `artifact_hash` |
| **5 — native source subgraph** | source in `EXTENSION-REVISION` (Active) via the thin CLI, instead of / alongside git; `source_ref` = a REVISION root; optional Lane-A builds | **no new repo** (REVISION Active + `workbench-go` CLI) | source end entity-native — but **near-term, a per-repo choice; git stays first-class** | a package built from a REVISION-native `source_ref` |

**MVP = Stage 1** (build/run a signed package); **Stage 2** makes distribution content-addressed. Stages 3–5 are
progressive trust/nativeness upgrades to an already-running system. You can stop at any stage and be strictly
better off than the previous one.

---

## §5 Who owns what (keep the lanes clean)

- **Arch (here):** the `system/package` type + the manifest/admission seam + the `make package`/`make verify` verb
  convention + the reproducibility discipline. Small, spec-shaped. This repo authors it; it lands where §7 decides.
- **Cohort:** the connection-node repo (off core-rust), `make package` in its build, the W-HOSTING runner. The
  actual Rust + build wiring.
- **Ops (operator):** the DO box — content store + package endpoint + runner, one box at first; the deploy step.

We specify the surface and the sequence; we do not build the repos or run the box (AGENTS boundary — stay in tree).

## §6 The first three concrete steps

1. **Author `EXTENSION-PACKAGE`** — ✅ **DRAFTED (`PROPOSAL-EXTENSION-PACKAGE`, 2026-07-24).** Post-verification, it
   shrank further: a package is a **`build-provenance` attestation kind on the Active `EXTENSION-ATTESTATION`** (not
   a new entity type) over a content-addressed artifact blob — the **Lane-B foreign-artifact on-ramp**. Distribution
   reconciles onto `EXTENSION-SUBSTITUTE` (Active) + REGISTRY §6a.7 (no new distribution surface). The only new
   normative surface is one attestation kind + its `properties` schema + a hermetic-build contract + the
   `make package`/`make verify` verb. Critical-path spec artifact: **one**, and it invents no type or wire.
2. **Create the connection-node repo** off core-rust; execute **Stage 0** (it runs on the DO box with a
   `system/device` management plane) — this is already the W-FLEET first milestone, so it is not new work, just
   sequenced here.
3. **Add `make package` + hermetic-build discipline** to the node's build (digest-pinned base, no in-build network,
   `SOURCE_DATE_EPOCH`, stripped output) and emit the first signed package — **Stage 1, the MVP.**

Everything past step 3 is the Stage 2–5 upgrade ladder, each with its own small gate.

## §7 Open decisions (need an arch or operator call)

- **Home of `system/package`** — **RESOLVED (operator, 2026-07-24): a system component** (a `system/*` extension,
  `EXTENSION-PACKAGE`), converging with `system/device` / `system/signaling` / `system/network`. The dividing line
  is now explicit: **infrastructure lands as system components (`system/*` extensions); L5 conventions are for apps**
  (chat / site / embed). A package is substrate consumed by many apps, so it is a system component — not
  REGISTRY-internal (REGISTRY is a *consumer* of packages for its release index), not a `REPOS` app convention
  (`REPOS` is the L5 *repo-hosting UX* over it).
- **Runner = the node itself, or a separate fleet-runner?** Self-hosting (the node also hosts packages) is fewest
  moving parts for Stage 3; a separate runner is cleaner at scale. *Lean: fold into the node first, split at a
  second service.* **Design call.**
- **DO-box role at bootstrap** — content store + package endpoint + runner all-in-one, or split? *Lean: all-in-one
  on one box; that is the whole point of "one repo, one build, one package, one box."* **Ops call.**
- **`source_revision_hash` at bootstrap** — git tree hash vs commit hash; the signing target. **Small, pin at
  Stage 1.**

## §8 Honest ledger

- **Grounded:** the outside-in order + the one-new-wire critical path rest on the confirmed primitive inventory
  (`…-SUPPLY-CHAIN` §2) and the make+podman convention (AGENTS-STANDARD). The repo roster in §2 is *stated from
  session context, not re-surveyed* — confirm the exact set with the cohort before acting on it.
- **New but small:** `system/package` (one entity type), two make verbs, the staging. No new runtime, no new wire
  core, no fork of the impls.
- **Deferred:** the scheduler / multi-node fleet / federation are past Stage 5 — do not pull them into the
  bootstrap; one box, one service first.
- **The gate (unchanged discipline):** not real until a package is actually built, signed, distributed by hash,
  and run — and (Stage 4) a *second builder reproduces its hash*. Prose does not prove a build pipeline.

## §9 References

- Vision + primitives: `EXPLORATION-CONTENT-ADDRESSED-SUPPLY-CHAIN` (§2 inventory, §2a build/package, §3a
  outside-in, §4 trust).
- Fleet / run: `WORKSTREAMS.md` W-FLEET + W-HOSTING; `HANDOFF-2026-07-24-connection-node-implementation-direction`.
- Source end: `EXTENSION-REVISION`, `GUIDE-SHELL-FRAMING` §9.2 (`entity-repo`).
- Distribution: `EXTENSION-REGISTRY` §6a.7 (signed manifest), `EXTENSION-SUBSTITUTE`, `GUIDE-REFERENCE-DEPLOYMENT`.
- Roadmap slot: `ROADMAP-APPLICATIONS` `REPOS`.

*The one sentence: we reach "building/running packages" by going outside-in — author one small `system/package`
wire, stand up the connection-node repo off core-rust and run it on the DO box (Stage 0), add `make package` for a
signed hermetic build (Stage 1, the MVP), then climb the independent upgrades — distribute-by-hash, manifest
admission, reproducibility checks, entity-native source — each shippable, on a running system, with the connection
node as the single first end-to-end proof.*
