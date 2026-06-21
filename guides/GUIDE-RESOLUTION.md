# GUIDE-RESOLUTION — how a name becomes bytes

**Status**: Active
**The meta-rule:** a resolution claim is **not validated until a cross-impl conformance test exercises the composed chain.** Prose — including this guide — does not catch resolution bugs. The `RESOLVE-CHAIN-*` vectors (§11) are the validation.

---

## 1. The two primitives everything rests on (pointer)

Both normative (V7 §1.4, §1.7; PRIMER):

- **(P1) Getting an entity is always two hops.** `tree: path → hash`, then `content: hash → bytes`. The tree is `path → hash`, **not** `path → entity`; the content store holds `hash → bytes` once, deduplicated. *Every* materialization is "resolve path to hash, then hash to bytes." This is why path-resolution and content-resolution are **two rungs with two different trust characters** (§5).
- **(P2) One universal address space; each peer holds a local *view*.** Every path is `/{peer_id}/{rest}`. A peer's tree holds `/{its_own_id}/…` (authoritative) and `/{other_id}/…` (its cache of others). Only the keyholder's data for `/{peer_id}/…` is canonical — you trust it by **signature, not by where you got it.**

---

## 2. The foundational principle: names are receiver-relative

**There is no global namespace.** A name has **no meaning on its own** — it means what *your* `resolver-config` says it means. The same string can resolve to different peer_ids for two users running different configs (REGISTRY §1, §4, §5: *"the substrate gates no name claims … whether a receiver TRUSTS that binding is the receiver's policy"*).

Three consequences that govern everything below:

1. **"Trusted" is a property of the *path you resolved through*, not of the name.** `alice` is trusted *for you* because your config pinned a registry whose signature you verify. For someone else it may be unknown or resolve elsewhere. (REGISTRY §7.4: **asserted** vs **verified**.)
2. **Uniqueness is per-receiver.** Two registries may bind the same name to different peers; *your* config's priority order is the tiebreak, and cross-registry ambiguity **fails closed** by default (REGISTRY §4.1.1, §8.3).
3. **This is DNS-shaped without a single root.** Delegated authorities, TTLs, records — but the "root" is whatever your distribution preloaded and you chose to keep (REGISTRY §7 bootstrap-with-precedes; §7.3 = the OS-root-CA analogy). Global *agreement* on a name is emergent (many receivers trusting the same registry), not built-in.

This principle is the same fact as §5's "a **name is authority-scoped**": a name has no self-evident meaning, so only an authority's signature gives it one — which is *why* its resolution ladder must be trust-ordered, never promiscuous.

---

## 3. What do I have → where I enter the ladder

Every entry into the system is a **starting reference**, and each drops you onto the chain at a different rung. "The different pathways in" is just: *which reference do I hold?*

```
  WHAT I HAVE                    ENTERS AT                   NEEDS
  ───────────────────────────────────────────────────────────────────────
  a name      alice          top:    name → peer_id      a registry (Layer A)
  a peer_id   z6Mk… (Base58)     middle: peer_id → transport reachability (Layer B)
  a transport wss://… / http URL lower:  connect directly    nothing — dial it
  a content-hash  b3…            bottom: hash → bytes        substitute / any source
  a URL       name@auth/path     top, composed               the whole chain
```

The registry's job is **the top rung only**: name → (peer_id + initial transports + trust_anchor). Everything below is NETWORK / TREE / SUBSTITUTE.

---

## 4. The one shape, four ladders

Every resolution is the same three-move shape — *try nearest/most-authoritative → fall through a preference-ordered ladder → honest typed dead-end* — differing only in target, rungs, and **how promiscuous the ladder may be** (§5).

