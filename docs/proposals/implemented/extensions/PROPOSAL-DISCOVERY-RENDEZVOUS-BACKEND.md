# PROPOSAL — `rendezvous` as a DISCOVERY backend on the SIGNALING carrier

**Status:** **IMPLEMENTED — FOLDED 2026-08-20**, `EXTENSION-DISCOVERY` 1.0 → 1.1. Ruled 2026-08-17;
layering confirmed, backend token minted, composition answered.
**Fold state: LANDED.** §2.1 enum · new §5.5 (five subsections) · §3.0.1 **rule 5** · §6 promotion +
QR shape-only warning · version bump. All verified against the tree before this marker (L3).
`entity-workbench-go`'s four build findings (`RENDEZVOUS-BACKEND-BUILD-RESULT-2026-08-19`) are folded
in full — §5.5.2 the TOFU reason *and* consequence, §5.5.3 deposit granularity *and* the `secret`-mode
credential rule, §3.0.1 rule 5 for a carrier-native departure signal.

> **A header line recording fold state is `entity-workbench-go`'s ask and it is adopted here.** They
> read a `RULED` stamp as landed, built, and had to grep to discover the spec was untouched — *"one
> thing that would have saved us a session."* **`RULED` says a decision was made; it says nothing
> about whether the text moved**, and the two were three weeks apart on this proposal. Every ruled
> proposal from here carries an explicit fold state in its header.
**Tier:** extensions. Normative delta is **one enum value** in `EXTENSION-DISCOVERY` §2.1 plus a
composition subsection; no wire change, no new entity type.
**Answers:** `entity-browser-rust` Q8 (`ROUTING-2026-08-16-g` §Q8, restated as their **#1 blocker** in
`ROUTING-2026-08-17-comprehensive` §4). Also disposes their Q3 residual and Q5.
**Read at:** browser-rust `ca3c760` · arch `6aab719` · legacy `PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE`
(the internal legacy corpus, read-only)

---

## §0 Summary

| Ask | Ruling |
|---|---|
| **Q8.1** — is rendezvous-at-a-key a DISCOVERY backend on the SIGNALING carrier? | **Yes.** Both specs already state this seam from opposite ends; browser-rust read them correctly. §1 |
| **Q8.2** — is there a `candidate.backend` token for it? | **`rendezvous`**, minted here. And a boundary they will need: of SIGNALING's four key modes, only **`tag` / `secret` / `lobby`** are discovery. **`pair` is not.** §2 |
| **Q8.3** — does `decision(grant-limited)` compose with a share audience? | **Yes — it is the *same* authoring path, not a parallel one.** Their instinct that an audience should be produced by a decision on a candidate is correct. §3 |
| *(free)* **Q3 residual** — is §6's QR backend settled enough to build against? | **No — shape-only.** But it is the same mold as `rendezvous`; build that one and QR follows. §4 |
| *(free)* **Q5** — may a met rendezvous tag become a REGISTRY local-name binding? | **The tag never becomes a name; the *identified peer* does.** §5 |

---

## §1 Q8.1 — the layering is confirmed, and both specs already say it

**`EXTENSION-DISCOVERY` §1** defines the substrate and names this growth path explicitly:

> *"Discovery is a **substrate with pluggable backends**, piecemeal by design: it starts small (mDNS on the
> local network) and grows to hold the other ways peers find each other and decide to connect — QR
> exchange, registry-assisted discovery, gossip/DHT, and the connect/share UX that goes with them."*
> *"**The unifying job:** surface candidate peers, and mediate the human decision to admit them."*

**The backend-pairs-with-a-carrier relationship is already in the text, twice, by name:**

- §5.2 — *"mDNS is WebRTC's `discovery:mdns` signaling carrier."*
- §6 — *"**QR / short-code backend** — out-of-band candidate exchange; **pairs with WebRTC `manual:qr`
  signaling**."*

So the relation browser-rust proposes for rendezvous is not a new pattern; it is the pattern the two
landed backends already use.

**`EXTENSION-SIGNALING` §1.1 draws the same seam from its side.** Its scope is the mailbox, the key
derivation, the coordination messages, the punch, and the unwrapped surface. **Candidate surfacing and
admission appear nowhere in it** — and §1.2's design principle is the mirror image of DISCOVERY §2:

