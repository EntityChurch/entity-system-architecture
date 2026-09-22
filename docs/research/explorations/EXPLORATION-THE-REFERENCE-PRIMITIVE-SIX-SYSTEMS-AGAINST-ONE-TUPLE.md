# EXPLORATION — the reference primitive: six systems against one tuple

**Status:** Exploration (design record). Not a proposal, not normative.

**Operator-requested, 2026-09-06 (second pass).** The brief: we are not inventing link-following once
for sites, again for feeds and again for content references — it gets standardized once. Survey Nostr,
ATProto, BitTorrent, Nix and whatever else can be gathered, so the design is right and the edge cases
are covered. The goal is that everything fits together and everything is reduced to its logical
primitive form.

**Predecessor:** `EXPLORATION-THE-LINK-AND-THE-WALK-…` (same day) established the problem — nine
reference shapes, five discriminators, two peer terms. **This document does the reduction, and it
corrects that document's largest claim.**

---

## §0 The result

**One tuple, four terms, and every system surveyed is a projection of it.**

```
reference = (identity, authority, hints, expectation)
```

| Term | Question | Mandatory when |
|---|---|---|
| **identity** | *what* | **always** — this is the reference |
| **authority** | *who vouches / who publishes* | the identity is **not** self-verifying, **or** the intent is *"whatever is current"* |
| **hints** | *where to look first* | **never** — droppable by construction, or it is not a hint |
| **expectation** | *what I saw when I linked* | you want the rug-pull guarantee |

**And there is a fifth thing that is not a term of the reference, which is the discovery every mature
system grew: the resolution record.** Nix's `narinfo`, ATProto's DID document, Nostr's `kind:10002`,
IPFS's provider record. **A reference does not carry where-to-look; it carries enough to *ask*.**

**The correction, and it is the most important thing in this document.** The predecessor filed D-38 as
*"'fetched from anywhere' names a mechanism the substrate does not have."* **That is too strong, and
opening `EXTENSION-NETWORK` is what showed it.** §6.5.5 **Mode A1** specifies the composition outright:

> *"On local content-store miss the dispatcher resolves `system/peer/transport/{publisher_peer_id}/*` →
> finds the `http-poll` profile entity (§6.5.3) with `tree_url_prefix`, `content_url_prefix`,
> `content_layout`… builds the URL; performs GET + hash-verify (no BRIDGE-HTTP import — Mechanism A);
> ingests."*

**So a reader holding `{peer, hash}` has a specified, normative path to the bytes, requiring no
preconfigured substitute source.** What remains true from D-38 is narrower and still real: nothing
answers *"who has this hash?"* for a publisher you cannot resolve or who is gone. **The gap shrank from
a subsystem to an edge case** — see §4.

**The second result, and it makes the two open items one item.** **Mode A1's input is a publisher
peer-id.** Seven of our nine reference shapes do not carry one. **So "the shape lacks a peer term" and
"the reader cannot fetch it" are not two findings — they are the same finding**, and D-37 subsumes the
useful half of D-38.

**The third result: the missing term is the one three deployed systems added independently, for the
identical stated reason.** Matrix: *"Room IDs are not routable on their own as there is no reliable
domain to send requests to"* → `via`. Nostr: a bare `note1` cannot be located → `nevent` with relay
hints. BitTorrent: an infohash names bytes and no holder → `tr` / `ws` / `xs` / `as` / `x.pe`.
**Ours is `hints`, and we have no slot for it.**

---

## §1 Six systems, one table

**Every row is the same decomposition.** Read down the columns: the shape of the gap is identical
everywhere, and the fill differs only in where it is stored.

