# EXPLORATION — the naming landscape, the DNS mapping, and the one record type we do not have

**Status:** Exploration (design record). Not a proposal, not normative.
**Read at:** arch `9ebdd9d` · go `5020e62` · rust `302b7f4` · py `5205895` · browser-rust `7bc1ccf` ·
workbench-go `7492fe8`
**Question it answers:** *does the registry design map onto DNS, the web, Nostr and AT Protocol — and
where are the gaps?* Asked by the operator 2026-08-20, the day before the registry v1 call.

---

## §0 The first finding is that this document did not exist

**`docs/research/` had no naming or registry landscape study.** RELAY has one — a six-system
prior-art survey that L11 was ratified over. REGISTRY, the extension shipping first, had none in this
corpus.

**It had one in legacy, and it is good.** Found by named search, read before any of §1–§7 was
written, per L11 (*read the study that produced a design space before ruling inside it*):

| Legacy document | What it establishes |
|---|---|
| `reviews/PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE.md` | The eight-point continuum, the resolver-handler contract, **§6 the address-discovery gap and its recommended fix** |
| `explorations/EXPLORATION-IDENTITY-PUBLISH-AND-REGISTRY-LANDSCAPE-cgid-10-217.md` | **The six-system survey** (DNS/DID:web · libp2p+IPNS · Nostr · ActivityPub · ATProto · ENS), the six primitive families, the mapping table, what we do and do not need to invent |
| `explorations/EXPLORATION-REGISTRY-CRUX-SYNTHESIS-cgid-10-217.md` | Round-1 question mapping |
| `proposals/implemented/PROPOSAL-EXTENSION-REGISTRY-{SUBSTRATE,PETNAME}.md` | The landed sources of §§2–6 of the spec |

All under
the internal legacy corpus (read-only), which is
**read-only** — another team's tree. This document carries the substance forward, per the precedent
`EXPLORATION-THE-NETWORK-FAMILY-CONSOLIDATED-DESIGN-RECORD` set.

**Why the absence mattered.** `SYSTEM-ARCHITECTURE` ¶561 names *"a registry-and-discovery landscape
review"* as existing. `EXTENSION-REGISTRY` §13 cites both source proposals by name. Every one of
those citations resolves to legacy, so the corpus reads as though it holds its own design record and
does not. That is `spec address`'s `dangling` class doing exactly its job, on the extension we were
about to call v1.

---

## §1 The DNS mapping, record type by record type

The honest way to stress-test this is not "are we like DNS" — it is to take DNS's record types, which
are 40 years of accumulated answers to real operational questions, and ask what answers each one.

| DNS | The question it answers | Ours | Verdict |
|---|---|---|---|
| **A / AAAA** | *where is this thing* | `system/peer/transport/{P}/*` — self-published in the peer's own namespace, resolved by the NETWORK §10 dispatcher | ✅ **and it is the better shape** — authoritative at the child, which is what DNS had to learn |
| **glue** | *where is it, when you cannot ask it yet* | `transports` on the binding, `[MUST]` non-empty for `peer-issued` (§6a.3) | ✅ present, and correctly bounded by `ttl` |
| **— (no DNS analog)** | *where is this thing, when I have only its id* | **nothing** | ❌ **the gap — §4** |
| **NS** | *who is authoritative for this subtree* | no delegation in v1. Direction pinned: sharded/delegated `binding-manifest`, explicitly on the **DNS-zone / TUF-delegated-targets** model (§6a.7) | 🟡 deferred, direction chosen |
| **MX** | *where does mail for this go* | `system/peer/inbox-relay`, self-certifying, per-peer — RELAY §3.5 calls it *"the MX-equivalent"* | ✅ |
| **SRV** | *where is service X, with priority* | `services` (§3b) for deployment-shared infra (STUN / TURN / signaling / inbox-relay); `transports` is ordered and NETWORK §6.5.1a does priority selection | 🟡 **split — §5** |
| **TXT** | *arbitrary attested string, usually domain-control proof* | the `dns-txt` **backend** — reads someone else's TXT record as a trust anchor. We do not serve TXT | ✅ correctly a backend, not a record — **§6** |
| **CNAME** | *this name is that name* | none | ✅ **correctly absent — §3.3** |
| **SOA + AXFR** | *enumerate the zone* | browse (§6a.3a), a `SHOULD` | ✅ **and DNS's own history says keep it a SHOULD — §5.3** |
| **NSEC / NSEC3** | *authenticated denial of existence* | `binding-manifest` `coverage: "complete"` ⇒ absence is authoritative — the spec cites **DNSSEC-NSEC / TUF** by name | ✅ format pinned, impl deferred |
| **CAA** | *who may issue for this name* | `accepted_trust_anchors` + issuer policy (§6a.9.2) | ✅ |
| **DS / DNSKEY** | *the signature chain* | native: `peer_id` **is** the pubkey (V7 §1.5); the binding verifies against the pinned root; there is no unsigned mode to fall back to | ✅ **structurally stronger — §3.1** |
| **negative TTL (RFC 2308)** | *how long may I cache "no"* | `neg_ttl` on `ResolutionResult`, pinned key in `hints` | ✅ |
| **EDNS / DoH / ODoH** | *do not leak the query* | §4.1 step 2's name-disclosure `[MUST]` — **at the substrate, not per-backend** | ✅ **stronger than DNS — §3.2** |

