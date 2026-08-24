# ANALYSIS — the "Entity Core" name collision: Valto Loikkanen's EIOS initiative

**Status:** Analysis — 2026-08-21. **NOT** a proposal, and it rules on nothing in the corpus.
Answers an operator question: *another project is publishing under "Entity Core" with an
"operating system" framing — did they take it from us, and what are they actually doing?*
Author: arch workspace.

**Provenance.** Every claim below is pinned to how it was observed. External facts come from live
fetches on **2026-08-21**: `github.com/entity-core` (org page + REST API `/orgs/entity-core`),
`entitycore.org` (homepage + `/eios`), and the **entire published EIOS corpus downloaded and
grepped locally** — 8 markdown files, 23,938 words, the full `main` tree of `entity-core/eios`
listed via the git-trees API so the search is exhaustive rather than sampled. Domain facts come
from RDAP/whois. Our own dates come from `git log --reverse` in this checkout and
`/orgs/EntityChurch`. Nothing here rests on a search-engine summary.

---

## The verdict in one line

**Independent convergence, not copying — and the collision was manufactured by a documented
rename inside a *different* project's lineage.** "Entity Core" is what Valto Loikkanen's PIOS term
**"Core Self"** became when he forked PIOS into an entity-scoped sibling on 2026-07-08. The rename
is written down, in their own §37 terminology-mapping table, alongside a collision check that names
**WHO's EIOS** and **EntityOS** as the brands they checked against — and does not name us. Their
corpus contains **zero** occurrences of `entitychurch`, `entity-core-protocol`, `ecdeos`,
`keystone`, `peer`, `hash`, `CBOR`, `merkle`, `conformance`, `opcode`, `cryptograph`, `libp2p`,
`IPFS`, `wire format`, `content-address`, or `blockchain`.

Beyond the two words in the name, **there is no technical overlap at all.**

---

## §1 What EIOS actually is

An **organizational knowledge-management framework** — prose only, no code, no protocol.

| Layer | Their name | What it is |
|---|---|---|
| Initiative | **Entity Core** | The public brand / org at `entitycore.org` |
| Framework | **EIOS** | *Entity Information Operating System* — the spec (39 sections, CC BY 4.0) |
| Implementation | **Keel** | Reference implementation. **Seed draft; no runnable distribution exists** |
| Blueprint | **Weave** | Naming/classification canon for entity information |
| Instance | **Entity Keel** | One organization's private running instance |

The thesis is *"Your AI is only as good as the context it can see"* — a company's institutional
memory (originals, events, decisions, agreements, financials, glossary, governance) should live in
one durable, governed, exportable store so that AI agents and humans both operate with full
context. The named problems are **corporate amnesia, key-person dependency, and tool lock-in.**

Its architecture is **five information zones** (Originals · Events · Knowledge · Derived · System)
and **five entity circles** (Entity Core → Internal → Extended → Served World → Outer). Section
titles include *Entity DNA*, *Entity Viability*, *Stakeholder Model*, *Financial and Economic
Context*, *Meetings and Communications*, *Self-Improving Entity Loops*. **`Keel` 105 hits ·
`governance` 96 · `stakeholder` 88 · `agent` 188.** It is a management-and-governance document.

**Scale and stage, so nobody over-reads it:** one person, five repos, **0 stars on every repo**, no
code, last commit **2026-07-10** — six weeks stale as of today.

---

## §2 The evidence trail — why "they scraped us" is refuted

| Date | Event | How observed |
|---|---|---|
| **2025-12-22** | the pre-split monorepo first commit — our "entity core" naming in use | `git log --reverse` |
| 2026-01-10 → 06-05 | `entity-core-{rs,py,go,rust}`, `-papers`, `-keystone` first commits | `git log --reverse` |
| 2026-04-17 | **`github.com/peecos`** created — PIOS, *Personal* Information Operating System | `/orgs/peecos` |
| 2026-06-10 | `entitycoreprotocol.org` registered | RDAP |
| 2026-06-14 | **`github.com/EntityChurch`** created | `/orgs/EntityChurch` |
| 2026-06-21 | Our v0.8.0 initial public release — 11 repos pushed public | `git log --reverse` |
| **2026-07-08** | EIOS forked from PIOS; names **EIOS / Keel / Weave / Entity Keel ratified** | their `spec` §39.3 |
| **2026-07-09 13:49Z** | `entitycore.org` registered | RDAP |
| **2026-07-09 15:47Z** | **`github.com/entity-core`** created — two hours after the domain | `/orgs/entity-core` |
| 2026-07-10 | EIOS v1.2; all five repos last touched | GitHub |