| | **identity** | **authority** | **hints** | **expectation** | **resolution record** |
|---|---|---|---|---|---|
| **BitTorrent** | `xt` infohash | — *none* | `tr` tracker · `ws` webseed · `xs` exact source · `as` acceptable source · `x.pe` peer | = identity | `.torrent` / DHT |
| **Nix** | store-path digest | `Sig` (`cache-key:sig`) | substituter list (config) | `NarHash` + `FileHash` | **`narinfo`** |
| **IPFS** | CID | — *none* | gateways, delegated routers | = identity | **provider record** |
| **Nostr** | event id / `d` tag | pubkey (`p` tag, `nevent` author TLV) | relay hints in `nevent`/`naddr` | = identity | **`kind:10002`** (NIP-65) |
| **ATProto** | rkey / CID | **DID** | — *(in the DID doc)* | `strongRef.cid` | **DID document** → PDS |
| **Matrix** | `event_id` / `room_id` | — *(room ids are opaque)* | **`via=`** | event hash | room directory |
| **Web** | path | origin (host) | — *(the host is the hint)* | — *none* | DNS |
| **Ours** | `content-hash` **or** `path` | `peer` — **`FEED` only** | **none** | `hash` / `seen` | REGISTRY binding *(peer axis only)* |

**Four observations, and the third is the design.**

**(a) Self-verifying identity removes the need for authority and creates the need for hints.** Look at
BitTorrent, IPFS and Nix: all three have no authority term on the reference (Nix's `Sig` is on the
*record*, not the link) — because the hash settles trust. **And all three have the richest hint
vocabularies in the table.** *Content addressing eliminates the need to trust the source and
simultaneously eliminates the ability to find one.* That trade is not a defect anywhere; it is the
shape of the thing, and every one of them paid for it with a discovery layer.

**(b) The systems with a strong authority term have the weakest hint vocabularies**, because the
authority *is* the route. ATProto puts nothing in the URI — *"the 'authority' part of an AT URI does not
indicate a network location… the hostname is only used for identity lookup"* — because the DID document
resolves to a PDS. The Web has no hint parameter because the origin is the host. **Ours is in this
family and that is the correct family**, which is why §4's answer is better than the predecessor
implied.

**(c) The hint taxonomy is ranked, and BitTorrent's is the most evolved.** `xs` (exact source) > `as`
(acceptable source) > `ws` (webseed) > `tr` (tracker, a *discovery service* rather than a source) >
`x.pe` (a bare peer address). **These are not five ways to say "location" — they are five confidence
levels**, and a client tries them in order. Our own ladder vocabulary already exists in the same shape
(`GUIDE-RESOLUTION` §4: *try nearest/most-authoritative → fall through a preference-ordered ladder →
honest typed dead-end*). **We have the ladder discipline and no ladder rungs to put in a link.**

**(d) Nobody puts the hint list *only* in the link.** Every one of them also has a maintained record.
§3.

---

## §2 What a resolution record is, and why Nix's is the one to study

**Nix's `narinfo` is the cleanest separation of the four terms in the entire survey**, and it is worth
reading as a data structure rather than as packaging:

```
StorePath: /nix/store/s549276qy…-hello-2.12.1     ← identity
URL:       nar/1490jfgjs….nar.xz                  ← hint (relative to the substituter)
Compression: xz                                    ← how to decode
FileHash:  sha256:1490jfgjs…                       ← expectation, of the compressed bytes
NarHash:   sha256:18xzh7bjn…                       ← expectation, of the content        [REQUIRED]
References: … …                                    ← the closure — outbound edges
Sig:       cache.nixos.org-1:irA3Fgc…              ← authority, over the whole record
```

**Five properties worth taking, each of which we either have or lack:**

1. **The identity is not the location.** `StorePath` is asked for; `URL` is answered. **Our
   `content_url_prefix + layout + hash` construction is the same move** — the hash is the question, the
   URL is derived. ✓ we have this.
2. **The integrity term is separate from the identity term, and it is mandatory.** Nix's store path is
   *input-addressed* (a hash of how it was built), so `NarHash` is genuinely independent and the parser
   *rejects the record without it*. **Ours are the same value**, because our identity is already
   content-addressed. ✓ ours is strictly simpler, and this is the one place we are ahead of Nix.
3. **The authority signs the record, not the bytes** — because the bytes are self-verifying and the
   *claim* is not. **This is exactly `SUBSTITUTE` §7.2's manifest rule** (*"the `path_index` is an
   authority claim about the publisher's tree; anyone serving the URL can forge it, so hash-verifying
   the manifest against its own hash proves nothing about authorship"*). ✓ independently derived, same
   sentence.
