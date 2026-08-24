# docs/research/ + docs/proposals/ — the authoring workspace

Post-split, **this repo is self-authoring**: pre-spec design work happens here, not in the
frozen the pre-split monorepo monorepo (archived at the v0.8.0 cut). This mirrors the old
`v7.0-core-revision/{explorations,reviews,proposals}/` workspace, lightly re-homed.

## The lifecycle (how work moves)

```
exploration / analysis  →  proposal (DRAFT)  →  review / absorption  →  ratify  →  fold into spec
   docs/research/            docs/proposals/       docs/research/         (edit the      specs/…
   explorations/                                   reviews/               spec header)
```

1. **Exploration** (`docs/research/explorations/EXPLORATION-<TOPIC>.md`) — research + analysis
   of a design space. No commitment; maps the problem, prior art, options, open questions.
   `ANALYSIS-*` / `CRITIQUE-*` are the same phase in a narrower/adversarial voice.
2. **Proposal** (`docs/proposals/PROPOSAL-<TOPIC>.md`) — a concrete DRAFT design with a
   `**Status:** DRAFT` header. Proposals are their own thing (not under `research/`) because
   they are the ratifiable unit.
3. **Review** (`docs/research/reviews/`) — cross-impl / cohort feedback absorbed against a
   proposal (`ABSORPTION-*`, `ARCH-RESPONSE-*`, `ARCH-CLOSEOUT-*`).
4. **Ratify + fold** — the spec edit is the second half (`AGENTS.md`: "ratified ≠ folded").
   A landed proposal moves to a `proposals/implemented/` (or is marked implemented) and the
   spec header becomes source of truth.

## Relationship to the rest of the tree

- `specs/` + `guides/` — the **folded, published** normative surface (what a mirror consumer
  reads). Edited only at fold.
- `docs/status/` — dated status / handoffs / pull-in maps (ephemeral, per AGENTS-STANDARD).
- `docs/research/` + `docs/proposals/` — **this workspace** (pre-fold; the design record).

## Notes

- **Naming:** `EXPLORATION-<TOPIC>.md`, `PROPOSAL-<TOPIC>.md`, `ANALYSIS-*`, `ABSORPTION-*`.
  (The old repo stamped an internal `cgid-NN-NNN` clock; we date-stamp in-doc instead for now.)
- The archived pre-split workspace (60 explorations / 189 reviews / 225 proposals, incl. DRAFT
  `PROPOSAL-EXTENSION-WEBRTC-TRANSPORT` and `PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY`) lives
  in the internal legacy corpus — read-only reference;
  bring items forward selectively, don't edit there.
