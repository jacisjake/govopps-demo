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
Gate attributes: `held` | `not_held` | `unknown`. Silence is never `not_held`.

## Working ticket vs freeze

- Interview **mutates** one working ticket.
- On score/search, copy it to an **immutable freeze**. That freeze is what retrieve and eval read.
- Re-score replaces the freeze.
- Turns are not individually versioned.

## Compiler (hybrid)

qwen may **propose** `query` expansions and a `search_plan`. A rules compiler **accepts or rewrites** before any HTTP call.

| Source | Who writes the plan |
|---|---|
| SAM listings / opportunities | Rules only (ncode, ptype, state, dates) |
| SBIR.gov | Rules only (agency, phase, keywords from query) |
| Grants.gov search2 | qwen proposes cats/elig/keywords; compiler drops unknown keys |
| USAspending | Rules from NAICS / keywords / award type |
| Utah table | qwen proposes themes; compiler matches known theme enum |

Forbidden in any plan: URLs, tiers, eligibility assertions, invented recipient names.

If a proposed relevance facet would collapse the retrieved set, the compiler does not apply it silently — it surfaces a tradeoff turn.

## Model may / may not

| May | May not |
|---|---|
| Extract profile from evidence | Call source APIs |
| Propose query + search_plan | Emit a tier or match % |
| Ask one question that changes the ticket | Infer set-aside / clearance from a name |
| Voice McKenna copy **after** scoring, from the record | Invent a URL or a recipient |

## Eval

Five brief cases are fixtures: raw input + expected ticket + expected map shape.

- Default: recorded HTTP fixtures. Deterministic.
- `--live`: hit real APIs. Same assertions.
- Failures: missing required facet, forbidden inference, source not queried, empty history block, URL not in retrieve set.

A pretty card with a wrong ticket is a fail. Eval does not grade McKenna’s prose.