| Resolving | → | Ladder (near → far) | Op / hook | Spec home |
|---|---|---|---|---|
| **name** | peer_id (+transports) | local-name → pinned peer-issued → dns-txt/did → aggregators → `not_found` | `system/registry:resolve(name)` | REGISTRY §2, §6, §7 |
| **peer_id** | reachable transport | self → cache → host-vouches → registry transports → `Unreachable` | dispatcher ladder; `system/peer/transport/{peer}/{profile}` | NETWORK §6.5, §10 |
| **content-hash** | bytes | local store → (pending) → substitute backends → `404` | transparent miss-hook `substitute_consult` | SUBSTITUTE §3, §5 |
| **tree-path** | hash | local tree → *(authority: owner's signed snapshot)* → `404` | `system/tree:get(path)` (local) | TREE §1; V7 §1.7 |

**A full URL is these four composed in sequence** — a dead-end at any rung is the honest answer for the whole chain, tagged by rung:

```
name ─REGISTRY▶ peer_id ─NETWORK▶ transport ─(connect)▶ tree-path ─TREE▶ hash ─SUBSTITUTE▶ bytes ▶ render
```

There is no separate "browse engine" — browsing *is* walking this chain and re-walking it on each click.

---

## 5. The placement rule: self-verifying vs authority-scoped

This is *why* substitute is a server-side extension but resolution-orchestration (`resolve()`) is the SDK — and it is structural, not a style call. Two questions decide where any resolver lives:

**Q1 — Is the target self-verifying?** A self-verifying target (content-hash → bytes: the bytes hash-match from *any* source) needs **no consumer-side trust state** — the hash replaces trust. Its ladder may run **server-side, transparently, promiscuously**. An authority-scoped target (name, path: value depends on *who owns it*, P2) requires the **consumer's own trust policy** at each step, which is consumer-side state.

**Q2 — Across how many authority boundaries, and whose capability?** Substitute fetches from **non-authoritative byte sources** (CDNs, dumb HTTP) — it never spends the consumer's authority against a third authority-bearing peer. Resolution crosses **multiple authority-bearing peer boundaries**, each gated by **the consumer's own** capability — which lives consumer-side.

| Resolver | Self-verifying? | Consumer walks it? | Lives | Status |
|---|---|---|---|---|
| **substitute** (content) | yes — hash anchor | n/a — transparent hook | **extension** | landed v1.0 |
| **`resolve()`** (the chain) | no — authority-scoped rungs | yes | **SDK** | proposal (Draft) |
| **`system/resolve` delegation** | no | no — thin client (browser-behind-NAT) | **extension forwarding the consumer's cap-chain** | deferred |

They are **one family at three points.** The delegation handler is *resolve-as-an-extension*, and it forwards the requester's cap-chain — the **same mechanism substitute uses** (`substitute_consult` threads `cap_chain`). Substitute skips the ceremony only because its target is self-verifying.

**The rule:** a resolver lives **server-side as an extension** when its target is **self-verifying**; **client-side in the SDK** when it **composes across authority-scoped boundaries**; and when a thin consumer can't walk it, it **delegates to a handler that forwards the consumer's cap-chain** — never one that acts with its own authority.

---

## 6. Name syntax & semantics — "name at registry"

**The mechanism is fixed; the syntax is a convention.** Every lookup is one call: `system/registry:resolve(name)`. Which backend answers is decided by `name_format_dispatch` (REGISTRY §4.1 step 2) — a list of **POSIX-glob → backend-kinds** rules matched against the name string — then by `resolver_chain` priority. So *the name's shape selects the naming authority*, and the substrate does not care what the shape is. This guide standardizes **four shapes** so apps, links, and registries all speak the same grammar. **No substrate change** — the grammar is realized by the default `name_format_dispatch` globs a distribution ships.

### 6.1 The four shapes (recommended grammar)

| Shape | Example | Means | Routes via glob (example) | Backend |
|---|---|---|---|---|
| **bare name** | `alice` | "resolve through my default chain" | `*` (catch-all) | local-name, then default registry |
| **registry-scoped** (email-shaped) | `alice@entity-church` | "this name **at** this registry" | `*@entity-church` | peer-issued (the named registry) |
| **domain-scoped** | `alice@example.org` | "resolve via this DNS domain" | `*@*.*` (dotted authority) | dns-txt / well-known-url |
| **scheme-typed** | `did:web:example.org`, `did:key:z6Mk…`, `example.eth` | a naming system with its own syntax | `did:web:*`, `did:key:*`, `*.eth` | did-web / did-key / consensus |
| **self-certifying** | `z6Mk…` (a Base58 peer_id) | the name **is** the key | decodes as V7 §1.5 peer-id | none — self-verifying |

This is exactly the user's framing: **`alice@entity-church`** reads "Alice *at* the Entity Church Registry" — the `@authority` names *which* registry, the local part is the name *within* it. Bare `alice` works too when the Entity Church Registry is your default — `@authority` is for **disambiguation / explicit targeting**, not required for everyday use.

### 6.2 Why `@` (email-shaped) for the primary registry-scoped form

- **Matches the mental model.** "Alice at Entity Church Registry" *is* `local-part@authority`.
- **Universally familiar.** Everyone already parses `name@domain` as "name within an authority."
- **Cleanly distinguishable.** Presence of `@` ⇒ scoped; absence ⇒ default chain; leading `scheme:` ⇒ typed system. One glance routes it.
- **The authority part is polymorphic, dispatched by *what it is*:** a known **registry handle** (`@entity-church` → a pinned peer-issued registry) vs a **DNS domain** (`@example.org`, dotted → dns-txt/well-known). The dispatch globs separate them.

**Alternative considered — a `reg:` scheme (`reg:entity-church/alice`).** Valid and a glob can route it; rejected as the *primary* form because it's heavier and less familiar than `@`. Kept available for tooling that prefers explicit schemes. (Design-space: the substrate accepts any convention; we recommend `@`, document `reg:` as also-valid.)

### 6.3 The targeting unification (name@X where X identifies the authority OR pins the peer)

`name@X` qualifies the name by `X`. The resolver dispatches on **what X is** — and this unifies *targeting* with the link-form's *verification pin* (PROPOSAL-UNIVERSAL-RESOLUTION §7):

- **X is a registry handle / domain** → "resolve `name` *using* authority X" (targeting — §6.1).
- **X is a Base58 peer_id** → "resolve `name` however my chain does, but the result **MUST equal** peer_id X" (verification pin — name for reach, peer_id for trust; the cross-publisher link form).

Because peer_ids are Base58 multikey (V7 §1.5) and registry handles/domains are not, the resolver tells them apart by structure. This is the **recommended unification** and the cleanest answer to "reverse-lookup when clicking around": **links carry `name@peer_id`** — name to stay a *forward* registry resolve through the host-vouches rung, peer_id to verify the answer. **Open design point for cohort** (§11): confirm the dispatch-on-X rule and whether one separator should carry both meanings or they want distinct separators.

### 6.4 Backend status — and where the community comes in

The grammar above names every backend the substrate is *designed* to carry. Most are **not built yet** — the substrate contract exists, the concrete backend doesn't. We are shipping the substrate + the conventions and **actively want community input** on the per-backend mechanics (especially the web-native ones), because how an entity peer maps onto DNS / well-known / DID is a place where existing-web interop matters more than our preferences. Honest status:

| Backend | Status | What's missing |
|---|---|---|
| **self-certifying** (name = peer_id) | **built** (trivial) | — |
| **local-name** (your own handles) | **built v1.0** | — |
| **pinned / out-of-band** | **built** | — |
| **peer-issued** (a registry vouches; the Entity Church Registry path) | **spec'd, backend not built** | the registration flow + the backend handler that fetches+verifies a signed binding (REGISTRY §7.4 specs the flow; the impl is the unblock) |
| **dns-txt** | **paper** — *working out the kinks* | record format; DNSSEC/DoH trust qualification; the resolver backend |
| **well-known-url** | **paper** — *working out the kinks* | the `.well-known/` path + binding artifact shape; the resolver backend |
| **did-web / did-key** | **paper** — *working out the kinks* | DID-document ↔ binding mapping; inbound resolve + outbound present-as |
| **consensus-anchored** (e.g. ENS) | **paper / future** | chain-record format; the resolver backend |
| **aggregator / federation** (registries-of-registries, live) | **deferred** | RELAY Mode A cross-peer subscription |

**Open invitation:** the web-native backends (dns-txt, well-known-url, did-web) are exactly where we want naming-services / authorities to propose the mapping that fits how *they* already publish identity. The substrate's contract (REGISTRY §2–§5) is the fixed part; the binding-artifact shape per pathway is open. A backend "works" once it speaks the resolver contract, returns a verifiable `ResolutionResult`, and a receiver chooses to trust its `trust_anchor`.

### 6.4a What the grammar is NOT

- **Not a new substrate surface.** It is `name_format_dispatch` globs + a documented convention. Promoting it to normative = a short proposal that pins the default globs + the dispatch-on-X rule, nothing in the kernel.
- **Not a global parser.** A deployment MAY ship different globs; the grammar is the *recommended interoperable default*, not a wire format. Two peers that ship the standard globs interoperate; one that doesn't simply routes its own way.

---

## 7. Trust model — whom you trust, and how registries relate

Trust is **receiver-side** (REGISTRY §5). You decide which authorities you accept, per `resolver-config`:

- **`accepted_trust_anchors`** per chain entry — which `trust_anchor` variants you'll accept from that backend (e.g. only `peer_issued:{entity_church_peer_id}`). A binding that doesn't match is rejected; the chain advances. Fail-closed.
- **Pinned bindings** override everything (REGISTRY §4.1.1) — your explicit `name → peer_id` assertions, auto-trusted.
- **Signature verification** on every non-local-name, non-self-certifying binding (REGISTRY §3) — the binding is verified against the issuing authority's key, located at `system/signature/{hex(binding.content_hash)}` (V7 §5.2 invariant-pointer).
- **Revocation honored** (REGISTRY §3.1) and **expiry** (`issued_at + ttl`) before any `resolved`.

**"We trust these registries; community runs their own."** This is just multiple backends in your chain, each with its own accepted anchors:

```
resolver_chain (priority asc):
  0  local-name            (your own handles — free, authoritative for you)
  1  peer-issued: entity-church-registry   accepted: [peer_issued:{ec_peer_id}]   ; the standard entity-native registry
  2  peer-issued: example-registry        accepted: [peer_issued:{ex_peer_id}]   ; a community registry you also trust
  3  did-web / dns-txt      (web-anchored names)         ; when those backends ship
```

**Consulting several registries = "registries of registries", three shapes** (EXPLORATION-REGISTRY-NETWORK §4):

1. **Chained config (BUILT today)** — the chain above *is* "consult several registries." Priority order + `name_format_dispatch` route the query. Most of the felt need is already here.
2. **Static aggregation via precedes (BUILT)** — a distribution preloads signed bindings from many registries, each verifiable against its own issuer (REGISTRY §7). A union baked at build time; works offline; only lacks live freshness.
3. **Live federation / aggregator (DEFERRED)** — a Mode-A relay subscribes to N registries and serves the live union as one backend (REGISTRY §8.2); deferred on cross-peer subscription. Does **not** re-sign — receivers verify originals.

**Registries vouching for registries** (cross-reference / hierarchy) is expressible via `issuer_attestation` on a peer-issued binding (REGISTRY §3 — "the registry's authority cert"): registry A can attest to registry B's authority, letting a receiver who trusts A extend trust to B. This is the consensus/hierarchy direction; **named, not yet specified** — lives with the peer-issued backend proposal (§8).

---

## 8. Publishing a name — how one gets *into* a registry

Resolution (§1–§7) is the **read** side. Getting a name *others* can resolve is the **write** side, and it splits sharply by what's built:

- **Self-asserted, local (BUILT — REGISTRY §6.5).** `local-name:bind(name, peer_id, transports)` writes a binding others-can't-see. Good for your own contacts; never leaves your peer (§6.7). Does *not* make the name globally resolvable.
- **Distribution-preloaded, signed (BUILT — REGISTRY §7.4).** The day-one release ships the **Entity Church Registry**: its identity key pinned as a trust root, a resolver-config entry for it, and signed **precedes** for the curated coral-reef set (Alice, entitycoreprotocol.org, entitychurchfoundation.org). First run resolves these **fully** — verifies each binding's signature against the pinned registry key — with **no live registry up** (the registry is itself a static coral-reef publisher over http-poll). *This is the working registration model the v1 demo runs on.*
- **Peer-issued registration against a live registry (GAP — the one real unblock).** The protocol for *"ask the Entity Church Registry to bind `alice → my_peer_id` and sign it"* — the request, the registry's **admission policy** (what it agrees to sign: first-come / review / allow-list — the registry's *own* cap surface, where anti-squatting lives), and an **ownership-proof challenge** (claimant signs a nonce to prove control of `target_peer_id`, so the registry can't be tricked into vouching for a peer it doesn't control) — is **undesigned.** It is the peer-issued backend, "its own proposal." Until it lands, names enter the system via preloaded precedes, which is honest and sufficient for release.

