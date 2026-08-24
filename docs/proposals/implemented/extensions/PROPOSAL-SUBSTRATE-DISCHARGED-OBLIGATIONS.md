# PROPOSAL — what a substrate supplies vs. what an implementation owes

**Status:** **RULED + FOLDED 2026-08-16** — `EXTENSION-SIGNALING` §11.5.1 and `EXTENSION-NETWORK`
Amendment 14 / §10.3. **Fixed in place, no rev bump** — both halves are cohort findings.
**Source:** `entity-browser-rust`, `ROUTING-2026-08-16-the-two-nat-rig-is-built-and-what-it-does-not-close.md`,
read from their worktree at `53898c1` *(the routing doc was untracked at read time — see §4)*.
**Domain:** conformance-claim scoping.

---

## 0. The one subject underneath two findings

browser-rust routed two things that look unrelated — a NAT rig that must measure itself, and a
keepalive MUST that a browser discharges for you. **They are the same question:** when a substrate
supplies a behaviour, what has the implementation proved, and what does it still owe?

§11.5.1 already answers the *claim* half — *"a gate claim is scoped by its substrate."* Neither half
below is new doctrine; both are that doctrine reaching a case it had not been pointed at.

## 1. §11.5.1 — the emulated row must MEASURE its mapping, not assert it `[FOLDED]`

**The fourth instance of the loopback-blindness class, and the first about the *harness*.** The
existing three are all a *peer* wrong in a mapping-dependent value. This is a **substrate** that
measures nothing and is believed.

**browser-rust's first rig accepted unsolicited inbound UDP.** That accept **confirms a conntrack
entry** whose reply tuple is exactly the one the peer's own outbound punch needs; the outbound then
loses the race for its advertised port and is remapped, **so both sides send from ports the other
never heard of and the punch cannot converge.** A real NAT *drops* that packet and keeps no state.
Measured, not theorized: a bare UDP punch reported NO-PACKETS while conntrack showed A leaving from
`:32941` having advertised `:60449`. Two symptoms — signaling `bucket_full` and
`addIceCandidate: Unknown ufrag` — were chased first and were **downstream of it, not app bugs.**

**Why it belongs in §11.5.1 and not in their README.** The table *grants* the emulated row a set of
proofs. **A rig that merely looks like that row inherits those grants silently.**

**Folded** as a qualifier on the row plus a fourth bullet in the class. Their wording is kept nearly
verbatim because it is better than a paraphrase: *"a rig that asserts them is loopback wearing a
NAT's clothes."*

**Also folded, from their §2:** the emulated row is a **port-restricted cone** NAT (conntrack
endpoint-independent mapping + address/port-dependent filtering) and **structurally cannot model
symmetric NAT** — so it cannot supply evidence for anything TURN exists for. That belongs in the
"cannot prove" column, which previously said only *"CGNAT, symmetric, carrier quirks"* without
saying why the row cannot reach them.

## 2. Amendment 14 obligation 2 — the substrate may discharge it, and the claim must say so `[FOLDED]`

**browser-rust asked the right question before running the gate rather than after.** On WebRTC the
browser runs **RFC 7675 ICE consent freshness** itself — STUN binding requests on the selected pair
every few seconds — so the NAT mapping is refreshed **whether or not the peer implements §5
keepalive**. Their question: does a green *survives-idle* run validate Amendment 14's obligation?

**It does not, and the landed text already contains both halves of the answer without joining them:**

- **§10.3 obligation 2** puts a `MUST` on the **mechanism** — *"it MUST run keepalive (§5)."*
- **§1822 v1 posture** gates the **outcome** — *"conformance gates the outcome … carries ordinary
  operations and survives idle."*

**Ruled: the obligation is on the property — the mapping stays alive across idle — and a substrate
that refreshes the mapping itself discharges it.** But:

1. **The implementation MUST know and declare which mechanism holds the mapping open.** An
   implementation that is silently relying on the substrate has not chosen anything, and will carry
   that assumption onto a substrate that does not supply it.
2. **A green survives-idle run on a self-refreshing substrate is NOT evidence for a substrate that
   is not.** A native punch on the *same* §11.5.1 row still owes §5 keepalive in full. This is
   §11.5.1's scoping rule applied to an obligation rather than to a property.

**So browser-rust's green run, when it comes, may claim §1822's outcome on the WebRTC substrate and
MUST NOT be recorded as closing obligation 2 generally.** Their reading was right, and they should
run the gate.

## 3. Not folded — recorded as their asks, answered elsewhere

- **The evidence row** (`PROPOSAL-SIGNALING-ICE-PROVISIONING-LIFETIME` §3 item 1, `INDEX.md`) —
  corrected there, not here.
- **Their §1.2 constraint** — *"whatever wins must reach a peer that resolves nothing"* — recorded in
  that proposal against the TURN-placement ruling. **It is a constraint on the answer, not a vote for
  an option, and they were explicit that they have no preferred shape.**
- **`SDK-OPERATIONS` §7.3 → `system/peer/status`** — folded there.

## 4. Provenance note, because the pin is unusual

The routing document was **untracked in browser-rust's worktree** when read (`53898c1`, two modified
files, the doc itself `??`). **It is quoted here as read, and the pin says so.** If it is amended
before it lands, this proposal's §1 and §2 are re-checked against the committed version — an
untracked source is a real reading, but it is not yet a durable one.