**Twelve of fifteen answered, two deferred with the direction chosen, one missing.** For a first
version that is a good result, and the missing one is the interesting one.

---

## §2 The five systems, and what we took from each

Condensed from the legacy survey (§3.1–§3.6 there), re-checked against the landed spec.

| System | Its lookup primitive | What we took | What we did not |
|---|---|---|---|
| **DNS / DID:web** | hierarchical name → record, transport-trusted (DNS+TLS PKI) | TTL semantics, negative caching, `.well-known` discoverability, the delegation *direction* | the hierarchy itself, and transport-trust as the trust model |
| **libp2p Kademlia / IPNS** | DHT, O(log N) hops; **records ARE multiaddrs** | signed-record + monotonic counter (our `seq`) | the DHT — and with it, **the peer-id → address answer it exists to give** |
| **Nostr** | *none at protocol level.* npub **is** the pubkey; clients dial relays directly | pubkey-as-identity (V7 §1.5), the relay pattern (RELAY Mode S), NIP-05 ⇒ our `well-known-url` kind | NIP-65's published relay list — **the same gap as above, one system over** |
| **ActivityPub / Mastodon** | WebFinger over HTTPS, per-instance authority | `well-known-url` covers WebFinger too; bilateral defederation as receiver policy | the instance-as-authority model — our peers are not tenants of a server |
| **AT Protocol** | `did:plc` ledger or `did:web` → **DID doc → PDS endpoint** | `peer-issued` ≈ did:plc; the role separation (publisher / relay / consumer) | did:plc's ledger, and **the DID document's endpoint field** — same gap, third system |

**The pattern in the right-hand column is not a coincidence and it is §4.**

---

## §3 Where the design is genuinely better than the landscape, and why

Three of these are worth stating precisely, because they are the answers to *"is the design right."*

### §3.1 The trust anchor is not the naming hierarchy

DNS's structural mistake, visible only in hindsight: **the thing that resolves the name is also the
thing you trust.** A zone operator can answer differently for different queriers and nothing in the
protocol notices. DNSSEC was the fix and it did not deploy — because it was optional, and because its
trust root is the same hierarchy.

Ours inverts it. `peer_id` **is** a public key (V7 §1.5), so a binding is verified against the pinned
root by the consumer, and a registry that lies produces a signature that does not verify. **There is
no unsigned mode.** That is not a better version of DNSSEC; it is the property DNSSEC was retrofitted
to approximate, available because we started from pubkey-as-name.

**The cost, stated honestly:** we inherit key management, which DNS does not have. That cost is real
and it is paid in EXTENSION-IDENTITY (rotation, quorum recovery), not here.

### §3.2 Query privacy is at the substrate, not per-backend

The founding plan §9 explicitly **deferred** query privacy: *"Real concern, but a layered concern —
encrypted DNS, oblivious DNS-over-HTTPS, etc., have analogs here. Per-backend."*

**We did not do that, and the divergence is the right one.** §4.1 step 2's name-disclosure `[MUST]`
plus §4.1b's broad-pattern classifier put it in the **dispatch filter** — before any backend is
consulted, on the configuration rather than on the query. DNS took twenty years to get from "resolver
sees everything" to DoH, and DoH still tells the resolver operator every name you look up; ODoH is
the actual fix and is barely deployed.

Ours is not a transport wrapper. It is a rule about which backends may be *told a name at all*, which
is a category DNS has no place to put. **This is the strongest single thing in the extension and it
is the one that is v1-IN.**

