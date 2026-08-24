# PROPOSAL — the static publish handshake: close the phantom, name the publish-side obligation, and say the freshness bound out loud

**Status:** **IMPLEMENTED 2026-08-18** — all seven deltas folded, each verified against the tree.
D1 `EXTENSION-NETWORK` §6.5.3.1 · D2 §6.5.6 · D3 §6.5.3 · D4 §6.5.5 (adjudicated first, per the
proposal's own *do not fold blind*: the phantom `EXTENSION-STORAGE-SUBSTITUTE-HTTP §3-RES.4` resolves
to landed `EXTENSION-SUBSTITUTE` §7.2, which owns the manifest-signature MUST, the `seq` freshness rule
and the `manifest_signature_invalid` / `manifest_stale_seq` codes) · D5 §6.5.4 + the
`manifest_url_prefix` reservation in §6.5.3 · D6 `EXTENSION-TREE` §3.3a · D7 `guides/GUIDE-SERVING-MODE`
§8. `EXTENSION-NETWORK` 1.7 → 1.8, `EXTENSION-TREE` 4.0.2 → 4.1.
**Tier:** extensions — `EXTENSION-NETWORK` §6.5.3 / §6.5.3.1 / §6.5.6, `EXTENSION-TREE` §3.3a.
**Answers:** `entity-browser-rust` `ROUTING-2026-08-17-d` Q1–Q4 (`dev` @ `65dfa58`), and a
conformance finding against `entity-workbench-go` `publish/publish.go` that their Q1 surfaced
without either of us looking for it.
**Read at:** arch `5f3a50b` · browser-rust `65dfa58` · workbench-go `ca2e0cb` · core-rust `1a6955b` ·
arch-tools `46c6e50`

---

## §0 Summary

| Ask | Answer |
|---|---|
| **Q1** — is layer 1 + layer 2 the intended composition for a *static* publish? | **Yes, and the static case is the primary case, not a variant.** But the document they were told to check against — `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE` — **has never existed**, and the publish-side obligation it was going to carry is genuinely absent. §1, §2 |
| **Q2** — is per-purpose tracked-prefix intended? | **Yes, and `EXTENSION-TREE` §3.3a already rules it explicitly** — "two conformant publishers may legitimately publish different extents; `prefix` is what distinguishes them." Track `sites/`. §3 |
| **Q3** — does `guarded_publish` need to coalesce? | **Coalescing is explicitly permitted; dropping the final root is not.** `EXTENSION-NETWORK` §6.5.6's bounded-convergence MUST (30 s maximum convergence delay) is the answer, and it makes their unreproduced suspicion a *conformance* question with a test. §4 |
| **Q4** — freshness vs rollback: is "withholding is undetectable" the v1 position? | **Yes — and that is the honest answer only because the mechanism that was going to change it was never written.** The spec must say so in its own words instead of pointing at a phantom. §5 |
| *(not asked)* | **`entity-workbench-go` advertises `signed_pointer` and emits no `published-root`.** Its `MANIFEST_GET` serves a transport-profile entity where §6.5.3.1 requires `system/peer/published-root`. §6 |

**The one-line version:** the static publishing surface is fully specified on the *read* side and
unspecified on the *write* side, because the write side was deferred to a document nobody wrote.

---

## §1 The phantom — `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE` does not exist

Three places in `EXTENSION-NETWORK` hand a normative obligation to that name:

| where | what it is handed |
|---|---|
| §6.5.3.1, the `MANIFEST_GET` **body MUST** | *"its revocation primitive is defined in `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE` (this section pins only the cache bound)"* |
| §6.5.6, the Amendment-10 **signed-root closure MUST** | *"participates in the §6.5.3 / `PEER-MANIFEST-STATIC-HANDSHAKE.md` §1.1 (planned) walk-from-signed-root threat model"* |
| §6.5.5, the consumer-mode **conformance contract** | *"signature-verify on the manifest (when present per `EXTENSION-STORAGE-SUBSTITUTE-HTTP.md` §3-RES.4 (planned))"* — a second absent document, same shape |

