# GovOpps — Sources

Every retrieve uses the frozen ticket’s `search_plan`. Snapshots and caches are allowed (brief: 2–4 quality sources beat a thin everything). We take all four official sources plus a curated Utah table.

## Federal (mandatory)

| Source | Use | Auth | Notes |
|---|---|---|---|
| Grants.gov `search2` | Current / forecasted grants | none | qwen may propose categories; compiler validates |
| SAM.gov Assistance Listings | Program universe (grants, loans, other) | API key | `code/sam_snapshot.py` already pulls this |
| SAM.gov Opportunities v2 | Procurement | API key | Same snapshot script |
| SBIR.gov | Solicitations, topics, awards | public | Topics for retrieve; awards for history |
| USAspending.gov v2 | Who else got this money | none | **Required on every served card** |

History block (USAspending and/or SBIR awards): similar recipients, amounts, years, Utah count if any. Empty history is an eval fail for served cards, not a silent omit.

## Regional (v1)

Hand-normalized table, ~50–150 rows: GOED / state finance / university tech-transfer / municipal pilots / state SBIR match. **No scrape.**

Every Utah row carries `source: utah`. The map header splits Federal / Utah. Retrieve only loads Utah rows when `search_plan.regional.themes` is non-empty.

## Normalization

Each retrieved row, regardless of origin:

```
source, source_id, url, retrieved_at
agency, award_name, mechanism          # revenue | capital | capital_debt
amount_min, amount_max
timing.status, close_date
cost_share_pct
eligibility / set_aside / geography
declared requirements[]
domain text, naics[], psc[]
```

`retrieved_at` drives the scoring staleness gate (`max_data_age_days` = 7).
