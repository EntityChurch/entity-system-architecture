# PROPOSAL — the load-bearing TTL ceiling has no config site, and four seats built four

**Status:** IMPLEMENTED — folded at `EXTENSION-REGISTRY` **v1.16** (no separate bump, per D6). D1–D5 verified 2026-09-06: `PINNED KEYS: max_ttl` in the §4 schema, the config site at `[MUST, v1.16]`, the `0`-is-undeclared rule, the null-`ttl` `local_max` arm, and `REG-TTL-RESOLVER-CEILING-1`.
**Tier:** extensions — `EXTENSION-REGISTRY` §4, §6a.9.1, §11.1
**Answers:** `entity-core-py` **`SA-PY-15`** (filed 2026-08-17; **never entered arch's board** — zero
occurrences in this tree until this proposal) · relayed in `entity-core-go`'s consolidated packet as GAP 2
**Read at:** arch `84cf677` · py `7a71fd4` · rust `39082d8` · go `921accc` · browser-rust `4583a35`

---

## §0 Summary

§6a.9.1 makes the **resolver-side** TTL ceiling a `[MUST when present]` and the spec itself calls it
*"the actual security property … the load-bearing one."* **It names no field to declare it in.**
§4's `resolver-config` schema has `backend_kind`, `backend_id`, `priority`,
`accepted_trust_anchors`, `hints` — and no ceiling.

So every seat invented a site. **This is the defect class py named, in its purest form: a `[MUST when
present]` with no declared config site is a rule two conformant peers cannot both implement.**

**Direct-read of every seat that implements it — four seats, three sites:**

| Seat | Tier | Site | Durable? |
|---|---|---|---|
| `entity-core-py` | engine | `resolver_chain[].hints.max_ttl` (`_resolver_ceiling`) | **yes** — config, read at resolution |
| `entity-core-rust` | engine | `resolver_chain[].hints.max_ttl` (`resolver_max_ttl_from_hints`) | **yes** — config, read at resolution |
| `entity-core-go` | engine | `WithLocalMaxTTL(ms)` — a **Go builder option** | **no** — process-lifetime, set in code |
| `entity-browser-rust` | **app** | `name_resolver_max_ttl_ms` in the deployment config | yes, but a **fourth** key nobody counted |

An operator handed a `system/registry/resolver-config` artifact therefore sets this ceiling on
**zero** implementations interoperably.

### §0.1 The correction to the packet that carried this to us

go's packet reported *"three impls, three homes — rust: a third location."* **rust and py are the
same location**, byte-for-byte the same key (`hints.max_ttl`), reached by the same argument. go had
not opened rust's tree and said so (*"rust per its report"*). The true picture is **better** than
reported — two of three engines already converged — and also **worse**, because a fourth seat in a
tier nobody searched carries a fourth key. Both halves matter: the convergence is what makes this
ratifiable rather than a design question, and the fourth key is what makes it urgent.

**That fourth key is L16 paying for itself twice in two days.** The app tier was not searched
because the resolver ceiling reads as an engine concern. It is not.

---

## §1 Why `hints.max_ttl` and not a new declared field

**py and rust independently chose `hints`, for the same stated reason, and it is the right one.**
Both wrote the argument down in source:

> §4's `resolver-config` schema declares no ceiling field and the ruling names none… `hints` is
> declared `<opaque object | null>` / backend-specific config and already carries `neg_ttl`, so this
> needs no new field on a content-addressed type. Declaring one would move
> `system/registry/resolver-config`'s type hash for a name no other seat would read.
> — `entity-core-py`, `_resolver_ceiling`

A declared top-level field moves the type hash of a content-addressed entity every deployment holds.
`hints` is the slot that exists precisely so backend-scoped config costs no type change, and it is
already load-bearing for `neg_ttl`. **Ratify the convergence; do not invent a third design.**

**Per-chain-entry is also the correct granularity, not incidental.** The ceiling bounds how long
*this* resolver will honor *this* backend's answers. A registry you barely trust and a registry you
operate do not deserve one number, and a single peer-global ceiling could not express that.

## §2 Why the value must be durable config, which is where go is non-conformant

rust wrote the security argument, and it decides go's builder option:

> A ceiling read at fetch time applies on a cold boot and silently does not on a warm one, and a
> security control present on one boot path and absent on the other is worse than absent: it tests
> green on whichever path the test happens to take.