**The end-to-end the user described works today via precedes:** `alice@entity-church` → peer-issued backend → fetch the registry's signed binding (precedes or http-poll) → verify signature against the pinned Entity Church Registry key → `ResolutionResult{peer_id, transports}` → connect over http-poll → fetch the lab's content by hash. The *registration* of that binding (how `alice` got into the registry) is the gap; the *resolution* of it is built.

---

## 9. The capability model of resolution (pointer)

A resolution chain is also a capability chain — each peer-crossing rung is gated independently **by the target peer**, presenting **the consumer's own** cap (PROPOSAL-UNIVERSAL-RESOLUTION §4):

- `name → peer_id`: no cap to resolve (trust is receiver-side); failure is `not_found`/`PolicyRejected`, not `Denied`.
- `peer_id → transport` / connect: pre-authorized (V7 §4.2); failure is `Unreachable`. **Resolution never confers admission — IDENTIFY is the gate** (REGISTRY §5.1).
- `tree:get(path)`: needs read cap (V7 §6.3); the §4.4 discovery floor grants read on `system/type/*` + `system/handler/*` **only**.
- `content:get` + substitute: three caps, cheapest-first, fail-closed (SUBSTITUTE §8).

**Two structural facts** (the typed `Outcome` must distinguish them):
- **Reachability ≠ authorization.** You can resolve a name, reach the peer, and still be **`Denied`** at content. Resolve-success ≠ access-success.
- **Public sites are not free on the floor.** A publisher serving a public site MUST grant **anonymous/broad read** on the site subtree + content (V7 §6.9a seed policy); the §4.4 floor reads types/handlers, not content. **Demo action item** — without it, every fetch returns `Denied`.

