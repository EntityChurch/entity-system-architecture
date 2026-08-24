# PROPOSAL — ICE provisioning: consumption lifetime, the TURN half, and merge precedence

**Status:** DRAFT (2026-08-14) — **§1.2 partially ruled 2026-08-16 (§1.2a); §1.1 and §1.3 still open**
**Target:** `specs/extensions/EXTENSION-SIGNALING.md` §13 item 5 → a resolved disposition; possibly a
new §6.6 (consumption model) and/or an `EXTENSION-REGISTRY.md` §3b.6. **Which spec receives each
half is itself part of what this proposal must decide** — see §4.
**Landed so far (partial, no `EXTENSION-SIGNALING` rev bump — the disposition is partial):**
`EXTENSION-SIGNALING` §13 item 5 disposition narrowed per §1.2a; **`EXTENSION-REGISTRY` §3b.0a
(new, v1.6)** — the absent `data_relay` credential channel named as a deferral, with the
refuse-rather-than-fail-silently SHOULD. That section is complete as what it is, which is why it
carries a rev bump where the §13 row does not.
**Provenance:** `entity-browser-rust` `ROUTING-2026-08-14-who-tells-a-browser-which-ice-servers-to-use.md`
§4 and §6; the arch reply `ROUTING-2026-08-14-ice-server-placement-answer-to-browser-rust.md`.
**Scope:** design work, deliberately *not* folded into `EXTENSION-SIGNALING` v1.1 or
`EXTENSION-REGISTRY` v1.5. Both of those closed the halves that were settled; this carries the rest.

---

## 0. What is already closed, so this proposal does not reopen it

Two ratifications on 2026-08-14 answered the discoverable-source question:

- **`EXTENSION-REGISTRY` §3b** (v1.5) — the deployment-wide signed set: `services.reflector`,
  `services.signaling`, `services.data_relay` (with `policy: open/members/metered`),
  `services.inbox_relay`, returned on `:resolve`.
- **`EXTENSION-SIGNALING` §4.5.1** (v1.1) — a signaling node publishes **its own** §9.3 STUN
  listener in `advertise`, for the peer that reaches a carrier directly and never resolves.

**Between them the STUN half is answered end to end.** §9.3 forbids a reflector requiring
authentication, so nothing in that path carries a credential, nothing expires, and a read at
initialization is sound. That is why it could ratify without this proposal.

## 1. The three open questions

### 1.1 Consumption lifetime `[the hard one]`

A consumer that reads provisioning **once, at initialization**, cannot hold a credential that
rotates. `entity-browser-rust` states the constraint concretely: `InitParams.webrtc` is Init-only, so
provisioning is consumed at worker-`Init` and a connector change already requires a reload
(`ROUTING-2026-08-14` §6). A TURN credential rotating hourly — the normal case, HMAC time-limited —
cannot ride that snapshot.

Their sketched fix is that the establisher **re-reads immediately before each negotiation** rather
than trusting the boot snapshot, which they note is feasible (they already `reach_node` per meet) but
is *a different consumption model* than `InitParams` implies. **They explicitly declined to invent
it, and were right to** — two implementations inventing it separately is how a cross-peer seam
diverges.

**What this proposal must decide:** whether the spec pins a consumption model at all, or leaves it
implementation-local. The test is `AGENTS.md`'s: *does a divergence cross a peer boundary?* A peer
holding a stale TURN credential fails to relay — but it fails **locally and loudly** (its own
candidates do not gather), not by desynchronizing a pair. That argues implementation-local. Against
that: the rung-2 gate cannot be written without *some* pinned expectation of when provisioning is
re-read, and "implementation-local" has previously meant "unbuilt in every tree."

### 1.2 The TURN half — and the connector-by-URL path it does not reach

TURN is **not a surface a signaling node serves**, so it does not belong in `advertise` and §4.5.1
correctly excludes it. `EXTENSION-REGISTRY` §3b's `data_relay` is its home, and `policy` already
answers whose credential and who pays.

**The gap is the path that never resolves.** A browser peer adds a connector as a
`(node_peer_id, node_addr)` pair typed in directly (`entity-browser-rust` `src/connectors.rs`); no
registry resolve occurs, so §3b's signed set never lands. That peer can now discover STUN (§4.5.1)
and still has no route to a TURN credential.

