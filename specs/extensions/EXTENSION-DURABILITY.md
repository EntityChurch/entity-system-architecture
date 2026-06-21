# Durability Extension — Exploratory Reference Specification

**Version**: 0.1

**Status**: Draft

**Provenance**: The content below is the EXTENSION-INBOX v5.7/v5.8 §10 material lifted verbatim into a standalone file. It was originally landed in EXTENSION-INBOX (v5.7 + Amendment 1 = v5.8), cross-impl validated against entity-core-go / entity-core-rust / entity-core-py, then extracted when the architecture team concluded the thread had broached too far into user-space semantics — it pattern-matched on log-system conventions (durability levels, response-status branching, supported-level advertisement) without an actual deployment driver, contradicting the project's minimize-to-bare-substrate posture. The material is preserved here as a reference design rather than discarded, because the cross-impl analysis behind it is real work and the question of whether anything in the durability shape eventually belongs as a system extension is left open.

**No deployment is required to install this extension.** It is not normative for V7. It is not normative for any other extension. EXTENSION-INBOX does not depend on it. V7 v7.46 does not depend on it. The 412 and 202 reservations in V7 §3.3 were reverted along with the extraction (412 is unused at the V7 level; 202 remains in use by EXTENSION-INBOX §7.1 — the inbox-ack semantics — independently of this file).

**Open question (not answered here):** whether durability — even as defined below — belongs as a system extension at all, or whether the response-observability concern it addresses is better solved by a much narrower protocol-level rule (e.g. "a peer that cannot honor a request-side marker MUST return an explicit refusal status, not silently drop the marker"). The latter is a one-line V7 amendment; the apparatus below is much larger and may not be needed to get the observability property.

