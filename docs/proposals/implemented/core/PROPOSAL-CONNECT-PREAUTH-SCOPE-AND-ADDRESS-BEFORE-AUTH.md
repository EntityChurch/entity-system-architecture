# PROPOSAL — two pre-establishment orderings the corpus never stated, one of which is holding a skip open

**Status:** **FOLDED 2026-09-03** — D1–D3 landed and verified: `ENTITY-CORE-PROTOCOL` **0.8.2.6** (§4.7 address-before-auth table; §4.2 bullet 1 state qualifier).
**Target:** `entity-core-protocol` `specs/ENTITY-CORE-PROTOCOL.md` §4.2 + §4.7.
**Filed by:** `entity-core-rust` (`ROUTING-2026-09-02-g` §1a and §4), both raised as *"ours to
implement, yours to write"* and neither changed unrouted. Both re-derived here.
**Read at:** core-protocol `d43d66d` (**0.8.2.5**) · rust `77916ed` · go `78a9f2d` · py `4c7a6bc`

---

## §1 Q1 — a foreign-namespace EXECUTE arriving before establishment: which gate answers?

`0.8.2.5` ruled the own-namespace case (**401 `authentication_failed`**). §4.7's `invalid_request`
paragraph independently names *"an EXECUTE naming a foreign namespace (§1.4)"* in its class. **Both
rules match a pre-establishment foreign-namespace EXECUTE and the corpus does not order them.**

rust implemented **address before authentication** and asked to have the reading checked rather than
accepted:

| Pre-establishment EXECUTE | Answer |
|---|---|
| `system/protocol/connect` | §4.7's rows, unchanged |
| own namespace, non-connect | **401 `authentication_failed`** (0.8.2.5) |
| **foreign namespace** | **400 `invalid_request`** |

### The ruling: address first. Three legs, none of them cohort behaviour.

**(a) The status has to select the right remedy, which is the argument `0.8.2.4` already made.**
When §4.7 split `connection_sequence_error` from `invalid_request` it gave the reason: *"because
clients key error handling off `result.data.code`, a code that selects the wrong remedy fails the
contract this table exists to provide."* **A 401 says: authenticate and retry.** For a
foreign-namespace address that retry is guaranteed to fail — the frame is refused at every
authentication state, so the 401 invites a caller into a loop that cannot terminate. 400 says *this
frame is not a request to me*, which is true and actionable.

**(b) The address refusal is a property of the frame, not of the sender.** §6.5 step 3 calls it *"a
gate, not an ordering preference"* and says the request *"never becomes an authorization question."*
Authentication state cannot change the answer, so evaluating it first can only mislead.

**(c) It is where the gate already sits.** §1.4's own text — folded at the PD-1h ruling — says of the
downstream permission check: *"the check is not redundant here; it is **unreachable** here."* The
foreign-namespace refusal happens at canonicalization (§6.5 step 3), which is **upstream of the
authentication boundary**, so address-first is the order the pipeline already has.

**rust's counter-argument, stated because they raised it and it is the honest objection:** §6.5 step
3 is described as firing *after the envelope is authenticated*, and pre-establishment nothing is — so
a reader could put the 401 first on the grounds that its precondition is the one actually met. **The
answer is that §6.5 step 3's placement describes the established-connection pipeline, not a
precondition on the rule**; the rule's own subject is the address, which is available on the first
byte. Leg (a) is what decides it, and it does not depend on where the check happens to sit.

**Corroboration, verified and cited last (L18):** go's `probeExecuteBeforeEstablished` targets the
responder's **own** namespace *"so the §1.4 foreign-namespace gate cannot be the refusing
mechanism"* — read at `78a9f2d`. That precaution is only necessary if the address gate wins, so go's
harness already assumes this order. If it did not, the argument above would not change.

**Why it is worth a sentence rather than leaving it to converge:** rust's point that this row is *the
control* is the strongest part of their filing. A peer that "fixes" CE-1 by relabelling its
pre-establishment catch-all to `authentication_failed` passes every own-namespace row and fails only
the foreign one — **so the foreign-namespace row is the only assertion that distinguishes a real fix
from a rename**, and it is currently unspecified.

## §2 Q2 — is `system/protocol/connect` still pre-authorized after establishment?

