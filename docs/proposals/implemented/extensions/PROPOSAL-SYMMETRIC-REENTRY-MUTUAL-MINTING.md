# PROPOSAL — symmetric origination authority for rendezvous-established peers

**Status:** **FOLDED 2026-08-05.** Both impls acked and built the classification half (core-rust `5596370`,
core-go `93995df`); arch applied §11 to `EXTENSION-NETWORK.md` §6.6 (carve-out), `EXTENSION-SIGNALING.md` §6.5
(the reciprocal-grant rule), and `guides/GUIDE-CAPABILITIES.md` §4a (the taxonomy) — no rev bump. This file is now
the **design/implementation record**; the specs are source of truth. The **mint mechanism** (§11.2) is built in
`entity-core-rust` (`0eccb3f` + F7 `9e77417`/`f11df7d`) and is what go/py build next, against the landed spec;
cross-impl + S5 validation continues, fixing the spec in place if it finds a defect.

> **Second cycle (2026-08-05) — landed (§12).** `entity-core-go` built the mint against the first fold and ran
> V3 (Go↔Rust, `8·0F @ e968ed1`), surfacing **four rulings + a new MUST + two spec gaps** — all folded in place,
> no rev bump. **§11 below is the *first-fold* record and is now partially superseded — the live specs are source
> of truth** (most materially §11.2 *Contents*: the reciprocal grant is the **assembled inbound-dialer grant**,
> not the flat floor). Arch design on the reciprocal grant is **discharged**; what remains is cohort build under
> the rulings + the **S5** two-browser gate — not arch spec work. See **§12**.

_History:_ **Rev 2** ruled core-rust's held items — narrowing is a **scope decision, not hygiene** (the universal
grant *is* consumed, §7), with **no fourth taxonomy row**. **Rev 3** reframed the §4.4 discriminator **onto the
rendezvous key** (core-go: "rendezvous-driven vs profile-driven" was Rust-specific and left `pair` mode never
minting) and added the **connection-scoped-authority precedence pin** (§9 Q2). **At fold (rev 4):** the precedence
ordering is ratified **connection-scoped-wins** (§9 Q2), and core-rust corrected the classification — the native
§7 punch derives its `pair_key` from the two peer-ids and therefore **mints**; only a direct dial to a resolved
endpoint is asymmetric (§4.4). No-fourth-row **converged (Go + Rust, independent enumerations)**. Retracts this
proposal's own earlier "installed but unconsumed" claim.
**Target (four homes, one fact each):**
- `specs/extensions/EXTENSION-SIGNALING.md` §6.5 — the reciprocal-grant rule, scoped to trigger (b).
- `specs/extensions/EXTENSION-NETWORK.md` §6.6 — a carve-out on the `minted_capability` MUST NOT.
- `guides/GUIDE-CAPABILITIES.md` — the three-way back-direction-authority taxonomy (the durable artifact).
- `guides/GUIDE-CONFORMANCE.md` §7a.2a — the carriage convention + the acceptor-originates vector.

**Provenance:** resolves the `§6.5 symmetric originate` ambiguity (`entity-core-rust`
`docs/SPEC-AMBIGUITIES.md`), surfaced by a live two-browser WebRTC rung-1 run in `entity-browser-rust`
(offerer→answerer `200`; answerer→offerer `no originating authority`). Ruling **mutual minting** selected by
the maintainer and built in `entity-core-rust` (`0eccb3f`) before landing here. **This revision reframes that
build on first principles (establishment symmetry) and narrows its scope** — the reference grants on *every*
dial; this proposal grants only on a §6.5 (b) symmetric establishment. See §7 for what core-rust must change.
**Scope:** connection-establishment authority direction. Does **not** touch the locked wire core, the §6.3
signed-blob SDP path, or the rendezvous key. Does **not** broaden authority on ordinary dial-by-address
connections.

---

## 1. Problem — the handshake mint assumes an asymmetry that rendezvous does not have

For B to **originate** a dispatch to A (not merely respond to A), B needs a capability granted **by** A: A owns
its handlers, and `verify_request` at A roots the chain at A's identity (`grantee == author`, chain roots at
`granter == A`). So the only question is ever *when, and why, does A grant that.*

The connection handshake answers it **one-directionally** (`EXTENSION-NETWORK.md` §6.6, the
`minted_capability` / `held_capability` pair): the **acceptor** mints a connection capability *for* the
**dialer** (`granter = acceptor, grantee = dialer`), and the acceptor holds nothing granted by the dialer. That
is correct for **dial-by-address** — a client contacts a server, the server authorizes the client, and there is
no reason to hand the server reach-back into the client. §6.6's *"the session entity … MUST NOT be built to
solve generic back-direction dispatch"* is defending exactly that case, and it should stay.

But the mint direction encodes a **hidden assumption: the dialer is the party that wanted service.** In
`EXTENSION-SIGNALING.md` §6.5 **trigger (b)** — two peers agree a rendezvous key out of band and meet at it —
that assumption is **false**. The `dialer`/`acceptor` labels there are an artifact of who sent HELLO
(§7.4.1), assigned by a **coin-flip** (the §6.5 perfect-negotiation offerer sort), *not* by who wants to invoke
whom. Both peers agreed the key; **that agreement is the mutual authorization act.** So after one negotiated
channel opens, §6.5 (b) says either peer may originate — but the authority is one-directional, and the acceptor
fails closed. **Observed:** rung-1, two browsers, roles as above.

Neither existing back-direction mechanism covers this. There is no `deliver_token` (neither peer subscribed) and
no inbound request to reenter from (§7a.2a caller-authorizes-reentry needs a live inbound call to ride). A
*spontaneous symmetric* pair falls through both. That is the real gap.

## 2. The principle — authority direction tracks establishment symmetry

One rule resolves it, and it composes with §6.6 rather than contradicting it:

> **The direction of connection-establishment authority follows the *symmetry of the establishment*, not the
> mechanics of who spoke first.**
>
> - **Asymmetric establishment (dial-by-address).** One party requested service; authority is one-directional,
>   acceptor → dialer. **§6.6 unchanged.**
> - **Symmetric establishment (rendezvous, §6.5 (b)).** Both parties signalled mutual intent by agreeing the key
>   out of band; authority is **bidirectional**. **New, and confined to this mode.**

**Bidirectional authority, one new grant.** The symmetry is in the *authority*, not in the count of mints. The
§6.6 handshake already supplies one direction — the acceptor grants the dialer authority to originate to the
acceptor. A §6.5 (b) establishment adds only the **mirror**: the dialer grants the acceptor authority to
originate to the dialer. That single reciprocal grant is the whole of what is new (§4); together the two make the
pair symmetric.