Candidate shapes, as tabled:

| # | Shape | For | Against |
|---|---|---|---|
| 1 | A `data_relay` field on `advertise` | one round trip, the path already taken | a **node** is not the **deployment** and holds no authority to vouch for deployment-wide infrastructure — §4.5.1's "never a directory" rule; and puts a rotating credential on a boot-time read (§1.1) |
| 2 | The connector row carries it, app-tier | ships without upstream; user-editable | users hand-entering rotating credentials is a bad surface, as browser-rust already said of their own option |
| 3 | A connector URL implies a registry to resolve | reuses §3b whole, keeps one signed source | invents a discovery step on a path whose whole point is not having one |
| 4 | Ruled out of scope — direct-connector peers get STUN only | honest; covers the cone-NAT majority | leaves symmetric-NAT peers on that path with no fallback, permanently |

> **Correction to shape 1's objection, made on a re-read of the canonical sources.** As first
> written this row said the shape "asserts a signaling node vouches for infrastructure it does not
> run — the exact thing §3b signs *against*." **That is not what §3b says.** `EXTENSION-REGISTRY`
> §3b.5 is *"a deployment vouches for the infrastructure it advertises"*, and
> `GUIDE-REFERENCE-DEPLOYMENT` §3.4 explicitly contemplates a deployment advertising a
> **commodity/community relay it does not run** — *"the advertisement makes this a config choice,
> not a code change."* **Vouching is not running**, and the corpus already settled that at the
> deployment level. The real objection is narrower and survives: **the node is not the deployment.**
> §4.5.1's rule is about the *node* surface — a node publishes what it itself serves because it
> cannot sign for a set it has no authority over. Stated as "does not run" the objection also refutes
> §3b itself, which is how you can tell it was the wrong statement of it.

#### 1.2a `[RULED 2026-08-16]` — shape 4 is out, and the question underneath it is not placement

**Two rulings, and then the finding that makes the rest of §1.2 unrulable today.**

**Ruling 1 — shape 4 is rejected. The connector-by-URL path MUST be able to reach a data relay.**
This needs no symmetric-NAT evidence, because it is a ruling about *permanence*, not about
mechanism. Three things force it:

- **§3b already commits to serving that path.** Its own "Why deployment-scoped and not per-peer"
  note ends: *"the connector-entered-by-URL path, which never performs a resolve at all, is served
  by `EXTENSION-SIGNALING.md` §4.5 rather than by this entity."* Shape 4 would truncate a path the
  ratified spec says is served.
- **It truncates §3b.4's ladder permanently for one class of peer.** The ladder is direct → punch →
  `data_relay` → store-and-forward. Shape 4 stops that class at rung 2 forever — not as a floor a
  deployment chooses, but as a spec rule it cannot opt out of.
- **It strands the class most likely to need the relay.** A browser peer is a likely occupant of
  carrier-grade and corporate NAT, which is the symmetric case relay exists for.

`entity-browser-rust` stated exactly one requirement and no preferred shape — *whatever wins must
reach a peer that resolves nothing.* **That requirement is accepted and is now a constraint on any
answer to §1.2**, not a vote among the four.

**Ruling 2 — shape 2 is rejected as a credential carrier; its trust-anchor half survives.** A human
cannot re-enter an hourly HMAC credential, and the filing seat raised that objection against their
own option. **But the objection is to carrying a *secret*, not to carrying an *identity*.** A
connector row carrying a stable, pasteable **deployment peer-id** — a trust anchor, not a
credential — is a good human surface for the same reason the credential is a bad one: it does not
rotate. That half is not rejected and is load-bearing below.

**The finding — and it is why shapes 1 and 3 are held rather than picked.**

**`EXTENSION-REGISTRY` §3b has no credential channel at all.** The schema is
`data_relay: [+ {endpoint, priority, policy}]`, and an exhaustive search of `specs/` and `guides/`
returns **no TURN credential mechanism anywhere in the corpus** — no username/credential fields, no
RFC 8489 long-term or ephemeral-credential binding, no minting operation. The only occurrences of
"credential" next to TURN are `EXTENSION-SIGNALING` §13 item 5 *describing the problem*.