**Searched exhaustively, and the negative is proven, not inferred.** `find -iname '*MANIFEST-STATIC*'`
and `-iname '*STATIC-HANDSHAKE*'` across the entire ecosystem checkout — all eleven repos,
not this one — return nothing; a content grep for the string across `*.md`, `*.rs`, `*.go`, `*.py`
returns nothing outside the citing text itself. It is not archived, not renamed, not in a sibling
tree. It was never written.

**`PROPOSAL-PUBLISHED-ROOT-PREFIX-AND-REPUBLISH` already ruled that it should go.** Its §7 delta
**D4** reads: *"Replace the `(planned)` pointer to `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE` with the
§3.3a citation. Same at the `MANIFEST_GET` body MUST."* D1, D2, D3, D5, D6 and D7 all landed. **D4 did
not** — and the proposal sits in `implemented/`. This is **L3** (a partial fold does not get a version
bump) with the version bump already taken, and the residue was not cosmetic: it is the citation two
app-tier seats then built against.

**Why nothing caught it.** `spec address` classed citations — a doc token *governing a `§`* — and the
§6.5.3.1 mention governs none, so the scan skipped the line entirely. The §6.5.6 mention *does* carry
a `§`, and was suppressed by the `(planned)` marker: the analyzer implemented the first clause of
`SPECIFICATION-FORMAT.md` §11.4 and not the second, which permits a forward reference **only when
confined to a non-normative note or an Extension Points section**. A `(planned)` pointer inside a MUST
bullet passed clean. Both holes are fixed in arch-tools `46c6e50` (new `absent-doc` class,
`_in_note_context` confinement), with the §6.5.3.1 sentence as the regression case. **The gate now
fails on the text this proposal exists to fix**, which is the order those two things should happen in.

---

## §2 Q1 — the static publish shape, confirmed; the publish-side obligation, missing

**Confirmed, with a correction to the framing.** `system/peer/transport/http-poll` (§6.5.3) is not a
live-peer profile that a static bucket approximates. Its own first sentence is the static case:
*"A publisher peer using this profile has published its tree + content to a static HTTP origin
(typically a CDN-backed bucket); consumers poll and fetch the bytes directly."* Everything in
§6.5.3.1 is shaped by that constraint and says so — listings are **named objects** with no trailing
slash *"(it doesn't survive static CDNs)"*, **no redirects**, `content_layout` is a pure hash layout.
The **live** case (§6.5.6) is the one described as a collapse of the two-process story into one box.

So `PublishedRootClient` over an HTTP `ContentFetcher` is exactly right, and the composition in their
§1 is the intended one. Three reads, three anchors, one signature:

- `MANIFEST_GET` at `{manifest_url_prefix}` ⇒ `system/peer/published-root` wire entity (§6.5.3.1),
  normatively defined at `EXTENSION-TREE` §3.3a, signature at the §5.2 invariant pointer.
- `CONTENT_GET` at `{content_url_prefix}/{layout}/{hex(H)}` ⇒ bare 2-key `ECF({type, data})`,
  pure-body rehash.
- `TREE_GET` — **not on the signed path.** The hash-chain walk goes from `published-root.root_hash`
  through trie nodes by `CONTENT_GET`; a served `path → hash` binding is exactly the thing the threat
  model refuses to trust. Their pipeline diagram has this right and it is worth keeping right.

**What is actually missing, and it is the real content of Q1.** The signed-root closure obligation —
*upload the CHAMP root, every interior node, every leaf-bound content hash, the `published-root`
entity and its signature* — lives **only** in §6.5.6, is scoped to a live peer's `serve_scope`, and is
phrased as a resolution duty ("MUST also resolve, under the same `serve_scope`"). **A static bucket
has no `serve_scope` and resolves nothing.** For a static publisher the identical requirement is an
*upload* duty, and no landed text states it. A publisher that uploads only its path-bound entities
produces a site whose manifest verifies and whose walk 404s on the first interior node — the
`signed_pointer` machinery non-functional in its premised security model, which is precisely the
failure §6.5.6 was written to prevent for the live case.

---

## §3 Q2 — per-purpose tracked prefixes are intended, and already ruled

`EXTENSION-TREE` §3.3a: *"**Two conformant publishers may legitimately publish different extents.**
`prefix` is what distinguishes them: a publisher at `prefix: "system/"` commits to a strictly smaller
set than one at `prefix: "/"`. A consumer MUST read the extent from `prefix` and MUST NOT infer it
from the publisher's identity or from what it happens to find."*