### §3.3 No CNAME, correctly

CNAME exists in DNS for three reasons, and each dissolves here:

1. **CDN fronting** — point `www` at the CDN's name. We publish content-addressed; the same root is
   served from any number of origins with no name involved.
2. **Service migration** — the host moved. `peer_id` is stable across rotation and across hosting, so
   the binding does not change when the address does.
3. **Vanity / apex aliases** — two names, one thing. Issue two bindings to the same `target_peer_id`;
   there is nothing to alias.

CNAME is a symptom of names being bound to *locations*. Ours are bound to *keys*. **Adding a
name→name indirection would import DNS's aliasing-loop and apex problems for no capability we lack.**

---

## §4 The gap: every id-first system in the survey has a peer-id → endpoint answer, and we do not

This is the finding. It is not stylistic and it is not new — **it was named in the founding plan,
given a concrete design, and never landed.**

### §4.1 What is actually missing

Three cases; two work:

| You have | Path | State |
|---|---|---|
| a **name** | REGISTRY `:resolve` → binding carries glue `transports` | ✅ works, three-way green |
| a peer you have **contacted** | its `system/peer/transport/{P}/*` profile, cached | ✅ works |
| **only a `peer_id`**, never contacted, no name for it | — | ❌ **nothing** |

### §4.2 It is a closed loop of pointers around an empty centre

Verified by exhaustive search of the corpus, not a partial grep:

- **`EXTENSION-NETWORK` §6.5.4:** *"Post-corridor: the EXTENSION-REGISTRY.md extension will define
  peer-ID → endpoint-set resolution."*
- **`EXTENSION-REGISTRY` §12 (out of scope):** *"Address-discovery (`runtime-peer-endpoint`) —
  reverse peer_id → endpoint lookup. **Lives in EXTENSION-IDENTITY amendment, separately
  authored.** `:resolve` is name-keyed by contract."*
- **`EXTENSION-IDENTITY`:** the string `endpoint` occurs **zero times**.
- **`runtime-peer-endpoint` occurs exactly once in the entire corpus** — in the REGISTRY line above,
  pointing at the amendment that does not exist.

NETWORK assigns it to REGISTRY; REGISTRY assigns it to an IDENTITY amendment; IDENTITY does not have
it. **This is AP-18's shape — a pointer at a phantom is indistinguishable from a pointer at a
document you have not opened** — and it is the same residue class as
`PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE`, which cost two app-tier seats two incompatible publishing
surfaces.

Meanwhile `EXTENSION-RELAY` §3.5 describes REGISTRY as *"the A+MX-in-one-zone resolution"* and routes
inbox-relay discovery through it — **which only works if you have a name**, because `:resolve` is
name-keyed by contract. RELAY's stated resolution path is unavailable for exactly the case it is most
needed in.

### §4.3 Why DNS is no guide here, and why the other systems are

**DNS never needed this, because DNS is name-first by construction.** Nobody holds a bare IP and asks
"what is this and how do I speak to it" — PTR exists and is vestigial. So the 40 years of DNS
experience has nothing to say about the case, and the analogy stops being useful precisely here.

**We are id-first by construction** — `peer_id` *is* the identity, so bare ids circulate as
first-class references in a way DNS names never do. Landed specs that hand you a bare `peer_id` and
expect you to reach it:

- **`runtime-peer-set`** — Alice's *other* runtime peers, by id. This is the founding plan §6's
  motivating case, verbatim.
- **`EXTENSION-RELAY` §3.5** — the destination's inbox-relay, resolved from the destination's id.
- **group rosters** — members by id.
- **capability grants** — a grantee by id.

**And every id-first system in the survey solves it, each in its own idiom:**

| System | Its answer |
|---|---|
| libp2p / IPNS | the DHT — *"records typically ARE multiaddrs; peer-ID → multiaddrs is the DHT's primary content"* |
| AT Protocol | the DID document's endpoint field → the PDS |
| Nostr | NIP-65, the user's own published relay list |

Three designs, one shape: **the endpoint set is published by the id-holder, signed by the id-holder,
and fetched by id.** That is also exactly what the founding plan recommended.

### §4.4 The fix already has a design, and it was reasoned

`PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE` §6 weighed three options and picked B:

