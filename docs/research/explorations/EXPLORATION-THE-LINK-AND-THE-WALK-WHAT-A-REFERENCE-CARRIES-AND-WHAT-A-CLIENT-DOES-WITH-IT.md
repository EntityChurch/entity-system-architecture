# EXPLORATION — the link and the walk: what a reference carries, and what a client does with it

**Status:** Exploration (design record). Not a proposal, not normative.

**Operator-requested, 2026-09-06.** *"What we're building is obviously something like hypertext,
hypermedia… right now when we put a site link in the current mode, it's just a name of a site, which
doesn't tell us enough to trace the route. How does this map to a peer? How does it map to a site's
definition?"* And: *"the client knows how to walk this… however we define the browser is the client
glue code that understands all these specifications and semantics and data models and the registry
flows and the lookups."*

**Why this is a new axis and not a re-run.** `REFERENCE-PRIOR-ART-…-BY-AXIS` §0 lists what it does not
cover, and `EXPLORATION-THE-RESOLUTION-CHAIN-…` §0 opens by finding that *"the prior-art library has
eight axes and the one this question needs is not among them."* The eight are relay, inbox/outbox, DHT,
federation, authority, user-facing identity, change detection, economics. **The link form itself — what
a reference carries and whether a client can act on it — is not one of them**, and it is what was asked
for.

**Companions, all read for this document:** `EXPLORATION-THE-HYPERTEXT-LINEAGE-…` (Xanadu, Hyper-G, the
Web's counter-design) · `EXPLORATION-THE-DURABLE-REFERENCE-…` (the three-part reference and the
three-way read) · `EXPLORATION-THE-READERS-LOOP-…` (§2, how four systems reconstruct a thread).
**Nothing below re-opens those; §6.4's decline of bidirectional links and §4's three-way read stand.**

---

## §0 The result, stated first

**The operator's complaint is correct, it is measurable, and it is narrower and worse than it sounds.**

**There are nine reference shapes in the application tier, discriminated five different ways, and only
two of them carry a `peer` term.** A reference without a peer term is not routable: there is no party
to ask. It is resolvable only if you already hold the bytes, or if some source you have already
configured happens to hold them.

| | Count |
|---|---|
| Reference shapes across `FEED`, `SITE`, `EMBED`, `SHARE` | **9** |
| Discrimination mechanisms among them | **5** — field name · explicit tag · CBOR type · a leading-`/` string rule · *"the renderer's classifier"* |
| Shapes carrying a first-class `peer` term | **2** — both in `FEED`, both drafted, neither landed |

**And the shape on the actual hypertext surface is the weakest one.** `APP-CONVENTION-SEMANTIC-CONTENT-SITE`
§4's `link-ref` — the target of a nav menu entry, the thing a user clicks — is `tstr` with the comment
*"a link the renderer's classifier resolves relative to the site root."* **There is no classifier
specified anywhere.** The word `classifier` occurs exactly once in the entire `specs/applications/`
directory, in that comment. Three forms are named as examples (`"./about"`, `"site:labs/intro"`,
`"entity://…"`) and no rule distinguishes them.

**Worse, one rung down: a prose link in a page body has no entity-native meaning at all.** §3.1 states
the convention *"adds no prose vocabulary — that's the base format's standard,"* so a markdown link in a
`site-page.body` is a CommonMark link handed to *"each front-end's own base renderer."* **The single most
common hypertext act in the system is unaddressed by any convention in the corpus.**

**The second result, and it is the one that changes sequencing.** The traversal story rests on three
mechanisms, and each is in a different state that the corpus currently describes with the same
confidence:

| Mechanism | Cited as | Actually |
|---|---|---|
| name → peer → transport → tree → hash → bytes | built | **built and strong** — `GUIDE-RESOLUTION` §4 |
| *"fetched from anywhere"* on a 404 | the capability that beats ATProto and Hyper-G | **overstated, but far less than this row first said — CORRECTED, see §4.1.** The publisher path is built and normative (`NETWORK` §6.5.5 Mode A1); only the unresolvable-publisher and publisher-is-gone cases are unbacked |
| a peer's own registry view, readable by a visitor | the operator's proposal | **not specified, and deliberately the opposite** — `REGISTRY` §4 stores `resolver-config` *"peer-local; not synced"* |

**None of that is a reason to build a DHT** — `REFERENCE-PRIOR-ART-…` §3 already measured that axis and
the finding holds (over 70% of IPFS provider records pointed at unreachable peers; every deployed fix
moves *back* toward addressability). **The finding is narrower: one sentence in a drafted convention
promises a mechanism the substrate does not have, and it is the sentence the design's advantage is
stated in.**

