# GovOpps — HTTP API

FastAPI, `root_path=/govopps`. Same origin as the dashboard. JSON only.

App factory: `create_app(db_path, static_dir=None) -> FastAPI` (tests use a temp sqlite file).

## Resources

| Method | Path | Body | Result |
|---|---|---|---|
| GET | `/api/groups` | — | `{groups: [{id, name}]}` |
| POST | `/api/sessions` | — | `{id}` |
| GET | `/api/sessions/{id}` | — | session |
| POST | `/api/sessions/{id}/intake` | `{text?}`, `{url?}`, `{ticket?}`, `{profile?}`, `{query?}` | session + interview state |
| POST | `/api/sessions/{id}/turns` | `{answer?}`, `{live?}`, `{ticket_patch?}` | session + interview state |
| POST | `/api/sessions/{id}/group` | `{id}` | seeded session (no score) |
| POST | `/api/sessions/{id}/score` | `{opportunities[]?}`, `{today?}` | `{freeze, portfolio, components, search}` |
| POST | `/api/sessions/{id}/accept` | `{source_id}` | `{workspace}` |
| POST | `/api/sessions/{id}/workspaces/{source_id}/items/{item_id}` | `{draft?}` / `{evidence?}` / `{deadline_status?}` | `{workspace}` |
Unknown `id` → 404. Unknown group → 404.

## Session

```
id, created_at
intake          # {text, url}
ticket          # working, mutated
freeze          # null until first score; then immutable copy of ticket
turns[]
last_question   # last McKenna line shown
portfolio       # last rank_portfolio, or null
workspaces      # {source_id: workspace} after accept
```

Interview state on intake / turns / group: `search_ready`, `next_question` (from `last_question` if set).

Free-text intake token-seeds a ticket, then calls qwen unless the client sent `ticket` / `profile` / `query` (passthrough — no LLM). `live: true` turns call qwen with the last six chat lines and a hunt digest from the last portfolio. Fail-soft keeps the rules ticket and a slot question.

Score snapshots `ticket` → `freeze`, compiles `search_plan`, retrieves unless `opportunities` is supplied, runs `score_pair` + `rank_portfolio`, persists portfolio. Thin tickets are allowed (no 409). `search.sources` is `[{id, count}]` for `grants_gov`, `sam_opps`, `sbir`, `usaspending`, `regional`.

The dashboard scores after the pitch, every send, and package edits.

## Components

Projection of the last portfolio. Card props include the score record plus registry voice (`voice_record`).

```
{ type: "portfolio_header", props: { served_count, funding_in_reach,
          honest_no_match, agency_counts, utah_count, federal_count,
          closing_90, tier_counts, revenue_options, capital_options } }
{ type: "card.strong" | "card.maybe" | "card.weak", props: <score record + voice> }
{ type: "no_match", props: { honest_no_match, served_count, no_match_message } }
```

Studio types (`studio.checklist`, `studio.narrative`, …) are added when Accept exists; see `studio.md`.

## Studio

`POST /accept` creates a workspace for one Strong or Maybe record. Weak/Poor → 409 `{detail: "tier cannot accept"}`. Missing opportunity or session → 404. Item writes set `draft`, append `evidence`, or set `deadline_status` in `{done, blocked, open}` without inventing `close_date`. GET session includes `workspaces`.

## Later (not v1 HTTP)

Mobile chat and embed reuse these resources; they do not get a second API.
