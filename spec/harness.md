# GovOpps — Harness

The harness turns conversational text and artifacts into **deterministic search features**. It is not semantic search over opportunities.

## Facet ticket

One JSON object. If a field is not on the ticket, search may not use it. If a field is on the ticket, eval can assert it.

```
ticket
  profile     who they are (intake schema; provenance on every field)
  query       translation: keywords_broad/precise, naics, agencies,
              mechanisms, geography, award_band, must_verify
  search_plan per-source parameters (compiled; see below)
```

Provenance: `stated` | `inferred` | `unknown`.
Gate attributes: `held` | `confirmed-held` | `not_held` | `skipped` | `unknown`. Silence is never `not_held`. `skip` / `blank` / `idk` on an open certs slot records `skipped` and stops re-asking.

## Working ticket vs freeze

- Interview **mutates** one working ticket.
- On score/search, copy it to an **immutable freeze**. That freeze is what retrieve and eval read.
- Re-score replaces the freeze.
- Turns are not individually versioned.

## Compiler (hybrid)
qwen may **propose** `query` expansions and a `search_plan`. A rules compiler **accepts or rewrites** before any HTTP call. The compiler does not currently refuse a plan that collapses the retrieved set — it compiles what it can and retrieve returns what it finds.

| Source | Who writes the plan |
|---|---|
| SAM listings / opportunities | Rules only (ncode, ptype, state, dates) |
| SBIR.gov | Rules only (agency, phase, keywords from query) |
| Grants.gov search2 | qwen proposes cats/elig/keywords; compiler drops unknown keys |
| USAspending | Rules from NAICS / keywords / award type |
| Utah table | qwen proposes themes; compiler matches known theme enum. No themes → no Utah rows. |

Forbidden in any plan: URLs, tiers, eligibility assertions, invented recipient names.

After the first score, later model turns also receive a **hunt digest**: `served_count`, `honest_no_match`, and the top five non-Poor cards (`agency`, `award_name`, `tier`).

## Model may / may not

| May | May not |
|---|---|
| Extract profile from evidence | Call source APIs |
| Propose query + search_plan | Emit a tier or match % |
| Ask one open question that changes the ticket | Infer set-aside / clearance from a name |
| Voice McKenna copy **after** scoring, from the record | Invent a URL or a recipient |

The system prompt is a short operating brief, not the full intake spec. `next_question` from the model is preferred; it is dropped only if it is an exact repeat, re-asks a recorded/skipped 8(a)/SDVOSB, or re-asks location after `profile.state` is set. Empty / unparseable model output falls back to a slot question.

## Eval

Five brief cases are fixtures: raw input + expected ticket + expected map shape.

- Default: recorded HTTP fixtures. Deterministic.
- `--live`: hit real APIs. Same assertions.
- Failures: missing required facet, forbidden inference, source not queried, empty history block, URL not in retrieve set.

A pretty card with a wrong ticket is a fail. Eval does not grade McKenna’s prose.
