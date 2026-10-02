# Jev source profiles — vectorizing & matching GovOpps opportunities

Question from `reports/State and territory funding sources.md`: across the state/territory
portals we could add, what does a real record actually contain, and how should that shape
our embedding + matching design?

## Method

`code/ts_source_profile.py` sends batches of 7 sample records to TypeSafe Jev (System One)
and asks one narrow judgment per dimension per record:

| Primitive | Dimension | What it decides |
|---|---|---|
| Choice `mech` | instrument type | grant / coop agreement / procurement solicitation / award record / loan / tax incentive / equity / unclear |
| Choice `revq` | economic quality for a for-profit SB | doing-business (`profit_bearing_work`) > supported-operating (`cost_or_no_margin_funding`) > learning (`assistance_with_strings`) > `repayment_or_credit` > `equity` |
| Noul `sb` | small business can apply/bid | founder-relevance filter |
| Score `vec` (0–3) | semantic text available for embedding | none / label-only / brief purpose / rich description |
| Choice `timing` | lifecycle state | open / closed / forecasted / rolling-unknown |
| Noul `amt` | amount present in record | structured-dollar coverage |
| Noul `elig` | eligibility/applicant-type present | filter-facet coverage |

State per item = `source`, `agency`, `title`, 450-char `text`, raw field-name list, and a
`hints` string of status/date/amount/eligibility `name=value` pairs pulled from the record.
Threshold for "yes" columns below: p ≥ 0.6.

Samples (n = 56) span every metadata *archetype* found in the 56-entity report — federal
baseline plus the best machine-readable state/DC sources:

- `grants_gov` (search2 listings, our corpus) and `sbir` (topic rows, our corpus)
- CA Grants Portal via CKAN datastore API — richest state grants source
- DC PASS Solicitations via ArcGIS GeoServices (layer 19)
- TX ESBD server-rendered listings (procurement + grant tabs)
- NY awarded-procurement Socrata feed and WY bid-award Google Sheets (post-award/history tier)
- Utah curated table (`code/data/utah_opportunities.json`) — our hand-normalized pattern

Raw per-record answers: `tmp/ts_samples/profiles.json`. Repro:

```
python3 tmp/ts_samples/build_samples.py        # normalize samples
python3 -m code.ts_source_profile tmp/ts_samples/samples.json
```

## Profiles

| Source (archetype) | n | mech | vec (0–3) | revq verdict | SB-apply | amount | eligibility |
|---|---|---|---|---|---|---|---|
| Grants.gov search2 listings | 6 | grant, unclear | **0.88** | unclear ×6 | 0% | 0% | 0% |
| SBIR.gov topic rows | 6 | grant | 1.50 | unclear ×6 | 100% | 0% | 0% |
| CA Grants Portal (CKAN) | 8 | grant | **2.33** | unclear ×6, cost ×1, strings ×1 | 38% | 38% | **100%** |
| DC PASS Solicitations (ArcGIS) | 6 | procurement | 1.15 | **profit-bearing ×5**, unclear ×1 | 100% | 0% | 0% |
| TX ESBD listings | 8 | procurement | **0.90** | **profit-bearing ×8** | 88% | 0% | 0% |
| NY awarded procurement (Socrata) | 8 | **award_record** | 1.02 | profit-bearing ×8 (history) | 88% | 100% | 25% |
| WY bid-award Sheets | 6 | **award_record** | 1.06 | profit-bearing ×5 (history) | 17% | 100% | 0% |
| Utah curated table (GovOpps) | 8 | grant | 2.06 | strings ×3, cost ×4, **prize-as-profit ×1** | 100% | 100% | 62% |

Two Jev behaviors worth noting for pipeline design:

1. **Instrument classification is near-deterministic** (`mech_p = 1.0` across all 56 records,
   no confusions between grants, solicitations, and award records). A single Choice question
   can auto-route a new source into the right tier.
2. **`timing` correctly tracked raw status vocabularies** (`Posted`, `active`, `OPEN`,
   `forecasted`, `Solicitation Status=Open`) without per-state mapping tables — Jev can serve
   as the normalizer for status strings we haven't seen yet, with a fallback rules layer.

## The revenue-quality finding: contracts answer, grants go silent