`WithLocalMaxTTL` is set in Go code at construction. An operator editing `resolver-config` cannot
reach it; a deployment that forgets the option gets **no ceiling** and looks identical to one that
declared none. That is the failure mode the MUST exists to prevent.

## §3 `0` is dropped, and all four seats already agree

Every seat independently refuses to honor a literal `0`, with the same diagnostic reasoning
(rust: `resolver_max_ttl_from_hints`; py: `_resolver_ceiling`; browser-rust:
`the_resolver_ceiling_is_read_from_the_deployment_and_zero_is_dropped`):

> Honored literally it expires every binding instantly and the operator sees *"no binding for this
> name"* — indistinguishable from a bad signature or a revocation, which is the worst possible
> diagnostic for a value that is almost certainly a typo or an unset field serialized as zero.

**Four independent implementations reached an identical rule that appears nowhere in the spec.**
That is exactly the cross-impl-observable surface §11's determinism bias says to pin, and it is
free — nobody has to change anything.

## §4 The sticky-binding arm, where rust is ahead of the text

rust handles a ceiling against a binding carrying **no** `ttl` (a pin, a local-name) by returning
`Some(local_max)` — an absent lifetime becomes the declared maximum. That is right: the ceiling is a
bound on *how long a value may be honored*, so applying it only to bindings that already have a
bound leaves the unbounded case unbounded, which inverts the control. **The spec says nothing**, and
`min(binding.ttl, local_max)` read literally has no arm for `null`. Pin rust's reading.

*(`peer-issued` never reaches this arm — §6a.4 requires a non-null `ttl` before a result surfaces.
It is reachable through `local-name` and `pinned`.)*

---

## §5 Deltas

| # | File | Section | Change |
|---|---|---|---|
| D1 | `EXTENSION-REGISTRY.md` | §4 schema | Document `hints.max_ttl` as the **pinned key** for the resolver-side ceiling, beside the existing opaque-`hints` declaration |
| D2 | `EXTENSION-REGISTRY.md` | §6a.9.1 resolver-side clause | Name the config site `[MUST, v1.16]`; require it be **durable config read at resolution**, not a process-lifetime setting; state the cold-boot/warm-boot argument |
| D3 | `EXTENSION-REGISTRY.md` | §6a.9.1 | `0` MUST be treated as **undeclared**, not as an instant-expiry ceiling |
| D4 | `EXTENSION-REGISTRY.md` | §6a.9.1 | The **null-`ttl` arm**: a ceiling applied to a binding with no `ttl` yields `local_max` |
| D5 | `EXTENSION-REGISTRY.md` | §11.1 | **`REG-TTL-RESOLVER-CEILING-1`** — the resolver-side vector the spec has never had for its own load-bearing property |
| D6 | `EXTENSION-REGISTRY.md` | header | **No further bump** — this folds into `1.16` alongside `PROPOSAL-REGISTRY-NAME-LEGALITY-INPUT-DOMAINS`, and every clause above is tagged `[v1.16]` |

**Deliberately NOT done:** no top-level `local_max` field on `resolver-config`. See §1 — it moves a
content-addressed type hash to restate what `hints` already carries.

---

## §6 Cohort impact — routed per seat, not filed in a table

| Seat | Owed |
|---|---|
| `entity-core-py` | **Filing seat (SA-PY-15).** Already conformant — `hints.max_ttl`, `0` dropped. **Their site is the ratified one.** Owed: confirm, and close SA-PY-15 |
| `entity-core-rust` | Already conformant on all four points, and D4 ratifies their null-`ttl` arm. Owed: confirm |
| `entity-core-go` | **Non-conformant on D2.** Move the ceiling off `WithLocalMaxTTL` and read `resolver_chain[].hints.max_ttl`; add the `0`-dropped and null-`ttl` arms. The builder option MAY stay as a seed for the entity (§6a.9.1's store-first rule), never as a parallel source at resolution |
| `entity-browser-rust` | **A fourth key.** `name_resolver_max_ttl_ms` is a *deployment* input, which is legitimate — but it MUST land in the chain entry's `hints.max_ttl`, or a browser and an engine reading the same `resolver-config` disagree. Owed: read, and say whether the deployment key maps onto the ratified site |
| `entity-workbench-go` | Read-and-report: no ceiling found in tree at `2de39b0` |
| all | `REG-TTL-RESOLVER-CEILING-1` is a new vector — unimplemented everywhere |