---

## §1 The census — nine shapes, five discriminators, two peers

**Read from the CDDL blocks, each section opened.** This is a census of the *slot* (*"how does an
application-tier document point at another thing?"*), not a count of a token — the distinction L8's
twentieth form exists to enforce.

| # | Convention · § | Shape | Peer term | Pin | Discriminated by |
|---|---|---|---|---|---|
| 1 | `FEED` §2.2 `reference` | `{peer, hash, ?path}` | **✓** | ✓ required | **field name** — `hash` xor `seen`, MUST reject both (§2.2.3) |
| 2 | `FEED` §2.2 `live-reference` | `{peer, path, ?seen}` | **✓** | expectation | **field name**, same rule |
| 3 | `SHARE` §2.2 `blob-target` | `{tag:"blob", hash}` | ✗ | ✓ | **explicit tag** |
| 4 | `SHARE` §2.2 `prefix-target` | `{tag:"prefix", path}` | ✗ | ✗ | **explicit tag** |
| 5 | `EMBED` §3 `pointer-payload` | `{tag:"pointer", hash}` | ✗ | ✓ | **explicit tag** |
| 6 | `EMBED` §3 `child-payload` | `{tag:"child", ref: (path / content-hash)}` | ✗ | either | tag at the payload; **CBOR type** inside `ref` |
| 7 | `EMBED` §7 `img-src` | `{tag:"hash"} / {tag:"inline"}` | ✗ | ✓ | **explicit tag** (*"godot — a hash IS bytes"*) |
| 8 | `SITE` §4 `site-page.embeds` | `[* (path / content-hash)]` | ✗ | either | **CBOR type**, untagged |
| 9 | `SITE` §4 `nav-node.target` (`link-ref`) | `tstr` | ✗ | ✗ | **nothing** — *"the renderer's classifier"* |
| — | `SITE` §3.2 `::embed{ref=…}` | string projection of #6 | ✗ | either | **leading `/`** (rule `F-1`) |

**Three observations, in ascending order of consequence.**

**(a) The tagging discipline is real and was applied unevenly.** `EMBED` §3's normative note is
explicit — *"`payload` is a tagged union… Decoders MUST reject an untagged/ambiguous payload (the
silent-divergence risk `G-PIN-2` closes)"* — and `SHARE` §2.2 carries the same `; TAGGED — no untagged
ambiguity` comment. **That discipline stops at the payload boundary and does not reach inside `ref`**,
where `path / content-hash` is discriminated by CBOR major type. Which is *sound on the wire* and is why
nobody caught it.

**(b) `SITE` §3.2 found this exact defect once, fixed it locally, and did not generalize.** `F-1`'s
finding is that *"at the wire layer CBOR types disambiguate `path` (tstr) from `content-hash` (bstr); in
the directive string they don't"* — so the directive got a normative leading-`/` rule. **The same
document's `link-ref`, twelve lines later in §4, is also a string, is also a union of those forms, and
got a renderer's classifier instead of a rule.** One string projection of a reference was hardened; the
other was not, in the same spec, by the same pass.

**(c) The peer column is the finding.** Shapes 3–9 identify *what* without identifying *who*. A
`content-hash` alone names bytes and no holder; a `path` carries `/{peer_id}/…` by V7 §1.4, so the peer
is *recoverable by string parsing* but is not a term. **`FEED` alone made "who published it" first-class,
and `FEED` is a DRAFT proposal** (`docs/proposals/active/applications/`). The landed app-tier specs have
no routable reference at all.

> **The one-sentence version, and it is the operator's sentence made precise: a reference is routable
> if and only if it carries a peer term. Two of nine do, and both are unlanded.**

---

## §2 What the deployed systems do — and Nostr's answer is the operator's complaint, verbatim

**This is the axis the prior-art library lacks, so it is sourced fresh.**

### §2.1 Nostr — two tiers of identifier, and the second exists *only* for sharing

NIP-19 defines bare identifiers (`npub`, `note`) and TLV-carrying entities (`nprofile`, `nevent`,
`naddr`, `nrelay`). The TLV types include **type 1 = relay** — *"for `nprofile`, `nevent` and `naddr`,
optionally, a relay in which the entity is more likely to be found"* — plus **type 2 = author** and
**type 3 = kind**.

**Their stated rationale is the whole finding:**

> *"When sharing a profile or an event, an app may decide to include relay information and other
> metadata such that other apps can locate and display these entities more easily."*

**The bare forms are for storage and protocol; the hinted forms are for links.** NIP-21's `nostr:` URI
scheme admits every NIP-19 form except `nsec` — so the thing you paste into a document is expected to be
the hinted one.

**That is the operator's complaint and Nostr's answer to it, arrived at independently.** A bare `note1…`
is *"just a name,"* and it does not tell you enough to trace the route; `nevent1…` bundles the id with
where to look and who wrote it. **Nostr shipped a second encoding rather than a second lookup system.**

### §2.2 ATProto — the reference is two-part, and the third part is a reserved empty slot

Already established in `EXPLORATION-THE-DURABLE-REFERENCE-…` §2 and not re-derived: `strongRef` is
`{uri, cid}` — *where to find it* plus *what you should find* — because *"content-addressed hashes aren't
resolvable by themselves in atproto; there's no global DHT you can throw a bare CID at."* Fragments are
*"supported in general syntax, not currently used,"* with JSON Path reserved.

**The relevant contrast for this document: ATProto's URI carries the authority (the DID) as a structural
term.** `at://did:plc:…/app.bsky.feed.post/…` is peer-first by construction. **Ours is peer-first in
`FEED` and nowhere else.**

