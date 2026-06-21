# SDK — Domain Roadmap (canonical, living)

**Target repo:** `entity-system-architecture` (the SDK sub-domain).
**What this domain is:** the **binding layer** between the protocol/extension substrate and the
languages that consume it. The SDK specs are cross-impl *conventions* over the type surface —
they pin operation names, result-carrier shapes, dispatch ergonomics, and host-language encoding
equivalence at the SDK boundary, so that Go/Rust/Python (and future) SDKs feel the same without
mandating one API. Not the protocol; the ergonomic contract above it.

> **Canonical & living.** Updated as the SDK surface tracks extension landings; the
> content/papers projects pull from it. Maturity (M0–M6) is defined in
> **`STATUS-RELEASE-SURFACE-AND-MATURITY-CANONICAL.md`**; this is the forward, stage/phase view.

**Owning workstream:** W1 (SDK/app). workbench-go owns the SDK
reference surface; the specs ratify the empirical baseline as a cross-impl target.

---

## Artifacts & maturity

| spec | version | hdr | maturity | scope |
|---|---|---|---|---|
| `SDK-OPERATIONS` | 1.10 | Draft | **M2** (working draft, tracks V7) | core SDK surface: put/get, dispatch, connection, handler registration |
| `SDK-EXTENSION-OPERATIONS` | 0.9 | Working draft | **M2** | per-extension SDK surface: Content closure (`EnsureClosure`), Continuation assembly, Subscription, Revision, Compute (expression builder + lowering toolkit) |
| `SDK-IDENTITY-INFRASTRUCTURE` | 0.5 | Draft | **M2** | identity-stack tooling: bundle wire-shape, Bootstrap-vs-Restore, role/identity SDK surface |
| **Guides** (7) | — | — | doc | `GUIDE-SDK-PATTERNS`, `GUIDE-IDENTITY-SDK`, `GUIDE-PERSISTENCE`, `GUIDE-SHELL-FRAMING`, `GUIDE-PEER-CONCERNS-AND-NAMESPACES`, `GUIDE-IMPL-DISCIPLINE`, `GUIDE-ENTITY-WORKBENCH-APP` |

The SDK specs sit at **M2** by design: they are *conventions*, not protocol — their gate is
"does the cross-impl SDK boundary agree," validated through workbench-go's reference surface and
the 3-impl SDK adapters, not through the keystone 15-peer protocol gate.

## Stages / phases

**Stage — Core SDK surface (DONE for v1).** `SDK-OPERATIONS` is pinned to the current V7
amendment line (result carrier + dispatch-surface equivalence, internal sub-dispatch
authorization, `included` preservation, concurrency paragraph, `put_cas` three-value
`expected_hash`). L1-default / L0-carve-out is the normative dispatch stance.

**Stage — Per-extension SDK alignment (TRACKING extension landings).**
`SDK-EXTENSION-OPERATIONS` absorbs each extension as it converges: Content materialization
(`EnsureClosure` / `AtPeer` / `Reassemble`), Continuation assembly modes + structural-transform
ops, Subscription `include_payload` + cross-peer mirror recipe, Revision diff/merge-config,
Compute expression builder (E7) + lowering toolkit (E7.1) + typed conveniences (E7.2). **Phase
discipline:** the SDK surface follows the extension, never leads it — an extension landing is
the trigger for its SDK section. DURABILITY is explicitly *not* normative SDK surface.

**Stage — Identity-stack SDK (CONVERGING).** `SDK-IDENTITY-INFRASTRUCTURE` ratifies the
Go-team empirical baseline (`ext/role/sdk/`, `ext/identity/sdk/`) as a cross-impl target. The v1
contract: IdentityBundle canonical wire shape (`system/identity/bundle/v1`), the
`BootstrapFromMaterials` vs `RestoreFromBundle` naming distinction (conflating them is
non-conformant), and the **security non-goal: bundles MUST NOT carry private key material**
(driven by the Rust v1 `keypair_pem` defect — v1 bundle bytes in the wild are compromised key
material, rotate). Layer-1 (cross-peer-deterministic cap verdict) vs Layer-2 (local policy)
separation is the framing SDKs SHOULD expose.

**Stage — Forward (post-release).** The `browse_*` SDK seam (the thin surface SITE/SPACES/REPOS
share) is W1 territory, pending L5 convention validation. GROUP v1.5 SDK rows are placeholder
pending the GROUP-V1.5 sweep.

## Notes for the content/papers project

- The framing: **the SDK is a convention, not an API mandate** — "your language, your idioms,
  but the boundary bytes and operation semantics agree." That's the cross-impl-without-lock-in
  story.
- These are working drafts (M2). Honest about it: the SDK surface stabilizes *behind* the
  extension landings it binds.