**So all four tabled shapes answer the same sub-question — where the `turn:` *endpoint* comes
from — and none of them answers where the *credential* comes from.** For `policy: open` that is
survivable. For `members` and `metered` — the two policies that exist precisely because relay
bandwidth is billed — an endpoint without a credential gathers no relay candidates. **Ruling
placement now would pin a delivery route for a value that cannot yet be used on arrival.**

**The implementation evidence agrees, and it is already shipped.** `entity-browser-rust`
`src/connectors.rs` is a *local* user-editable list of signaling nodes — despite its module name, it
performs **no `EXTENSION-REGISTRY` `:resolve`**; a tree-wide search for `service-advertisement`,
`ResolutionResult`, and `data_relay` in `src/` returns nothing, so §3b's signed set genuinely never
lands there. And `src/session_config.rs` `parse_ice_urls` **refuses `turn:`/`turns:` outright**, with
the reason stated in-tree: *"A TURN server needs a username and credential, and there is nowhere to
put them here yet … Accepting one would build an `RTCIceServer` that silently gathers no relay."*
**That refusal is correct and this proposal endorses it** — it is the honest floor while the channel
is missing, and it is a better posture than accepting a URI and failing silently.

**Direction this proposal now favors, stated as a direction and not a ratification.** The trust in
§3b comes from the **signature over the advertisement entity**, verified against a pinned deployment
identity (§3b.0) — **not from the resolve that transported it.** A resolve is a *transport*, not an
*authority*. If a peer can obtain the signed entity by any route and verify it itself, it gets the
identical guarantee. That makes a node a **courier, not a directory**: it hands over bytes it cannot
forge, adds no assertion of its own, and therefore does not violate §4.5.1's rule — which forbids a
node *asserting* a third-party set, not *relaying a signed one*. Combined with shape 2's surviving
trust-anchor half (the connector pins the deployment identity), this reaches a peer that resolves
nothing, without inventing a discovery step and without a node vouching beyond its authority.

**It is not ruled, because it is not yet evidenced, and the missing evidence is the credential
channel — which is upstream of it.** §3b must grow one before any delivery route is worth pinning.

**Evidence gate for §1.2, replacing "pick one of four":**

1. **A credential channel for `data_relay`, specified in `EXTENSION-REGISTRY` §3b** — the actual
   blocker, and a §3b gap rather than a §4.5 one.
2. **A rotating-credential trial** (§3 item 3, unchanged) against whichever channel that is.
3. **Then** the delivery route, of which node-as-courier is the current favorite and shapes 1 and 3
   remain live alternatives.

### 1.3 Merge precedence

§4.5.1 pins a **SHOULD**: merge rather than replace, dedupe by endpoint bytes as published. That is
safe today because §9.3 already requires consulting several reflectors and requiring agreement, so
more sources strictly improve the conclusion.

**The half that was broken is fixed `[2026-08-14]`.** As first written the dedup was *undefined*, not
merely under-specified: §4.5.1's field was a `primitive/string` while §3b's was a `NETWORK` §6.5
endpoint **object**, and there is no byte-dedup across a string and an object. Both are now the
§3b.0 URI string, so the dedup is well-defined. (`entity-browser-rust` Finding 3, 2026-08-14 — they
noted it resolves the moment Finding 1 does, and it did.)

**What remains open** is precedence proper, which the type fix does not touch: two sources can still
**disagree** in a way dedup cannot arbitrate — a deployment set naming a relay the node contradicts,
or a `policy` that differs between them. Then precedence is a real rule. Not urgent: no peer holds
both sources yet, and browser-rust consumes §4.5.1 only until it builds a registry client.

## 2. Why these were held back rather than folded

Each fails a different bar that the two ratified halves passed:

- §1.1 has **no evidence** — no implementation has re-read provisioning per negotiation, so pinning
  a model now pins an untested one.
- §1.2 was held as a **placement question with four live candidates** and no clear winner; picking
  one in a fold would be inventing, not folding. **§1.2a now rules two of the four out and shows the
  remaining two answer the wrong sub-question** — the blocker is a missing credential channel in
  `EXTENSION-REGISTRY` §3b, upstream of any delivery route.