The discriminator is **"was this a rendezvous-driven symmetric establishment,"** which is checkable at the
`establish_live` seam (the mode is known there) and is exactly the condition §1 names. The out-of-band key **is**
the signal of mutual intent; least-authority holds because symmetric intent is *signalled* (by using rendezvous),
never *assumed* for every connection.

**What a peer gets once it is in this flow, and what it needs (the maintainer's ask, pinned).** Entering a §6.5
(b) establishment, each peer knows precisely what it is granting and receiving: **the same default connection
grant it would issue an inbound dialer**, mirrored. The reciprocal grant's floor is the default connection grant
— *no broader* — because symmetry means the acceptor receives the mirror of what the dialer received, and a peer
already knows what that is. A peer that wants to extend a counterpart *additional* access does so **explicitly**,
as an attenuation-upward via policy (§4.2), never as an open-ended default. **Bounded-and-known at entry;
widened only by deliberate act.**

## 3. The three back-direction authorities (the durable artifact — GUIDE-CAPABILITIES)

The defect beneath both the impl's over-scope and §6.6's blanket prohibition is that **"back-direction dispatch"
was treated as one thing.** It is three, each with a distinct authorizer, trigger, and home. Naming them is what
turns this miss into a strength — the next symmetric-establishment question resolves by table lookup:

| # | Need | Granter → grantee | Granted at | Scope | Home |
|---|---|---|---|---|---|
| 1 | **Dialer invokes acceptor** | acceptor → dialer | connection handshake | full default connection grant | `EXTENSION-NETWORK.md` §6.6 |
| 2 | **Scoped reentry delivery** — subscription notify, continuation advance | subscribee → subscriber | subscribe / continuation time | one delivery target | `deliver_token` (INBOX / SUBSCRIPTION) |
| 3 | **Symmetric spontaneous origination** — rendezvous peers | dialer → acceptor (adds the §6.6 mirror) | §6.5 (b) establishment | default connection grant, policy-attenuable | **this proposal, `EXTENSION-SIGNALING.md` §6.5** |

The universal build (§7) installs a #3-shaped grant on **every** connection — and per core-rust's code read it
**is** consumed, not idle: `get_or_connect`'s reuse of the accepted inbound endpoint (the §6.11(b) profile-less
path) dispatches over it at three sites. `0eccb3f` is the first build that ever made that path authorize (it
previously drew `401 unresolvable_grantee`), so nothing green depended on it — but "installed but unconsumed"
(an earlier claim of this proposal, now retracted) was wrong. Narrowing is therefore a **scope decision, not
hygiene** (§7): it returns a live path to fail-closed, and the fix is to route each consumer to its **correct**
authority (row 2 where a trigger exists; fail-closed where none does), **not** to keep a broad row-3 grant on
asymmetric connections. The taxonomy is the naming artifact that makes that routing decidable — and the answer is
**no fourth row** (§7).

## 4. Mechanism — one reciprocal grant, scoped to §6.5 (b)

The §6.6 handshake already gives the **dialer** authority to originate to the **acceptor** (the acceptor's
authenticate-response cap). The only missing direction is the mirror, so **only one new grant** exists:

> On a §6.5 (b) establishment, the **dialer** mints a capability for the **acceptor**: `granter = dialer`,
> `grantee = acceptor`, `grants = <default connection grant>` (§4.2), authored under the connection's active
> `content_hash_format`, and **signed** (signer = the dialer's granter identity, target = the cap's content hash
> — a single-sig root cap is rejected `missing_signature` at verify-time, V7 §5.5 chain walk).

