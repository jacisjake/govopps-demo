# GovOpps — Architecture

Version 0.1 · 2026-08-15

## Product

GovOpps is a founder studio for federal (and Utah) opportunities. Match is a door, not the win. Accept opens a workspace; the end state is a submission packet, not a match %.

McKenna is the copilot. Jess-from-New-Girl voice, patriotically optimistic. She never guarantees eligibility. She never invents a URL.

Deployed at `goed.ut.gitsum.rest/govopps`. Headless JSON API. Web dashboard is client #1. Iframe embed and a mobile McKenna-chat client consume the same API later.

## Pipeline

```
founder input (URL / deck / pitch / chat)
        │
        ▼
  Harness runtime (qwen via LiteLLM)
  extracts / confirms → working ticket
        │
        │  freeze (immutable snapshot)
        ▼
  search_plan compiler
    SAM / SBIR …………… rules only
    Grants.gov / Utah … qwen proposes, rules accept
        │
        ▼
  Deterministic search
    Grants.gov · SAM listings+opps · SBIR.gov
    USAspending · Utah table (source: utah)
        │
        ▼
  score_pair + rank_portfolio   (code/scoring.py, schema v0.3)
        │
        ▼
  component manifest
        │
        ▼
  Opportunity Map  ──accept──►  Studio
                                 checklist
                                 narrative · evidence · deadline
```

The model never searches, never assigns a tier, never invents a URL. Scoring is off-model.

## Runtime shape (v1)

One process: FastAPI serves `/api/*` and the built Vite SPA. Caddy on `goed.ut.gitsum.rest` reverse-proxies `/govopps` (`root_path=/govopps`).

| Port / host | Role |
|---|---|
| this process | API + dashboard |
| `127.0.0.1:14100` LiteLLM | `qwen2.5-14b` — intake, interview, McKenna copy, Grants.gov/Utah plan propose |
| sqlite | sessions, freeze snapshots, Utah table, optional SAM/SBIR caches |

LLM is optional at the HTTP boundary: intake may accept a prebuilt `ticket` patch so eval and tests do not call qwen.

## Information architecture

- **Web dashboard** — award-quality map + studio. Default view after score is the Opportunity Map (count, funding-in-reach top-3, agencies, closing-in-90, Federal / Utah split). Studio opens on Accept.
- **Mobile (later)** — McKenna chat. Different IA. Writes the same session; dashboard reads it.
- **Embed (later)** — `/embed.js` injects an iframe of the dashboard.

## What v1 does not build

- Live SAM.gov submit
- React Native / Capacitor
- Scraping startup.utah.gov
- Vector retrieve (facets are the retrieve features)
- Next.js

## Status

| Piece | State |
|---|---|
| Scoring v0.3 | Implemented (`code/scoring.py`) |
| API + sessions | In progress |
| Harness / retrieve / dashboard / studio / deploy | Specified, not built |
