# EXPLORATION — the relay landscape, the prior art, and the scope I was ruling inside without seeing

**Status:** landscape audit, **part 1 of 2**. **No dispositions are ruled here.** It retracts one that was.
**Occasioned by:** operator correction, 2026-08-17 — *"there's nothing wrong with an intermediary carrying
opaque envelopes, that's totally valid relay functionality… you're supposed to do the research/landscape
analysis audit to determine how smtp/nostr/activitypub/etc do this in modern forms and figure out where our
relay system fits into that… we need to understand the scope before we start making rulings."*
**Read at:** arch `c7cddcb` · legacy corpus (read-only, `entity-lab-legacy-meta`) ·
`EXTENSION-RELAY.md` v1.0 in this tree

---

## §0 What I got wrong, stated plainly

I produced **three** arguments about RELAY's mode set across two days, and **read none of the prior art
until now.** The sequence:

| # | what I claimed | what was actually true |
|---|---|---|
| 1 | Mode A is incoherent — *"you cannot serve a queryable view over bytes you may not decode"* — so evict it (disposition 1, RULED) | The MUST NOT is scoped to the **inner** envelope. §9 says *"there are two envelopes; only one is decoded"* and the **relay envelope IS decoded**. An aggregator indexing outer routing fields is exactly conformant |
| 2 | Downgraded to PROVISIONAL on re-reading §9 — but still framed disposition 1 as the leading answer | Correct instinct, under-weighted. I had the right reading and did not follow it |
| 3 | *"An aggregator is not an intermediary carrying opaque envelopes — it is a peer with a large local view"* | **An intermediary carrying opaque envelopes is the core relay function.** Encrypted store-and-forward, CDN-hosted encrypted messages, live circuits for unreachable peers — all of it. Writing "it is not X, it is Y" about one mode read as displacing X, and X is the point of the extension |

**The method failure, which is the one that matters.** `HANDOFF-2026-08-17-relay-audit` §3a named eleven
legacy documents, *"start here"* on the study that **produced the four-mode model**. I wrote three
arguments before reading any of them. **A decomposition argument is not a landscape analysis** — showing
that one shape of aggregation is expressible another way says nothing about what relay functionality the
system needs or how the field solves it.

**This document is the reading I should have done first.** It covers **5 of the 11** documents; §7 names
the 6 outstanding and what each is expected to bear on. **Nothing is ruled here.**

---

## §1 The scope, from the study that produced the mode set

`EXPLORATION-RELAY-AND-AGGREGATOR-PATTERN-cgid-10-217`, §0, verbatim:

> **"Relay is the universal intermediary primitive.** 'Carry a signed, capability-bearing envelope from
> sender to receiver via an intermediary.' Every form — active forward, passive store-and-poll,
> aggregator, NAT-circuit, federation server — is one configuration of this primitive. **One mechanism;
> many modes.**"

And the operator framing that commissioned it: *"Relay is just so important to everything in the system…
We have how all the other systems do it and then we should be able to find a solution that works for the
entity system."*

**Its §1 table is the scope statement I was missing.** Every near-term capability has relay underneath:

| Capability | Mode |
|---|---|
| Static-hosted publisher (CDN) | **S** — the static host is a passive intermediary |
| NAT-bound peer accepting messages | **C** (circuit) or **F** (forward to its inbox) |
| Intermittent peer (phone offline) | **S** for inbox; **F** when up |
| Registry resolution from a hosted authority | **S** static; **A** live aggregator |
| Multi-registry meta-resolver (federation) | **A** |
| **Encrypted messaging through an untrusted hop** | **F or S** with ENCRYPTION |
| Group-shared registry | **A** |

> *"Every near-term capability has relay underneath. **If we get relay right, we get most of the
> connectivity story right.**"*

**And the standing pin the study was written under —
`[[feedback_design_space_not_authority]]`: *relay modes are design-space; flag trade-offs; don't pick a
single canonical.*** I did the opposite: I picked, and I picked eviction.

---

## §2 The landscape, as surveyed — and it is not thin

