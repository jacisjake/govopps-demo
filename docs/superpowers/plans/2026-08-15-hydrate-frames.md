# Hydrate Frame Components Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Six frames become shells; `score.components[]` hydrates header, cards, no-match, and studio. No baked awards in App.

**Architecture:** Presentational components take score-record props. `Manifest` mounts known types. `App` owns session step: Landing → Working → Map | NoMatch → Studio. `createSession` + `intake` + `score` (+ `accept`) are the only network calls.

**Tech Stack:** React 19, Vitest + Testing Library, existing FastAPI `/govopps/api`.

**Spec:** `docs/superpowers/specs/2026-08-15-clickable-frames-design.md`

## Global Constraints

- Unknown `type` → skip. Poor is never a card.
- Strong CTA: Begin Application. Maybe: Go. Weak: none.
- Maybe SWOT is S+W only.
- Missing McKenna prop → omit bubble. No Cyberdriven/Rallytime fallback.
- App default route contains no award names.
- Offline retrieve empty → honest NoMatch is valid.
- Visual language: existing indigo/white frame CSS class names.

---

### Task 1: Score-shaped Manifest components

**Files:**
- Create: `web/src/components/Header.tsx`
- Create: `web/src/components/Card.tsx`
- Create: `web/src/components/NoMatchBanner.tsx`
- Create: `web/src/components/Tray.tsx`
- Modify: `web/src/registry.tsx`
- Test: `web/src/registry.test.tsx`

**Interfaces:**
- Consumes: `Component = { type: string; props: Record<string, unknown> }`
- Produces: `Manifest({ components, onAccept?: (sourceId: string) => void, selectedId?: string, onSelect?: (sourceId: string) => void })`

- [ ] **Step 1: Failing tests** — extend `registry.test.tsx`:
  - `card.strong` with `{award_name, agency, match_pct: 92, amount_max: 1800000, cta: "Begin Application", source_id: "af"}` shows award, agency, 92, Begin Application.
  - `card.weak` with `cta: "none"` has no button.
  - `portfolio_header` with `{served_count: 7, funding_in_reach: {value: 4200000, label: "top 3 combined"}, federal_count: 6, utah_count: 1}` shows 7 and $4,200,000.
  - `invented.layout` skipped.
  - `no_match` shows /no strong match/i.

- [ ] **Step 2: Run** `cd web && npx vitest run src/registry.test.tsx` — FAIL until Header/Card render those fields.

- [ ] **Step 3: Implement Header/Card/NoMatchBanner/Tray; point REGISTRY at them.** Card uses `match_pct`, `agency`, `award_name`, `amount_max`, `timing.days_remaining`, `timing.close_date`, `timing.status`, `cta`, `source_id`, `tier`. Money via `Intl.NumberFormat`.

- [ ] **Step 4: PASS vitest registry tests.**

- [ ] **Step 5: Commit** `feat: render score components from props`

---

### Task 2: Optional score body + accept client

**Files:**
- Modify: `web/src/api.ts`
- Test: `web/src/api.test.ts`

**Interfaces:**
- `score(id, body?: { opportunities?: unknown[]; today?: string })`
- `accept(id, sourceId: string): Promise<{ workspace: Record<string, unknown> }>`

- [ ] **Step 1: Test** `score("ses_1", {})` POSTs `/govopps/api/sessions/ses_1/score` with `{}`. `accept("ses_1", "af")` POSTs `.../accept` with `{source_id:"af"}`.

- [ ] **Step 2: Run** `npx vitest run src/api.test.ts` — FAIL (score requires opportunities; no accept).

- [ ] **Step 3: Implement.**

- [ ] **Step 4: PASS.**

- [ ] **Step 5: Commit** `feat: optional score body and accept client`

---

### Task 3: App session flow

**Files:**
- Modify: `web/src/App.tsx`
- Modify: `web/src/screens/Landing.tsx` — textarea + submit
- Modify: `web/src/screens/Interview.tsx` — Working shell (searching), no hardcoded cards
- Modify: `web/src/screens/Map.tsx` — `<Manifest />` in results; tray from selected card props
- Modify: `web/src/screens/NoMatch.tsx` — `<Manifest />` for no_match + weak cards
- Modify: `web/src/screens/Confirm.tsx` — unused or fold into Working; do not mount from tabs
- Modify: `web/src/screens/Studio.tsx` — checklist from workspace
- Modify: `web/src/styles.css` — drop tab bar
- Test: `web/src/flow.test.tsx`

**Interfaces:**
- App steps: `landing | working | map | nomatch | studio`
- Landing `onSubmit(text: string)`
- After score: if any `type === "no_match"` → `nomatch`, else → `map`

- [ ] **Step 1: flow.test.tsx**
  - render App: heading Let's find some money; queryByText SBIR Phase II is null; no tab named Confirm.
  - mock fetch: sessions → {id}, intake → {}, score → header + card.strong AFWERX. Submit "we build sensors". Wait for Here's what we found and AFWERX. Landing pitch example may remain; award names only after score.
  - mock score `{components:[header, no_match]}`: submit → No strong match; no AFWERX.

- [ ] **Step 2: `npx vitest run src/flow.test.tsx`** FAIL (tabs still there; hardcoded SBIR).

- [ ] **Step 3: Wire App + shells. Kill tab bar.** Composer after score is no-op.

- [ ] **Step 4: PASS flow + registry + api tests.**

- [ ] **Step 5: Commit** `feat: hydrate frames from session score`

---

### Task 4: Accept → Studio

**Files:**
- Modify: `web/src/App.tsx`, `web/src/screens/Studio.tsx`, `web/src/screens/Map.tsx`
- Test: `web/src/flow.test.tsx`

- [ ] **Step 1: Test** click Begin Application → accept called with source_id; Studio shows checklist label from workspace; crumb returns to Map.

- [ ] **Step 2: FAIL.**

- [ ] **Step 3: Wire onAccept.**

- [ ] **Step 4: PASS.**

- [ ] **Step 5: Commit** `feat: accept opportunity into studio shell`

---

### Task 5: Build, verify, deploy Blanco

- [ ] `cd web && npm test && npm run build`
- [ ] Browser: `/govopps/` landing has no award names; submit any pitch; Working then Map or NoMatch from live API.
- [ ] Rsync ui-deploy (exclude .git .venv node_modules) to `~/apps/govopps/`; restart uvicorn 8787. No Caddy edit.