**The `grantee` is an ordinary capability grantee `[correctness — MUST]`.** It MUST be exactly the identifier the
core verify contract resolves and compares to the author (`ENTITY-CORE-PROTOCOL.md` `verify_request` / chain
walk) — today, the grantee **identity entity's content hash** — and **not** the §3.2 rendezvous
`system/peer-id`. Conflating the two (the #67 id-encoding thread) mints a cap that fails
`grantee_mismatch` / unresolvable-grantee at the far side. This MUST names the grantee by *what verify compares*,
so its direction does **not** hinge on how #67 ultimately resolves the `peer-id` ↔ identity-hash encoding.

The acceptor, on receipt, verifies `granter == the peer it authenticated on this connection`, that the cap
hash-validates, and **SHOULD verify the granter signature at acceptance** (fail-fast, rather than deferring the
only signature check to first use). It installs the cap plus its supporting entities (signature + granter
identity) as this connection's originating authority, injecting those supporting entities into the envelope's
included set when it later originates (its handshake `auth_included` does not contain them). Result: the acceptor
now holds a cap granted by the dialer, so — with the §6.6 direction — **either** peer can originate;
`verify_request` at the far side is satisfied unchanged.

### 4.1 Delivery — post-handshake, every mode; carriage per §7a.2a

The new grant is **dialer → acceptor**, and the dialer learns the acceptor's authored identity only in the
**authenticate-response** — the final handshake leg. So the reciprocal cap is authored and delivered **after**
the handshake completes, over the open connection, in **every** mode (this is what the reference does). There is
no earlier leg to carry it: even in `pair` mode, where both `system/peer-id`s are known out of band, the grantee
the cap must name is the acceptor's *identity-entity content hash* (§4 MUST), which the dialer holds only once the
authenticate-response arrives.

> **`pair`-mode in-leg minting is aspirational and fully #67-contingent.** It becomes possible *only if* #67
> resolves such that the grantee is nameable from the out-of-band `peer-id` alone (no separate identity entity)
> **and** an earlier handshake leg survives to carry it. Neither holds today — do not build against it; it is
> recorded as a future optimization, not a v1 path.

**Timing `[cross-peer seam]`.** The grant lands ≈1 round-trip after channel-open; a peer that originates in that
window has no authority yet. §6.5 (b) origination **gates on grant-received** — a state condition, not a timer —
**with a bounded wait**: if the reciprocal grant does not arrive within the bound, the peer **fails closed** (no
originating authority), which is exactly the pre-adoption graceful degradation (§6). The bound MUST exist so a
non-adopting cross-impl counterpart — which never sends a grant — degrades to fail-closed rather than **blocking
forever** (F6). The reference **already implements exactly this** — `serve_traversed_connection` polls
`originating_capability()`, breaks the moment the grant lands, and falls through to fail-closed at the bound; the
"≈1s timer-race" phrasing was a doc error in core-rust's own proposal, since corrected, and there is nothing to
"replace." The rule, stated once: *authorized once the grant is in hand; fail-closed if the bound elapses first.*

> **The grantee-encoding precondition (#67).** The §4 grantee MUST and the aspirational `pair` optimization both
> ride the `system/peer-id` ↔ identity-hash contract whose ambiguity already produced a silent never-meet
> (`ROUTING-2026-08-03-id-encoding`) and whose stale Ed25519 spelling still sits in `ENTITY-SYSTEM-REFERENCE.md`
> (:75/:601, the open #67 thread). Pinning that contract is a precondition for anything that names a grantee from
> a `peer-id`; until then, follow the §4 MUST — name the grantee by what verify compares (the identity content
> hash).

### 4.2 Contents — default connection grant floor, policy MAY attenuate

The reciprocal cap's grants are the **default connection grant** — the mirror of what an inbound dialer receives,
so entry authority is bounded and known (§2). A peer MAY **narrow** it via `system/capability/policy/{peer}`
(the "reach me, but not everything" case) and MAY **widen** it later by minting a further attenuated-upward cap
as an explicit act — never as a default. Policy is **optional**: default-floor keeps the establishment hot path
free of a mandatory lookup; the policy hook is there for deployments that want asymmetric trust inside a
symmetric establishment. (Contents rule: default grants as the floor, not policy-first.)

### 4.3 Carriage — reuse §7a.2a's ratified in-band shape, not a new frame

`GUIDE-CONFORMANCE §7a.2a` **already ratified** how reentry authority travels: **in-band params** (shape (a) —
`reentry_capability` / `reentry_granter` / `reentry_cap_signature`), chosen cross-impl over the included-set
because it is self-contained and does not depend on the session API exposing `included`. The reciprocal grant is
the *same concept applied proactively at establishment*, so it MUST use the **same** carriage — not a new
first-class `reentry-grant` frame and not a new intercept lane. This keeps the **locked wire core unchanged**
(the grant is an ordinary EXECUTE the acceptor intercepts, self-verifying via the enclosed granter signature)
**and** keeps one reentry-authority convention instead of two. (Carriage: neither a new frame nor a new lane —
the existing §7a.2a shape.)

### 4.4 The discriminator is the rendezvous key — locally-derived, never wire-carried `[cross-peer seam — MUST]`

The mint fires iff the establishment was reached **by meeting at a §3 rendezvous key** (`pair` / `tag` / `secret`
/ `lobby`). That is the precise content of "symmetric": a §3 key is one **both** peers brought independently, and
§3.4's rendezvous-hash routing (§3.2's `pair_key` for the `pair` case) means they **meet only if both used the
same key** — so the joint bringing of the key *is* the mutual-authorization act, **regardless of why either peer
showed up.** The asymmetric no-mint case is its mirror: an establishment reached by **dialing a resolved
transport endpoint** (a directly-dialed `tcp`/`http` profile or a raw address — **not** an establishment that
lands on a `pair_key` rendezvous), where no §3 key was mutually brought — §6.6's one-directional mint stands.

**Do not classify on "rendezvous-driven vs profile-driven," and do not classify on substrate** (`entity-core-go`,
2026-08-05 — this corrects rev 2). That framing was a Rust-shaped fact stated as a general one: the native §7
punch is **not** inherently asymmetric — in **both** `entity-core-go` and `entity-core-rust` it derives its
`pair_key` from the two peer-ids and therefore **mints** (rev 2's "profile-driven, asymmetric" was not even true
of Rust) — and in a dialer's seat "drove off a key" and "drove off resolution" can be **both true of one
event**, so an exclusive
either/or misfires and, worse, can leave `pair` mode — §6.5's *natural default* — never minting. The test is
**positive and on the key**: *was a §3 rendezvous key mutually brought?* If yes, mint; a co-occurring profile
resolution does not demote it. ("Traversed" is likewise not the discriminator — a punch may itself be
key-established.)

**Locally-derived, never wire-carried `[MUST]`.** Each peer classifies from **its own** use of the key — it knows
whether it established this connection by meeting at a §3 key it holds. It **MUST NOT** be a wire field the
counterpart sets (a one-sided field is the §7.4.1 failure shape; core-rust flagged this, correctly). It need not
be: because meeting proves both brought the **same** key (§3.4 / §3.2 `pair_key`), each peer classifies
**independently** and they **agree by construction**. Each `establish_live` implementation sets its **own** local
`established_via_rendezvous_key` flag from its own establishment path; the flag is an input to the mint decision,
not a claim on the wire. (This is the additive seam field the cohort held for a ruling — additive, locally-set on
both sides, never trusted from the peer.)

## 5. The §6.6 carve-out — no latent contradiction

Folding §6.5 (b) reciprocal-grant while §6.6 still reads a bare *"MUST NOT solve back-direction at the
handshake"* plants a contradiction the tree-hygiene gate rejects. §6.6 gains a one-line carve, and §6.5
back-references it — one canonical home per fact, cross-referenced:

> **§6.6 (added).** *This one-directional mint is the **asymmetric** case (dial-by-address). A §6.5 (b)
> **symmetric** rendezvous establishment adds one reciprocal grant (the dialer mints the acceptor's mirror of
> this cap, `EXTENSION-SIGNALING.md` §6.5), yielding bidirectional authority by design — that is not the "generic
> back-direction dispatch" this MUST NOT forbids, but a match to the establishment's own symmetry (back-direction-
> authority taxonomy row 3). The prohibition still holds for every asymmetric connection.*

## 6. Why this was not anticipated (the ratchet — a feature must make us stronger)

1. **The establishment-symmetry axis was never named.** "dialer/acceptor" (a HELLO role, §7.4.1) silently stood
   in for "client/server" (a trust direction). They coincided in every establishment until rendezvous split
   them — an equivalence that holds locally then springs apart at a cross-peer seam, which is the **catalogued**
   *cross-peer seam equivalence-collapse* shape (AGENTS.md). We had the ledger entry and still walked in.
2. **§6.5 (b) was specified as connectivity; its authority consequence was never traced.** We folded "either can
   originate" (the transport claim) without asking "with what authority?" We pinned the on-wire seam (the §6.3
   coordination envelope, `b99304d`) and missed the capability seam directly behind it.