**We predate them by roughly seven months on the naming and three weeks on the GitHub org.** There
is a 17-day window (our 06-21 public release → their 07-08 name ratification) in which contact was
theoretically possible. Four things close it:

1. **The rename has an internal source.** Their §37 table derives `Entity Core` from PIOS's
   `Core Self`, `Keel` from PIOS's `Core`, and `Weave` from PIOS's `Cotton` — a fiber metaphor
   extended to fabric. The whole name set is one mechanical entity-scoping pass over an inherited
   personal-scoped vocabulary. Nothing needed to be borrowed.
2. **They ran a collision check and published it.** §37.1 explains why the *public* brand is
   Entity Core rather than EIOS: *"the acronym EIOS collides with existing public usage (notably
   WHO's 'Epidemic Intelligence from Open Sources', eios.org), and 'EntityOS' is an existing
   product brand."* Someone who checked two collisions and wrote both down would have written down
   a third.
3. **Zero lexical trace.** Exhaustive grep of their full corpus for our identifiers and our
   technical vocabulary: **all zero** (list in the verdict above). The only hits on words we also
   use are ordinary English — `capability` appears 23 times and every occurrence is *business*
   capability ("capability map", "capability development"); `extension` appears twice, neither
   about software; the acronym `DID` appears **zero** times case-sensitively.
4. **We were hard to *find*, though this is the weakest of the four and does not carry the
   argument.** Our published prose is real and static, but it sits under `/sites/<peer-id>/…`
   with no crawl path from any apex (§4.3), and a search for our own project name today returns
   EIOS, `entityos.io`, and Microsoft's EF Core rather than us. That makes accidental encounter
   unlikely; it does not make it impossible — which is why the case rests on 1–3.

---

## §3 Why the terminology overlap feels so strong anyway

Both projects independently reached for the same three moves, because both are answering
*"what is the durable substrate for a thing that outlives its participants?"* at very different
altitudes:

| Move | Us | Them |
|---|---|---|
| Call the unit an **entity** | The addressable typed thing in the tree | The organization (company, co-op, foundation, public body) |
| Call the substrate an **operating system** | Literal — a wire protocol, kernel, extensions, capability dispatch | Metaphor — a governed information layer |
| Call the middle **core** | The protocol below the extension family | The innermost trust circle **and** the public initiative |

Two genuinely convergent *design* commitments are worth naming, because they are the reason the
sites read alike:

- **Append-only event log as the spine, with originals preserved immutably and derived views
  rebuildable.** That is their zone model and it rhymes hard with content-addressed storage plus
  reconstructible state. They arrive at it from audit/compliance; we arrive at it from
  content-addressing and dedup.
- **Portability as a survival property.** Their §35 requires an Entity Keel be *"exportable,
  recoverable, and runnable outside the infrastructure provider currently hosting it"* — the same
  instinct that makes ours a protocol with independent implementations rather than a product.

**The convergence stops there.** They have no wire format, no peers, no cryptography, no
conformance notion, no addressing model — the five load-bearing invariants this repo foregrounds
have no counterpart in their corpus, because their document never descends to a layer where those
questions exist. Their "operating system" is the sense in which *"a company runs on an OS"*; ours
is the sense in which a machine does.

**Structurally the two are orthogonal, and arguably stacked.** EIOS describes *what an organization
should keep and who may see it*; entity-core describes *how typed state is addressed, stored, and
moved between peers*. An Entity Keel is roughly an L5 application — the layer this repo's
`specs/applications/` conventions are for. Nothing to defend against; no overlap to resolve.

---

## §4 What the collision actually costs us

Three items, all real, **none of them arch's to fix** — recorded here so they route rather than sit.