```
system/identity/runtime-peer-endpoint := {
  runtime_peer:  system/hash,
  endpoints:     [<opaque endpoint string>],
  expires_at:    primitive/uint?,
  supersedes:    system/hash?,
}
```

signed by **the runtime peer itself**, at `system/identity/public/runtime-peer-endpoint/{peer}/{hash}`.

Its rejection of Option A ("put addresses in the identity attestation") is the part worth keeping,
because **we shipped Option A anyway** in the form of mandatory `transports` on a registry-signed
binding:

> *every address change forces Public_alice to re-sign and republish… couples address-rotation
> cadence (frequent, per device) to identity-attestation cadence (rare). Also, addresses are
> properties of runtime peers, not facts about identity — Public_alice attesting to addresses she
> might not even know is a **category mistake**.*

Substitute "the registry" for "Public_alice" and that is §6a.3's `[MUST]` non-empty `transports`.
**That MUST is still correct** — it was ruled on a real argument (a static publisher has no profile
to discover, so an issued binding with no transports is operationally worthless) — but it is glue,
and DNS's whole lesson about glue is that it is a **bootstrap hint that goes stale**, which is why
the child's own A record is authoritative. We have the child's record (`system/peer/transport/*`).
We have the glue. **What we never built is the path from an id to the child's record.**

### §4.5 Scope

**Not v1, and not close to it.** It reopens nothing under `COHORT-OPEN-ITEMS` §0a: no name reaches a
third party, no binding verifies against anything unsigned.

> **§4.5a — the first version of this section said it is IDENTITY's, and that was wrong
> `[corrected 2026-08-20, operator]`.** It read: *"It is not even REGISTRY's — by REGISTRY's own §12
> it is IDENTITY's, and that scoping is right: the id-holder publishes its own endpoints."*
> **The second clause does not support the first.** "The id-holder signs it" is an argument about
> *who signs*; it says nothing about *which extension owns the type*, and I used one to conclude the
> other. What actually happened is that I read `EXTENSION-REGISTRY` §12's pointer and adopted it —
> the citation was the whole derivation. **L0's first line is reason it out from the ground up, not
> from what a peer produced and not from an outside standard; a landed sentence of our own is the
> same trap wearing a citation.**
> **And the pointer it defers to is circular.** §2.3 says *"Reverse `peer_id → binding` lookup is
> address-discovery and is scoped to EXTENSION-IDENTITY per §12."* §12 says *"`:resolve` is
> name-keyed by contract (§2.1 / §2.3)."* **Each cites the other and neither derives it.** The one
> real argument in the neighbourhood is §2.3's, and it is about something narrower and correct — see
> §4.7.
> The full derivation is §4.6. **It is NETWORK's.**

### §4.6 Whose is it? Derived, not cited

**Test the claim directly: what would a peer need the IDENTITY extension *for*, in order to be told
where another peer is?** Nothing survives the question.

1. **`peer_id` is substrate, not identity.** V7 §1.5 — every peer has one, whether or not
   EXTENSION-IDENTITY is installed. The query key is not an identity-extension concept.
2. **The answer is a NETWORK type.** `system/peer/transport/{peer_id}/{profile-id}` is defined in
   `EXTENSION-NETWORK` §6.5.1, lives in NETWORK's namespace, and is consumed by NETWORK's §10
   dispatcher. There is no identity object anywhere in the question or the answer.
3. **IDENTITY's actual job is a different mapping.** It maps a durable user identity to the set of
   peer-ids that speak for it, **across rotations, over time** — `Public_alice → {peer_id}*`. This
   is `peer_id → {address}` **now**. Different key, different value, different lifetime, different
   signer, different failure mode. They compose (you may run one then the other); they are not the
   same layer.
4. **Making it IDENTITY's would break NETWORK's own model.** §6.5.1a says a peer **SHOULD**
   self-publish its profiles and consumers **MUST NOT** assume the path exists. A bare peer with no
   identity extension is squarely in scope there. Scoping its reachability to IDENTITY means a peer
   that never adopted an optional extension cannot be reached by id — while `EXTENSION-DISCOVERY`
   simultaneously promises that *"an admitted peer is dialed over whatever transport profiles it
   advertises (§6.5)"* and that *"identity verification is post-admission and out of scope."*
   **DISCOVERY explicitly puts identity after the dial. IDENTITY cannot be a prerequisite of it.**

