# PROPOSAL — `MUST NOT mint a synonym` has three live violations, and the third one hides a conformance gap two releases old

**Status:** **FOLDED 2026-09-02** — all five §7 delta rows landed and verified in the tree:
`ENTITY-CORE-PROTOCOL` **0.8.2.5** (§4.7 note + version) and `EXTENSION-SIGNALING` **v1.2**
(§4.3, §9.2 scope note, header + fold note). The `EXTENSION-TYPE` half is **not** folded here — it is
`PROPOSAL-TYPE-OPERATION-ERROR-TAXONOMY`, DRAFT, routed to `entity-core-rust`.
**Target:** `entity-core-protocol` `specs/ENTITY-CORE-PROTOCOL.md` §4.7 (0.8.2.4 → 0.8.2.5) ·
`specs/extensions/EXTENSION-SIGNALING.md` §4.3 + §9.2 (v1.1 → v1.2).
**Filed by:** `entity-core-go` (`ROUTING-2026-09-02-g` §2, relaying `entity-core-py`'s SA-PY-33) and
`entity-core-go` (`ROUTING-2026-09-02-h`, the CE-1 three-way measurement). Both re-derived here.
**Read at:** core-protocol `98e8bdd` (**Version: 0.8.2.4**) · arch `60da9ef` · go `4936cf2` ·
rust `1ded022` · py `4c7a6bc` · keystone `6dcbd22`

---

## §1 The clause, and what it actually binds

`ENTITY-CORE-PROTOCOL` §4.7 landed this at **0.8.2.4** (line 1895):

> **`invalid_request` — the generic malformed-request code (normative, 0.8.2.4).** … Extension
> specifications use this code for the same class and **MUST NOT mint a synonym.**

§3.3's 400 row (line 830) states the complement: `invalid_request` is the default 400 code, "the generic
400 code **an extension handler uses** for a structurally invalid request", with a closed set of
more-specific codes where one applies — `invalid_path`, `invalid_params`, `unexpected_params`,
`chain_depth_exceeded`, `signature_path_conflict`.

So the clause binds **extension specifications and extension handlers**, not just the connect path.
It was folded as a sentence about the connect handshake and its reach is the whole 400 class.

## §2 The sweep — by the rule's SUBJECT, not its tokens (L23)

Grepping `bad_request` finds one spec. That is the token sweep and it is not the rule's subject. The
subject is *"a code minted for the generic-400 class."* Swept `specs/`, `guides/` and
`entity-core-protocol/specs/`, then every implementation tree for string-literal wire codes:

| Minted code | In the corpus? | Emitted by | Sites |
|---|---|---|---|
| `bad_request` | **yes** — `EXTENSION-SIGNALING` §4.3, §9.2, §9.2's key-length rule | go, py | go `ext/signaling/node/node.go:282`; py `entity_handlers/signaling/node.py:111` |
| `connection_required` | **no — minted** | go | `cmd/.../execute.go:159` |
| `handshake_failed` | **no — minted** | rust | measured on the wire (§4) |
| `bad_request` (TYPE handlers) | **no — minted** | rust | `extensions/type-system/src/{validate,constraint}.rs`, 8 sites |

`invalid_params` in `DOMAIN-LOCAL-FILES` and `EXTENSION-REVISION` is **not** a violation — §3.3 names it
in the sanctioned more-specific set. It is in this table's negative space deliberately: the sweep had to
distinguish *a sanctioned specific code* from *a minted synonym*, and only the second is the defect.

**`entity-browser-rust`: zero sites**, searched at the same commit.

## §3 SIGNALING — the surfaces already differ, and only one line leaks

`EXTENSION-SIGNALING` has two surfaces (§2.2): **wrapped** (ordinary EXECUTE to `system/signaling`) and
**unwrapped** (the non-entity TCP protocol of §9). §2.2's MUST is:

> The three verbs are **semantically identical on both surfaces.** A verb completable on one and not the
> other is non-conformant.

That is a MUST about **verb completability and semantics**, not about error representation — and the two
surfaces already carry errors in structurally different shapes by construction. The unwrapped surface
answers `{ ok: false, error: tstr }`: a bare string in a protocol with no status line. The wrapped
surface answers an EXECUTE_RESPONSE with `(status, code)`. **§4.7 governs EXECUTE codes. It does not
reach a `tstr` in a non-entity protocol.**

So §9.2's closed enum is **already correctly scoped** by its own section placement — §9 is titled *The
Unwrapped Protocol* — and `bad_request` is legal there. Nothing in §9 is the defect.

**The defect is one line: §4.3**, which sits in §4 *Handler and Operations* — the **wrapped** handler
section — and reads:

> Errors: `message_too_large` (§5 pin 5), `bucket_full` (§5 pin 5), `bad_request` (malformed input or
> wrong key length), `rate_limited` (§8.2).

That is an EXECUTE error list naming a synonym, and it is **why both implementations emit it on the
wrapped surface.** go's own docstring says so outright: *"renders the … `bad_request` code **on the
wrapped surface**."* Both seats did exactly what the spec told them.

**The translation preserves SIGNALING's existing granularity exactly.** §9.2 gives all four causes
(malformed frame or CBOR, unknown `op`, wrong key length, missing field) **one** code. The minimal
divergence-free wrapped-surface equivalent is therefore **one** code — the generic `invalid_request`,
which §4.7 claims for precisely this class. Splitting the four causes across `invalid_request` and
`invalid_params` would invent a boundary the corpus does not draw and that two conformant peers will draw
differently: a tightening that closes no hole and costs interop (**L25**).

## §4 CE-1 — the answer is landed text, and it landed at 0.8.1

`entity-core-go` measured a non-connect EXECUTE arriving **before the handshake completes**, three-way on
the wire, with a probe that targets the responder's **own** namespace so the §1.4 foreign-namespace gate
cannot be the refusing mechanism (**L8**'s two-mechanisms-one-observable trap, correctly avoided):

| impl | observed | in a spec? |
|---|---|---|
| go | `403 connection_required` | no |
| rust | `400 handshake_failed` | no |
| py | `403 capability_denied` | code yes, **status wrong for the class** |

This was filed as unruled, and **it is not.** `ENTITY-CORE-PROTOCOL` **§4.2 Pre-Authorization Rules**,
bullet 3:

> EXECUTE targeting any other path without valid authentication MUST be rejected per §5.2a: a
> missing/unverifiable `author` or signature is auth-class **401**; an authenticated request lacking a
> covering capability is authz-class **403** *(0.8.1, F32 — this rule previously said a blanket 403,
> contradicting §4.4/§5.2a)*.

A pre-establishment non-connect EXECUTE **is** "EXECUTE targeting any other path without valid
authentication": no handshake has completed, so there is no verified signer, so §5.2a's discriminator
(line 2367 — *auth-class when the EXECUTE itself cannot be authenticated*) puts it in the **401**
column. §5.2a's table gives the code for that class: **`authentication_failed`**.

**So all three implementations are running the pre-F32 text.** go's and py's 403s are exactly the blanket
403 that **F32 retired at 0.8.1**; rust's 400 is a third answer. The question was answered two releases
ago and no seat found it.

**§4.7's precedence clause does not conflict.** It reads: *"For connection-handshake failures this table
is authoritative … §4.2, §5.2a and §6.12 are cross-references and MUST NOT be read as independent
registries."* A non-connect EXECUTE is **not a connection-handshake failure** — every row of §4.7 is
about a connect operation — so §4.7 does not cover this input and §4.2 is not competing with it.

**Why three seats missed it, and it is structural.** The implementer is in the connect handler. They look
at §4.7, the table that declares itself the MUST-emit contract for that surface. §4.7 **has no row for
this input and no note saying it is elsewhere.** §4.2 states the rule in the vocabulary of
*authentication* and *pre-authorization*; nothing in it says *"connection not yet established."* This is
**L23's second shape** — the home that states the rule is not the home the reader is in — and the remedy
is **L23's fourth shape**: a restatement that names its authority, so the next reader finds it.

The remedy is therefore a **note, not a row**. Adding a row would classify this input as a
connection-handshake failure, which it is not, and would put §4.7 and §4.2 into exactly the
two-registries state the precedence clause forbids.

## §5 `EXTENSION-TYPE` declares no error codes at all — filed, not folded

`EXTENSION-TYPE` §8.3 / §8.4 declare `system/type/validate-request` and `system/type/validate-result`
and the §7 analysis operations. Grepped for `invalid_request`, `invalid_params`, `bad_request`,
`Errors:` and `error code`: **zero hits in the whole document.** rust has built these handlers and minted
`bad_request` at 8 sites; go and py have not built them, so there is no measured divergence yet — only a
latent one, and the shape is the one three seats just demonstrated.

**Two halves, and only one is forced.** rust's fix is forced **today** by §4.7's clause regardless of
TYPE's silence — `bad_request` → `invalid_request`. The **taxonomy** is design work: it needs the failure
set of each op enumerated before codes are assigned, and inventing codes without that enumeration is how
`bad_request` happened. Filed as `PROPOSAL-TYPE-OPERATION-ERROR-TAXONOMY`, owned by arch, with the
derivation started — **not** left as a bare blocker (**L13** fourth axis).

## §6 What this does not change

- **§9.2's closed enum stays exactly as it is.** No unwrapped-surface behaviour moves.
- **No connect-path row moves.** §4.7's table is untouched; one note is added below it.
- **`invalid_params` stays sanctioned** where a named param is present and malformed.
- **No new obligation is created by D1.** It restates a rule landed at 0.8.1 in the table the reader is
  actually in. The conformance gap it exposes predates this proposal by two releases.

## §7 Delta table

| # | Document | Section | Change |
|---|---|---|---|
| **D1** | `entity-core-protocol/specs/ENTITY-CORE-PROTOCOL.md` | §4.7, after the pre-hello `authenticate` note | Add the pre-establishment non-connect EXECUTE note: not in this table; §4.2 bullet 3 + §5.2a govern → **401 `authentication_failed`**. Name `connection_required` and `handshake_failed` non-conformant, on the `incompatible_key_type` / `invalid_signature` precedent. |
| **D2** | `entity-core-protocol/specs/ENTITY-CORE-PROTOCOL.md` | line 3 | **Version** 0.8.2.4 → **0.8.2.5** |
| **D3** | `specs/extensions/EXTENSION-SIGNALING.md` | §4.3 | `bad_request` → `invalid_request` in the wrapped-surface error list; name the surface split. |
| **D4** | `specs/extensions/EXTENSION-SIGNALING.md` | §9.2 | Scope note: this enum is the **unwrapped** `error: tstr` value set, not EXECUTE codes; the wrapped surface uses §4.7's. Names §4.7 as the authority for the wrapped side (**L23** fourth shape). |
| **D5** | `specs/extensions/EXTENSION-SIGNALING.md` | header | **Version** 1.1 → **1.2** + fold note |

## §8 Cohort impact

| Seat | Owed | Status |
|---|---|---|
| `entity-core-go` | signaling `bad_request` → `invalid_request` (1 prod + 2 test sites); `execute.go:159` `403 connection_required` → **401 `authentication_failed`**; tighten `execute_before_established_refused` to assert the pair | routed |
| `entity-core-rust` | pre-establishment `400 handshake_failed` → **401 `authentication_failed`**; type-system `bad_request` → `invalid_request` (8 sites) | routed |
| `entity-core-py` | signaling `CODE_BAD_REQUEST` → `invalid_request`; pre-establishment `403 capability_denied` → **401 `authentication_failed`** | routed |
| `entity-core-keystone` | vendor 0.8.2.5; the generated peers' pre-establishment answer is unmeasured on this axis | routed |
| `entity-browser-rust` | none — zero sites, searched | — |

**This is a flag day on nothing.** D1 changes what a peer **emits**, not what it **accepts** (L21's third
shape): a peer emitting 401 where a counterpart expects 403 breaks no connection, because no peer
branches on this code — it is a terminal refusal on an unestablished connection. Stated explicitly so
the seats can sequence freely.

## §9 Open items

None. Both halves are derived from landed normative text; neither waits on a measurement.