3. **The validating topology masked it.** An all-symmetric browser rig makes an over-broad grant and a correct
   grant indistinguishable, and the acceptor-*originates* vector had never run — the same single-impl blindness
   that hid `fire_at`, the §7.4.1 handshake role, and the id-encoding never-meet. The meta-rule ("not validated
   until a cross-impl conformance test exercises it") was in force and unmet; the reference had to *add* the
   acceptor-originating regression (§8).
4. **"Back-direction" was one undifferentiated concept.** Without the §3 taxonomy, the impl reached for the one
   broad hammer and §6.6's blanket MUST NOT looked like it forbade the legitimate §6.5 case too. **That missing
   taxonomy is the actual defect** — deeper than the impl's over-scope or the proposal's original mis-citation
   (which attributed the mint to §7.4.1; it is §6.6).

## 7. Scope ruling + what core-rust changes

The reference (`0eccb3f`) fires the reciprocal grant from `perform_connect_with_dispatch` — the **shared** client
handshake used by `connect_and_pool` (all TCP/HTTP dials) **and** the WebRTC dialer — so **every** full-peer dial
grants the acceptor the default connection grants back (the **universal** reading, §2). core-rust answered this
proposal's cohort ask by reading the code: that grant **is** consumed. `get_or_connect` (`core/peer/src/remote.rs`)
falls back to the accepted inbound endpoint when the counterpart has no dialable profile (the §6.11(b) path), and
three sites dispatch over it with `dispatch_cap = None` — `connection.rs::make_execute_fn`,
`connection.rs::process_async_delivery`, `relay_forwarder.rs` — all plain dial-by-address. So **narrowing returns
a live path to fail-closed: a scope decision, not hygiene.**

**Ruling — resolution is not authority; no fourth taxonomy row.** Reaching a profile-less peer over a reused
inbound endpoint is a §6.11 / §10.3 **resolution** mechanism; it confers **no** authority of its own. Dispatch
over that resolved path is authorized by the same taxonomy (§3) as any dispatch — so the three sites decompose,
none needing row 3:

- **`process_async_delivery` → row 2.** It already holds a `deliver_token`; pass it as `dispatch_cap`. Correct
  under any ruling and unblocked today — core-rust flagged this as the immediately-useful move; **take it now.**
- **`relay_forwarder` → pass-through, not origination.** A relay forwards an envelope carrying its **own**
  end-to-end capability chain; it does not originate under its own authority. Confirm the forwarded chain is what
  authorizes the far side (it should be) — then no back-direction grant is involved at all. **Confirmed with a
  distinction (core-rust):** the *payload* is pass-through, but the outer forward EXECUTE is the forwarder's own
  **row-1** connection authority to the next hop — the hop is row 1, not "no authority."
- **`make_execute_fn` (generic) → its trigger's row-2 authority, else fail-closed.** A dispatch to a profile-less
  peer always has *some* trigger (subscription, continuation, inbox, application intent), each with a row-2
  delivery authority (`GUIDE-CONFORMANCE §7a.6` — "autonomous origination has no core trigger"). A genuinely
  **triggerless** spontaneous dispatch to a peer reachable only because it dialed in, over an **asymmetric**
  connection, is exactly the generic back-direction dispatch §6.6 forbids — and **fails closed** (its
  pre-`0eccb3f` behavior, which was correct). Row 3 stays reserved for §6.5 (b) symmetric rendezvous.

**So the change is:** gate the mint on §6.5 (b) rendezvous-symmetric (§4.4), route `process_async_delivery` to its
`deliver_token`, confirm `relay_forwarder` is payload-authorized, and let any triggerless generic back-dispatch
fail closed. It is protocol-visible authority — scope it before go/py mirror it. **Verification ask back to
core-rust:** enumerate `make_execute_fn`'s callers and confirm each maps to a row-2 trigger; a caller with a
*legitimate* need and no row-2 home would be new evidence that reopens this — but per §7a.6 none should exist.

**Converged (Go + Rust), 2026-08-05.** core-go ran the enumeration independently against its own tree and the
ruling holds: one resolution funnel (`getRemoteConnection`), every dispatch mapping to a dial, a per-delivery
trigger, or verbatim relay pass-through — **no fourth row**, with one honestly-named triggerless edge (a
best-effort close notification) that already fails closed and logs. Two independent enumerations agreeing makes
this **converged, not cohort-consistent** (§10). The same pass found a Go defect of this proposal's *exact* shape
(`deliverToInbox` built a `deliver_token`-authorized envelope then dispatched bare on the remote branch — green
by accident on a dialed connection, `401` on the profile-less path) — **fixed**, and it is row 2 reached
backwards. **Routed follow-on (not this fold):** **both** impls independently hit a same-shape latent bug one
layer up — cross-peer subscription notify authorizes with the notifier's self-grant instead of the subscriber's
`deliver_token` (core-go `DispatchLocalEnvelope`; core-rust `mint_delivery_grant` mints `granter == grantee ==
subscriber`), green only because the gate publishes a dialable profile as setup (the masked-topology lesson
again). Routed to the SUBSCRIPTION owner — **row 2**, authorize with the subscriber's token — with core-go's note
that it has **two halves**: the envelope's capability is dropped in **core** (`DispatchLocalEnvelope`) before the
delivery function runs, so fixing the authorization alone would be discarded a layer down — whoever picks it up
needs both. Corroborates the taxonomy; does not gate §11.

## 8. Conformance additions (`GUIDE-CONFORMANCE §7a.2a` + `EXTENSION-SIGNALING §6.5`)

1. **§6.5 (b) symmetric originate.** After a §6.5 (b) establishment in which the reciprocal grant has landed, the
   acceptor MUST be able to originate a cross-peer dispatch to the dialer over the same channel; a fresh acceptor
   with no grant MUST fail closed (no bearer / self-authorization) — and MUST fail closed rather than block
   indefinitely if the grant never arrives (§4.1 bounded wait).
2. **Asymmetric non-regression.** A peer reached by ordinary **dial-by-address** MUST NOT gain reciprocal
   originating authority (the §7 narrowing, made a gate — this is the vector the universal build would fail).
3. **Reciprocal cap validity.** `granter = dialer` (the authenticated counterpart), `grantee = acceptor` named as
   the core verify contract resolves it (identity content hash, §4 — **not** the §3.2 peer-id), signed by the
   dialer; the acceptor MUST reject a grant whose granter is not the authenticated peer, and SHOULD verify the
   granter signature at acceptance (F7).
4. **Vector shape.** A two-peer, genuinely-different-role exchange over a real negotiated channel — invisible to
   any same-implementation loopback or single-role suite, which is precisely how the gap shipped.

> **§8.3 — Acceptance-check scope `[correctness — MUST; landed F7]`.** The acceptor's at-acceptance check covers
> **only the legs the frame carries**: a signature targets the cap, its signer **is** the granter, and it verifies
> under the granter's key. It **MUST NOT** run the full chain walk (`verify_capability_chain`) — that walk also
> resolves the **grantee** (the acceptor itself), whose identity entity is **not** in a grant the dialer authored,
> so the full walk would reject **every valid grant**. Landed in core-rust (`connection.rs::verify_grant_signature`,
> four tests incl. a forged-signature case the structural checks alone miss). **Doubly grounded (core-go,
> 2026-08-05):** Go's full chain walk rejects at the same point for an *independently-arising* reason (its walk,
> too, fails at grantee resolution) — two implementations, two distinct paths to the same MUST NOT, which is
> stronger than the single-impl argument. *(Note: `0eccb3f`'s doc claimed
> "signature valid" at this step; the code checked hash + granter only until F7 added the signature verification —
> so the acceptance-time signature check is genuinely new, not a restatement.)*