rust reads §4.2 (*"the sole pre-authorized path"*, no state qualifier) against §5.1 (*"author and
capability required on all authenticated EXECUTE"*), concludes a post-handshake `ping` must be
signed, and has logged it in their `SPEC-AMBIGUITIES.md`. **It is holding a conformance row open:**
`connect_ping_before_hello` SKIPs at their seat because the applicability control
`pingServedOnceEstablished` sends an *unauthenticated* post-handshake ping, gets a non-200, and
disables the 409 row.

### The ruling: it stays pre-authorized, in every connection state. This is landed text.

**§3.3, line 773:**

> All EXECUTE requests MUST include `author` and `capability` — **except** requests targeting the
> connection path (§4).

**The exception is on the path and carries no state qualifier.** §4.2's *"sole pre-authorized path"*
says the same thing in the same shape. And §5.1's requirement is scoped to *"every **authenticated**
EXECUTE"* — a class line 773 defines by **excluding** the connection path. **The two texts are not in
tension; §5.1's subject simply does not include this frame.** rust read §5.1 as a general rule with
§4.2 as an exception to it; it is the other way round.

**It is also coherent on the security side, which is worth saying because "unauthenticated frame on
an established connection" reads alarming.** By the time the connection is established **both peers
have authenticated** (§4.6) and the frame arrives on that authenticated connection. Per-frame
`author`/`capability` on the connect path would be re-proving, at every keepalive, an identity the
connection already carries. That is why `ping` is on the pre-authorized path at all.

### This resolves rust's §4 item against rust, and go's control is correct as written

rust proposed that go either send the control's ping authenticated or state that its criterion is
*unauthenticated* ping. **Under this ruling neither is needed:** an unauthenticated post-handshake
ping MUST be served, so `pingServedOnceEstablished` is asking exactly the right question and a peer
that answers non-200 is the non-conformant party. **go's harness needs no change; rust's intercept
moves ahead of `verify_request` and the skip closes.**

rust's general point still stands and is kept on the record — *a control whose criterion is answered
by a narrower probe than the rule silently disables a row against a peer that implements it* — it
simply does not fire here, because the narrower probe is the rule.

## §3 What this does not do

- **No new obligation in Q2.** §3.3 line 773 has said it since before this cohort existed. §4.2 gains
  a state qualifier so the ambiguity has nowhere to live.
- **Q1 is new normative text**, and it is the only new obligation in this proposal. It is one row of
  a table two seats have already built consistently.
- **Neither is a flag day (L21 third shape).** Q1 changes what a peer emits for an input that is
  refused either way. Q2 makes a peer *accept* something it may currently refuse — which is the
  direction that can partition — but the only affected frame is an unauthenticated connect-path
  operation, and a peer refusing it today is refusing a keepalive, not a connection. **Divergence
  unit: a `ping` between a strict peer and a lenient one; no handshake, no established connection is
  affected.**

## §4 Delta table

| # | Document | Section | Change |
|---|---|---|---|
| **D1** | `ENTITY-CORE-PROTOCOL.md` | §4.7, in the 0.8.2.5 note | Add the address-before-authentication ordering row and its derivation. |
| **D2** | `ENTITY-CORE-PROTOCOL.md` | §4.2 bullet 1 | *"the sole pre-authorized path"* → gains **"in any connection state, before and after establishment"**, citing §3.3 line 773 as the authority (**L23** fourth shape). |
| **D3** | `ENTITY-CORE-PROTOCOL.md` | line 3 | Version → **0.8.2.6** (shared with `PROPOSAL-STATUS-TABLE-DEFAULT-CODES`). |

## §5 Cohort impact

| Seat | Owed |
|---|---|
| `entity-core-rust` | Q1: already implemented — no change. Q2: move the §5.1 intercept ahead of `verify_request`; `connect_ping_before_hello` un-skips. Close the `SPEC-AMBIGUITIES.md` entry. |
| `entity-core-go` | Q1: harness already assumes it — no change. Q2: `pingServedOnceEstablished` is correct as written — **no change**, and do not "fix" it. |
| `entity-core-py` | Both: confirm the post-handshake unauthenticated `ping` is served, and the foreign-namespace pre-establishment row. Neither measured at this seat. |
| `entity-core-keystone` | Vendor 0.8.2.6. The generated cohort is unmeasured on both rows; the ping row is cheap and 46 peers wide. |

## §6 Open items

None. Q2 is landed text made unambiguous; Q1 is derived from §4.7's own remedy-selection rule and is
already built at the seat that raised it.