### §2.3 HTML — the base URI is the mechanism, and it is why relative links work

The Web's answer to *"what does `./about` mean"* is not a classifier; it is a **base URI** resolved by a
specified algorithm, defaulting to the document's own address and overridable by `<base>`. Every
relative reference resolves against it deterministically, and two conformant browsers agree.

**`SITE` §4's `link-ref` has the relative form and no base-URI rule.** *"Resolves relative to the site
root"* names the base informally in a CDDL comment; it does not say what the site root is at resolution
time (§2's placement is publisher-chosen), what happens across a `passthrough_of` republish, or how
`site:` and `entity://` are told apart from `./`. **The Web needed a specification for exactly this and
wrote one; we have a comment.**

### §2.4 The pattern across all three

| System | Compact form | Shareable form | What the shareable form adds |
|---|---|---|---|
| Nostr | `note1`, `npub1` | `nevent1`, `nprofile1` | **relay hints**, author, kind |
| ATProto | bare CID | `strongRef` `{uri, cid}` | **the authority** (DID) + the pin |
| Web | relative path | absolute URL | **the origin** |
| **Ours** | `content-hash` | `FEED` `reference` `{peer, hash, ?path}` | **the peer** — *drafted, not landed* |

**Every deployed system separates a compact internal identifier from a routable external one, and the
difference is always the same thing: the authority you can go ask.** We derived the same atom in `FEED`
§2.2 and did not propagate it to the tier that actually renders links.

---

## §3 The walk — what a client must understand, and what is missing from it

**`GUIDE-RESOLUTION` §4 is better than its reputation and it answers half of this.** It specifies one
shape — *try nearest → fall through a preference-ordered ladder → honest typed dead-end* — over four
ladders (name, peer-id, content-hash, tree-path), and states the composition:

```
name ─REGISTRY▶ peer_id ─NETWORK▶ transport ─(connect)▶ tree-path ─TREE▶ hash ─SUBSTITUTE▶ bytes ▶ render
```

with the line that answers the operator's *"is there a browse engine?"* directly: *"There is no separate
'browse engine' — browsing **is** walking this chain and re-walking it on each click."*

**That is the vertical walk: one identifier down to bytes. It is specified, and §3's entry-point table
(*what I have → where I enter the ladder*) is exactly the right frame.**

**What has no home is the horizontal walk: given the bytes you just rendered, what are the outbound
edges and how do you follow one?** That question is answered per-convention, nine different ways, per
§1 — and for `link-ref` and prose links, not at all. **The chain guide assumes you have an identifier;
the gap is that the document you just rendered may not have given you one you can use.**

### §3.1 So is a client standard owed? — yes, and it is smaller than it sounds

The operator's framing: *"we may need to have some kind of a standard too… you know, kind of like the SDK
guide. It may not be specifications in the formal sense, but your browser is going to support this level
of functionality, this level of understanding."*

**The corpus has the vertical half (`GUIDE-RESOLUTION`) and the per-type semantics (each convention).
What it lacks is the seam between them**, and that seam is one table plus one rule:

1. **An edge inventory** — for each app-tier type, which fields are outbound references, and which shape
   each one is. This is §1's census turned normative, and it is what lets a client walk a document it
   does not have a renderer for.
2. **A resolution contract per shape** — what a client does with each, including the three-way read
   (`FEED` §2.2.4, already specified) and a base-URI rule for the string forms (absent).

**Note what this is not.** It is not a new extension, not a new entity type, and not a browser
conformance suite. **It is the document that makes "the client knows how to walk this" checkable** — and
per `GUIDE-RESOLUTION`'s own meta-rule, the thing that would *validate* it is a vector, not prose.

---

## §4 The three gaps the walk rests on

### §4.1 *"Fetched from anywhere"* overstates — and this section was itself too strong

> ### CORRECTED 2026-09-06, same day, by opening `EXTENSION-NETWORK` — which this document listed under *what is unread*.
> **The heading of this section originally read *"names a mechanism that does not exist."* That is
> wrong.** `EXTENSION-NETWORK` **§6.5.5 Mode A1** specifies the composition: *"on local content-store
> miss the dispatcher resolves `system/peer/transport/{publisher_peer_id}/*` → finds the `http-poll`
> profile entity (§6.5.3) with … `content_url_prefix` … builds the URL; performs GET + hash-verify
> (Mechanism A); ingests."* **A reader holding `{peer, hash}` has a specified, normative path to the
> bytes and needs no preconfigured substitute source.**
> **What survives is narrower and still real:** nothing answers *"who has this hash?"* for a publisher
> you cannot resolve (§6.5.4 defers peer→transport discovery to REGISTRY and calls it out of scope for
> v1) or who is gone. **The fix is a one-line scoping edit, not a subsystem** — see
> `EXPLORATION-THE-REFERENCE-PRIMITIVE-SIX-SYSTEMS-AGAINST-ONE-TUPLE` §4, which supersedes this section.
> **The lesson is the ordinary one and it is why the unread list exists:** this document named
> `EXTENSION-NETWORK` as unopened in §8 and then reasoned about the substrate's capabilities anyway.
> **A stated limit that a conclusion is published on top of is not a limit** (L4).

**The paragraph below stands as originally written, because the observation about the *wording* is still
correct and it is what D-38 now covers.**

**`FEED` §2.2.4 row 3:** on a 404 at the path, *"fall back to `seen`, **fetched from anywhere** — author,
mirror, cache, stranger."* §2.2.1 says the same of the pin: a consumer *"may obtain them **from
anybody**."* And the claim built on it: *"Row 3 is where the second naming layer earns its keep and it is
the row the surveyed field does not have… the property Hyper-G obtained with a server-owned link
database, obtained here without one."*

**What the substrate provides is `EXTENSION-SUBSTITUTE` §1: *"an ordered chain of substitute sources…
The model follows Nix substituters: ordered, priority-driven."*** A `system/substitute/source` (§2.1) is
a **configured entity** naming a `source_peer_id` and an endpoint. §10 puts *"wildcard / bare-hash
substitution"* and *"transitive substitute following"* **out of scope for v1**.

> **So "from anywhere" means "from any source you have already configured, or that the publisher
> endorsed." There is no mechanism that answers *who has this hash?*** If the author's path 404s and no
> configured source carries the bytes, a reader with a valid, verifiable, permanent content hash has
> nowhere to send the question. **"Stranger" is not reachable.**

**This is L12's shape** — *a mechanism cited in a ruling must be reachable by the actor the ruling
assigns it to; name its input and how that actor obtains it.* The actor is the reader, the input is a
holder, and the corpus does not say how the reader gets one.

**Three things keep this from being a large finding, and they matter:**

- **The seam exists and is named.** `SUBSTITUTE` §2.1 lists `substitute_type: "peer-to-peer"` as a
  contemplated value and §6's convention-dispatch (`system/substitute/<type>:try`) is the plug point.
  **This is an unfilled slot, not a missing architecture** — and per L8's nineteenth form, an unfilled
  slot in a generator's input set is a fact about the input set.
- **A DHT is probably still the wrong answer**, and the corpus has the measurement: `REFERENCE-PRIOR-ART-…`
  §3 records >70% of IPFS provider records pointing at unreachable peers, and that *"a DHT's unique value
  is answering lookups when there is no always-on origin — its worth is inversely proportional to how
  well the origin tier works."* Our origin tier is the strong part.
- **The honest fix is probably a sentence, not a subsystem.** Row 3's real content is *"the hash stays
  valid and remains fetchable from any source that has it"* — which is true, useful, and does not promise
  discovery. **The overclaim is the word *anywhere* doing the work of a mechanism.**

**The reason to fix it now rather than later:** it is currently written as the property that distinguishes
us from ATProto and Hyper-G, in a proposal under review, and two seats build from these documents.

### §4.2 A peer's registry view is not publishable — and this is **D-27**, already on the board

**The operator's framing today:** *"If I publish stuff, I'm going to publish my registry data — these are
the registries I have, this is my ranking. So if I'm navigating through a specific peer, I see what they
have… I can at least understand their view from their perspective. I can reconstruct if I want to build
that closure."*

**This is not a new item and must not be filed as one.** `docs/status/WORKSTREAMS.md` **D-27** —
*"The publishable resolver-config"* — was raised by the operator on **2026-09-05** and already carries
the same framing, including the load-bearing distinction: *"a config is deliberately peer-local and not
synced so nothing can install itself; **publishing one voluntarily is a different act** — an offer others
may adopt, exactly like a block list or a ranking."* It is filed **small, and the highest-value item of
its group.** *(Nearly minted here as a duplicate; caught by reading the board. L7.)*

**`EXTENSION-REGISTRY` §4 stores `resolver-config` at `system/registry/resolver-config` and marks it
*"(peer-local; not synced)."*** So the artifact exists, is well-specified, and is deliberately not
shared — which is D-27's premise, re-verified this session.

**What this document adds to D-27 is a second motivation it did not have**, because it is the missing
input to
§4.1: *when I cannot resolve a reference this peer published, the peer that published it is the party
most likely to know how.* **A published resolver view is a routing hint attached to the publisher rather
than to each link** — which is Nostr's relay hint moved from the link to the author, and it composes with
every one of the nine shapes without changing any of them.

**Two things it must not become**, both visible from the corpus's own rules:

- **It is not a trust import.** `GUIDE-RESOLUTION` §2 — names are receiver-relative; §7 — trust is
  receiver-side. Reading Alice's view tells you *how Alice resolves*, and adopting it is a separate,
  explicit act. The operator's framing already says this (*"it doesn't mean I'd necessarily translate
  that stuff into my own"*).
- **It is a disclosure.** A resolver-config is a map of whom you trust and, via `pinned_bindings`, of
  whom you have met. §4's *not-synced* is not an oversight — REGISTRY §4.1's whole name-disclosure
  apparatus exists because naming metadata leaks. **A publishable view is therefore a curated projection
  of the config, never the config**, and that distinction is the design.

### §4.3 The `link-ref` classifier and the prose link

§1(c) and §2.3. **The cheapest correct move is the one HTML made**: specify the base and the
discrimination, not a classifier. `SITE` §3.2's `F-1` rule is the local precedent and it is four lines
long.

---

## §5 What this puts on the docket

**Nothing here is proposed and nothing is folded.** Ordered by the operator's standing method test — *name
the proposal, implementation, or tree it will be checked against* (`HANDOFF-2026-09-06` §5).

| # | Item | Checked against | Size |
|---|---|---|---|
| **D-36** | **`link-ref` has no classifier and prose links have no rule.** Specify the base and the discrimination, per §3.2's `F-1` precedent | `SITE` §4 · `entity-browser-rust`'s renderer, which must already do *something* | **small** — a rule, not a mechanism |
| **D-37** | **The reference-shape census (§1).** Nine shapes, two with peers. Decide whether the app tier converges on a peer-bearing atom or states why not | `FEED` §2.2 vs `SITE`/`EMBED`/`SHARE`, all four on disk | **medium** — and it is a convergence call, not an invention |
| **D-38** | **"Fetched from anywhere" (§4.1).** Either scope the sentence to what the substrate does, or name the `peer-to-peer` substitute convention as the thing that would back it | `FEED` §2.2.4 · `SUBSTITUTE` §1/§10 | **small if scoped**, large if built — and scoping is probably right |
| **D-39** | **The client walk contract (§3.1).** The edge inventory + per-shape resolution. The *"NED browser standard"* | `GUIDE-RESOLUTION` §4 · both app-tier seats | **medium** — mostly assembly of what exists |
| **D-27** | **not new — already on the board.** §4.2 adds a second motivation: a published resolver view is the cheapest partial answer to §4.1, being Nostr's relay hint attached to the publisher instead of to each link | `REGISTRY` §4 · the five live static sites | unchanged: **small** |

**Sequencing note, and it is the only one this document is confident about: D-37 gates D-39.** An edge
inventory over nine inconsistent shapes documents the inconsistency; over a converged atom it is a table.
**D-36 is independent and is the cheapest real improvement on the board.**

**Two connections to items already open.** D-37 is the same question as **D-31** (the anchor — what a
durable reference pins) one level up: D-31 asks what a reference points *into*, D-37 asks what it carries
*at all*. They should be read together. And §4.1 is upstream of **D-10** (conversation assembly) for the
same reason D-35 is — assembly presumes fetchability.

---

## §6 The identifier scheme — flagged only, per the operator

> *"I do find it convenient when we talk about Nostr, they kind of ID their proposals… But there's no
> canonical identifier for the issue. I wouldn't dig into it — mostly just flag it."*

**Flagged, not analyzed.** What is true today, from the tree: proposals are identified by **filename**
(`PROPOSAL-APP-CONVENTION-FEED`), docket items by a **`D-N` local to `docs/status/WORKSTREAMS.md`**, and
routing packets by **date plus a letter** (`ROUTING-2026-09-06-b`). The `D-` numbers are already on their
second numbering — `4b7be23`'s commit message is *"a docket so findings stop being renumbered every
arc"* — which is the evidence that the need is real and internal before it is external.

**The comparison the operator drew is exact:** NIP-19 and NIP-21 are *numbers*, cited as numbers, stable
forever, and that is why this document could cite them in one token each. **We would have written "the
Nostr identifier-encoding spec."**

**Not started, and it should not be until the design work settles** — a stable-identifier scheme minted
over a corpus that is still being renumbered would need renumbering. **The trigger to watch for is
external feedback**, which is the operator's own framing (*"once we really want the community to get more
involved"*).

---

## §7 Sources

**Opened this session, primary:**
[NIP-19 — bech32-encoded entities](https://github.com/nostr-protocol/nips/blob/master/19.md) ·
[NIP-21 — the `nostr:` URI scheme](https://github.com/nostr-protocol/nips/blob/master/21.md)

**Relied on from prior in-corpus reads, each verified against four primary specs at the time and not
re-opened here:** ATProto `strongRef` and the AT-URI fragment reservation
(`EXPLORATION-THE-DURABLE-REFERENCE-…` §2, §7).

**In-corpus, opened this session by section:** `GUIDE-RESOLUTION` §§1–11 (full) ·
`PROPOSAL-APP-CONVENTION-FEED` §2–§2.2.4 · `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §2, §3.1, §3.2, §4,
§4.1, §4.2, §11 · `APP-CONVENTION-EMBED` §3 (the CDDL block and its normative notes) ·
`APP-CONVENTION-SHARE` §2.2–§2.4 · `EXTENSION-SUBSTITUTE` §1, §2.1, §2.2, §10 · `EXTENSION-REGISTRY` §4
(storage line), §4.1, §4.3 · `EXTENSION-DISCOVERY` §6 (deferred backends) ·
`REFERENCE-PRIOR-ART-…-BY-AXIS` §0, §3 · `EXPLORATION-THE-HYPERTEXT-LINEAGE-…` (full) ·
`EXPLORATION-THE-DURABLE-REFERENCE-…` (full).

## §8 What is unread, named so nobody assumes coverage

- **`EXTENSION-ROUTE` and `EXTENSION-RELAY` in full.** Read only via grep for the DHT/`resolve_next_hop`
  seam. Both are cited above for a single fact each (the seam is deferred); **neither was opened**, and
  any claim about what they specify beyond that is not made here.
- **The `entity-browser-rust` and `entity-workbench-go` trees.** §1's census is of the **corpus**, not of
  what the seats emit. *"Their renderer must already do something with `link-ref`"* is an inference, and
  **D-36 should open the tree before it is written** — a naming or shape divergence has no discovery path
  except someone reading both trees (L21's fifth shape).
- **`APP-CONVENTION-EMBED` §§4–6 and §10.** The ladder and the fallback discipline. Read §3 and §7's
  `img-src` line only; the census rows are from opened CDDL, the surrounding semantics are not claimed.
- **W3C URL / RFC 3986 base-URI resolution in detail.** §2.3 states the mechanism at a level any web
  developer would confirm; **the exact algorithm was not opened**, and D-36 should cite it properly rather
  than from this document.
- **NIP-05 and the rest of the NIP corpus.** Only 19 and 21 were opened. `GUIDE-RESOLUTION` §6.0's NIP-05
  reading is prior work and was not re-verified.