4. **`References` makes the closure explicit** — the record names its own outbound edges, so a client
   can complete a fetch without parsing the content. **This is the edge inventory D-39 asks for**,
   solved at the record layer rather than the type layer. **We have no equivalent**, and it is the
   cheapest known answer to *"what do I need to fetch next?"*
5. **The record is served at a well-known key derived from the identity** — `<hash>.narinfo`. **So
   discovery is a construction, not a search.** ✓ this is precisely our `content_url_prefix` shape.

> **The reduction Nix demonstrates: you do not need a distributed lookup system if you have (a) a
> derivable location and (b) a list of places to try.** Nix has ~one substituter for most users and a
> global cache; it never built a DHT and does not need one. **Our position is Nix's position, not
> BitTorrent's**, and §4 is why.

---

## §3 The two homes for a hint — and the mature answer is both

**A hint can live on the link or on the identity, and the field split on it before converging.**

| | **On the link** | **On the identity** |
|---|---|---|
| Examples | magnet `tr`/`ws`/`xs`, Matrix `via`, Nostr `nevent` relay hint | Nostr **NIP-65** `kind:10002`, ATProto **DID doc**, Nix substituter config |
| Works for a stranger | **yes** — it travels with the reference | only after you can resolve the identity |
| Freshness | **frozen at link time; rots** | **maintained by the party who knows** |
| Update cost | rewrite every link ever published | one record |
| Failure | dead tracker, dead relay | none, if resolvable |

**Nostr is the instructive case because it has both, deliberately.** NIP-19 puts relay hints in
`nevent`/`nprofile` *"such that other apps can locate and display these entities more easily"*, and
NIP-65 publishes a per-author read/write relay list so *"clients should send events to the author's
**write** relays, and consult a tagged user's **read** relays when fetching mentions."* **The link hint
is the bootstrap; the published list is the steady state.** They are not competing designs and the
protocol carries both.

> **This settles a question that was about to be argued as either/or on our side.** **D-27 (the
> publishable resolver view) is the *on-the-identity* home, and it is NIP-65.** The hint term this
> document proposes for the reference atom is the *on-the-link* home, and it is `via`. **Both, and the
> reason for both is that they fail in opposite directions.**

**One caution the field also supplies, and it is why hints must be droppable.** Nostr's relay hints are
widely observed to rot, and the ecosystem's answer was not to strengthen them but to add NIP-65 beside
them. IPFS measured the same decay (`REFERENCE-PRIOR-ART-…` §3: >70% of provider records pointing at
unreachable peers, *"record staleness"* and *"address staleness"* as distinct failures). **A hint that a
consumer is allowed to trust is not a hint; it is an under-specified authority term.** Hence: **hints
are advisory, unsigned, unverified, and a conformant consumer that ignores every one of them MUST still
reach the same answer, more slowly.** That sentence is the whole safety property.

---

## §4 Our chain, re-measured — and D-38 was too strong

**This section corrects the predecessor. The correction came from opening `EXTENSION-NETWORK`, which
that document listed under *what is unread*.**

**What actually exists, verified by section:**

| Step | Mechanism | State |
|---|---|---|
| `{peer, hash}` → publisher's transport profile | `system/peer/transport/{peer_id}/{profile-id}`, **self-published** (NETWORK §6.5.1a) | **SHOULD**, and *"consumers MUST NOT assume the self-published path exists"* |
| profile → content URL | `http-poll` profile carries **`content_url_prefix`**, `content_layout` (NETWORK §6.5.3) | **built, normative** |
| URL → verified bytes | **Mechanism A** — inline GET, hash-verify, discard on mismatch (`SUBSTITUTE` §7.1) | **built, "the load-bearing baseline"** |
| the whole composition | **NETWORK §6.5.5 Mode A1**, dispatcher-driven | **specified; impls SHOULD provide it** |
| peer → transports, when not self-published | registry binding carries transports (`GUIDE-RESOLUTION` §4 Layer B, §8) | **built** for a resolvable name |

**So the honest statement is the opposite of the predecessor's.** A reader holding a `FEED` `reference`
— `{peer, hash, ?path}` — resolves the peer, reads `content_url_prefix`, constructs the URL, fetches,
and hash-verifies. **No configured substitute source, no DHT, no manifest.** The pieces were built to
fit: `SUBSTITUTE` §2.2 says its endpoint shape matches the NETWORK profile *"so that a peer's published
transport profile and its substitute entries describe the same URL space without duplication,"* and
**opening NETWORK §6.5.3 confirms the field name and role are identical.**