**Depends**: ENTITY-CORE-PROTOCOL.md v7.46+. (If installed, this extension reintroduces the 412 status code and the durability use of 202 *within its own surface* — they are not part of V7's core status table.)

---

## 1. Model and trust posture

"Durable" means: store the request reliably and let the sender find it again. Three request knobs are orthogonal — *wait or don't*, `deliver_to`, and the durability marker (§2); setting none yields prior behavior unchanged. The receiver's own configured policy decides what durability it can provide; the sender does not blindly trust the receiver's outcome and vice versa — **both sides are observable** (a mistrust model, not a trust model). The durable entry is found again by the receiver-returned handle (§6).

**Durability vs. persistence.** Durability is the property that a request can be **reliably retrieved by id** after the response. Persistence (local-survives-restart) is one substrate; cross-peer hosting / replication is another. The spec contract is the response (§5) and the handle (§6); substrate is the receiver's choice (§4), and receivers MAY advertise it via zone characteristics (§3). The inbox is **one example** of a durable store — not the canonical store; a receiver MAY preserve a durable entry anywhere in its tree.

## 2. Request-side durability marker (optional)

```
durability_request: {type_ref: "system/durability-request", optional: true}

system/durability-request := {
  fields: {
    level:     {type_ref: "primitive/string"}
                 ; the requested durability level; vocabulary illustrative, not a frozen enum (§7)
    must_have: {type_ref: "primitive/bool", optional: true}
                 ; default false. false = best-effort (take less, observably). true = required (refuse if unmet, §5).
  }
}
```

**Extends**: `system/protocol/execute`. Independent of `deliver_to`/`deliver_token`.

## 3. Advertise supported durability (optional discovery)

A receiver **MAY** expose the durability levels it supports and the **zone characteristics** under which it provides them (e.g. `same-host`, `same-network`, `same-availability-zone`, `cross-region` — receiver-defined, extensible; the spec does not prescribe the vocabulary, same posture as strength-level values per §7). Probe-via-request (the §5 contract itself) is the canonical fallback when advertisement is absent — a sender sends what it wants with `must_have: false` and reads the response's `applied` / `reason`. Absence of advertisement does not change the response contract.

## 4. Reconcile

On accepting an EXECUTE carrying `durability_request`, the receiver reconciles the request against its own policy: it provides `min(requested, policy-supported)` for self-determinable strengths, decided **at acceptance from its own configuration** — not a prediction of another peer's future state. What it then reports is §5.

## 5. The response: status + the `durability` field

The response is the **status code plus a pinned `durability` field**. One distinct meaning per status number — no overloading.

```
durability: {type_ref: "system/durability-result", optional: true}

system/durability-result := {
  fields: {
    requested:     {type_ref: "primitive/string"}
    applied:       {type_ref: "primitive/string"}
                     ; the durability PHYSICALLY IN PLACE at the moment of this response, or "none".
                     ; ONE meaning in every row — it never names a promise.
    committed:     {type_ref: "primitive/string", optional: true}
                     ; a strength committed to a pathway that completes ASYNCHRONOUSLY. Present ONLY with status 202.
    max_available: {type_ref: "primitive/string", optional: true}
                     ; the best the receiver could offer. Present ONLY with status 412.
    handle:        {type_ref: "system/path", optional: true}
                     ; absolute tree path where the durable entry can be read (the sender's lookup address).
                     ; Present when applied != none; on 202, names where the committed entry will land.
                     ; The RECEIVER chooses the path — the spec does NOT prescribe layout. See §6.
    reason:        {type_ref: "primitive/string", optional: true}
                     ; reason code. Spec-enumerated cases use PINNED spellings; other diagnostic
                     ; codes remain implementation-defined (§7):
                     ;   "no_durable_store"          — receiver has no durable store      (§5 row 3, 200)
                     ;   "durability_required_unmet" — must_have unmet, recognized level (§5 row 4, 412)
                     ;   "unknown_level"             — requested level not recognized    (§5 rows 6/7, 200/412)
                     ;   "duplicate_request_id"     — (author, request_id) already preserved (§5 row 8, 409)
  }
}
```

| Situation (requested level X; `must_have`?) | Status | `durability` |
|---|---|---|
| receiver can do ≥ X | **200** (final — nothing to watch) | `{ requested: X, applied: X, handle: P }` |
| weaker Y, **not** must-have (best-effort) | **200** (final) | `{ requested: X, applied: Y, handle: P }` |
| no durable store, not must-have | **200** (final) | `{ requested: X, applied: none, reason: no_durable_store }` |
| **must-have**, cannot meet — *recognized* level (weaker only, none, or topology mismatch) | **412** (refused — operation **not performed**) | `{ requested: X, applied: none, max_available: Y or none, reason: durability_required_unmet }` |
| a strength the receiver **is configured for** but that completes later (replication-class — not self-certifiable at acceptance) | **202** (accepted; completes asynchronously) | `{ requested: X, applied: <physically in place now>, committed: X, handle: P }` |
| **unknown level**, not must-have (best-effort fallback) | **200** (final) | `{ requested: X, applied: none, reason: unknown_level }` |
| **unknown level**, must-have (fail-closed) | **412** (refused — operation **not performed**) | `{ requested: X, applied: none, max_available: <strongest recognized> or none, reason: unknown_level }` |
| **duplicate (author, request_id)** — pair matches a previously preserved entry | **409** (conflict — operation **not performed**) | `{ requested: X, applied: none, reason: duplicate_request_id }` |

Status meaning is fixed and is the only branch a consumer needs:

- **200** — the durability outcome is **final**; nothing to watch. The not-must-have cases are genuine, settled successes; `applied` is the level achieved (possibly `none`); `handle` is present whenever `applied != none`.
- **202** — accepted; the `committed` strength completes **asynchronously** and is observable via `handle` at the request-id address (§6). This reuses EXTENSION-INBOX.md §7.1's existing 202 semantics (accepted, async, observed elsewhere); it is not a new sense. `applied` still reports only what is physically in place at response time; `committed` carries the async target.
- **412** — a *required* durability precondition could not be met. Two sub-cases: (i) the receiver recognizes the requested level but cannot meet it (weaker only, none, or topology mismatch) → `reason: durability_required_unmet`; (ii) the receiver does **not recognize** the requested level and `must_have` is true → `reason: unknown_level` (**fail-closed** — never silently downgrade an unrecognized must-have level; §7 strength values are receiver-defined and one peer's vocabulary may not match another's). In both cases the operation is **not performed** — refused at acceptance, **safe to retry elsewhere, no double-execution**. 412 is *not* a post-hoc failure report on a completed operation. (412 vs. 501: 501 = the operation itself is unsupported; 412 = the operation is supported but the request's required-durability precondition failed, so it was refused — exactly "Precondition Failed".) Within this extension only — V7 v7.46 does not reserve 412 at the core level.
- **409** — the request's `(author, request_id)` matches a previously preserved entry (409 is already in V7's reserved set for `duplicate_request_id`); the receiver enforces uniqueness over the pair regardless of storage layout. The operation is **not performed**; the prior entry stands. `reason: duplicate_request_id`. Idempotency caching (returning the cached verdict before issuing 409) is implementation-defined.

**Invariant (the contract's whole point).** `applied` always means *durability physically in place at the moment of the response* — one meaning in every row, it never overstates. A promise of later durability lives only in `committed`, gated to status 202. No response ever claims a durability it does not yet have. Replication-class is therefore inherently `202`-then-observe even when required — no receiver can synchronously prove a second peer holds the data; "required replication" means "the sender verifies it landed (§6)", not "the receiver blocks until it has."

`deliver_to` interaction: a `deliver_to` target with no handler for the delivered operation loses the result — see EXTENSION-INBOX.md §4.1 / §4.3 (`receive`), the `system/delivery-spec.operation` override, or the embedded-peer composition (continuation re-dispatch). State the precondition; this is the silent-loss class the durability contract is intended to make observable.

## 6. Finding it again (the handle)

**Handle-in-response.** The receiver returns the address in the response: when `applied != none` (or on `202`, where the committed entry will land), `system/durability-result.handle` carries the absolute tree path of the durable entry. The sender reads it as any tree path — `tree:get` / sync / subscription. The receiver chooses storage layout; the sender follows the path.

**The receiver decides layout, not the spec.** The inbox is **one example** of a durable store; a receiver MAY preserve a durable entry anywhere in its tree (under `system/inbox/...`, under a per-tenant namespace, behind a content-addressed path, anywhere it pleases). A receiver that wants offline-constructable addresses — so senders can derive paths without first seeing the response — MAY use an invariant-pointer-style scheme per ENTITY-CORE-PROTOCOL.md §3.5. Optional, not required. (The spec does not mandate the inbox path, an invariant-pointer scheme, or any specific layout.)

**Receiver-side uniqueness.** `(author, request_id)` uniquely identifies a request (ENTITY-CORE-PROTOCOL.md §3.2/§3.3); a stored inbox delivery carries `original_request_id` (EXTENSION-INBOX.md §2.1). The receiver enforces uniqueness over the pair **regardless of storage layout** — a duplicate arrival is **409 `duplicate_request_id`** (§5). The `handle` is the sender-side address; the pair is the receiver-side key.

**Topology and freshness.** Which peer physically holds the freshest copy is a routing/partial-projection concern; the address and content hash are stable. Sync provides freshness; retention is the hosting deployment's storage choice (a topology role), not something sync provides.

## 7. Strength-level vocabulary is illustrative; reason codes are pinned

Two distinct carve-outs:

- **Strength-level vocabulary** (the values inside `requested` / `applied` / `committed` / `max_available`; the zone characteristics in §3) is **illustrative, not a ratified enum**: the field *shape* in §5 is pinned for cross-impl determinism; the *level vocabulary* is named basic and derived from real use, not frozen. Receivers define what they offer; senders ask for what they want; the response says what was achieved. Unrecognized levels are handled per §5 (fail-closed on `must_have`, best-effort fallback otherwise). Refuse-at-acceptance (412) applies only to strengths the receiver self-determines at acceptance; replication-class is `202`-then-observable.
- **Reason codes for spec-enumerated cases** are **pinned**: `no_durable_store`, `durability_required_unmet`, `unknown_level`, `duplicate_request_id` (§5). Other diagnostic `reason` strings (additional impl-specific codes attached to non-spec-enumerated situations) remain implementation-defined.

## 8. Conformance (within this extension)

These obligations apply only to peers that install this extension. A peer that does not install it is unaffected; its response surface is V7 v7.46 unchanged.

**MUST**

- Never silently discard a `durability_request`; always answer with status + the `durability` field per §5.
- `applied` MUST report only durability physically in place at response time; MUST NOT report a not-yet-achieved (promised) level in `applied`. A promised strength MUST be carried in `committed` with status 202 only.
- Status 412 MUST mean the operation was not performed (refused at acceptance). 412 is returned when (i) a `must_have` request's required durability cannot be met at a *recognized* level, or (ii) the requested level is **not recognized** by the receiver and `must_have` is true (§5 fail-closed rule).
- The `durability` field MUST follow the pinned `system/durability-result` shape; `committed` MUST appear only with 202; `max_available` only with 412.
- The `handle` field MUST be present in the response when `applied != none`, and on 202 (naming where the `committed` entry will land); MUST be absent otherwise. The `handle` is the sender's lookup address (§6); the receiver chooses the path.
- A durable request whose `(author, request_id)` matches a previously preserved entry MUST be rejected with **409 `duplicate_request_id`** (§5). The receiver enforces uniqueness over the pair regardless of storage layout. Idempotency caching (returning the cached verdict before issuing 409) is implementation-defined.
- A `durability_request` with a `level` value the receiver does not recognize MUST fail closed: `must_have: true` → 412 with `reason: unknown_level` and `max_available` = the strongest level the receiver does recognize (or `none`); `must_have: false` → 200 with `applied: none, reason: unknown_level` (best-effort fallback) (§5).
- Reason codes for spec-enumerated cases MUST use the pinned spellings (§5 / §7): `no_durable_store`, `durability_required_unmet`, `unknown_level`, `duplicate_request_id`.

**SHOULD**

- Use 202 (not 200) whenever the achieved durability is not yet final at response time.

**MAY**

- Advertise supported durability levels and zone characteristics (§3) — optional offline discovery; probe-via-request is the canonical fallback when absent.
- Use an invariant-pointer-style scheme (ENTITY-CORE-PROTOCOL.md §3.5) for the `handle` path, so senders can construct lookup addresses offline (§6).

**Implementation-Defined**

- The concrete durability `level` vocabulary and zone-characteristics vocabulary (§7); the receiver's policy for which levels it offers under which zones.
- The storage layout / path layout the receiver chooses for durable entries (§6 — the receiver returns this in `handle`).
- The receiver's durability policy and which replication topologies it is configured for (§4).
- Diagnostic `reason` code spellings for situations not enumerated in §5 (spec-enumerated reason codes are pinned per the MUST above).
- Idempotency caching for duplicate `(author, request_id)` requests (returning the cached verdict before issuing 409).
