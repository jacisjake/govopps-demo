# GovOpps — UI mockup set (Claude design phase)

2026-08-15 · Approved plan for the end-to-end mockups. Target: one `.pen` file,
six desktop frames (1440px). Content: Cyberdriven (hero path) + Rallytime
(honest no-match). Authority for behavior stays with `spec/` — this doc only
fixes what the mockups show.

## Visual language — "warm civic studio"

Revised 2026-08-15 after review: the original dark-navy direction read dreary.
Re-skinned to a light, warm studio mood (inspiration: cream-ground dashboard
with floating cards; mood only, colors our own).

- Ground: warm cream `#F3F0EA`; floating white cards, 14–20px radii, soft
  shadows, minimal borders.
- Ink: deep navy `#1F2A44` for text and display; navy `#2C3E6B` for stat
  numbers.
- Action: coral `#E4632E` (CTAs, active states) with honey-gold `#F2B33D` in
  the McKenna avatar gradient. Action color is reserved for actions.
- Type: serif display for headlines, clean sans for body/labels/numbers.
- Tier color is reserved for card status: green (Strong), amber (Maybe),
  neutral/slate (Weak); soft-filled pill chips. Red appears only in the
  suppression aside and the `not held` chip.
- McKenna voice only in persona copy. Numbers, labels, CTAs plain
  (`components.md`, `rag_registry.md`).

## Frames

### 1 · Landing — chat-first hero
- Kicker: `GOED · GovOpps`. Headline: "Tell McKenna about your company."
- Sub: "Paste a pitch, drop a deck, or share your site. She'll translate it
  into government language and map what's fundable."
- One large input = the page. Drop-zone affordance inside it (doc / link /
  pitch icons). Primary CTA: "See the map".
- Trust line: "Utah GOED · StartupState". No marketing sections.

### 2 · Confirm — accuracy gate (Cyberdriven)
- Founder message: the Cyberdriven elevator pitch (22-person Utah company, AI
  threat detection, $2M ARR, Series A, wants federal/defense customers).
- McKenna reply: extracted Company Profile card with provenance chips —
  `stated` (concepts, UT, 22 employees), `inferred` (small business, NAICS
  541512/541519, intent), `unknown` (set-asides, clearances, certs,
  match-funding).
- Confirm control: "Look right?" → per-field confirm/edit affordance.

### 3 · Interview — the hero beat
- Chat state, one question at a time. McKenna: a retrieved DoD contract is an
  8(a)/SDVOSB set-aside — "Is Cyberdriven 8(a) or SDVOSB certified, or
  veteran/minority-owned at 51%+?"
- Founder: "No, we're not."
- Profile rail flips `8a: unknown → not_held` (visually: chip state change).
- Question counter/quiet progress — sufficiency, not a form.

### 4 · Opportunity Map — payoff (Cyberdriven)
- `portfolio_header` stat band: served count, funding-in-reach (top-3 sum),
  agencies, closing-in-90, Federal/Utah split.
- Tier stack:
  - `card.strong` — DoD/AF SBIR Phase II, cyber. Spine: Agency/Award ·
    $/timing · SWOT scorecard. CTA **Begin Application**.
  - `card.maybe` — DHS/CISA program with Missing block (CMMC status,
    match-funding). CTA **Go**. SWOT diminished (S+W only).
  - `card.weak` — adjacent grant with criteria checklist. No CTA.
- McKenna aside (the visible RED beat): one DoD set-aside contract was
  suppressed because Cyberdriven confirmed it isn't 8(a) — honest, not hidden.
  No Poor card is ever rendered.

### 5 · Honest no-match — Rallytime
- `no_match` component: McKenna's message blames the difficulty of the system,
  never the founder; hopeful, not cynical.
- Surviving adjacent cards (weak, workforce/education-adjacent) listed below.

### 6 · Studio — accepted workspace
- Entered by Accept on the Strong card. Breadcrumb back to Map (header stats
  stay on the Map).
- Checklist grouped by class: registration → read → gap-closing → upgrade →
  prepare → engage. Statuses: open / in_progress / done / blocked.
- Tool shelf right rail: **Narrative** (draft on a prepare item), **Evidence**
  (facts on a gap item), **Deadline** (close_date, days_remaining; read-mostly).
- McKenna assists the open item. She never invents solicitation URLs.

## Out of scope for this pass

Mobile frames, embed, animation, and any implementation in `web/`.