**Where the IDENTITY framing came from, since it was not invented here.** The founding plan's §6 was
written from inside the identity stack, solving *"reach Alice's **other** runtime peers."* That is
two steps — `Public_alice → {peer_id}` (IDENTITY's) then `peer_id → {address}` (generic) — and the
plan placed the record for the second step at `system/identity/public/runtime-peer-endpoint/…`,
inside the first step's namespace. **The plan's own critique of its Option A is the argument against
its Option B's placement:** *"addresses are properties of runtime peers, not facts about identity."*
It correctly moved the **signer** to the peer and left the **record** in the identity namespace.

**Conclusion: the record is NETWORK's.** REGISTRY may host a *lookup backend* over it — that is
REGISTRY's competence and §4.7 explains why the contract already accommodates it — but the type,
the signature rule and the freshness rule belong beside `system/peer/transport/*`, which is where
every other statement about how to reach a peer already lives.

### §4.7 §2.3's argument is good, and it is not the argument it was used for

§2.3 pins that the **transport-fallback loop** re-resolves the **original name** held in the NETWORK
§6.6 session, not the peer-id. **That rule is correct and this exploration does not touch it:** the
name is the thing with a revocable, attributable binding behind it, and re-resolving the id would
silently drop the binding's provenance and its revocation path.

**But that is a rule about one loop.** It was widened, in one sentence and with no derivation, into
*"`:resolve` is name-keyed by contract"* → *"reverse lookup is address-discovery"* → *"scoped to
EXTENSION-IDENTITY."* **Four steps; only the first is argued.** Adding an id-keyed lookup takes
nothing away from the fallback loop.

### §4.8 And the id-keyed lookup is the *easiest* one in the system, which makes excluding it worse

A `name → peer_id` binding needs an authority, because a name is not a key: somebody must vouch that
`alice` means this peer, and that vouching is where the registry's entire trust apparatus lives.

**`peer_id → address` needs no authority at all.** The query **is** the key. The answer is signed by
that key. A consumer verifies against the thing it already typed in. There is no trust anchor to
configure, no issuer to pin, no revocation to chase, and **no name to disclose** — which is the one
thing v1's central privacy MUST exists to protect.

It is also **already grammatical**. §4.1a row 2 dispatches `did:key:*` to the `self-certifying`
backend, and a Base58 peer-id is a `did:key` in different clothing — the §2.4.1 vocab table already
carries `self-certifying` as a binding kind. **The resolver contract can express this query today;
what does not exist is a backend with an answer.**

### §4.9 What exactly is an endpoint? — three types, one word, and one of them is loose

The question is fair, and the corpus does not answer it consistently:

| Where | What it actually is |
|---|---|
| `system/peer/transport/*` `.endpoint` | a **transport-specific object**; live transports pinned to `{url: "<scheme>://…"}` by §6.5.1a D4 |
| `services[].endpoint` (§3b) | a **URI string** — `stun:` / `turn:` / `wss:` — and §3b.0 is an entire subsection ruling that it is *NOT* a §6.5 endpoint object |
| `binding.transports` (REGISTRY §3) | prose says `[<endpoint per NETWORK §6.5>]`. **Every implementation carries `[system/hash]`** — go `[]hash.Hash` (`registry_ext.go`), py `array_of system/hash` (`definitions.py`). It is a **hash array of profile entities** and the spec never says so |

**The third one is a live spec-vs-impl gap of exactly the class §3b.0 was written to close** — that
subsection exists because two readings of "endpoint" produced different bytes, different hashes, and
two peers that never meet. It got pinned for `services` and left loose one field over. Filed as R-24.

**And the answer that matters for §4:** because `transports` is a **hash array**, a binding does not
contain addresses — it contains *references to profile entities*. Which surfaces the real structural
problem, below.

### §4.10 The deeper finding: a transport profile is not self-authenticating

`system/peer/transport/*` carries `peer_id` **inside the entity** — "the peer this profile is for."
Nothing signs it. Its authority is **positional**: it is trustworthy because it sits at
`{peer_id}/system/peer/transport/…` in that peer's own namespace.

**Position is exactly what is lost when the entity travels.** Fetched by hash from a registry, a
relay, or a cache, all a consumer has is a content-addressed blob asserting `peer_id: P` — and the
hash proves the bytes, never the authorship. Anyone can mint `{peer_id: P, endpoint: <mine>}` and its
hash is perfectly valid. What stands between that and a consumer today is **the registry's signature
over the binding**, which means: *the registry is vouching for where P is.*

That is a category the corpus already rejected once. §6.5.1a's own model is that a peer
**self-publishes** where it is. The registry was supposed to say *who* `alice` is, not *where* P is —
and it ends up saying both because the peer's statement has no portable, self-authenticating form.

**The prior art here is direct and we have not read it.** libp2p shipped unsigned address records in
its DHT, found them forgeable and unattributable once they left the origin, and introduced **signed
peer records** — a signed envelope over `(peer_id, addresses, seq)` — as a protocol change. *(External
knowledge, not an in-corpus citation: `ANALYSIS-CONNECTIVITY-LIBP2P-COMPARATIVE-AUDIT-AND-LANDSCAPE`
maps reflection and ICE candidates and **has no row for peer records at all**, which is itself a gap
in that audit.)* ATProto's DID document and Nostr's NIP-65 relay list are the same object in two other
idioms. **All three are: signed by the id-holder, carrying a sequence number, servable by anyone.**

**We are where libp2p was before that fix**, with the damage bounded by the registry's signature
rather than open to anyone — which is better, and is still the wrong party signing.

---

## §5 The two smaller gaps, and one non-gap

### §5.1 "What does this peer serve?" has no mechanism — and `services` is not it

`services` (§3b) is **deployment-shared infrastructure** — STUN reflector, TURN data-relay,
signaling carrier, shared inbox-relay. §3b says so directly (*"Reflector and signaling are shared
infrastructure, not attributes of any one peer"*). It is an SRV record for the **deployment**.

The per-peer question — *does this peer serve a site, and where* — is answered today by **convention
plus browse**, which is to say by nothing normative. DNS answers it with SRV per name; ATProto with
the DID document's service array; Nostr does not answer it at all.

**This is one field on an answer that already exists, not new architecture** — and it wants to be a
per-peer record in the peer's own namespace (like `system/peer/inbox-relay`, which already is one),
not another field on a registry-issued binding. **The MX we have is the template for the SRV we
don't.**

### §5.2 Delegation (NS) — direction pinned, mechanism absent

`resolver_chain` is a **client-side ordered list**, which is `/etc/resolv.conf`, not NS. There is no
way for a registry to say *"names under `*.acme` are answered by that registry."* Today that is
configured per-consumer, so it scales with the number of consumers rather than the number of zones.

The direction is chosen (§6a.7's sharded/delegated manifest, on the DNS-zone / TUF model) and
federation is RELAY Mode A, deferred. **This is the right thing to defer** — DNS's hierarchy is also
its central point of political failure, and every surveyed post-DNS system deliberately declined to
rebuild it. But it should be deferred *knowingly*: the thing that breaks first at scale is not
resolution, it is **configuration distribution**.

### §5.3 Browse as a `SHOULD` — DNS already ran this experiment, and reversed it

Whether §6a.3a browse should become a `MUST` is a live v2 question. **DNS answered it: AXFR was
open by default and is now closed by default nearly everywhere**, because enumerating a zone is a
reconnaissance gift and the operational need is rare. DNSSEC's NSEC then re-leaked it by accident and
NSEC3 was invented to patch it.

**A registry that must be enumerable is a registry that must publish its whole customer list.** Our
`coverage: complete` manifest already gives the one property enumeration was wanted for —
authenticated denial — without the enumeration. **Keep browse a `SHOULD`.** The zero-consumer state
that `entity-browser-rust` measured is not evidence it should be promoted; it is evidence nobody has
needed it yet.

### §5.4 The non-gap: identity rotation propagation (legacy Q-C)

The founding plan left this open. It is closed by construction: `peer_id` is the **stable
cross-rotation identifier**, so a rotation does not change the binding at all. Only a *compromise
recovery* that replaces the identity does, and that is the quorum path in IDENTITY, TTL-bounded here
like everything else. **No action; recorded so the question is not re-opened a third time.**

---

## §6 Which unbuilt backends are worth building, from the survey

All four web-native backends are **paper** (`GUIDE-RESOLUTION` §6.4) and all four are cut to v2. They
are not equal, and the survey ranks them:

1. **`well-known-url` — build this one first.** It covers **DID:web, Nostr NIP-05 and ActivityPub
   WebFinger with one backend**: three ecosystems, one HTTPS fetch, one JSON parse, ~200 LOC per
   impl by the legacy estimate. It is the single highest coverage-per-effort item in the naming
   stack.
2. **`dns-txt` — lower value than it looks, and this answers the "is the TXT thing relevant"
   question directly.** It does the *same job* as `well-known-url` — prove control of a DNS domain —
   by a worse route: a second resolver dependency, DNSSEC-or-DoH trust qualification to specify, and
   a 255-byte-per-string record format to fight. **Its one genuine advantage is that it works
   without an HTTPS server**, which matters for a domain owner who has DNS but no web host. Build it
   when someone asks for exactly that.
3. **`did-web`** — mostly subsumed by (1); the delta is the DID-document schema and inbound/outbound
   `present-as`. Worth it if DID interop is a product goal, not otherwise.
4. **`consensus-anchored` (ENS)** — real demand exists in one community, requires an Ethereum RPC
   dependency, and `.eth` is already the only typed suffix v1 admits. Genuinely last.

**And one the survey says we should probably not build ourselves: the DHT.** libp2p Kademlia is the
canonical implementation and the legacy guidance is to wrap it rather than reimplement. Note that a
DHT would also close §4 — that is not a coincidence, it is what DHTs are *for* — but it is a large
dependency for a problem a signed self-published record solves locally.

---

## §7 Verdict

**The design maps, and the mapping is deliberate rather than lucky.** Twelve of fifteen DNS record
types have an answer; the two deferred ones have their direction pinned to a named prior-art model;
the three places we diverge from the landscape (signature-anchored rather than transport-trusted,
query privacy at the substrate, no name→name aliasing) are all divergences **toward** the stronger
property, and each survives being argued against its DNS counterpart.

**The decomposition is the strongest part** and it is the thing the surveyed systems mostly get
wrong: *name → peer* (REGISTRY), *what peers exist* (DISCOVERY), *how do I reach a peer* (NETWORK)
are three questions with three mechanisms. DNS answers 1 and 3 together and has no 2. mDNS/DNS-SD
answers 1+3 and bolts 2 on. Nostr answers none of them and pushes all three to the client.

**Zooko's triangle is the frame this corpus never named, and we should — because we pass it.** Human-
meaningful, secure, decentralized: pick two, said the 2001 formulation. Our answer is not to beat it
but to expose all three corners as *different binding kinds under one resolver contract*:
`self-certifying` (secure + decentralized, not memorable) · `local-name` (memorable + secure, not
global) · `peer-issued` (memorable + global, trust the issuer). **The user picks the corner per
name, and the dispatch filter keeps the corners from leaking into each other.** That is the petname-
system answer (Miller/Stiegler, already cited in the petname proposal) and it is a better statement
of what we built than anything currently in `GUIDE-RESOLUTION`. Worth folding.

**The one real hole is §4**, it was known in 2001-vintage form before this repo existed, it was named
in the founding plan as *"still the load-bearing small piece,"* it has a four-field design and three
landscape templates, and it is currently a triangle of pointers around nothing.

---

## §8 What this exploration proposes doing about it

Nothing before the release. In order after:

| # | Item | Where |
|---|---|---|
| 1 | **R-22** — the address-discovery hole. **In `EXTENSION-NETWORK`, not IDENTITY** (§4.6): a signed, portable, `seq`-bounded statement by a peer about its own transports, plus a lookup path for a holder of only a `peer_id`. Drafted as `PROPOSAL-PEER-TRANSPORT-SET`; fixes the NETWORK §6.5.4 / REGISTRY §12 / §2.3 pointer loop as part of it | proposal, NETWORK tier |
| 1b | **R-24** — `binding.transports` is prose-typed `[<endpoint per NETWORK §6.5>]` and is `[system/hash]` in every implementation (§4.9). Same class §3b.0 closed for `services` | hygiene, REGISTRY §3 |
| 2 | **R-23** — per-peer service advertisement, on the `system/peer/inbox-relay` template rather than as a binding field | proposal, NETWORK or a new per-peer record |
| 3 | Fold Zooko's triangle + the three-corner framing into `GUIDE-RESOLUTION` §6 | guide, hygiene |
| 4 | Rank the paper backends per §6 in `GUIDE-RESOLUTION` §6.4 so *"all four are paper"* stops implying they are equal | guide, hygiene |
| 5 | Record in `SYSTEM-ARCHITECTURE` and `EXTENSION-REGISTRY` §13 that the source proposals are **legacy**, so the citations stop reading as in-corpus | hygiene, part of R-20 |