**What is genuinely still open, and it is now three narrow cases rather than a subsystem:**

1. **A peer-id you cannot resolve.** §6.5.4 is explicit: *"How a consumer learns of a publisher's
   transport profiles is out of scope of this section. For v1: out-of-band… Post-corridor:
   EXTENSION-REGISTRY will define peer-ID → endpoint-set resolution."* **A raw peer-id with no registry
   binding and no self-published profile has no route.** ← **this is what a `via` hint fixes, and it is
   Matrix's exact sentence.**
2. **A publisher who is gone.** The pinned hash stays valid forever and there is no way to ask anyone
   else for it. **This is the only case that would need a content-routing layer**, and the DHT axis was
   already measured and declined for good reasons.
3. **Seven of nine shapes cannot enter step 1 at all**, because Mode A1's input is a publisher peer-id
   and they carry none. **This is D-37, and it is the case that actually bites**, because it is not an
   edge case — it is `SITE`, `EMBED` and `SHARE`.

> **So the sentence *"fetched from anywhere"* is still wrong, but the fix is smaller than filed.** It
> should read: **"fetchable from the publisher's declared content origin, or from any source that has
> it and that you can reach."** True, precise, still a real advantage over ATProto (which has no
> equivalent of step 2–3 for a moved record), and it promises no discovery.

---

## §5 The reduction — one atom, and our nine shapes are projections of it

**The claim: all nine app-tier shapes are the same tuple with terms omitted, and the omissions are not
deliberate.**

| Shape | identity | authority | hints | expectation |
|---|---|---|---|---|
| `FEED reference` | `hash` | `peer` | *(`path` is doing this job)* | `hash` |
| `FEED live-reference` | `path` | `peer` | — | `seen` |
| `SHARE blob-target` | `hash` | — | — | `hash` |
| `SHARE prefix-target` | `path` | — | — | — |
| `EMBED pointer-payload` | `hash` | — | — | `hash` |
| `EMBED child-payload` | `path`\|`hash` | — | — | — |
| `EMBED img-src` | `hash` | — | — | `hash` |
| `SITE embeds` | `path`\|`hash` | — | — | — |
| `SITE link-ref` | *(a string)* | — | — | — |

**Three things fall out.**

**(a) `FEED`'s two atoms are already the primitive, minus the hint term.** The reduction does not
invent a shape — it says the app tier has one and seven documents do not import it. **The work is
promotion, not design**: lift the atom to a shared home the whole tier references, per the same rule
`FEED` §2 states for type tags (*"the cross-impl contract is the type tag, not the path"*).

**(b) The two-shape encoding survives the reduction and should not be collapsed.** In the frame above,
`reference` and `live-reference` look like one tuple with a different term marked authoritative — which
invites merging them into one shape with an optional `hash`. **`FEED` §2.2.3 already refuted that**, and
the refutation is right: under an optional hash, an absent one is indistinguishable between *"the author
wants live"*, *"the implementation did not populate it"* and *"the author only had a URL"* — **one intent
and two bugs.** Corroborated across four systems, which all express the split as a distinct shape or name
and none as an optional field. **The unified frame explains *why* there are two shapes; it does not
license one.**

**(c) `path` is currently doing two jobs in `reference`, and separating them is the cleanest thing in
this document.** In `FEED reference`, `path` is *"a starting point, not an address of record"* — which
is, exactly and precisely, **a hint**. It is the only hint in the corpus and it is (i) singular,
(ii) untyped as a hint, and (iii) the same field name that is *authoritative* in `live-reference`.
**Naming it as a hint costs nothing and makes the atom regular:**

```cddl
; the shape the reduction suggests — NOT PROPOSED, illustrative
reference = {
  peer: peer-id,               ; authority
  hash: content-hash,          ; identity + expectation (they coincide; that is our advantage)
  ? via: [* hint]              ; ADVISORY, droppable, unsigned — a conformant reader may ignore all of them
}
hint = tree-path / url / peer-id     ; ranked by position, magnet-style
```

