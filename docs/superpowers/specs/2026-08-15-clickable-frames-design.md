# GovOpps — Frame components, hydrated by score

2026-08-15 · Revised after review: Cyberdriven / Rallytime copy in the
six frames is **example data**, not product content. Build the visual
components. McKenna + `POST /api/sessions/{id}/score` hydrate them
from the current search results.

Visual language stays as drawn (indigo/white, 1440 chrome). Authority
for types and CTAs: `spec/components.md`. Authority for record fields:
`code/scoring.py` `score_pair` / `rank_portfolio`.

## Decision

The UI does not invent layouts or invent opportunities. It mounts
`components: [{type, props}]` from the score payload. Unknown `type`
is skipped. Poor is never a card.

## Shells vs components

**Shells** (chrome only — logo, McKenna rail, composer, user chip):

| Shell | When |
|---|---|
| Landing | no session / no intake |
| Working | intake done, score not back (empty map + “Waiting on a few basics” / “Searching live sources…”) |
| Map | score returned and `honest_no_match` is false |
| NoMatch | score returned and `honest_no_match` is true |
| Studio | workspace accepted |

**Components** (props only; no hardcoded awards):

| type | Renders |
|---|---|
| `portfolio_header` | stat band: served_count, funding_in_reach, agencies, closing_90, federal_count, utah_count |
| `card.strong` | match_pct, agency, award_name, amount_max, timing, CTA **Begin Application** |
| `card.maybe` | same + requirement_gap / missing; CTA **Go**; SWOT S+W only |
| `card.weak` | same; no CTA; “Long shot” |
| `no_match` | honest empty-map copy; surviving Weak cards still listed |
| `studio.checklist` | items[] by class |
| `studio.narrative` | item_id, draft |
| `studio.evidence` | item_id, facts[] |
| `studio.deadline` | close_date, days_remaining, status |

Detail tray is the selected card’s props, not a second data source.

McKenna bubbles are props (`opener`, turn `next_question`, rag strings).
If a prop is missing, the bubble is omitted — no fallback story.

## Data flow

1. Landing submit → `POST /api/sessions` → `POST …/intake` `{text}` → Working.
2. Working → `POST …/score` `{}` (retrieve uses server transport; offline
   default is empty). Manifest renders returned `components`.
3. Strong/Maybe CTA → `POST …/accept` `{source_id}` → Studio shell +
   studio.* components from the workspace.
4. Studio crumb → Map with the same score `components`.

No second canned Rallytime route. No-match is whatever score returns.

## Empty / loading

- No components yet: Working empty-map (“Your opportunity map builds here”).
- Score in flight: Interview search bar + skeletons; no fake cards.
- `components === [portfolio_header, no_match]`: NoMatch shell.
- Composer send after score is a no-op this pass (no `…/turns` UI).

## Files

- Add: `web/src/components/{Header,Card,NoMatch,Tray,StudioChecklist,StudioNarrative,StudioEvidence,StudioDeadline}.tsx` — presentational, props-only.
- Change: `web/src/registry.tsx` — map catalog types to those components.
- Change: `web/src/App.tsx` — session + step; no tab bar.
- Change: `web/src/screens/*.tsx` — shells only; children = `<Manifest />`.
- Change: `web/src/api.ts` — add `turns` / `accept` only if Studio is in this pass; otherwise Studio is a shell that lists workspace items from `accept` response.
- Test: `web/src/registry.test.tsx` extended — render Header/Card/NoMatch from **synthetic** score-shaped props (Cyberdriven numbers allowed as fixture, not as App defaults).
- Test: `web/src/flow.test.tsx` — Landing submit calls `createSession` + `intake` + `score` (mocked fetch); Map shows whatever mock `components` return; empty mock → NoMatch.
- Do not bake awards into `App.tsx` or screens.

## Out of scope

Cream/coral reskin, live `GOVOPPS_LIVE` requirement (offline empty map
is valid), parsing answers via `…/turns`, file upload, inventing
solicitation URLs, mobile.

## Proof

- Vitest: Manifest with Strong+Maybe+Weak+header matches frame fields
  (agency, award_name, CTA). Unknown type skipped. `no_match` without
  cards. App default route is Landing with no “SBIR Phase II” text.
- Browser: submit any pitch; page shows Working then Map or NoMatch
  from the live `/govopps/api` score. No Cyberdriven string unless the
  founder typed it.
