# REFERENCE — prior art by axis: relay · inbox/outbox · DHT · federation · authority · identity

**Status:** Reference (design record). Not a proposal, not normative. **A library, not an argument.**
**Companion:** `EXPLORATION-THE-OPERATING-MODELS-AND-THE-ALWAYS-ON-TIER` — the synthesis. This is the
decomposition it rests on, kept separately because a synthesis is re-argued and a source list is not.

**What this is for.** When a design question arrives — *how should relays work? what does an inbox
mean? where does a DHT fit? how do independent instances find each other? who decides what a group
may do?* — this file answers *"what does each existing system actually do on that axis, and where do
I read the primary source."* **The goal is not to copy any of them.** It is to make the trade-offs
legible before a decision, so that a design choice is a choice rather than an accident.

**How to use it.** Find the axis. Read the row for the two or three systems closest to the shape you
are considering. **Then open the source** — the summaries here are deliberately short and are not a
substitute for the spec. Every claim traceable to a URL carries one; claims marked *(unsourced)* are
this record's own reading and should be treated as weaker.

**Currency warning.** Figures are dated where known. Several are 2024–2025 vintage against systems
that have grown since; **re-measure before sizing anything.** A number in this file is a starting
point for a measurement, never a substitute for one.

---

## §1 Axis: relay — *who carries a message the sender cannot deliver directly*

| System | What a "relay" is | Trust required | Who runs it | Failure mode |
|---|---|---|---|---|
| **SMTP** | the recipient's **MX** — an always-on mailbox host, named in DNS by the recipient's domain | receives plaintext unless TLS+DKIM; the MX is *authoritative for delivery* | the recipient's domain | open delivery ⇒ spam; admission control retrofitted at enormous cost (SPF/DKIM/DMARC) |
| **Nostr** | a **dumb store** — accepts events, serves queries; no routing, no forwarding | can silently omit; cannot forge (events are signed) | anyone; many are paid, many shut down | omission is undetectable ⇒ clients hedge to big relays ⇒ centralization (§4) |
| **Mastodon** | two distinct things: **instance inboxes** (`POST /inbox`, addressed, push) and **fediverse relays** (unaddressed re-broadcast between instances) | the instance is fully trusted by its own users | instance admins; relays are opt-in | relay lists rot; small instances stay invisible |
| **ATProto** | the **Relay** — crawls all PDSes, emits a merged firehose | none for correctness (records are signed); trusted for *completeness* | anyone (~$34/mo full-network) | bandwidth-bound; grows as events × subscribers |
| **libp2p** | **circuit-relay-v2** — a rendezvous byte pipe for peers that cannot dial each other | none (end-to-end encrypted through it) | any public peer | reservation limits; relay churn |
| **Tor** | multi-hop onion circuits — relay as *anonymity*, not reachability | none per-hop by construction | volunteers | latency; exit-node scarcity |

**The distinction most often collapsed:** *addressed forwarding* (SMTP, libp2p circuit) and
*unaddressed re-broadcast* (fediverse relays, ATProto firehose) are different primitives with
different loop control, different dedup and different failure modes. **A single word covers both in
most documentation.** This design record separates them as *relay* (addressed) vs *gossip/expansion*
(unaddressed), and that separation is load-bearing.