**Note what the hint slot buys that nothing else does: it is the only term that can carry a route for a
peer the reader cannot resolve** (§4 case 1) **and the only one that survives the publisher going away**
(§4 case 2, partially — a mirror named in the link outlives its origin). **Both of the two genuinely
open cases are addressed by the one term the field converged on, and we have no slot for it.**

---

## §6 Edge cases the field has already hit — the operator asked for these explicitly

**Each is sourced from a system that hit it in production, with our exposure named.**

| # | Edge case | Where it was learned | Our exposure |
|---|---|---|---|
| 1 | **Hints rot silently** — a dead relay/tracker looks like an absent object | Nostr relay hints; IPFS *"address staleness"* | **would be new** with `via`. Mitigation is the safety property in §3: advisory, and ignoring all hints reaches the same answer |
| 2 | **The identity is not routable on its own** | Matrix room IDs, verbatim | **live now** — §4 case 1 |
| 3 | **Handle reassignment invalidates a reference** | ATProto: *"if reassigned, the URI becomes invalid"* — hence DIDs for durable refs | **live**, and it is **D-31**: a peer-id is a handle key and handle keys rotate. `quorum_id` is our DID-equivalent |
| 4 | **The rug-pull** — quote something, author overwrites it | ATProto `strongRef`'s published rationale | **closed** — `FEED` §2.2.3 keeps `reply` pinned and refuses to widen |
| 5 | **Compression/encoding changes the location but not the identity** | Nix: NAR named by `FileHash`, prompting a proposal to name by `NarHash` so the URL is reconstructible | **avoided** — our URL is derived from the content hash, not from a transfer encoding |
| 6 | **The record claims a closure it does not serve** | Nix `References`; our own §7.3 publishing-order discipline | **named** — `SUBSTITUTE` §7.3 (blobs first, manifest last) and NETWORK Amendment 10's closure MUST |
| 7 | **A signed record answers the wrong question** | our own REGISTRY §6a.4 `binding.name == norm` | **closed**, and it is the same class as Nix verifying `Sig` before trusting `URL` |
| 8 | **Deep-linking into a partially-fetched object** | BitTorrent `so` (select-only), *"deep links to a particular file before metadata download completes"* | **open and unasked** — this is the anchor ladder (D-31 / durable-reference §5), rung 1 |
| 9 | **The discovery service becomes the centralization** | IPFS delegated routing exists *because* the DHT is too slow for light clients — and it is an HTTP server | **watch item.** Our equivalent is the aggregator/Mode-A relay, already deferred |

**Edge case 9 is the one worth stating as a principle**, because it is Hyper-G's lesson arriving on a new
noun: **every content-routing layer surveyed has degraded toward "ask a well-known HTTP server."** IPFS
built a DHT and added delegated routing over HTTP; BitTorrent had trackers, added a DHT, and webseeds
came back. **Our origin tier is that HTTP server, natively, from day one.** *The convergent endpoint of
every P2P discovery design is the thing we already have* — which is the strongest argument in this
document for not building content routing, and for spending the effort on §5's hint term instead.

---

## §7 What this changes on the docket

**No new items. Two existing ones change materially and one is subsumed.**

| Item | Change |
|---|---|
| **D-37** (the census / one atom) | **Promoted to the primary item.** §5 gives it the target shape and the argument: the app tier already has the atom, seven documents do not import it, and **Mode A1's input is the term they omit** — so this is the same finding as the fetch gap |
| **D-38** (*"fetched from anywhere"*) | **Substantially reduced, and partly withdrawn.** Mode A1 exists and is normative; the composition is built. What remains: **scope the sentence** (a one-line edit, still worth doing, three homes) and **the two narrow cases in §4**. **It is no longer a candidate for new machinery** |
| **D-27** (publishable resolver view) | **Strengthened and named: it is NIP-65.** §3 shows the on-the-identity hint home is a deployed pattern with a published rationale, and that it is **complementary to, not competing with,** a link-carried hint |
| **new, folded into D-37** | **The `via` hint term.** Not a separate item — it is the one term §5 adds, and proposing the atom without it would repeat the omission this document exists to find |
| **D-39** (the walk contract) | Unchanged, still gated on D-37. **§2 point 4 adds one input**: Nix's `References` is the edge inventory solved at the record layer, which is worth weighing against solving it at the type layer |

