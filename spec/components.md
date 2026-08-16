# GovOpps — Component registry

The UI does not invent layouts. The API returns `components: [{type, props}]`. The client mounts known types.

Voiced McKenna strings, when present, are **props** produced after scoring from `rag_registry.md`. Numbers, labels, and CTAs stay plain.

## Catalog (v1)

| type | When | Props (min) |
|---|---|---|
| `portfolio_header` | always after score | served_count, funding_in_reach, honest_no_match, agency_counts, closing_90, federal_count, utah_count |
| `card.strong` | served Strong | score record + voice_record (opener, swot) |
| `card.maybe` | served Maybe | record + missing_block / voice |
| `card.weak` | served Weak | record + criteria_checklist / hedge_copy |
| `no_match` | honest_no_match | no_match_message; surviving Weak cards still listed |
| `studio.checklist` | accepted workspace | items[] |
| `studio.narrative` | shelf | item_id, draft |
| `studio.evidence` | shelf | item_id, facts[] |
| `studio.deadline` | shelf | close_date, days_remaining, status |
Unknown `type` → skip, do not crash. Do not guess a card.

## Rules carried from the registry

- Poor / non-match is never a card type.
- Strong CTA: Begin Application. Maybe: Go. Weak: none.
- Maybe SWOT is diminished (S+W only).
- All URLs must have been retrieved this session.
