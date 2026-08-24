# PROPOSAL — Availability Descriptor (an honest field primitive for partial/heterogeneous data)

**Status:** **IMPLEMENTED — FOLDED 2026-08-15 as `SPECIFICATION-FORMAT` §8.2a** (with vector
requirements at §8.5). Landed as an **authoring standard** rather than an extension spec, because it is a
reusable field shape with no operations and no wire mechanism of its own — `§8.2 New Types`' neighbourhood
is its home, and extensions reference it rather than each redefining it. **Unblocks
`PROPOSAL-SYSTEM-DEVICE`**, its first consumer. Authored 2026-07-12; it sat DRAFT for a month with nothing
blocking it.
**Scope:** a small, reusable **field shape** for values that may be absent, unsupported, withheld, or
unmeasured — so a consumer can always tell *why* a field has no value. No wire change (an ordinary ECF
map). **Lands first**; `PROPOSAL-SYSTEM-DEVICE` is its first consumer, but it is deliberately general.
**Research:** `docs/research/explorations/EXPLORATION-SUBSTRATE-INTROSPECTION-DEVICE-AND-STORAGE.md §4`,
`ANALYSIS-DEVICE-STANDARDS-ALIGNMENT-REDFISH-OTEL.md §2`.

---

## §1 Problem

Any surface that spans heterogeneous platforms or grant scopes has fields that are sometimes present
and sometimes not — and "absent" hides **four distinct causes** a consumer must distinguish:

- the platform **cannot** produce it (a browser peer has no NIC enumeration),
- the caller is **not permitted** to see it,
- it **hasn't been measured** yet,
- or it genuinely **is** some value.

Collapsing these to "field missing" or "zero" is the exact dishonesty our substrate floor forbids
("deliver-or-signal, never silently drop"). Every heterogeneous introspection surface will re-invent
this distinction unless we standardize it once.

## §2 The descriptor

An **availability descriptor** wraps an optional typed value `T` as an ECF map:

```
availability<T> := {
    state:     "value" | "unsupported" | "denied" | "unknown",   ; required
    value:     <T>,        ; present IFF state = "value"; absent otherwise
    fidelity:  "exact" | "coarse"   ; OPTIONAL; only meaningful when state = "value"
}
```

- `state` (snake_case key) carries a kebab enum value per `STYLE-NAMING-CONVENTIONS`.
- `value` is present **only** when `state = "value"`; consumers MUST NOT read `value` for any other
  state (an impl MUST omit it, not send null).
- `fidelity` distinguishes a measured value from a privacy-coarsened / bucketed one (§4) so a consumer
  never mistakes a bucket for a measurement.

## §3 The four states

| `state` | Meaning | Consumer reads it as |
|---|---|---|
| `value` | present and known (see `fidelity`) | use `value` |
| `unsupported` | this platform/build **cannot** produce this field | "not available here" — a permanent, structural no |
| `denied` | producible, but the caller's grant does **not** cover it | "not allowed" — may change with a broader grant |
| `unknown` | not measured / not yet probed (distinct from zero) | "no reading" — may become a value later |

The split is exhaustive and each state drives different consumer behavior (retry-with-grant for
`denied`; never-retry for `unsupported`; re-poll for `unknown`).

## §4 Fidelity

When a value is deliberately coarsened for privacy (bucketed, rounded, clamped — see
`PROPOSAL-SYSTEM-DEVICE §5`), `fidelity: "coarse"` MUST be set; an exact measurement is
`fidelity: "exact"` (or omitted, defaulting to `exact`). This lets a consumer render "≈8 GB"
(coarse) distinctly from "8 589 934 592 bytes" (exact) and never treats a bucket as precise.

## §5 Prior art (this is a synthesis, not an invention)

The descriptor is the **union of three established models**, each of which covers only one axis:

- **Support axis** — Godot `OS.has_feature(tag)`: a pre-flight "can this platform do it" query →
  our `unsupported`.
- **Permission axis** — W3C Permissions API `PermissionStatus.state ∈ {granted, denied, prompt}` →
  our `denied`.
- **Presence/state axis** — DMTF Redfish `Status.State ∈ {Enabled, Absent, UnavailableOffline, …}` →
  our `value` / `unsupported` / `unknown`. (Redfish also carries `Health`; we omit it — health/alerting
  is monitoring, out of scope for a read-only field primitive.)

No surveyed system unifies support + permission + measurement into one honest field; that union is the
contribution.

## §6 Naming

Per `STYLE-NAMING-CONVENTIONS` (normative): map keys `state`/`value`/`fidelity` are snake_case;
the enum string values (`value`, `unsupported`, `denied`, `unknown`, `exact`, `coarse`) are kebab (all
single tokens here). Entity type name for a standalone descriptor result, where one is needed:
`system/availability` (kebab path segments).

## §7 Where it's used

- **`system/device`** — every host field (first consumer).
- Reusable for any partial/heterogeneous surface: network reachability results, discovery outcomes, a
  `MAY` a peer can't honor, capability-probe results. Extensions SHOULD reuse this shape rather than
  inventing per-field "missing" conventions.

## §8 Non-goals

- Not a health/monitoring signal (no `Health` axis — Redfish's job, not ours).
- Not an error type — `denied`/`unsupported` are normal field states, not operation failures. (An
  operation-level failure still returns the ordinary coded error.)
- No new wire mechanism — it is an ordinary ECF map read/written like any entity field.

## §9 Conformance (when promoted)

- An impl that emits an availability-typed field MUST set `state`; MUST include `value` iff
  `state = "value"`; MUST NOT emit `value` for other states.
- MUST set `fidelity: "coarse"` when the value is coarsened.
- A reader MUST treat an unrecognized `state` as `unknown` (forward-compat: future states degrade
  safely), per the MUST-ignore-unknowns discipline.
- Cross-impl fixtures — pin **both** conditional-field rules, not only the four states (native-provider
  review, entity-core-go 2026-07-13 §6):
  - one vector **per `state`** (`value` / `unsupported` / `denied` / `unknown`);
  - a **`coarse`-fidelity** vector (`{state: value, fidelity: coarse}`) — pins the coarsen signal;
  - a **`value`-with-`fidelity`-omitted** vector — pins the **default-`exact`** rule (a reader MUST read an
    omitted `fidelity` as `exact`, never as absent/unknown);
  - the **"omit, not null"** rule exercised directly: a non-`value` state MUST serialize with `value` **absent**
    (a conditional `MarshalCBOR`, confirmed feasible native-side), never `value: null`.