**The sequencing this suggests, and it is now short:** D-37 (promote the atom + add `via`) → D-38's
one-line scoping edit alongside it → D-39. **D-27 is independent and can go any time.**

---

## §8 Sources

**Opened this session, primary:**
[NIP-65 — Relay List Metadata (the outbox model)](https://github.com/nostr-protocol/nips/blob/master/65.md) ·
[NIP-19 — bech32-encoded entities](https://github.com/nostr-protocol/nips/blob/master/19.md) ·
[NIP-21 — the `nostr:` URI scheme](https://github.com/nostr-protocol/nips/blob/master/21.md) ·
[ATProto — AT-URI scheme](https://atproto.com/specs/at-uri-scheme) ·
[Matrix spec — appendices, `matrix.to` and `via`](https://spec.matrix.org/latest/appendices/) ·
[IPFS — Delegated Routing V1 HTTP API](https://specs.ipfs.tech/routing/http-routing-v1/) ·
[Nix — store path specification](https://nix.dev/manual/nix/2.24/protocols/store-path.html) ·
[BEP 53 — magnet URI `so` parameter](https://www.bittorrent.org/beps/bep_0053.html) ·
[Magnet URI scheme — full parameter list](https://en.wikipedia.org/wiki/Magnet_URI_scheme) ·
[A Nix Binary Cache Specification (narinfo fields)](https://fzakaria.com/2021/08/12/a-nix-binary-cache-specification) ·
[NixOS Wiki — Binary Cache](https://wiki.nixos.org/wiki/Binary_Cache) ·
[nix `src/libstore/nar-info.cc` — required fields](https://github.com/NixOS/nix/blob/master/src/libstore/nar-info.cc)

**In-corpus, opened this session by section:** `EXTENSION-NETWORK` §6.5.1a, §6.5.3, §6.5.4, §6.5.5
(**the correction in §4 comes from here**) · `EXTENSION-SUBSTITUTE` §7.1, §7.2, §7.3, §7.4 ·
`REFERENCE-PRIOR-ART-…-BY-AXIS` §3. **Carried from the predecessor**, opened there:
`GUIDE-RESOLUTION` (full) · `PROPOSAL-APP-CONVENTION-FEED` §2–§2.2.4 · `APP-CONVENTION-*` shape blocks ·
`EXTENSION-SUBSTITUTE` §1/§2.1/§2.2/§10 · `EXTENSION-REGISTRY` §4.

## §9 What is unread, named so nobody assumes coverage

- **The magnet parameter table is from Wikipedia, not from a BEP.** BEP 53 was opened and covers only
  `so`. **`xs`/`as`/`ws`/`x.pe` have no single normative home** — they are conventions across BEP 9,
  BEP 19 and client practice. §1's ranking of them is **this document's reading**, defensible and not
  quoted from a spec. Before any of it is cited normatively, read BEP 19 (webseeding) directly.
- **Nix's narinfo has no formal specification.** The field table is from `nar-info.cc` via a secondary
  source plus the NixOS wiki. The *required-fields* claim (StorePath, NarHash, URL, non-zero NarSize)
  is from that reading of the source, **not from opening the source**.
- **ATProto DID resolution end to end** — `did:plc` directory mechanics and the PDS discovery hop.
  §1's DID-doc row is from the AT-URI spec's statement that the authority is *"only used for identity
  lookup"*; **the resolution spec itself was not opened.**
- **NIP-01** — the base event and tag model (`e`, `p`, `a` tags and their relay-hint positions). §1's
  Nostr row is assembled from 19, 21 and 65; **the tag-level hints were not read at the source.**
- **Our own `EXTENSION-CONTENT` §2.4 descriptors and §5.3 path-index-by-listing**, named by
  `SUBSTITUTE` §7.4 as a third publisher pattern. **Not opened**, and §4's table would likely gain a
  row from them.
- **`entity-browser-rust` and `entity-workbench-go` trees.** Still unmeasured. §5's claim that the
  omissions *"are not deliberate"* is an inference from the specs; **a seat may have already worked
  around this**, and D-37 should open both trees before the atom is proposed (L21's fifth shape).