Two legacy documents carry it. The first (`§4`) surveys relay architecture; the second
(`EXPLORATION-RELAY-RECEIVE-SIDE-OPACITY-AND-CROSS-PROTOCOL-cgid-10-229` §5) surveys the *opacity*
mechanism protocol by protocol. Consolidated:

| System | Relay parses payload? | Opacity kind | Encryption opacity | Mode(s) |
|---|---|---|---|---|
| **SMTP / email** | **No — RFC 5321 MUST NOT parse body**; routes on envelope `RCPT TO` | Role | STARTTLS is hop-by-hop (**relay sees plaintext**); S/MIME / PGP is E2E, headers still plaintext | **F**, with **S** fallback when the destination is offline |
| **NNTP** | No | Role | — | **S + A** — each server aggregates upstream and serves downstream |
| **Nostr (public)** | Partial — parses event header + tags, verifies sig, **indexes tags**; `content` an opaque string | None (public) | n/a | **A + S** |
| **Nostr DM (NIP-17/44/59)** | No — sees ciphertext, ephemeral-key sig, recipient `p` tag | **Cryptographic** (gift-wrap) | Full; recipient tag visible for routing | **A** carrying encrypted payloads |
| **AT Protocol** | **No** — verifies signatures + Merkle proofs on CID-addressed commits, **never interprets records** (the AppView does) | **Role — kept even though the data is public** | n/a | **S** (PDS) + **A** (Relay firehose) |
| **ActivityPub / Mastodon** | **Yes, fully** — every receiving server parses JSON-LD | **None** | None — TLS in transit; admins can read DMs | **F** + **S**; Mastodon relay servers are **A** |
| **Matrix (E2EE)** | No — sees routing/DAG metadata; content is Megolm ciphertext | Cryptographic | Full for payload; metadata visible | **F/S** store-and-forward |
| **libp2p circuit relay v2 / TURN** | No | Cryptographic (E2E identity at endpoints) | — | **C** |
| **Tor** | **No** — each hop decrypts one layer = next hop only | Cryptographic, **layered** | Full per-hop; no hop knows both ends + content | **F** along a pre-built circuit |
| **entity-core RELAY** | **No** — raw-frame, content-addressed inner, never decoded at **any** hop incl. terminal | **Role (raw-frame) + optional cryptographic (ENCRYPTION)** | Full when ENCRYPTION is used; `destination` / `recipient_key` visible | **F + S** shipped; **A + C** deferred |

**Three findings from this table that bear directly on the question I was ruling:**

1. **Every surveyed system mixes modes.** ATProto = S+A. Mastodon = F+S (+A). Nostr = A+S. SMTP = F+S.
   NNTP = S+A. **Not one of them implements a single mode.** A four-mode enumeration is not an
   over-broad taxonomy; it is the minimum needed to describe any real deployment.
2. **Two distinct kinds of opacity exist and must not be conflated** — *cryptographic* (Tor, gift-wrap,
   Megolm: the relay **cannot** read) and *architectural/role* (ATProto, SMTP: the relay **could** read
   but is contractually barred). **We use both**, and the ENCRYPTION layer is the cryptographic half.
3. **We are on the better side of the ActivityPub fork.** AP has no opaque-payload concept at all — every
   receiving server fully parses, and DMs are addressing-convention with no crypto. Content-addressing
   gives us role opacity for free.

---

## §3 The contradiction I ruled on is mostly not there

The argument that started this — filed by browser-rust, adopted and ruled by me — was:

> §1 says a relay carries **opaque** envelopes and the intermediary is transport; §10.4 says MUST NOT
> **decode the inner envelope** in the *"forward / store / **aggregate** / terminal-delivery path"*; §1's
> Mode A serves *"a queryable view."* **You cannot serve a queryable view over bytes you may not decode.**

**Read §9 as written and it dissolves:**

> *"**There are two envelopes; only one is decoded.** The **relay envelope** (`forward-request` /
> `store-entry`) **IS decoded** — that is how the relay reads its outer routing fields (`destination`,
> `next_hop`, `namespace`, `ttl_hops`). The **inner envelope** … MUST be held as opaque bytes.
> Implementations MUST NOT decode it as part of forward, store, **aggregate**, or terminal-hop
> delivery."*