**Sources:** [ActivityPub spec](https://www.w3.org/TR/activitypub/) ·
[Mastodon ActivityPub](https://docs.joinmastodon.org/spec/activitypub/) ·
[Fediverse relays](https://joinfediverse.wiki/Relays) ·
[Bluesky Relay ops](https://docs.bsky.app/blog/relay-ops) ·
[libp2p circuit-relay-v2](https://github.com/libp2p/specs/blob/master/relay/circuit-v2.md)

---

## §2 Axis: inbox / outbox — *where does a message go so it is found*

**This axis is the one with the most confusion in the wild, because three systems use the same two
words for different objects.**

| System | "Outbox" means | "Inbox" means | Who resolves it |
|---|---|---|---|
| **ActivityPub** | an **actor's public collection** of activities they produced (a readable URL) | an **endpoint you POST to** to deliver to that actor | the actor's server, via WebFinger |
| **Nostr (NIP-65)** | the relays a user **writes to** — read these to get their posts | the relays a user **reads from** — write here to reach them | the client, from the user's published relay-list event |
| **SMTP** | (no concept — the sender's MTA is transient) | the **mailbox** behind the MX | DNS MX record |
| **ATProto** | the user's **repo on their PDS** (a signed, ordered log) | (no per-user inbox; interaction records live in the *actor's own* repo) | DID document → PDS endpoint |

**The critical difference:** in ActivityPub the inbox is *push* — delivery is the sender's job and
failure is the sender's problem. In Nostr it is *advisory* — the sender writes where it thinks you
read, and nothing guarantees you look there. In SMTP it is *authoritative* — one MX, one mailbox,
delivery is defined.

**The measured lesson (Nostr, and it is the most important row in this file):** an *advisory* inbox
with no reconciliation between stores does not converge. Clients query the same 10–15 large relays
and assume replication carried the rest; strict outbox enforcement was tried, notes went missing,
and enforcement was **rolled back**. NIP-65 now gets *"lip service in readmes and silent override in
production."* The stated cause is that **the gossip layer NIP-65 presumes was never built.**

**Sources:** [ActivityPub inbox/outbox](https://www.w3.org/TR/activitypub/#inbox) ·
[NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md) ·
[Outbox model explained](https://nostrify.dev/relay/outbox) ·
[Critique of outbox in practice](https://leonacosta.medium.com/nostr-is-centralizing-by-design-da67b8f53966) ·
[NIP-64 inbox-model draft](https://github.com/nostr-protocol/nips/blob/inbox-model/64.md)

---

## §3 Axis: DHT — *distributed lookup, and why it decays*

**A DHT answers "where is K?" It is a lookup index, not a distribution mechanism.** Conflating it
with gossip is the single most common category error on this axis — IPFS uses its DHT for both,
which is why the confusion is so widespread.

| Property | Kademlia / IPFS Amino |
|---|---|
| Lookup cost | O(log N) hops |
| Redundancy | **K = 20** peers hold each provider record |
| Record validity | **48 h**, republish every **~22 h** to outrun churn |
| What is stored | *provider records* — `CID → peer that claims to have it` — plus addresses |
| Verification | the **bytes** are verifiable (content-addressed); the **pointer** is not |

**The decay, measured.** In a documented Kubo investigation, **over 70% of provider records pointed
at unreachable peers**, the remainder mostly reachable only via a relay. Two distinct failures:
*record staleness* (the record expires) and *address staleness* (the record survives, its addresses
rot). **NAT'd and residential nodes are named as the largest contributor** — announced but
undialable. K=20 is derived from *observed average churn*; when real churn exceeds the assumption
the guarantee degrades **silently**.

**The direction of the fixes is the finding:** every deployed mitigation moves *back toward
addressability* — delegated routing over HTTP, cached address books with active probing, provider
prioritization, IPNI as a complement. The design philosophy is explicit that a DHT trades speed and
predictability for resilience.

**Why this matters for any design with an origin tier:** a DHT's unique value is answering lookups
*when there is no always-on origin for the thing*. Its worth is therefore inversely proportional to
how well the origin tier works — and its characteristic failure (a well-formed, signed answer
pointing at nothing) is indistinguishable at the consumer from a withholding peer.

**Sources:** [IPFS Kademlia DHT spec](https://specs.ipfs.tech/routing/kad-dht/) ·
[Unreachable providers (Kubo #9982)](https://github.com/ipfs/kubo/issues/9982) ·
[Delegated routing caching](https://blog.ipfs.tech/2025-delegated-routing-caching/) ·
[Provide Sweep](https://ipshipyard.com/blog/2025-dht-provide-sweep/) ·
[Delegated Routing V1 HTTP API](https://specs.ipfs.tech/routing/http-routing-v1/)

---

## §4 Axis: federation — *how independent instances become one network*

| System | What joins you to the network | Is it automatic? | Cold-start experience |
|---|---|---|---|
| **Mastodon** | **a local user following a remote account.** That transitive closure is *all* your instance can see | **no** | an empty room; timelines *"feel empty and disconnected"* |
| **Mastodon + relay** | admin sends a `Follow` to a relay actor's `/inbox`; the relay re-broadcasts participants' public posts | opt-in, **admin-only** | better, but public relay lists **go stale and many endpoints error** |
| **ATProto** | your PDS is crawled by relays; you are in the firehose by default | **yes**, structurally | a new PDS is visible immediately — the relay does the joining |
| **Nostr** | you write to relays others read | partially | depends entirely on relay overlap |
| **SMTP** | an MX record in public DNS | **yes** | works on day one — the global namespace does the joining |

**The clearest contrast on this axis:** SMTP and ATProto make joining *structural* — a global
namespace or a crawling tier does it for you. Mastodon makes it *social* — somebody must follow
somebody. **Mastodon's model demonstrably works for governance and demonstrably fails for
discovery**, which its own community names as the weak point against centralized platforms.

**The transferable warning:** a design that ships independent clusters *without* a directory and a
relay-equivalent inherits Mastodon's cold-start problem **without** its decade of accumulated
workarounds (directories, relay lists, admin lore).

**Sources:** [Mastodon federation docs](https://docs.joinmastodon.org/spec/activitypub/) ·
[Fediverse relays](https://joinfediverse.wiki/Relays) ·
[ActivityPub relay list](https://github.com/brodi1/activitypub-relays) ·
[ATProto self-hosting](https://atproto.com/guides/self-hosting) ·
[Firehose](https://docs.bsky.app/docs/advanced-guides/firehose)

---

## §5 Axis: authority — *who decides what, for a group or an instance*

| System | Authority unit | What it controls | User's exit cost |
|---|---|---|---|
| **Mastodon** | the **instance admin** | membership, moderation, federation allow/deny (*limited federation mode* = allowlist), and the user's identity itself (`@user@instance`) | **high** — identity is the instance; migration moves followers, not history |
| **ATProto** | split deliberately: **PDS** holds data · **AppView** curates · **DID** holds identity | moderation is an AppView/labeler concern, separable from hosting | **low by design** — "credible exit" is a stated goal; DID survives PDS migration |
| **Nostr** | **the key holder**, plus per-relay policy | relays choose what to store/serve; no global moderation | **low** — take your key elsewhere; but your *reach* is relay-dependent |
| **SMTP** | the **domain owner** | everything for that domain | **high** — address is the domain |
| **Email + own domain** | **the user** (owns DNS) | portability across providers | **low** — this is the pattern the others reinvent |

**The pattern worth extracting:** exit cost is determined by **whether identity is separable from
hosting.** Mastodon binds them (`@user@instance`) and exit is painful. ATProto separates them (DID ≠
PDS) and calls the result *credible exit*. Owning your own email domain achieves the same thing by
accident and has for thirty years.

**For a design where the identifier is a public key**, this axis is largely resolved by
construction — identity is not issued by a host and cannot be revoked by one. **The cost paid
instead is key management**, which is a real and different problem (see
`EXPLORATION-ROTATION-AUTHORITY-AND-KEY-LIFECYCLE`).

**Sources:** [Mastodon moderation/secure mode](https://docs.joinmastodon.org/admin/config/) ·
[ATProto self-hosting & credible exit](https://atproto.com/guides/self-hosting) ·
[did:plc spec](https://web.plc.directory/spec/v0.1/did-plc)

---

## §6 Axis: user-facing identity — *what a person types, shows and shares*

| System | Handle | Underlying id | Does the handle prove anything? |
|---|---|---|---|
| **Mastodon** | `@user@instance.tld` | account on that instance | proves the instance vouches; the instance is the authority |
| **ATProto** | `alice.com` or `alice.bsky.social` | `did:plc:…` / `did:web:…` | **yes for `did:web`** (domain control); `did:plc` is ledger-attested |
| **Nostr** | NIP-05 `name@domain` (optional) | `npub…` (the pubkey) | NIP-05 proves domain control; **the npub is the identity** |
| **SMTP** | `user@domain` | none — the address *is* the identity | proves nothing beyond domain routing |

**The recurring shape:** a memorable handle is always a *claim* that resolves to a durable
identifier, and the interesting question is always **what the resolution proves.** ATProto's
`did:web` and Nostr's NIP-05 both prove *domain control*, via the same mechanism (a well-known HTTPS
path). Mastodon's proves *instance membership*. **A handle that looks like a domain but proves only
issuer attestation is the incoherent case** — it reads as a domain-control proof and is not one.

**This is Zooko's triangle in operational form** (Miller/Stiegler, 2001): a name can be
*human-meaningful*, *secure*, and *decentralized* — historically pick two. Each row above is a
different corner, and the mismatch cases are where a handle's appearance and its guarantee diverge.

**Sources:** [NIP-05](https://github.com/nostr-protocol/nips/blob/master/05.md) ·
[did:web](https://w3c-ccg.github.io/did-method-web/) ·
[WebFinger RFC 7033](https://datatracker.ietf.org/doc/html/rfc7033) ·
[Petname systems](https://files.spritely.institute/papers/petnames.html)

---

## §7 Axis: change detection — *how a reader learns something is new*

| System | Mechanism | Cost to publisher | Works when publisher is offline? |
|---|---|---|---|
| **RSS/Atom** | reader **polls** a URL; `ETag`/`If-Modified-Since` make "unchanged" nearly free | one static file | **yes — offline is not a concept** |
| **WebSub** | publisher pushes to a hub, hub fans out to subscribers | hub infrastructure | no (hub must be up) |
| **ActivityPub** | publisher POSTs to each follower's inbox | **O(followers) per post** | no |
| **Nostr** | client holds open subscriptions to relays | relay infrastructure | yes (relays hold events) |
| **ATProto** | firehose stream; consumers subscribe to the relay | relay bandwidth | yes (PDS holds the repo) |

**The scaling contrast is stark and is the reason RSS never broke:** ActivityPub delivery is
`O(followers)` work *for the publisher on every post*, which is why large fediverse accounts are
operationally expensive for their instance. RSS is `O(1)` for the publisher and pushes all cost to
readers, who can cache, coalesce and back off. **Every system that scaled moved cost off the
publisher and onto an always-on tier plus caching.**

**Sources:** [RSS 2.0](https://www.rssboard.org/rss-specification) ·
[Atom RFC 4287](https://datatracker.ietf.org/doc/html/rfc4287) ·
[WebSub](https://www.w3.org/TR/websub/) ·
[Jetstream](https://docs.bsky.app/blog/jetstream)

---

## §8 Axis: economics — *who pays, and what happens when they stop*

| System | Who pays | Observed sustainability |
|---|---|---|
| **RSS** | the publisher (a web server) | 25+ years, essentially free |
| **SMTP** | the domain owner | stable; spam defence is the real cost |
| **Mastodon** | instance admins, from donations | **high churn** — instances close, taking identities with them |
| **Nostr** | relay operators | **free relays shut down routinely**; guidance is to expect churn and support paid relays |
| **ATProto** | PDS ~cheap; **Relay bandwidth-bound**, ~$34/mo full-network post-Sync-1.1 (was ~$150 in 2024) | viable for organizations; explicitly framed as an entry cost to a services market |
| **IPFS** | whoever pins | unpinned content vanishes; pinning services are the de-facto answer |

**The recurring failure is not technical.** In every federated system above, the most common way a
user loses data or reach is that **the volunteer running their tier stopped paying for it.** RSS is
the exception, and it is the exception because the tier is a plain web server the publisher already
had for other reasons.

**Design implication worth carrying:** an always-on tier that is *also useful for something else*
(a website, a CDN bucket) survives; one that exists only to serve the network is a donation with a
half-life.

**Sources:** [Full-network relay for $34/mo](https://whtwnd.com/bnewbold.net/3lo7a2a4qxg2l) ·
[Relay operational updates](https://docs.bsky.app/blog/relay-ops) ·
[Nostr relay guidance](https://nostr.co.uk/learn/nostr-relays-explained/)

---

## §9 What is still unread — the honest gaps in this file

**Named so the next reader knows the edges rather than assuming coverage.**

> **Four of the five system-shaped gaps below were closed on 2026-09-04** and their verdicts live in
> **`EXPLORATION-THE-CONVERGENT-DESIGNS-WILLOW-SSB-USENET-MATRIX-AND-THE-FIVE-VERDICTS`**, which is
> the authority for those four. **This file stays the eight-axis source library** — find a system
> here, find its verdict there. The rows are kept rather than deleted because *what a gap was, and
> that it predicted its own relevance four times before anyone read it,* is the more useful record.

- ~~**Matrix**~~ **— READ.** State resolution v1 → v2 (2018) → **v2.1, room version 12, 2025**. The
  finding is that it is *still* being repaired seven years on, and that we avoid all of it by never
  making a shared object mutable.
- ~~**Scuttlebutt (SSB)**~~ **— READ, and it was the highest-value omission as predicted.** The
  mandatory append-only chain is a natural experiment with a published cost: onboarding friction,
  impossible deletion, unresolved forks, and the ICN paper's own admission on immutability as a
  harassment vector. **Diaspora, Farcaster and Lens remain unexamined.**
- ~~**BitTorrent's tracker→DHT→PEX progression**~~ **— READ, and the framing here was wrong.** It is
  **not a progression**; PEX structurally cannot bootstrap, and no layer won.
- ~~**Usenet/NNTP flood-fill**~~ **— READ.** Flood-fill, `Message-ID` dedup, `ihave`/`sendme`, and the
  retention-becomes-a-price verdict.
- **`Willow` / Earthstar was not on this list and should have been** — it is the nearest living
  neighbour to this design and the only surveyed system whose authors published a point-by-point
  rationale against every other. **A gap list is only as good as the systems it knows to name**, and
  this one missed the most important entry by not looking for systems designed *after* the ones it
  already knew.
- **Moderation and admission** are touched only in §5. **Still open.** For any social tier this is a
  first-class axis and it has no section here.
- **Xanadu, Hyper-G, Microcosm and the hypertext lineage** were never on this list at all — the file
  is organized around *deployed federated systems* and therefore could not see the *undeployed* ones.
  Read 2026-09-04; authority is
  `EXPLORATION-THE-HYPERTEXT-LINEAGE-XANADU-HYPER-G-AND-WHAT-THE-WEB-DECLINED-TO-SOLVE`.
- **Schema and vocabulary evolution has no axis here** and is a first-class interop concern —
  Protobuf/Avro/Thrift, ATProto Lexicon, Nostr's kind registry, Matrix room versions. Read
  2026-09-04; authority is `EXPLORATION-EVOLVABLE-SCHEMAS-AND-THE-VOCABULARY-PROBLEM`.
- **2026 figures were not re-measured.** ATProto throughput numbers are April-2025 vintage
  (~600 events/sec typical, 2000/sec peaks) and are certainly low today.
- **§7's ActivityPub `O(followers)` characterization** is this record's reading of the delivery
  model *(unsourced)* — it matches operator reports of large-account cost but was not confirmed
  against an implementation.