Ranked by founder value — *a path to doing business beats a path to learning how to do
business* — the `revq` ladder came out sharp on one side and empty on the other:

- **Contract-side records are decidable from the listing alone.** TX ESBD 8/8, DC 5/6, and
  the NY/WY award feeds all read `profit_bearing_work` at revq_p 0.70–0.96. A solicitation
  says "we will buy deliverables"; that is enough signal to rank it.
- **Grant-side records answer `unclear` at 75–100% (Grants.gov, SBIR, CA).** The economics of
  a grant (no fee/profit line, reimbursement cadence, cost share) are *program policy, not
  record content*. The listing never states them, and Jev correctly refuses to invent them.
  revq for assistance money must be **joined from the program-level (curated) row** — the
  exact Utah pattern, where every record resolved off `unclear` and one went further: the DNR
  water-loss **challenge prize** is filed `mechanism: grant` upstream but Jev read
  `profit_bearing_work` — recognizing that a prize won by shipping a product is doing
  business, not learning. Trust the judgment over the source label; consider a
  `prize_competition` option in revq to make that tier explicit.

## Absorption: the ladder is portfolio-relative, not absolute

`revq` ranks a single opportunity's economics. Value of the *next* opportunity depends on
what the founder already carries:

- **Assistance money has a capacity ceiling.** Each additional grant stacks roughly fixed
  overhead on a small team — reporting, indirect-rate negotiation, reimbursement cash-flow
  bridging, audit triggers — so marginal value decays with the portfolio. For most early
  companies the 3rd–5th concurrent grant is worth less than the 1st.
- **Profitable work scales with delivery capacity and compounds.** Contract revenue grows
  with the ability to perform (staff, subcontracting — DC PASS even carries
  `SUBCONTRACTING_REQ` per solicitation) and each performed contract becomes past
  performance that wins the next one. Its marginal value rises with success, unlike grants.
- **Consequence for ranking:** the primary axis is founder absorption state, not instrument
  quality. Keep the raw `revq` judgments reusable and put decay/scale logic in code —
  capital value decays per concurrent award of the type; work value scales with capacity and
  with past-performance gaps in the founder's NAICS/PSC profile. The interview ticket needs
  two additions for this: active-awards-by-mechanism count and a delivery-capacity signal.
- **Target mix, not one flat ranking.** Caps are per mechanism: the realistic ask is
  *one well-chosen grant plus N contracts* — the grant slot should buy the riskiest learning
  (best strategic fit, ideally funding work that de-risks the contract pipeline), while the
  contract slots scale until delivery capacity binds. So scoring should rank *within*
  mechanism and assemble a portfolio: a grant slot (0–2), revenue slots (headroom under
  capacity), history/capital_debt as supporting context. Founders who say "I just want
  revenue" collapse the grant slot; founders pre-revenue fill it first. The ticket should
  capture desired mix, or infer it from current holdings.
- **The grant slot exists to feed the contract slots.** The typical shape is one R&D-ish
  grant funding capability that makes contracts deliverable — "win the NIH/NSF Phase I,
  now you can bid the three VHA/DoD solicitations that required the evidence it produces."
  So grant value is partly *optionality*: score each grant card by how many of the founder's
  open contract candidates it de-risks (closes a past-performance or capability gap on the
  same theme/NAICS/PSC), and present the result as a paired plan — grant card on top,
  "contracts this unlocks" beneath — not two independent lists. The join is structured
  (theme + NAICS/PSC codes + agency buyer), so pairing is code, not inference; Jev's theme
  classifications from `classify_corpus.py` are already the shared key on both sides.

## What this says about vectoring

- **The sparse-listing problem is real and cross-cutting.** The listing rows that most portals
  expose (Grants.gov search2, TX ESBD, DC PASS) score vec ≈ 1: title + codes only. Embedding
  them as-is indexes labels, not meaning. These need a **detail-fetch step** before embedding
  (Grants.gov synopsis text, solicitation abstracts) or must be matched on codes, not vectors.
- **The two best vector sources are the two richest ones** — CA (Purpose + Description prose)
  and our Utah table (hand-written descriptions) at vec ≈ 2.1–2.3. Note the symmetry:
  the curated pattern *beats the raw federal feed*. A ~150-row editorial table per state gives
  a denser embedding corpus than scraping that state's live listings.
