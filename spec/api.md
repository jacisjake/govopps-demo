# GovOpps — HTTP API

FastAPI, `root_path=/govopps`. Same origin as the dashboard. JSON only.

App factory: `create_app(db_path) -> FastAPI` (tests use a temp sqlite file).

## Resources

| Method | Path | Body | Result |
|---|---|---|---|
| POST | `/api/sessions` | — | `{id}` |
| GET | `/api/sessions/{id}` | — | session |
| POST | `/api/sessions/{id}/intake` | `{text?}`, `{url?}`, `{ticket?}` | session |
| POST | `/api/sessions/{id}/turns` | `{answer?}`, `{ticket_patch?}` | session |
| POST | `/api/sessions/{id}/score` | `{opportunities[], today?}` | `{freeze, portfolio, components}` |

Unknown `id` → 404.

## Session

```
id, created_at
raw_intake
ticket          # working, mutated
freeze          # null until first score; then immutable copy of ticket
turns[]
portfolio       # last rank_portfolio, or null
```

Intake stores raw text/url. If `ticket` is present, it is applied (passthrough — no LLM required). Turns append and may patch the working ticket. Score snapshots `ticket` → `freeze`, runs `score_pair` + `rank_portfolio`, persists portfolio.

## Components

Projection of the last portfolio. No voiced copy.

```
{ type: "portfolio_header", props: { served_count, funding_in_reach,
          honest_no_match, agency_counts, utah_count, federal_count } }
{ type: "card.strong" | "card.maybe" | "card.weak", props: <score record> }
{ type: "no_match", props: { ... } }          # only if honest_no_match
```

Studio types (`studio.checklist`, `studio.narrative`, …) are added when Accept exists; see `studio.md`.

## Later (not v1 HTTP)

`POST /api/sessions/{id}/accept` → workspace.  
Mobile chat and embed reuse these resources; they do not get a second API.