---

## 10. Built vs paper (target work at what's real)

| Piece | State |
|---|---|
| Two-hop path→hash→bytes (P1) | **built / normative** |
| Registry substrate + meta-resolver + binding + resolver-config | **landed v1.0** |
| local-name backend | **landed v1.0** |
| bootstrap-with-precedes + Entity Church Registry worked example | **landed v1.0** (§7.4) — the demo's registration model |
| Substitute content ladder (HTTP Mechanism A) | **landed v1.0** |
| Receiver-relative naming model (§2) | **implied across REGISTRY §1/§4/§5; named here** |
| Name grammar `@`/scheme/bare (§6) | **recommended convention** — needs a small proposal to pin default globs |
| peer-issued backend + registration flow (§8) | **gap** — its own proposal (the real unblock) |
| dns-txt / did-web / well-known / consensus backends | **paper** — contract defined, each its own proposal |
| Live federation (Mode A aggregator) | **deferred** on cross-peer subscription |
| `resolve()` SDK seam + typed `Outcome` + `RESOLVE-CHAIN-*` vectors | **designed, not built** (Draft proposal; W1) |
| DID/DNS outbound (present-as did:key / did:web) | **deferred** (REGISTRY §12) |

---

## 11. Open questions

1. **Name grammar (§6):** ratify the `@`/scheme/bare default globs + the §6.3 dispatch-on-X rule (registry-handle vs peer_id-pin) as a small proposal. Confirm one-separator-two-meanings vs distinct separators.
2. **Peer-issued registration (§8):** the unblock. Admission policy (registry's own cap surface) + ownership-proof challenge. Its own proposal.
3. **`resolve()` SDK contract:** route the Draft to cohort; is the typed `Outcome` set (`NotFound`/`PolicyRejected`/`Unreachable`/`Denied`/`Pending`) complete + minimal across impls?
4. **`Denied` vs `NotFound` confidentiality:** the SDK MUST surface the *peer's* status faithfully, never synthesize a more-informative `Denied` where the peer chose `NotFound` (V7 §5.5a). Confirm cross-impl.
5. **Registry-vouches-for-registry (§7):** the `issuer_attestation` cross-registry trust direction — scope with the peer-issued proposal.
6. **Validation:** the `RESOLVE-CHAIN-*` vectors (full `name@registry/path → bytes` walk) — the only thing that *validates* this guide per the meta-rule.

---

## Spec homes (canonical — normative text lives here, not in this guide)

- **Two-hop / address space / authority:** V7 §1.4, §1.5, §1.7
- **name → peer_id (substrate, binding, config, caps, trust, bootstrap):** EXTENSION-REGISTRY §2–§7
- **federation / aggregator (deferred):** EXTENSION-REGISTRY §8; RELAY §11.1a
- **DID/DNS bridges (deferred outbound):** EXTENSION-REGISTRY §12
- **peer_id → transport (Layer B):** EXTENSION-NETWORK §6.5, §6.6, §10
- **content-hash → bytes:** EXTENSION-SUBSTITUTE §3, §5, §7 (HTTP Mechanism A)
- **tree-path → hash:** EXTENSION-TREE §1; V7 §1.7; NETWORK §6.5.3.1
- **caps / discovery floor / seed policy:** V7 §4.2, §4.4, §5.5/§5.5a, §6.3, §6.9a
- **SDK `resolve()` + Outcome + vectors:** PROPOSAL-UNIVERSAL-RESOLUTION
- **model + registry-network analysis:** the universal-resolution model and the registry-network-and-naming exploration