That is the answer, and it is stronger than "permitted" — the `prefix` field is REQUIRED and exists
*because* extents legitimately differ. Tracking `sites/` is the intended usage. The three cohort
impls already publish three different prefixes (`system/`, `/{peer_id}/`, `/`) under
`PROPOSAL-PUBLISHED-ROOT-PREFIX-AND-REPUBLISH` §8.

**The reason a publisher would want `/` instead is narrow and worth stating:** the universal root is
the right choice only when the *consumer* needs an authenticated answer about the absence of a path
outside `sites/`. Under `prefix: "sites/"` a consumer learns nothing, authenticated or otherwise,
about `system/…` — which for a published website is the correct and desirable scope. Their instinct
(one root move per publish, not per autosave) is also the write-amplification argument §6.5.6 makes
for coalescing, arriving at the same place from the other side.

---

## §4 Q3 — coalescing is permitted; losing the final root is a conformance failure

`EXTENSION-NETWORK` §6.5.6, **bounded-convergence republish `[MUST; added 2026-08-08]`**:

> A publisher advertising a `signed_pointer` **MUST** republish its signed root after a change to the
> tracked prefix, such that the published root **converges** to the tracked root within a **maximum
> convergence delay of 30 s**. […] **Coalescing / debouncing is explicitly permitted** — a signature
> per `tree:put` is write-amplifying and is not the intent; the requirement is convergence, not
> immediacy (a cascade SHOULD produce one republish, not one per binding).

This resolves Q3 without needing the reproduction they correctly declined to claim:

- **Returning `None` while a publish is in flight is fine.** That is debouncing, and the MUST is
  written to permit exactly it.
- **Returning `None` and forgetting the newer root is not fine** *if* the tracked root can then sit
  unequal to the published root past 30 s. The obligation is on the **convergence**, not on any
  individual call. *"The in-flight one subsumes this call"* is true only when the in-flight publish
  will sign a root at least as new as the caller's — and, as they observed, it cannot, because it
  captured the older root before the caller ran.
- **So the guard must remember the pending root and re-publish on release** — or establish, at the
  seam, that the window is empty. Their own candidate reason (`publish` runs synchronously inside
  `on_tree_change`, single writer) may well be a sound proof; it is a proof about *their* scheduler,
  and the moment a background writer, a batch import, or a WASM task queue exists it lapses silently.

**This is testable, and the vector already exists.** `entity-core-go`'s `seed_republished` observes
convergence within a 10 s window three ways; §6.5.6 records 30 s as 3× headroom over that measurement.
A "burst then go quiet, assert published == tracked before the deadline" case is the shape that would
fail a coalescer with no pending slot. Recommended to the cohort as a conformance addition, not
authored here.

---

## §5 Q4 — freshness: "withholding is undetectable" is the v1 position, and the spec must say it