- **Procurement records are code-shaped, not prose-shaped.** DC/TX carry NIGP commodity codes
  and agency codes but almost no semantic text and no amounts/eligibility. Vector search adds
  little; matching should go **NIGP/PSC → NAICS bridge + hard filters** (status, due window,
  citywide/set-aside flags), with the title embedding only as a weak recall channel.
- **Award records must not enter the opportunity index.** NY/WY history rows look superficially
  like listings, but Jev reads them as `award_record` + `closed_or_past` + non-dilutive 0%.
  They belong in a separate history store that feeds the "who else got this money" intelligence
  block (USAspending pattern), keyed by agency + NIGP/program, never ranked as candidates.

## Matching-process recommendations

1. **Two indexes, one gate.** `opportunities` (grants/SBIR/solicitations) and `history`
   (awards/USAspending/state transparency feeds). Route new source rows with the `mech`
   Choice; anything answering `award_record` skips the candidate pool entirely.
2. **Embed what the record can actually support.** Compose embedding text per archetype:
   grants → title + purpose/description (+eligibility sentence if present); SBIR topics →
   title + topic description; procurement → title + NIGP description string; curated rows →
   full description. vec ≥ 2 sources embed directly; vec ≈ 1 sources embed only after
   detail-fetch enrichment.
3. **Filter facets come from metadata, not from vectors.** amount, timing, geography,
   set-aside, applicant-type → structured columns (our normalization schema in
   `spec/sources.md`). Jev's `amt`/`elig` coverage numbers per source tell us which columns
   will be null before we ingest: SBIR topics, DC, TX listings will *never* fill
   `amount_min/max` from the record itself — those must come from the program-level
   (curated) row, reinforcing the Utah pattern.
4. **Rank by revenue quality, not dilution presence.** The old `nondil` binary conflated two
   things founders care about separately: whether money is dilutive *and* whether it is
   business or training. `revq`'s ordered Choice replaces it and maps directly to
   `spec/sources.md`'s `mechanism` column — `profit_bearing_work` → `revenue`,
   `cost_or_no_margin_funding`/`assistance_with_strings` → `capital`, `repayment_or_credit` →
   `capital_debt`, `equity` → excluded. Weight `revq` per the absorption section: a founder
   with no funded work still ranks doing-business above learning, but a founder already at
   their assistance capacity ranks an incremental grant below almost anything — the two
   instruments are equals in aggregate, and the founder's portfolio decides which one is
   the better *next* card.
5. **`sb` still filters, but as a gate not a rank.** Threshold 0.6: it separates
   founder-addressable records from university/Nonprofit-only money. CA shows why — 100% carry
   an `ApplicantType` facet, but only 38% plausibly admit a for-profit small business; Jev
   agrees with the structured facet. Use `sb` to pre-filter, `revq` to rank what survives.
6. **Ranking composition.** In the rerank stage, reuse this question set on query↔candidate
   pairs (TypeSafe rerank cookbook): `sb` as a gate, `revq` as the primary value axis
   (doing-business > learning), `vec` as an evidence-strength prior, `timing` driving the
   staleness/deadline boost. Weights stay in code, so a "show me revenue contracts first"
   toggle costs no re-inference.
7. **Cost.** 56 records profiled in 8 requests (7 records × 7 questions per request,
   parallel). Per-source onboarding of a new state ≈ 1 request per 7 sample rows; ongoing
   classification at ingest matches the existing `code/classify_corpus.py` batch economics.

## Caveats

- n = 6–8 per source; TX rows were transcribed from the live listing (server-rendered HTML,
  no CSV endpoint probed); Grants.gov/SBIR rows come from our frozen corpus, several archived.
- `vec` scores measure text available *in the listing payload* (450-char cap), not text
  available behind detail pages/attachments.
- CA's low `sb` rate reflects the sampled programs (environment/arts flow-throughs to
  local agencies), not a judgment on the whole portal.
- `revq = unclear` dominance on Grants.gov/SBIR rows is partly a sample-size effect (n = 6),
  but the mechanism is real: assistance economics are not in listing payloads.
- Territory/state portals with no structured sample (everything else in the 56-entity report)
  are assumed to match one of the four archetypes above — validate by running this profiler
  on ~7 scraped rows from each before trusting it.