| | |
|---|---|
| SIGNALING §1.2 | *"**The key introduces; it never authorizes.** … A shared key gets you to the meeting point; it never authorizes the session."* |
| DISCOVERY §2 | *"Discovery is the **initiator** of the grant, never the **authority** — the cap roots at the granting peer exactly as normal."* |

**Two specs, written separately, stating one seam from either end.** That is the confirmation asked for.

**The legacy landscape agrees and is not in tension.** `PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE` §5 splits
*lookup* (*"given a name, return the peer ID"* — REGISTRY) from *discovery* (*"find peers matching some
criteria"*), and §9 deliberately **did not pin discovery mechanisms** — deferring them to "a separate
landscape," which is what `EXTENSION-DISCOVERY` became. The study predates SIGNALING and says nothing
about carriers, so **the landed extensions are the authority on this seam, and they are consistent with
it.** *(Read in the internal legacy corpus, read-only.)*

### §1.1 Ruling

**Confirmed. DISCOVERY owns candidate surfacing and admission; SIGNALING owns the carrier.**
`src/rendezvous.rs` is a DISCOVERY backend implemented on the SIGNALING carrier, and building it as
neither is the duplication browser-rust already diagnosed. The refactor is toward the spec, not away.

---

## §2 Q8.2 — the token is `rendezvous`, and `pair` mode is not discovery

**Token: `rendezvous`.** Kebab-case single word per `STYLE-NAMING-CONVENTIONS` (enum/string values are
kebab). It joins the open enum at `EXTENSION-DISCOVERY` §2.1:

```
backend: <"mdns" | "qr" | "rendezvous" | ...>
```

**Why `rendezvous` and not `signaling`.** The existing values name **how a candidate is obtained** —
`mdns` is a multicast scan, `qr` is an out-of-band exchange. They do not name the carrier the resulting
connection uses. `signaling` would name the carrier and break that axis; the same backend could later run
over a different carrier and the token would then lie.

### §2.1 The boundary browser-rust will need: not all four key modes are discovery

`EXTENSION-SIGNALING` §3.2 defines four rendezvous key modes. **They do not all belong to this backend**,
and the split is clean once stated:

| mode | Who meets | Discovery? |
|---|---|---|
| `pair` | *"exactly those two peers"* — both peer-ids are **inputs** to the key | **No.** You already hold the counterpart's identity. That is connect, not *"what peers are out there that I don't already know?"* (§1) |
| `tag` | *"anyone who knows the tag"* | **Yes** |
| `secret` | *"anyone who knows the secret"* | **Yes** |
| `lobby` | *"anyone on that service, right now"* | **Yes** |

**A `rendezvous` candidate is surfaced only from `tag` / `secret` / `lobby`.** A `pair`-mode meeting
surfaces no candidate because there is nothing to admit that was not already known — it is an ordinary
NETWORK connection to a known peer (DISCOVERY §5.1).

### §2.2 What a rendezvous candidate MUST look like

This is the part that makes the refactor mechanical, and it maps directly onto browser-rust's in-memory
`Discovered { peer_id, verified }`:

1. **`identity_hint` MUST be absent (TOFU).** SIGNALING §1.2 and §3.4 are explicit that reaching a key
   proves nothing about identity — *"the key introduces; it does not authorize."* A backend MUST NOT
   synthesize an `identity-claim` for a peer that merely stood at a tag. Per §2.2.1, absent `identity_hint`
   means TOFU and *"the §2 grant decision IS the trust anchor."*
2. **Two entities, not one field.** Their `verified` boolean is §2.2's successor pattern:
   - `candidate_0` — `peer_id` **absent**, `backend: "rendezvous"`, `endpoint_hint` = the bucket/key
     locator. Surfaced the moment a counterpart is observed at the key.
   - IDENTIFY completes over the admitted channel (`EXTENSION-NETWORK`).
   - `candidate_1` — `peer_id` populated, `supersedes: candidate_0.content_hash`. `candidate_0` remains
     as the observation record; entities are immutable.
   - `decision.candidate` SHOULD reference the head of the chain.
3. **Absent, not explicit-null** — §2.1's wire convention erratum is a MUST and is load-bearing for
   cross-impl byte equality (three impls picked two encodings on `candidate.peer_id`; ECF hashes null and
   absent differently).

**Their stated exposure is discharged.** They wrote that *"emitting candidate/decision entities alongside
the current in-memory type is additive; making them the only path is a refactor we would rather do
once."* The shape above is the once.

### §2.3 Normative delta

- `EXTENSION-DISCOVERY` §2.1 — add `"rendezvous"` to the `backend` enum.
- `EXTENSION-DISCOVERY` §5 — add **§5.5 EXTENSION-SIGNALING**, carrying §2.1's mode split and §2.2's
  TOFU + successor requirements.
- `EXTENSION-DISCOVERY` §6 — move *rendezvous* from the unnamed "…" of the staged-growth list into a
  landed backend.

No new entity type, no wire change, no change to SIGNALING.

---

## §3 Q8.3 — a share audience and a discovery decision are one authoring path

**Yes, and they are not merely compatible — they are the same object.**

- `DISCOVERY` §2.1: *"The `decision.grant` field references a `system/capability/grant` entity (per V7
  §6.2 capability-mint) by bare hash."*