**The answer they asked for.** Yes. `seq` is a rollback defense, not a freshness proof
(`EXTENSION-TREE` §3.3a: *"`seq` monotonicity is the rollback defense: a consumer MUST reject `seq`
lower than one it has already accepted"*). A static origin serving `seq = N` forever is
indistinguishable from a publisher whose tree has not changed. There is no root `ttl`, no liveness
channel, and no landed text that obliges an origin to be current.

**The trap in the obvious fix, stated so nobody builds it.** `published-root` carries
`published_at`, inside the signed artifact — so a consumer *does* get a signed lower bound on the
artifact's age. It is tempting to read an old `published_at` as evidence of withholding. **It is
not.** `published_at` only advances when the publisher publishes, and a quiet tree legitimately holds
an old one indefinitely. A quiet publisher and a withholding origin are byte-identical to a consumer.
The 30 s maximum convergence delay does not help either: it binds *publisher tree → publisher root*,
never *publisher root → what an origin serves*.

**What to tell users — the surface word.** Not *"verified"* and not *"current"*, but
**"verified as of `published_at`"**, with the age visible. That claim is exactly true and exactly as
strong as the cryptography: the signature proves the publisher authored this root, the hash chain
proves every byte below it, and nothing proves a newer one does not exist. Their §4 item 4 — *"a page
that failed its hash check and one that was never checked must not look alike"* — is the right
discipline and this is its third state: **checked, and stale-able.**

**Where this bites, and it is the load-bearing part.** For revocation — a withdrawn share, a retracted
page — undetectable withholding means an origin can pin a consumer to the last root that still
contained the thing. Publishing is therefore **append-durable, not retract-durable**, against a
hostile origin. That is a real property of the v1 design and it belongs in the spec as a stated
position, not as a pointer at a document that does not exist.

**Not proposing a mechanism.** They did not ask for one and this proposal does not invent one. The
design space (a signed heartbeat republish with an unchanged `root_hash` and an advertised interval; a
root `ttl`; a liveness channel) is named here only so the next session does not re-derive it — and
per **L11**, choosing inside it requires reading the exploration corpus first, which this proposal has
not done and does not pretend to.

---

## §6 The finding their Q1 surfaced — `entity-workbench-go` publishes a manifest that is not one

**Source-read at `ca2e0cb`, clean tree.** `publish/publish.go` `writeManifest` builds a
`types.HTTPPollProfileData`, ECF-encodes it as a `system/peer/transport/http-poll` entity, and writes
it to `{outDir}/manifest` — the file served at `ManifestURLPrefix = {origin}/manifest`. In the same
struct it sets `SignedPointer: "system/peer/published-root"` and
`Freshness: "static-immutable+signed-pointer"`.

Three consequences, and the guards on the path are accounted for — there is no other writer:
`grep -rn 'published-root|published_root|PublishedRoot|PublishRoot' --include=*.go` over the whole
repo returns that one line, and `publish.go` performs no signing at all.

1. **`MANIFEST_GET` returns the wrong entity type.** §6.5.3.1 pins that route's body to
   `system/peer/published-root`. A conformant consumer decodes a transport profile where a signed
   root should be, and there is no signature to verify.
2. **Advertising `signed_pointer` incurs obligations nothing satisfies** — the §6.5.6 Amendment-10
   closure MUST and the bounded-convergence republish MUST both trigger on advertisement, and neither
   is reachable without a `published-root` to converge to.
3. **The `published_at` / `seq` freshness surface of §5 does not exist on a workbench-go site**,
   so the honest user-facing word there is currently *"unverified"*, not *"verified as of T"*.

**This is not carelessness on their part, and the framing matters.** The comment at
`publish/publish.go:126` names that file *"manifest — signed-handshake wire entity"*. They built to
the same absent document browser-rust asked about, reached a different reading of it, and shipped.
**Two app-tier seats, one phantom, two incompatible static surfaces** — which is exactly the failure
mode `APP-CONVENTION-CHAT` was created to stop, arriving one tier down. The remedy owed by arch is
§7's deltas; the remedy owed by workbench-go is a `published-root` emitter and a decision about where
the transport profile lives (it is discovered out-of-band per §6.5.4 — a sibling file is fine, the
`{manifest_url_prefix}` slot is not).

---

## §7 Spec deltas (executable form — the edits this proposal authorizes at fold)

| # | File | Section | Edit |
|---|---|---|---|
| **D1** | `EXTENSION-NETWORK.md` | §6.5.3.1, `MANIFEST_GET` body MUST | **Land D4 of `PROPOSAL-PUBLISHED-ROOT-PREFIX-AND-REPUBLISH`, at last.** Delete *"its revocation primitive is defined in `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE`"*; keep the cache bound. Replace with the §5 position, normatively: the manifest is mutable and MUST NOT be `immutable`-cached; `seq` monotonicity is a rollback defense and **not** a freshness guarantee; **no landed mechanism obliges an origin to serve the newest root, and a consumer cannot distinguish a withholding origin from a quiet publisher.** |
| **D2** | `EXTENSION-NETWORK.md` | §6.5.6, Amendment-10 closure MUST | Replace the `PEER-MANIFEST-STATIC-HANDSHAKE.md §1.1 (planned)` citation with `EXTENSION-TREE.md §3.3a`. The threat model it names is defined there. |
| **D3** | `EXTENSION-NETWORK.md` | §6.5.3, new bullet | **The publish-side closure obligation for the static case** (§2). A publisher that advertises `signed_pointer` on an `http-poll` profile MUST upload, under `content_url_prefix`, the transitive hash-linked closure of `published-root.root_hash` — trie root, interior nodes, leaf-bound content hashes — plus the `published-root` entity and its `system/signature` at the `ENTITY-CORE-PROTOCOL.md` §5.2 invariant pointer. Same obligation as §6.5.6's, in the form a bucket can satisfy: **resolve** becomes **upload**. Cross-reference both directions so neither reads as the only home. |
| **D4** | `EXTENSION-NETWORK.md` | §6.5.5 | The `EXTENSION-STORAGE-SUBSTITUTE-HTTP.md §3-RES.4 (planned)` citation is the same phantom class — resolve it to the landed home (`EXTENSION-SUBSTITUTE` §7 / §6.5.3.1's `MANIFEST_GET` MUST) or drop the parenthetical. **Adjudicate before folding; do not fold blind.** |
| **D5** | `EXTENSION-NETWORK.md` | §6.5.4 *Discovery and freshness* | One sentence pinning where a **static** publisher's transport profile lives — out-of-band per this section, **and specifically not at `{manifest_url_prefix}`**, which §6.5.3.1 reserves for the signed root. This is the workbench-go collision, stated as a rule rather than as a peer's build state. |
| **D6** | `EXTENSION-TREE.md` | §3.3a | Two sentences on `published_at`: it is a signed lower bound on the artifact's age and **MUST NOT** be read as evidence of origin freshness, because a quiet publisher and a withholding origin are indistinguishable. Consumer-facing wording guidance ("verified as of `published_at`") is guide material, not here. |
| **D7** | `guides/GUIDE-SERVING-MODE.md` | — | Carry the D6 wording guidance and the three display states (never checked / failed / verified as of T) where an app author will find it. |

**Not in scope, deliberately:** any freshness *mechanism*; any change to `seq`, `predecessor`, or the
30 s maximum convergence delay; anything about `TREE_GET` on the signed path.

---

## §8 Cohort impact

| Seat | Owed at fold |
|---|---|
| **`entity-browser-rust`** | Nothing blocking — D3 confirms the closure upload their pipeline needs, and Q1–Q4 are answered here. Their sequencing call (ship signing, let names wait) is endorsed in §9. |
| **`entity-workbench-go`** | Emit a real `published-root` + signature, or stop advertising `signed_pointer`. Move the transport profile off the `{manifest_url_prefix}` slot (D5). Both are §6 findings against landed text, not new obligations. |
| **`entity-core-{go,rust,py}`** | Nothing. D1–D7 touch the static publish surface and the citation graph; the republish MUST, `seq`, and `prefix` are unchanged. |
| **oracle** | Optional: a burst-then-quiet convergence vector (§4). `seed_republished` covers the steady case already. |

**D1–D7 cost no implementation a behavior change** except workbench-go, whose change is to become
conformant with text that landed 2026-08-08.

---

## §9 The sequencing question they raised, endorsed

Their call — **signing does not depend on the registry; ship (1) emit and (2) consume, let names
wait** — is correct and follows from the layering rather than from the timeline. A signed tree
verified against a peer-id already held needs no name resolution: `verify_signed_root(…,
expected_peer_id)` closes without a registry in the loop. The registry converts *"a key you already
have"* into *"a name you can type," and a name over an unverified tree is the half that carries no
guarantee. **Shipping names over unverified trees would be the wrong half** — agreed, and it is the
same argument `EXTENSION-REGISTRY` §6a.1 makes when it says the backend's only registry-specific
substance is trust verification.

---

## §10 Open for review

1. **D4's adjudication** — is `EXTENSION-STORAGE-SUBSTITUTE-HTTP §3-RES.4` a rename of a landed
   `EXTENSION-SUBSTITUTE` section, or a second never-written document? Not resolved here; resolving
   it by guess is how the first one propagated.
2. **D3's placement** — §6.5.3 (with the profile) versus a new §6.5.3.2. Placed with the profile on
   the reasoning that a publisher reads the profile definition and may never reach §6.5.6, which is
   titled for a live peer.
3. **Does D6 belong in `EXTENSION-TREE` or `EXTENSION-NETWORK`?** `published_at` is declared in TREE
   §3.3a and the misreading is a *consumer* behavior, which is NETWORK's surface. Placed at the
   declaration on the §3.3a principle that the interpretation of a signed artifact stays inside it.