## 9. Arch questions — ratified 2026-08-05

1. **Expiry/renewal — RATIFIED: no `expires_at`.** The reciprocal cap mirrors §6.6's connection cap (no expiry);
   symmetry with row 1.
2. **Revocation direction — RATIFIED: drop-on-disconnect for v1.** The grant is establishment-scoped, in-memory,
   re-granted on reconnect; it is **not** recorded to the session tree entity. Symmetric persistence (a dialer-side
   mirror of §6.6's `minted_capability`) is a deferred future item, not v1.
   **Precedence pin (core-go, 2026-08-05).** Because the grant is *not* in `system/peer/session/{peer}`, an impl
   whose §10 authority lookup reads **only** that durable entity's `held_capability` cannot see it — "a grant that
   isn't persisted is invisible to the one lookup that decides authority." So the reciprocal grant is
   **connection-scoped originating authority**, and the origination path **MUST** consult connection-scoped grants
   **in addition to** the durable `held_capability`. The two are **distinct slots** — the session entity's
   `held_capability` is the durable reconnect-skip authority (dial-by-address); the reciprocal grant is the
   live-establishment authority — and MUST NOT be conflated. Not persisting is deliberate **security**: a stale
   grant reused on reconnect *without re-meeting at the key* would authorize outside the establishment that
   justified it.
   **Ordering — RATIFIED at fold: connection-scoped wins.** Where both a durable `held_capability` and a
   connection-scoped grant exist for a peer, the connection-scoped grant is authoritative — a durable cap can
   predate the live establishment and MUST NOT shadow it (core-go's catch; reverse order shadows intermittently,
   the worst shape). core-rust's model — durable **never** consulted for origination — is the clean form; core-go's
   connection-scoped-wins fallback is equivalent in outcome. Both converge; pinned in `EXTENSION-SIGNALING.md`
   §6.5 so browser-rust/py do not guess.
3. **Taxonomy home — RATIFIED: `guides/GUIDE-CAPABILITIES.md`.** §3 lands there as a new §4a — cross-cutting,
   out of any single owner, referencing (not restating) the three normative homes.

## 10. Reference implementation + validation

- Impl: `entity-core-rust` — positive path `0eccb3f`; **F7 acceptance-check landed** (`9e77417` / `f11df7d`),
  workspace **2423/0/12**, clippy clean, wasm32 feature build green. **Universal scope still to be narrowed per
  §7.** Design / implementation record (demoted, points here):
  `entity-core-rust/docs/PROPOSAL-SYMMETRIC-REENTRY-MUTUAL-MINTING.md`; routing back to arch:
  `entity-core-rust/docs/status/ROUTING-2026-08-05-the-6.5-narrowing-has-a-consumer-to-arch.md`.
- Proven **bidirectional 2/2** on `entity-browser-rust`'s rung-1 two-browser WebRTC rig, roles swapped between
  runs (behavior follows the role, not a fixed peer).
- Native regression guard `test_s65_acceptor_originates_after_reciprocal_grant` builds the acceptor's originating
  path end-to-end; the one-directional handshake invariant still holds. Grant is in-memory on the reentry endpoint
  (§9 Q2), not persisted to the session tree entity — reconnect re-handshakes and re-grants.
- **Cohort corroboration (`entity-core-go` `27ce5ce`, 2026-08-05):** independent enumeration confirmed **no fourth
  row** (§7) — *converged, two impls*; §11.1 + §11.3 foldable as written; §11.2's mint contract mirrors code Go
  already runs (grantee = identity content hash, active format, signature at the invariant-pointer path); §8.3's
  MUST NOT independently corroborated. Go **blocked §11.2 on the rev-2 discriminator** ("I'm not implementing one
  that doesn't classify our own establishment") — resolved by the rev-3 rendezvous-key reframe (§4.4), which is
  Go's own proposed fix, so Go is unblocked by construction. Review: `entity-core-go`
  `docs/status/ROUTING-2026-08-05-symmetric-reentry-reviewed-and-our-row-2-was-bare.md`. *(Go also flagged an
  unrelated pre-existing `relay_offline_delivery` FAIL, verified not a regression, un-bisected — core-go's tree,
  not this surface.)*
- **browser-rust (heads-up):** `LivePath`'s new `established_via_rendezvous_key` field is **required**, so a
  `LiveEstablish` impl will not compile until it classifies — deliberate (a defaulted flag is the one-sided-field
  hazard §4.4 rules out; a compile error beats a silent behavior change). browser-rust classifies it for the §6.5
  (b) leg when it rebuilds against the landed spec.

## 11. Fold text — APPLIED 2026-08-05 (three files, verbatim)

**Applied** to the three files below. The landed text incorporates the fold-time corrections — **connection-scoped
wins** (§9 Q2), **the §7 punch mints** (its `pair_key` is peer-id-derived, §4.4), and **relay hop = row 1** — so
the spec is authoritative and this section is the fold record. No rev bump on any target —
cohort-finding-fixes-in-place (`AGENTS.md`). The normative MUSTs live in the specs; the GUIDE §4a edit
**references** them and restates nothing.

### 11.1 `specs/extensions/EXTENSION-NETWORK.md` §6.6 — carve-out (append to the `minted_capability` bullet)

Append, after *"…it MUST NOT be built to solve generic back-direction dispatch."*:

> **Carve-out — symmetric establishment.** This one-directional mint is the *asymmetric* case (dial-by-address).
> A **§6.5 (b) symmetric rendezvous** establishment (`EXTENSION-SIGNALING.md` §6.5) adds one reciprocal grant —
> the *dialer* mints the acceptor's mirror of this cap — yielding bidirectional authority *by design*. That is
> not the "generic back-direction dispatch" this MUST NOT forbids, but a match to the establishment's own
> symmetry (back-direction-authority taxonomy, `guides/GUIDE-CAPABILITIES.md` §4a). Reaching a profile-less peer
> by reusing a connection it opened (the V7 §6.11 reentry seam surfacing in §10 dispatch; `GUIDE-CONFORMANCE.md`
> §7a.2a) is **resolution, not authority**: a dispatch over it still needs a row-1/2/3 cap, and a *generic*
> dispatch to a profile-less peer over an
> asymmetric connection is authorized by its trigger's `deliver_token` where one exists and **fails closed**
> otherwise — the prohibition holds for every asymmetric connection.

### 11.2 `specs/extensions/EXTENSION-SIGNALING.md` §6.5 — the reciprocal-grant rule (new bullet after "Offerer determination", before §7)

> - **Symmetric origination authority (trigger (b)) `[cross-peer seam — MUST]`.** Trigger (a) is an *asymmetric*
>   dial: the §6.6 handshake mints one-directionally (acceptor → dialer) and that is the whole authority. Trigger
>   (b) is *symmetric* — both peers agreed the rendezvous key out of band, which is the mutual-authorization act —
>   so after the channel opens **either** peer may originate (the trigger-(b) premise above), and authority must be
>   bidirectional. The §6.6 handshake already supplies dialer → acceptor; a trigger-(b) establishment adds the
>   **one** missing mirror.
>   - **The mint `[MUST]`.** The **dialer** mints a capability for the **acceptor** (`granter = dialer`,
>     `grantee = acceptor`, `grants =` the default connection grant, authored under the connection's active
>     `content_hash_format`) and **signs** it (signer = the dialer's granter identity, target = the cap's content
>     hash — a single-sig root cap is otherwise rejected `missing_signature`, `ENTITY-CORE-PROTOCOL.md` §5.5). The
>     `grantee` is an ordinary capability grantee: it MUST be exactly what the core verify contract resolves and
>     compares to the author — the grantee identity entity's **content hash** — and **not** the §3.2 rendezvous
>     `system/peer-id`; conflating them mints a cap that fails `grantee_mismatch` (the §3.2 id-encoding hazard,
>     one field over). The acceptor MUST reject a grant whose `granter` is not the peer it authenticated on this
>     connection, and **SHOULD verify the granter signature at acceptance** — but MUST check **only the legs the
>     frame carries** (the granter signature), never a full chain walk, which resolves the grantee (the acceptor
>     itself, absent from a dialer-authored grant) and would reject every valid grant.
>   - **Contents.** The grant's floor is the **default connection grant** — the mirror of what an inbound dialer
>     receives, so entry authority is bounded and known. A peer MAY narrow it by policy
>     (`system/capability/policy/{peer}`) and MAY widen it later only as an explicit attenuation-upward, never as a
>     default.
>   - **Carriage.** The grant travels as the ratified reentry-authority shape (`GUIDE-CONFORMANCE.md` §7a.2a
>     in-band params), **not** a new frame and **not** a new intercept lane — one reentry-authority convention, and
>     the locked wire core is untouched.
>   - **Delivery + timing.** The dialer learns the acceptor's authored identity only in the authenticate-response,
>     so the grant is authored and delivered **after** the handshake, over the open connection, in every mode.
>     Origination **gates on grant-received**, with a **bounded wait**: if the grant does not arrive within the
>     bound the peer **fails closed** (no originating authority) — it never blocks. A non-adopting counterpart
>     therefore degrades to one-directional, not to a hang.
>   - **Where the grant lives `[MUST]`.** The reciprocal grant is **connection-scoped** originating authority, held
>     with the live connection — **not** written to `system/peer/session/{peer}` (it is establishment-scoped and
>     drop-on-disconnect; `EXTENSION-NETWORK.md` §6.6 records only the durable handshake cap). The origination path
>     MUST therefore consult connection-scoped grants **in addition to** the session entity's durable
>     `held_capability`, or the grant is invisible to an impl whose authority lookup reads only the durable entity.
>   - **The discriminator is the rendezvous key, locally-derived, never wire-carried `[MUST]`.** The mint fires iff
>     the establishment was reached by **meeting at a §3 rendezvous key** (`pair`/`tag`/`secret`/`lobby`) — a key
>     **both** peers brought independently, since §3.4's rendezvous-hash (§3.2's `pair_key`) means they meet only if
>     both used it. Classify **positively on the key**, not on "rendezvous-driven vs profile-driven" (not
>     exclusive — one event can be both) and not on substrate (a punch may itself be key-established). It MUST NOT
>     be a wire field the counterpart sets (a one-sided field is the §7.4.1 failure shape); it need not be, since
>     meeting proves both brought the same key, so each peer classifies **independently** and they agree by
>     construction. Each peer sets its own local `established_via_rendezvous_key` flag from its own path.
>   - **Not generic back-direction dispatch.** This grant is confined to trigger (b). It is **not** the authority
>     for an acceptor to originate to a profile-less dialer on an *asymmetric* connection (dial-by-address, native
>     punch); that back-direction, where legitimate, is the per-delivery `deliver_token` (`EXTENSION-INBOX` /
>     `EXTENSION-SUBSCRIPTION`), and absent such a trigger it fails closed (`EXTENSION-NETWORK.md` §6.6). See the
>     back-direction-authority taxonomy, `guides/GUIDE-CAPABILITIES.md` §4a.

### 11.3 `guides/GUIDE-CAPABILITIES.md` — new §4a (insert between §4 and §5)

> ## 4a. Back-direction authority: who may originate to whom
>
> *"What lets peer B originate a dispatch to peer A?"* has one shape at the floor — B needs a capability granted
> **by** A (A owns its handlers; `verify_request` roots the chain at A). Which mechanism supplies that cap depends
> on **why** B is originating, and there are three, each with a distinct granter, trigger, and normative home.
> They are not interchangeable; reaching for the wrong one is how back-direction authority gets over-broad.
>
> | # | Need | Granter → grantee | Granted at | Scope | Normative home |
> |---|---|---|---|---|---|
> | 1 | **Dialer invokes acceptor** | acceptor → dialer | connection handshake | default connection grant | `EXTENSION-NETWORK.md` §6.6 |
> | 2 | **Scoped reentry delivery** — subscription notify, continuation advance, inbox | subscribee → subscriber | subscribe / continuation time | one delivery target | `deliver_token` — `EXTENSION-INBOX` / `EXTENSION-SUBSCRIPTION` |
> | 3 | **Symmetric spontaneous origination** — rendezvous peers | dialer → acceptor (adds the §6.6 mirror) | §6.5 (b) establishment | default connection grant, policy-attenuable | `EXTENSION-SIGNALING.md` §6.5 |
>
> **The load-bearing distinction is establishment *symmetry*.** Row 1 is one-directional because a dial-by-address
> is asymmetric — a client contacted a server, and nothing entitles the server to reach back. Row 3 is
> bidirectional because a rendezvous is symmetric — both peers agreed the key out of band, and that agreement *is*
> the mutual-authorization act. Connection-authority direction follows the establishment's symmetry, not who sent
> the first handshake byte.
>
> **Resolution is not authority.** Reaching a peer that has no dialable transport profile by reusing a connection
> it opened (the V7 §6.11 reentry seam; `GUIDE-CONFORMANCE.md` §7a.2a; `EXTENSION-NETWORK.md` §10 dispatch) is a
> *resolution* mechanism; it confers no authority of its own. A dispatch over such a reused endpoint is
> authorized by the same three rows as any
> dispatch. In particular, a generic spontaneous dispatch to a profile-less peer over an *asymmetric* connection
> is **not** row 3 — it is authorized by its trigger's row-2 delivery token where one exists, and **fails closed**
> where none does (`EXTENSION-NETWORK.md` §6.6's prohibition on solving generic back-direction dispatch at the
> handshake). Row 3 is reserved for the symmetric rendezvous case.

---

## 12. Post-fold rulings — second cycle (2026-08-05)

`entity-core-go` built the mint against the §11 fold, ran V3 (Go↔Rust), and surfaced two rounds. All folded **in
place** (cohort-finding-fixes-in-place — `AGENTS.md`; no rev bump). The **live specs carry them and are source of
truth**; this section is the durable design record. Where §11's verbatim text differs, §12 + the live spec win.

### 12.1 The new invariant — reach-back serving `[MUST]`

The §11 fold pinned the *mint* and never pinned the *serve*. Go's client-side reader dropped an inbound EXECUTE
on the connection **it dialed** as an orphan (V7 §6.11(b) dialer-side reentry, served server-side only) — the
reciprocal grant verified and reached no handler. **Minting authority is the visible half; serving the reach-back
is the invisible half**, and it is loopback-invisible (every step reports success), like `fire_at`. A peer that
serves inbound EXECUTE only on *accepted* connections has built **half** of trigger (b). Now a MUST in
`EXTENSION-SIGNALING.md` §6.5 (b) ("Reach-back serving") + `GUIDE-CAPABILITIES.md` §4a. **`entity-core-py` (from
scratch) MUST build both sides.** *(Go fix is in-impl, not a spec defect; it was correct to fix, not route.)*

### 12.2 The four rulings (core-go's audit, first round)

- **Q1 — Carriage.** §7a.2a in-band params stay; the fold omitted the **receive-construction** — install into the
  connection-scoped slot **and serve** the reach-back (§12.1). Wire-shape + install + serve = the whole path.
- **Q2 — Contents `[both impls change]`.** The reciprocal grant is the **assembled inbound-dialer grant** (§4.4
  handshake union: floor ∪ policy(peer), advertisement-filtered) — **not** the flat floor. Minting the bare floor
  while the inbound direction grants the assembled set gives *the establishment whose justification is symmetry*
  asymmetric authority (Go measured it: reciprocal `403`s out-of-floor). **The mirror is symmetric *construction*,
  not identical grant sets.** (a) advertisement discipline **applies — MUST**; (b) **no third assembly direction**
  — "MAY narrow by policy" dropped; policy unions at handshake per §4.4. *(This supersedes §4.2 and §11.2's
  "default connection grant floor, policy MAY attenuate.")*
- **Q3 — The bound.** Pin a **floor, not a value** (2s recommended); conformance-vector timing concern, not a wire
  MUST. *(Refines §4.1 / §11.2's "bounded wait.")*
- **Q4 — Scope.** Reciprocal `system/capability:request` widening is the **same §4.4 op** as inbound (no new
  surface); a **separate vector, in v1, NOT gating** the S5 two-browser gate. Owned by the capability-handler
  track (reuse the §4.4 request fixture).

### 12.3 The two closures (core-go's V3 re-pin round)

- **Carriage, made concrete (was internally inconsistent — blocked py).** §11's Carriage said "§7a.2a in-band
  params," Delivery+timing said "rides the included set" — two mechanisms. Resolved to a two-phase shape — and
  then **corrected again in §12.6** (the §7a.2a framing was itself wrong). The **final** shape: **(1) Delivery**
  dialer→acceptor is the minted cap's **content hash**, once, post-handshake; **(2) Wielding** acceptor→dialer is
  an **ordinary EXECUTE rooted at `capability` = the cap hash** — no in-band triple, no included-set chain; the
  dialer resolves from the minted-and-delivered ledger it authored the cap into. See **§12.6**.
- **Advertisement-filter matching rule (Q2(a) was pinned-but-mechanically-open).** Pinned against **existing
  machinery, not invented:** an assembled entry is retained iff the advertised served-scope **covers** it under
  the same four-axis `scope_subset` relation the chain uses for attenuation (`ENTITY-SYSTEM-REFERENCE.md`
  capability-verification; entry ⊆ advertised); uncovered entries **drop, not narrow**. Exact-op-match and
  namespace-prefix-match are both **non-conformant** (they split on `foo/bar` vs advertised `foo/*`). Governs §4.4
  inbound assembly identically.

### 12.4 Routed upstream

The **V7 §4.4 core-text pin** for the advertisement-filter matching rule: the extension states it at the
reciprocal-mint site, but the core capability handler needs the same sentence so the inbound-dialer assembly it
already governs is unambiguous cohort-wide. **Owner: the capability-handler / V7 track** (this repo authored the
v7.62 amendment; the core-text mirror transfers there). Not a blocker on this fold.

### 12.5 Convergence state — NOT yet converged; S5 is the gate

Arch design is discharged. **Go↔Rust has now converged (2026-08-05);** browser/py remain:

| Impl | State |
|---|---|
| `core-go` | **`3fa57d3` — V3 4/4 both directions, reach 200.** Mint + carriage (EXECUTE-root) + dialer-side reentry all built |
| `core-rust` | **`f227df8` — V3 4/4 (Go↔Rust, built from real trees, not scratch).** Minted-ledger resolution fix (§12.6); §7 narrowing landed |
| `core-py` | mint **assembled + both-sided** from scratch (§12.1/§12.2), carriage = EXECUTE-root — **unstarted** |
| `browser-rust` | recompile against `established_via_rendezvous_key`; S5 two-browser run — pending S3 wasm WebRTC |

**Go↔Rust is *converged*, not cohort-consistent** — two independent trees, both directions, real builds (the
strong claim, `ADR-0012`). **Honest caveat:** V3 exercises the floor-level grant + reach-back + carriage; the
**policy-union and advertisement-filter branches (Q2/§12.3) are not exercised** (no policy entries in the V3
fixture) — those converge only when a policy / narrowed-advertisement fixture runs. **What remains:** py's mint,
the policy/advertisement fixture, and the **S5 two-browser gate** (loopback-blind; the different-role reciprocal
vector only runs there). None of it is arch spec work. **Arch is ready; the arc closes on py + S5.**

### 12.6 Carriage correction — the §7a.2a framing was wrong (2026-08-05, third pass)

The V3 build (Go↔Rust, both directions green, from real trees) settled the carriage question **against** the
§7a.2a framing that rode from the first fold through §12.3:

- **The §7a.2a triple gates nothing here.** It lives in the `system/validate/dispatch-outbound` *conformance
  handler's* params, where a handler **body** receives a reentry cap **as data** to sign a later outbound EXECUTE.
  Connection authority does not ride there and never needed to.
- **The working carriage needs no new fields.** The acceptor wields the grant with an **ordinary EXECUTE rooted at
  `capability` = the cap hash** — that field *is* the reference. The dialer resolves it from the
  minted-and-delivered ledger it authored the cap into. Folded: `EXTENSION-SIGNALING.md` §6.5 (b) Carriage +
  Delivery+timing. *(This supersedes §4.3, §11.2's Carriage, and §12.3's "triple as references.")*
- **The Rust minted-ledger bug `[durable catch — single-impl-invisible]`.** `verify_request_with_ctx` walked the
  leaf twice: chain-verify read `envelope.included`; `is_revoked` re-walked a **store-only** resolver. A reciprocal
  cap is never in the content store (it lives in the minted ledger), so the cap Rust *itself minted* was
  unresolvable on the second walk, and verify read the unresolvable chain as **revoked** → `403 capability_revoked`
  on valid authority. Fail-closed, invisible to any inlined-chain frame, fires only on the reciprocal shape. Fix:
  seed both walks from the same minted-and-delivered ledger before either runs. **Pinned as a spec MUST**
  (§6.5 (b) Carriage "Resolution caution") so py does not re-derive it. The `403 capability_revoked` code located
  it — reachable only *after* verification already succeeded, which ruled out both receive halves.


---

## 13. Post-fold rulings — the third and fourth cycles `[ledger updated 2026-08-15]`

**This proposal's ledger stopped at 2026-08-05 and three further spec changes extended this arc after
it.** They are recorded here rather than in a new document, per the reconstruction ledger
(`docs/status/PLAN-2026-08-15-spec-provenance-reconstruction.md`, arc A): the proposal exists, it is
the design record for this surface, and a second document covering the same surface would split it.

**None of the three is a correction to what folded.** All three are *new obligations discovered by
driving the folded surface on a substrate that had never driven it* — which is this arc's recurring
theme and the reason §12.5 named S5 as the gate.

### 13.1 §11.5.1 — a gate claim is scoped by its substrate `[2026-08-02, `55779b7`]`

**Source:** `entity-core-go`'s endpoint-binding rung (`6d01afe`).

A **purpose-built violating peer** — one that advertises one endpoint and punches from another, a real
§6.7.3 violation — was run against two substrates with identical flags:

| Substrate | Result |
|---|---|
| loopback | **VERIFIES** — violation undetected |
| two NATs | **FAILS** — `dialed_outbound: true`, no path |

**On loopback the violating peer's dial still lands on the counterpart's listener, so every observable
exchange reads green.** A loopback-only harness certifies a peer that cannot work behind a NAT.

Folded **generally rather than as two instances**: a §11.5 conformance claim **MUST name the substrate
it was obtained on**, because *green on a substrate that cannot exercise a property is not evidence
about that property*; plus a substrate table (what loopback / emulated dual-NAT / two real NATs each
prove and each structurally **cannot**), and the **blindness class** — with no NAT in path every packet
arrives regardless of which socket sent it.

**This is the same family as `GUIDE-CONFORMANCE` §5.2c** (a surface reached only by accident) seen from
the other end: not a check that passes by luck, but a *substrate* on which no check could fail.

### 13.2 NETWORK §10.3 obligation 5 — single-flight establishment per peer `[2026-08-06, `4c0803c`]`

**Source:** `entity-browser-rust`, the first implementation to drive the WebRTC leg.

Core-rust's cold-WebRTC path opened a channel **only by brute force**: ~470 concurrent
`get_or_connect` pool-misses each spawned a fresh `establish_live` — fresh `RTCPeerConnection`,
`session_id`, and offer — piling into the rendezvous bucket until two happened to overlap.

**The subtlety is why this was a gap and not a violation.** Obligation 4 bounds the **retry** axis and
*explicitly grants each standalone §10 dispatch its own budget* — so N concurrent dispatches are **N
individually-conformant establishments** whose *sum* is exactly the multiplicative third-party load the
discipline exists to prevent. **Obligation 4 had patched one instance of "bounded third-party load per
peer"; the fan-in instance was never stated.** *(Instance-patching versus stating the invariant once —
the shape this corpus names as a recurring failure, found here by a fourth implementation.)*

Ruled as **obligation 5 `[MUST]`** — single-flight establishment per peer, stated as the **general**
invariant obligation 4 instantiates for retries. Concurrent or repeated triggers coalesce onto one
in-flight establishment; the **observable** requirement is a bounded deposit count and the coalescing
**mechanism is implementation-idiomatic**. `SIGNALING` §11.5's S5 gate gained teeth in the same change:
a peer that opens a channel by depositing N and getting lucky **FAILS**, even though a channel opened.

### 13.3 §11.5 deposit-bound counting semantics `[2026-08-06, `78fdd13`]`

browser-rust landed and validated the fix (core-rust `e17c2ad`, gate teeth `a7dc456`): **~470 → 4 per
side, channel at t=1s where it had been 29s**, with no non-WebRTC regression.

**Their open question was answered against arch's own text and arch was the party corrected.** They
asked whether `SIG_DEPOSIT_BOUND=16` matched §7.2. It does not: **§7.2's exchange-attempt budget is 3,
not 16** — the "read of §7.2" was a misattribution. **Harmless, and the reason is the ruling:** the gate
counts *offer-deposits-over-run*, a larger quantity than *exchanges-per-establishment*, since each
exchange deposits an offer plus a bounded glare-rollback / ICE-restart re-offer. So the conformant
figure is ~4/side, and **16 is a safe ceiling because it is fixed, not because it is 16 — the property
that fails brute force is that the ceiling is O(1) and never scales with poll, dispatch or tick count.**

§11.5 now pins the counting semantics cross-impl: **offer deposits, at the node vantage, per side, per
establishment.** And the ceiling is **substrate-scoped** — 3 is correct on the native punch (which has
neither glare-rollback nor ICE-restart) and would be wrong on WebRTC, where one legitimate glare
rollback exceeds it. **A claim MUST name its substrate's value**, which is §13.1's rule reappearing one
layer down, in the same week, on a different surface.

### 13.4 What this ledger update does not change

**No ruling above is reopened**, and §12.5's convergence state is unchanged: **py's mint and the S5
two-browser gate remain the arc's close conditions.** 13.1–13.3 add obligations to the folded surface;
they do not alter the reciprocal-grant model, the carriage correction (§12.6), or the four rulings
(§12.2).

**Build state is deliberately not restated here.** The figures in 13.2/13.3 are `entity-browser-rust`'s
measurements as reported at the time and pinned to the commits named; they are **dated observations,
not standing facts** (D1/D8), and `entity-browser-rust` is at `c5f89b5` as of 2026-08-15.