- `PROPOSAL-SHARE-AS-GRANT-AND-THE-AUDIENCE-CARRIER` §1.3 (ruled today): a share's audience is carried by
  the **`grantee`** of a minted token — one token per audience member.

A `decision(grant-limited)` on a candidate mints exactly such a grant, against the identity IDENTIFY
established (§5.3, *"the grant is issued against that identity"*). **So an `Audience::Peer(...)` entry
*is* the grant a discovery decision produced.** browser-rust's own reading — *"an audience ought to be
produced by a decision on a candidate rather than typed into a form"* — is correct, and it closes the
affordance they filed twice as missing (buildout item 22; `PLAN-OF-RECORD-capability-enforcement` §9).

**One boundary, so the two do not swallow each other.** DISCOVERY §10 excludes *"fetching a candidate's
public manifest / sites BEFORE the user decides"* as **L5 territory**, and holds the substrate at
find-and-prompt. So:

| Concern | Home |
|---|---|
| Surface a candidate; mediate admission; mint the grant | **DISCOVERY** (§2) |
| Decide *what* to share and title it; browse what a peer offers | **L5** — `APP-CONVENTION-SHARE` |

The share convention **consumes** discovery decisions. It does not replace them, and DISCOVERY does not
grow a share catalog.

---

## §4 Q3 residual — the QR backend is shape-only, and that is now cheap

`EXTENSION-DISCOVERY` §6 calls QR *"near-term, small"* but carries **no entity shape, no token, and no
composition subsection** — it is named for shape, not settled. **Do not build against it as if it were.**

**It costs little now:** QR is the same mold as `rendezvous` — out-of-band candidate exchange, TOFU or a
claimed `identity_hint`, then §2.2's successor chain. The difference is that a QR payload *may* legitimately
carry an `identity-claim` (the displaying peer asserts its own peer-id in the payload), where a tag
rendezvous may not. Build `rendezvous` in the §2.2 shape and QR is an additive second backend, not a
second design.

---

## §5 Q5 — a tag is a meeting point; the identified peer is what gets a name

`EXTENSION-SIGNALING` §3.2 types `tag` as a *"**public label** — discovery convenience, **not** access
control"*, and §3.4 adds that a tag meeting is *"concurrent, on one pool"* with TTL-reaped buckets.
Nothing persists it, and whoever is standing there is whoever is standing there.

**So binding a *tag* as a REGISTRY local-name is a category error** — the name would resolve to a meeting
point, not to a peer, and the next occupant inherits it.

**Binding the *peer you met* is ordinary and intended.** After IDENTIFY, the §2.2 successor candidate
carries a verified `peer_id`; binding that peer-id to a local petname is REGISTRY's local-name backend —
Level 1 of the legacy landscape's continuum (*"local petnames"*). The UI phrasing follows the mechanism:
not *"remember the tag `alice`"* but *"you met this peer at `alice` — remember **them** as Alice?"*

---

## §6 Fold

1. `specs/extensions/EXTENSION-DISCOVERY.md` — §2.1 enum value, new §5.5, §6 list update (§2.3). Version
   bump; this adds a backend, which is a normative surface addition.
2. `guides/GUIDE-EXTENSION-DEVELOPMENT.md` — nothing owed; the DISCOVERY/SIGNALING seam is stated in the
   specs themselves per §1.

**Not in scope:** the `data_relay` credential channel (Q18), Mode C, and the control plane
(`:advertise`, limits, `EXTENSION-ROUTE`) — named by browser-rust so they are not folded into this.