1. **`entitycore.org` is gone, and we had planned uses for it.** It is named in
   `entity-core-keystone`'s protocol-generator packaging docs as the intended **pub.dev verified
   publisher namespace** (`dart/profile.toml`, `A-DART-007`) and as the **DNS TXT anchor for the
   `org.entitycore` Java reverse-DNS namespace** (`java/status/ARCHITECTURE-REVIEW.md`). Both
   assumed a domain we do not own and cannot now obtain. *Owner: the keystone / devops seats — a
   substitute namespace has to be chosen before either package ecosystem is published.*
2. **The GitHub org slug `entity-core` is taken.** No live conflict — we publish under
   `EntityChurch` — but the obvious org name is now permanently unavailable, and our repos are all
   named `entity-core-*`, so `github.com/entity-core` is where a reader will guess first and land
   on them. *Owner: operator.*
3. **The search collision is asymmetric — and the cause is a missing crawl path, not missing
   content.**

   > **Correction, 2026-08-21 (operator).** The first version of this item claimed our sites
   > *"expose no crawlable text."* **That is false**, and it was false in the exact shape
   > `AGENTS-STANDARD`'s *prove a negative* rule and **L4** describe: the absence was measured by
   > fetching `/` plus four guessed paths, and published as a claim about the whole site. **The
   > content is there and it is statically hosted.** Five apexes are live —
   > `entitycoreprotocol.org`, `entitychurchfoundation.org`, `ecdeos.org`, `entitychurchregistry.org`,
   > `billslab.com` — each serving the Entity Browser SPA **with a content projection beneath it at
   > `/sites/<peer-id>/<site>/`**: the protocol spec, CBOR encoding, native type system, machine
   > spec, glossary, keystone conformance matrix, `validate-peer`, and formalization, as ordinary
   > crawlable HTML. The protocol index alone is 488 words of prose over twelve linked pages.

   **What is true is narrower, and still worth fixing: the apex is not a path to it.**
   `https://entitycoreprotocol.org/` contains **zero** occurrences of `/sites/` — not in markup,
   not in script — and carries no `<noscript>` block, so its visible text is the 24-word
   WASM-failure fallback. `sitemap.xml` is 404. `robots.txt` is Cloudflare's default managed file
   with no `Sitemap:` directive and no `Disallow`. The 404 page is R2's stock page, linking only
   to Cloudflare docs. **A crawler that lands on an apex has no edge to follow and stops.** The
   prose exists, is unblocked, and nothing points at it. *(Measured at the edge 2026-08-21 by
   `curl`, all five apexes.)*

   **Owner: `entity-browser-rust`, verified in their tree rather than inferred** — the shell is
   their `index.html` (`1005db76`), `grep -rn '<noscript'` returns **nothing tree-wide**, and the
   `/sites/` projection is emitted by `src/content_site/static_export.rs`, the same file as
   publication-gate item **B-2**. **Tracked as `COHORT-OPEN-ITEMS` §0b B-4**; still owed a direct
   delivery to that seat (app tier — raised directly, never through the core packet).

---

## §5 What is not claimed

- **Not claimed: that they have never heard of us.** Absence of lexical trace in a published
  corpus is not absence of awareness. It is strong evidence against *derivation*, which is the
  question asked.
- **Not claimed: that PIOS predates our naming.** It does not — `peecos` was created 2026-04-17,
  four months after our first `entity-core` commit. The lineage argument is about *where their
  word came from*, not about who used it first.
- **Not claimed: anything about their non-public work.** Only the published corpus was searched.
- **Dates expire.** Their repos were last touched 2026-07-10 and their stage ("no runnable
  distribution") is a build-state claim measured today. Re-take it before citing it.

---

## §6 Disposition

**No action in this corpus. Nothing to rule on, nothing to defend, no proposal owed.** Two words
of English collided at two different altitudes, and the technical surfaces do not touch anywhere.

The one durable item is §4.3, and it is not about them: **the published surface exists, is static,
and is good — and nothing links to it.** They registered a domain six weeks ago, put prose at the
apex, and will index cleanly. We have a full protocol specification statically hosted on five
domains with no crawl path from any of them, six months after we started using the name.

**And the second lesson is about this document.** §4.3's first version was a negative measured from
one path and published as a claim about a whole site — the failure `AGENTS-STANDARD` names outright
and that **L8's thirteenth form** already records on a different noun. It survived because it was
*plausible*: the apex really does render 24 words. **An absence found at the front door is not an
absence.**