**`aggregate` is in that list deliberately, and it is scoped to the *inner*.** An aggregator may read
every outer routing field; it may not open the payload. **A queryable view over outer fields is exactly
conformant** — and it is precisely what the closest analog in the survey does:

> *"AT Protocol's relay … verifies signatures/hashes on content-addressed data, **forwards bytes it never
> interprets** — a role boundary kept **even when the data is public**."*

ATProto's Relay aggregates N PDSes into a firehose, serves it, and never interprets a record.
**Mode A is that.** Nostr goes slightly further (it indexes tags), and the study records that as a
*different point in the design space*, not as evidence of incoherence.

### §3.1 What IS defective — and it is one sentence

**§1's opening line, not §9 and not §10.4:**

> *"A relay is an intermediary that carries opaque, signed, capability-bearing envelopes **between two
> endpoints**."*

**§2's own modes table, eight lines later, contradicts it:** Mode A is `multi-publisher`, routing
`broadcast or filtered`. A broadcast multi-publisher intermediary is not "between two endpoints."

**So §1's one-line definition was written from the routed modes (F and C) and does not cover the design
space §2 tabulates.** That is the real defect: **an over-narrow definition, not a miscategorised mode.**
The fix is to widen §1 to match §2 — which is **disposition 2** (narrow/repair §1's wording, keep the
mode), the disposition I listed and did not take.

**I am not ruling that here.** §7's outstanding reading could still change it, and the operator's
instruction is to establish scope before ruling. But the coherence argument that disposition 1 rested on
**does not survive reading §9**, and disposition 1 is retracted as a *ruling* below.

---

## §4 What the modes actually buy — including everything the operator named

**Encrypted messages through an untrusted hop.** The inner MAY be encrypted (RELAY §3.1, §6.6); the relay
*"sees only an opaque content-addressed blob it never decodes."* The receiver needs ENCRYPTION + its key
— **but not RELAY** (§3.1.1: the destination *"MUST NOT need the RELAY extension installed merely to
receive"*). That floor is load-bearing for the security model, not ergonomics: the cap chain must verify
**exactly as on a direct connection**, which forces byte-identity, which forces raw-frame.

**Static CDN hosting encrypted relay messages.** `DESIGN-STATIC-TRANSPORT-AS-RELAY` ruled it and the
study carries it forward: *"a static store is just a relay intermediary that is **passive** instead of
**active**… The protocol cannot tell the difference — same envelope, same opacity, same verification on
receipt."* The stress test held on all three axes (envelope path byte-identical; async delivery inherits
relay's INBOX+CONTINUATION composition; the nonce/liveness handshake *does not apply* because
store-and-forward has no session — each envelope is self-authenticating).

**And the encrypted variant is a named, deliberately-deferred thread, not an oversight** — §4 of that
same document: *"**Encrypted long-term static exchange**: two peers each with a static endpoint, pushing
encrypted envelopes the other pulls each cycle — async encrypted relay. This needs the encryption + relay
+ (a)symmetric-rendezvous dig and is a **separate, later thread**."* **It is on the map. My rulings had
no business narrowing the surface it lands on.**

**The honest limit, which should be stated and not implied** (opacity study §4): the relay sees **who**
even when it cannot see **what** — `destination` in Mode F, the namespace *is* the destination peer-id in
Mode S, and `recipient_key` is a plaintext routing hint. **Same visibility as Nostr NIP-17 and SMTP
`RCPT TO`; weaker than Tor.** Correct for the v1 threat model (passive observation of the relay path, not
metadata anonymity). The frontier if that ever becomes a goal is NIP-59's throwaway outer signature and
Tor's per-hop-only knowledge — post-v1, and an additional layer rather than a substrate change.

---

## §5 The three primitives, and the two we dropped

`EXPLORATION-INFORMATION-TRAVEL-RELAY-ROUTING-GOSSIP-cgid-10-229` draws the boundary that keeps relay
clean, and it is directly relevant to what Mode A is even for:

| Primitive | Addressing | Question | Home |
|---|---|---|---|
| **Unicast relay** | addressed (one destination) | *"carry this to D"* | **EXTENSION-RELAY** |
| **Routing** | the decision, not the message | *"to reach D, who is next?"* | resolver seam → future EXTENSION-ROUTE |
| **Gossip / scatter** | **unaddressed** | *"spread this to whoever cares"* | **future EXTENSION-GOSSIP — does not exist** |

> ### ⚠ CORRECTION — "dropped" was wrong, and the guide that already maps this exists
>
> **`EXTENSION-ROUTE` v1.0 is Active in this corpus** — the routing plane returned as a *storage
> plane* (holds `system/route` next-hop entities; computes nothing, owns no resolver registry), and
> its own Related line already names *"DISCOVERY / **gossip backends** (producers)"*. **RELAY v1.2
> ships source-routed multi-hop** with precedence *source route > route table > direct*. Both were
> written up below as absent, from a legacy document that predates them. **The legacy note was
> stale and I amplified it into a loss.**
>
> **Gossip is not abandoned either** — `PROPOSAL-EXTENSION-GOSSIP.md` exists in the legacy tree; it
> simply did not cross the V8 publish split, along with `PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY.md`.
> `GUIDE-NETWORKING-MODEL` §7 cites both as if present, which is what made the lineage read as
> discarded rather than deferred. **A missing design record is a corpus gap, not a design gap.**
>
> **And `GUIDE-NETWORKING-MODEL` already is the boundary map** — five concerns kept separate, a
> deployment-topology table, and a v1-vs-roadmap status board. **L7 again**: the map existed while
> this section was reconstructing one. It has been corrected in place rather than duplicated here.
>
> The paragraph below is retained because its *lineage* claim is still the useful part.

**V2.0 had all four as separate extensions** (`system/routes`, `system/relay`, `system/gossip`,
`system/replicate`). Relay and routes are now landed; gossip and declarative replication are not:

> *"**`system/gossip` did NOT survive** — no gossip extension in v7. Subsumed in part by
> EXTENSION-SUBSCRIPTION + Merkle/manifest sync, but **true epidemic peer-to-peer spread is genuinely
> absent.** This is a real gap for the federation/mesh vision (**and Mode A leans on cross-peer
> subscription that doesn't exist**)."*

**This is the strongest argument in the corpus for the thing I was gesturing at, and it is not the
argument I made.** The tension around Mode A is not that an aggregator is miscategorised — it is that
**Mode A sits at the seam between *addressed* relay and *unaddressed* spread**, and the unaddressed
side is the part still on the roadmap. That is a scope question about the extension family, and it
deserves the audit's sequencing rather than a same-day disposition.

### §5.1 The boundary, as the operator states it — and it is the load-bearing frame

> *"Relay is more focused on the peer-to-peer specific messaging than the expansion of shared data
> through a gossip network or a bittorrent-like CDN."*

**This is the decomposition, and everything above lines up behind it:**

| | question | home | state |
|---|---|---|---|
| **Addressed messaging** | *"carry this to D"* | **RELAY** (F/S/C) + **ROUTE** + **NETWORK** + **REGISTRY/DISCOVERY** | **shipping** |
| **Unaddressed spread** | *"spread this to whoever cares"* | **GOSSIP** | roadmap; proposal in the legacy tree |
| **Content distribution** | *"many peers serve the same bytes"* | **CONTENT** — blob manifest is *"analogous to a torrent file; any peer with the chunks can serve them"* | transfer model **shipping**; chunk-peer discovery open |
| **Structured replication** | *"keep these subtrees in step"* | SUBSCRIPTION + REVISION + GROUP | composable today; `system/replicate` as a declarative policy is tracked, not built |

**And the operator's SMTP point is the sequencing argument.** SMTP is a complete, useful,
decades-proven addressed-relay system **with no gossip and no BitTorrent underneath it.** The
landscape table (§2) agrees: SMTP is F+S and nothing else. So the addressed-messaging tier is
**shippable and valuable on its own**, and the unaddressed tiers can arrive later without
invalidating it — which is exactly what `GUIDE-NETWORKING-MODEL` §3 already says about a LAN or
VPN deployment: *"No NAT traversal, no gossip, no DHT needed. Fully covered today."*

**What this frame does NOT do is settle Mode A** — it sharpens *why* Mode A is the hard one. Mode A
is the only mode that is multi-publisher, and multi-publisher is the defining property of the
unaddressed tier. Whether that makes it a relay mode with a widened §1, or a member of the
dissemination family, is the question part 2 answers with `PLAN-EXTENSION-LANDSCAPE` in hand.

---

## §6 Retraction

**`PROPOSAL-RELAY-COMPLETE-THE-MODE-SET`'s disposition-1 ruling (aggregate leaves RELAY) is RETRACTED as
a ruling**, not merely downgraded. It rested on a coherence argument that does not survive §9, and it was
made without the prior art. The proposal returns to **open, with three live dispositions and no lean
recorded from this seat** until §7's reading is done.

**What survives, unchanged and still true:**

- **Something is wrong at §1** — three seats agree, and §3.1 now names it precisely: §1's definition is
  narrower than §2's own table. **That is a wording defect with a small fix.**
- **Nothing is built for Mode A in any tree**, so re-filing (in either direction) is still ~zero churn.
- **Circuit mode's deferral condition has fired** — browser-rust is the driver. Untouched by any of this.
- **The three void deferrals** (Mode A's substrate blocker, Mode C's driver condition, §7's
  signed-mutable-pointer gate) remain void on landed text. That finding was independent of the
  disposition and stands.

---

## §7 What I have NOT read — named, so this is not overclaimed again

**Read (5):** the aggregator-pattern study · receive-side opacity + cross-protocol · information-travel /
relay-routing-gossip · static-transport-as-relay · `PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE` (for Q8).

**Outstanding (6), and what each is expected to bear on:**

| Document | Expected to bear on |
|---|---|
| `ANALYSIS-RELAY-CAPABILITY-CHAIN` | The transport-not-authority ruling at source. Load-bearing and **must not be disturbed** by any disposition |
| `ANALYSIS-AND-REVIEW-EXTENSION-RELAY-PRE-IMPL-cgid-10-228` | Re-derived eleven decisions and **left the deferrals untouched** — the closest thing to a prior audit of this exact question |
| `ARCH-CLOSEOUT-NETWORK-RELAY-CYCLE-cgid-10-230` · `SESSION-HANDOFF-cgid-10-229` | How the cycle actually closed, and why the deferrals were written as they were |
| `EXPLORATION-NETWORK-EXTENSION-LANDSCAPE-AND-DELIVERY-MODEL` · `-NETWORKING-SUBSYSTEM-COMPREHENSIVE-LANDSCAPE` | Where the network family's boundaries were drawn — bears on whether Mode A belongs to RELAY, SUBSCRIPTION, or an absent GOSSIP |
| `reviews/PLAN-EXTENSION-LANDSCAPE` | **The overall extension architecture** — the operator named this as needed, and it is the frame for any "which extension owns this" question |
| `proposals/implemented/PROPOSAL-EXTENSION-RELAY` | The spec's own rationale, §11.1a included — why the deferral was written |

**Part 2 of this audit reads those six and then, and only then, takes a disposition.**

## §8 What this document does NOT claim

- **Not** that disposition 2 is correct. §3.1 shows the coherence argument for disposition 1 fails; that
  is not the same as establishing 2, and §5's gossip-seam question could still support 3.
- **Not** that the landscape survey is freshly verified. The legacy study states its own limit — the
  protocol details are *"from research knowledge, not freshly re-verified per-RFC."* **Anything that
  becomes load-bearing for a ruling must be re-verified against the primary source before it is spent.**
- **Not** that the mode set is complete. §5's gossip gap is real and unowned.
- **Not** a claim about any implementation's current state. No build-state assertions appear above.