- §1.3 is **not yet a problem** — it becomes one only when a second source exists in a shipped peer.

Folding any of them to look complete is the failure `AGENTS.md` names as a capability note being
*ahead* of reality. §13 item 5 names them in-spec so the absence is explicit deferral rather than
oversight.

## 3. Evidence this needs before it can ratify

1. ~~**A two-NAT rig.**~~ ✅ **BUILT AND GREEN 2026-08-16** — `tools/e2e/webrtc-rung1/nat_topology.sh`,
   driven by `make e2e-webrtc-traverse`: two peers each masqueraded by **its own** router onto a
   transit network, so they present **two distinct external addresses**, with the STUN responder on
   transit rather than the host (on the host, rootless podman masquerades a second time and both
   peers read as one address — the rig degrades to `split` silently, and did on the first run).
   **It probes its own controls and fails the run if any disagrees.** *(This row read "Not built
   yet" for two days because it faithfully recorded the filing seat's own last word; they built it
   hours after saying they would not and did not send the update. Their correction, their framing.)*

   **What it does NOT close, and this matters to §1.2 below.** The rig is a **port-restricted cone**
   NAT (conntrack endpoint-independent mapping + address/port-dependent filtering). **It
   structurally cannot model symmetric NAT** — the case relay/TURN exists for — so it supplies **no
   evidence for the TURN half of this proposal** and must not be read as unblocking it. The three
   gates it was said to unblock resolve differently: §11.5.1's **S5** gets its substrate half but
   keeps its cross-impl caveat (both peers are the same WebRTC implementation — **no topology closes
   that**); the §10.3 seam gate is **partial** and now scoped by the substrate-discharge rule
   (`EXTENSION-NETWORK` §10.3 obligation 2); and §6.7.5's dial-back was **never
   implementation-blocked at all** — see the correction below.
2. **One implementation of `EXTENSION-REGISTRY` §3b**, so §1.3's merge question has two real sources
   instead of one hypothetical.
3. **A rotating-credential trial** on whichever consumption model §1.1 favors, before it is pinned.
4. **A credential channel for `data_relay` in `EXTENSION-REGISTRY` §3b** `[named 2026-08-16, §1.2a]`
   — §3b's `data_relay` carries `{endpoint, priority, policy}` and **the corpus specifies no TURN
   credential mechanism anywhere**. `policy: members`/`metered` are unusable without one, so this is
   upstream of §1.2's delivery-route question and of §1.1's consumption-model question alike: **there
   is no rotating credential to have a lifetime yet.**

## 4. `[ASK-ARCH]` — the decisions owed

- Does the consumption model get pinned, or is it implementation-local with only the gate
  expectation stated? (§1.1)
- ~~Which of §1.2's four shapes serves the connector-by-URL path — or is STUN-only the ruled answer
  there?~~ **Answered in part, §1.2a.** STUN-only (shape 4) is **rejected**: that path MUST be able
  to reach a data relay. Shape 2 is rejected as a credential carrier, its trust-anchor half kept.
  **The remaining choice is deferred behind a newly-named blocker** — §3b carries no credential
  channel, so no delivery route is rulable yet. **New decision owed: what mints a `data_relay`
  credential, and where it is specified.**
- Does §1.3 stay a SHOULD until two sources ship, or get precedence now?
- **Which spec owns the result.** A consumption-model rule is arguably `EXTENSION-NETWORK` §10.3
  seam territory rather than either of the two amended here.

## References

- `specs/extensions/EXTENSION-SIGNALING.md` §4.5.1, §9.1, §9.3, §13 item 5; §2.2 (both surfaces)
- `specs/extensions/EXTENSION-REGISTRY.md` §3b (esp. §3b.4's SHOULD and `data_relay.policy`)
- `docs/proposals/implemented/extensions/PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT.md` — the ratified
  deployment-wide half
- `docs/status/ROUTING-2026-08-14-ice-server-placement-answer-to-browser-rust.md`
- `guides/GUIDE-REFERENCE-DEPLOYMENT.md` §3.4 (the data-relay as the metered Tier-2 role)
