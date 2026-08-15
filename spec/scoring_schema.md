# GovOpps — Scoring Schema

Version 0.3 · the "Scoring" box in the schematic

Scoring turns a `(company profile, opportunity)` pair into a **tier**, a **relevance
score**, and a **data record** that the response registry renders into a voiced card.
It assumes the Matching layer upstream has already retrieved candidates and (optionally)
computed a domain-relevance signal.

The order is fixed and each stage has a distinct job:

> **gates filter → relevance ranks → evidence substantiates → registry voices**

**Changes in 0.3** — five rule errors closed, all in the same direction: *can the founder
actually pursue this?* Relevance answered "is it a fit," never "is it reachable."
Cost-share is now a requirement gap (§2.1) · rolling deadlines PASS instead of being
punished (§2) · deadline proximity and data staleness gate timing (§2) · the funding
headline is top-3, not a sum of everything (§6) · results are capped at two per agency (§6).

*(0.2 reconciled the schema against `rag_registry.md` and the `example_*.txt` records:
`unknown` became a ceiling rather than a floor, three attribute states, the no-match render
rule, and the §11 record contract.)*

**Canonical companion files.** `spec/rag_registry.md` (rules + voice) and
`spec/example_{strong,maybe,weak,poor-non-match}.txt` (data only).
`rag_response_registry.md` and `rag_response_registry_persona.md` are superseded by that
pair. **This document is the sole source of truth for the scoring rules.** Implementation:
`code/scoring.py` (rebuilt 2026-08-15 from this schema; `_to_delete/scoring.py` is not used).


## 1. Inputs

**Company profile** (built from onboarding — scrape + confirm + interview):

| field | example | used by |
|---|---|---|
| concepts (expanded tags) | cybersecurity, threat detection, AI, intrusion | domain relevance |
| NAICS | 541512, 541715 | domain relevance |
| state | UT | geography gate |
| employees / small-business flag | 22 / true | eligibility + set-aside gates |
| entity type | for-profit | eligibility gate |
| **attributes / credentials** | set-asides (woman, veteran, hubzone, 8a, native, rural), certifications (CMMC, FedRAMP, AS9100, ISO), clearances (FCL, cleared staff), registrations (SAM/UEI, CAGE) | set-aside + requirement gates |
| **founding & leadership team** | founders, ownership %, veteran/woman/disadvantaged ownership, individual clearances, PI credentials (PhD/PE/MD), prior federal experience | ownership-derived set-asides, key-personnel & clearance requirements, relevance/precedent |
| intent | capital · revenue · both | mechanism relevance + ranking |
| funding need (min–max) | $1M–$3M | award-size relevance |
| **match-funding capacity** | can commit ~$250K in matching funds · cannot · unknown | cost-share requirement gap (§2.1) |

The profile is **partial by design.** Onboarding (scrape → confirm → interview) fills what
it can; everything else stays unknown until an opportunity's own requirements make it worth
asking. See §2.1.

**Three states, not two.** Every attribute is `confirmed-held`, `confirmed-not-held`, or
`unknown`. The distinction drives §2: *not held* is a gate **fail**; *unknown* is a gate
**unknown**. Silence in the profile is never read as denial.

**Opportunity facets** (normalized from Grants.gov / SAM listings / SAM contracts / SBIR):
eligible entity types, set-aside code, geographic restriction, NAICS, domain text,
award min/max, **open status** (`open` · `closed` · `rolling` · `unknown`), close date,
**cost-share %**, mechanism (grant · cooperative agreement · contract · R&D · loan),
**`retrieved_at`**, **and declared requirements** — certifications, clearances,
key-personnel/PI criteria, past-performance expectations, registrations.

`cost_share_pct` and `retrieved_at` are load-bearing as of 0.3, not metadata: the first
decides whether the founder can afford to accept the money, the second decides whether we
are allowed to claim the opportunity is still open.


## 2. Gates — hard filters, evaluated first

Each gate returns **pass · unknown · fail**. `unknown` is a first-class outcome, not
a soft pass — it is what keeps the system honest. These three values are the only legal
gate results; records must not invent others (`not failed`, `n/a`).

| gate | pass | unknown | fail |
|---|---|---|---|
| **Eligibility** | company's entity type / small-biz status is allowed | opportunity doesn't state eligibility | restricted to types the company isn't (e.g. nonprofit-only) |
| **Set-aside** | open competition, or company holds the required status | set-aside code unrecognized, **or status simply unconfirmed** | reserved for a status the company is **confirmed not to hold** |
| **Geography** | no restriction, or company's state matches | HUBZone / local-area — depends on address | restricted to a different state |
| **Timing** | **rolling**, or open with ≥ `min_days_to_apply` remaining | status unconfirmed · deadline inside the window · **data staler than `max_data_age_days`** | closed |

All four gates are evaluated on every pair and all four are carried in the record (§11),
including `timing` — a closed opportunity is a RED suppression, and the record has to say
why.

### 2.0 The timing gate, in full (new in 0.3)

0.2 collapsed four different situations into two answers, and got both wrong at the edges.

**Rolling is a PASS, and it is good news.** 0.2 conflated *"we don't know the status"* with
*"it's rolling,"* returning UNKNOWN for both — which capped every rolling opportunity at
Maybe, permanently. Rolling is the *most* founder-friendly timing there is: no deadline
pressure, apply when ready. It passes, and its reason string should say so.

**A deadline the founder cannot meet is not a pass.** An opportunity closing in four days
was scored identically to one closing in four months. That let a Strong card carry a
**Begin Application** CTA on something un-applyable — precisely the expensive false positive
§9 is built to prevent. So: fewer than `min_days_to_apply` (default **21**) remaining →
**UNKNOWN**, which caps at yellow and emits the verify item *"closes in N days — confirm you
can assemble a submission in that window."* We don't refuse it; we stop promising it.

**Stale data cannot support an open claim.** Deadline arithmetic is only as good as
`retrieved_at`. If the snapshot is older than `max_data_age_days` (default **7**), the gate
returns UNKNOWN regardless of what the record says — we can't assert "open" from a
fortnight-old pull.

*Known simplification:* one deadline threshold for all mechanisms. Contract response windows
are routinely shorter than grant windows, so 21 days is conservative for procurement. Split
the tunable per mechanism if the demo shows contracts being over-hedged.

**Gate effects on tier:**

- Any **fail** → **RED** (suppressed, not shown — see §10).
- Any **unknown** → the tier is **capped at YELLOW**. A cap is a ceiling, never a floor:
  an unknown can only pull a tier *down*, never lift one *up*. A pair scoring 0.82 with one
  unconfirmed requirement is a YELLOW; a pair scoring 0.29 with the same unknown stays
  ORANGE. We never label something "Likely Fit" when we could not confirm the founder is
  eligible. *(Brief: never present an AI assessment as a definitive eligibility
  determination.)*

### 2.1 Requirement gaps — where the "verify" and upgrade prompts come from

The set-aside and eligibility gates are not a fixed checklist. Each opportunity **declares
its own requirements** — a security clearance, a certification (CMMC, FedRAMP, AS9100), a
key-personnel/PI criterion, an ownership status, an active registration, **a cost-share
obligation**. Scoring diffs those requirements against the **partial profile**:

- requirement **met** by a known attribute → gate **pass** (and past performance can also
  *boost relevance*, not just pass a gate)
- requirement **contradicted** by a known attribute → gate **fail**
- requirement **not covered** by the profile → **an open requirement-gap unknown** → becomes
  a **verify / upgrade prompt** on the card

**Requirement-gap unknowns cap the tier just like the four named gates.** They are a fifth
unknown surface: an opportunity can pass eligibility, set-aside, geography, and timing and
*still* be capped at yellow because a declared requirement (CMMC, cost-share, a clearance) is
unconfirmed. In the record this reads as all four `gates` = PASS but `unknowns: present`,
sourced from a non-empty `requirement_gap`. The cap rule in §2 ("any unknown → capped at
YELLOW") ranges over both surfaces.

This is why a Weak or Maybe card can climb tiers (§10): the prompt asks *exactly* the
unconfirmed requirements this opportunity imposes, and a confirmed answer re-runs the gates.
It's "ask only what it needs," done lazily — we don't interrogate every founder about
clearances up front, only when an opportunity that needs one surfaces.

**The requirement gap is generated, never hardcoded.** The Maybe card's *Missing* block and
the Weak card's *criteria question* are both projections of this gap for that specific
opportunity. The checklist in `example_weak.txt` is an **illustrative fallback pool**, used
only when an opportunity declares no requirements the profile can't already answer — it is
not the list to render by default.

**Cost share is a requirement, not a footnote (new in 0.3).** Through 0.2 `cost_share_pct`
only produced a sentence in *concerns* — a 50%-match opportunity and a 0% one scored
identically. But a 20% match on a $2M award is **$400,000 the company has to find**, which
for a ten-person business with $500K of revenue is not a caveat, it's the whole answer.
It is now a requirement-gap item like any other:

| founder's match-funding capacity | result |
|---|---|
| **unknown** (the default) | gate **unknown** → capped at yellow → verify item: *"requires ~$X in matching funds — can you commit that?"* |
| **confirmed sufficient** | **pass**; the obligation still appears in *concerns* |
| **confirmed insufficient** | **fail** → RED, suppressed |

The obligation is quoted against **`award_min`** — the smallest version of the commitment,
so the number we put in front of a founder is never inflated.

*Ruling, worth naming:* confirmed-cannot-fund **suppresses** the opportunity. Money the
company cannot accept is not an opportunity, and serving it spends the trust §9 is built to
protect. This routes an affordability question through the existing gate machinery — no new
facet, no reweighting, no threshold disturbance.

**Ownership-derived gates come from the team.** Most socioeconomic set-asides are
determined by *who owns and controls the company*, not a company-level flag: WOSB needs
≥51% women ownership, SDVOSB ≥51% service-disabled-veteran ownership, 8(a) ≥51%
disadvantaged-individual ownership, HUBZone adds an employee-residency test. So the
**founding & leadership team** is the source of truth for those gates — and for
key-personnel requirements (e.g., SBIR requires the PI be primarily employed by the company
at award) and for individually-held clearances. Team backgrounds also feed relevance:
prior federal awards, agency relationships, and domain credentials strengthen the
agency-precedent signal.


## 3. Relevance — weighted ranking, evaluated when gates don't fail

Weighted sum of four facets, each scored 0–1. Weights are tunable; these are the defaults:

| facet | weight | how it scores |
|---|---:|---|
| **Domain fit** | 0.45 | semantic relevance from the Matching layer; falls back to concept + NAICS overlap. The dominant signal. |
| **Mechanism fit** | 0.20 | capital-vs-revenue alignment to the founder's intent. Revenue-seeking + a contract = full marks; off-thesis = penalized; `both` = neutral-high. |
| **Award-size fit** | 0.20 | overlap between the opportunity's award range and the stated funding need; graceful falloff by distance. |
| **Agency precedent** | 0.15 | count of similar historical awards for this agency/program (from USAspending / SBIR evidence). |

Weights sum to 1.0. Domain carries nearly half because matching quality is 25% of the
rubric and usefulness another 30% — relevance is where most of the score lives.

**Each facet emits its sub-score and a one-line reason** into the record (§11). The
registry composes those reasons into SWOT prose; it does not re-derive them. Agency
precedent in particular must surface its evidence (*"NIH has made 43 similar SBIR awards
since 2020"*) — a 0.15-weight facet that never appears on a card is invisible intelligence.

**Relevance answers "is this a fit," never "can they pursue it."** Reachability —
affordability, deadline feasibility, credentials — lives entirely in the gates. Keeping the
two apart is what lets thresholds stay stable while reachability rules change: every 0.3 fix
is a gate change, and not one of them moved a number in this table.

**Domain fit is a dependency, not an assumption.** If the Matching layer supplies no
`domain_score`, the fallback is literal concept/NAICS overlap, and green then requires
roughly ⅔ of the profile's concepts to appear in the opportunity text. The tier system
degrades toward yellow rather than breaking — but a demo run with the semantic signal
unwired will look uniformly hedged. Check for greens before blaming the thresholds.


## 4. Tiers — the four confidence bands

Evaluated in order; the first matching row wins.

| tier | internal | user-facing | rule |
|---|---|---|---|
| 🔴 **RED** | Probably Not a Fit | **Poor / non-match** | any gate **fails**, or relevance < 0.25 |
| 🟢 **GREEN** | Likely Fit | **Strong match** | all gates pass, **no unknowns**, relevance ≥ 0.70 |
| 🟡 **YELLOW** | Potential Fit — Verify Eligibility | **Maybe match** | relevance ≥ 0.45 |
| 🟠 **ORANGE** | Adjacent Opportunity | **Weak match** | relevance ≥ 0.25 |

**Unknowns apply as a cap after this table, not as a rule inside it.** A pair that qualifies
for GREEN but carries any unknown gate is demoted to YELLOW. A pair at ORANGE with unknowns
stays ORANGE.

| relevance | unknowns | tier |
|---|---|---|
| 0.82 | none | 🟢 Strong |
| 0.82 | present | 🟡 Maybe *(capped)* |
| 0.51 | present | 🟡 Maybe |
| 0.29 | present | 🟠 Weak |
| 0.29 | a gate **fails** | 🔴 suppressed |

Colors stay internal; the founder sees the plain-language label. The displayed **match %**
is `relevance × 100`, rendered on Strong and Maybe cards only, and suppressed on Weak
(where a precise number would overstate a hedge). Section 10 defines how each tier renders
— including that **Poor / non-match is never served as a card**.


## 5. Capital vs. revenue — the thesis, as a knob

Every opportunity is tagged with a **mechanism class**, carried on the record (§11):

- **Revenue** — government contracts (procurement). *Government as customer.*
- **Capital** — grants, cooperative agreements, SBIR R&D. *Non-dilutive capital.*
- **Capital (debt)** — loans and loan guarantees. Non-dilutive but repayable.

When the founder's intent is **revenue-seeking**, a `revenue_bias` term lifts contracts
above equally-relevant grants in the ranking. This is "revenue > funding rounds"
expressed as a tunable weight, not a hard rule. The same tag drives the mechanism-fit facet
(§3) and the portfolio header's revenue-vs-capital split (§6) — which is why it is a
required field, not a display nicety.

`revenue_bias` applies **within a tier**, never across one: ranking is `(tier, relevance +
bias)`, so a biased Maybe can never outrank a Strong. Note that mechanism alignment is
already scored at 0.20 in relevance, so the bias is a second application of the same
preference — deliberate, but it means the default (0.5) is large relative to the 0–1
relevance scale and should be reviewed against real result sets before the demo.


## 6. Portfolio — rank, and be honest about a weak field

After scoring every candidate:

1. **Rank** by `(tier, relevance + revenue_bias)`.
2. **Cap at `agency_cap` (default 2) results per agency** in the served set. Federal sources
   emit near-clones — one SBIR cycle can yield a dozen topics from a single agency, and
   rank-by-score alone will hand a founder five variations of the same solicitation and call
   it a portfolio. Overflow collapses into a single *"N more from this agency"* affordance
   rather than being discarded, so nothing is hidden — it's just not five slots.
3. **Honest no-match test** — fires when **no candidate reaches GREEN _and_ the best
   surviving relevance is below 0.55**. Note this can fire with YELLOW cards on screen: a
   field topping out at 0.50 is a weak field even though it cleared the Maybe floor.
4. **Header stats** — counts per tier, revenue vs. capital split, and funding in reach
   (below). Stats count only *served* cards; suppressed REDs are invisible here as everywhere.

### 6.1 The funding headline (rewritten in 0.3)

Through 0.2 the header summed `award_max` across **every** non-RED result. Twenty hedged
leads at $1.5M rendered as **"$30,000,000 potential funding."**

That number is indefensible three ways: it sums maxima the founder would never all receive,
it counts opportunities we ourselves labeled *verify eligibility*, and it implies a
portfolio strategy no company executes. It is the single most attackable claim in the
product, and it sits in the largest rubric category.

**The rule now:** `funding_in_reach` = the sum of `award_max` across the **top
`top_n_funding` (default 3) served results by rank**, labeled *"top 3 combined."* Never the
whole set, never oranges alone, and always with the count of served options beside it so the
breadth of the search is still visible.

A founder realistically pursues one to three federal opportunities at a time. The headline
should describe that founder, not a hypothetical one who applies to everything.

### 6.2 What the no-match test renders

- The honest-read message **heads the results**; it does not replace them. Surviving
  YELLOW/ORANGE cards still render beneath it, correctly labeled. Suppressing them would
  punish the founder twice — once for a weak field, again by hiding the only leads in it.
- **Only when zero cards survive** (everything RED) does the message stand alone.

Message content is the registry's job (`rag_registry.md`, Tier: Poor → whole-portfolio).
Scoring supplies the trigger, the surviving set, and the set-aside prompts drawn from the
aggregated requirement gap.


## 7. Explanations — derived here, voiced in the registry

**Two layers, kept separate.** Scoring emits a **data record** — gate results, facet
sub-scores and reasons, requirement gap, SWOT factors as neutral bullet facts, next-step
actions. The **response registry** owns all wording, tone, icons, and CTAs. Numbers, data,
buttons, and field labels stay plain in both layers. A voiced sentence must never originate
in scoring, and a fact must never originate in the registry.

Every gate and facet emits a reason string, composed into the four explanation blocks the
brief requires:

- **Why you're a potential fit** — passed eligibility/set-aside gates + strong facet reasons
- **Potential concerns** — failed gates, **cost-share obligations**, weak domain or size fit,
  **tight deadlines**
- **What to verify** — every `unknown` gate becomes a verification item, drawn from the
  opportunity's **requirement gap** (§2.1): unconfirmed certifications, clearances,
  key-personnel/PI criteria, ownership status, registrations, **match-funding capacity**, or
  **submission feasibility inside the remaining window** — not a fixed list

**Next steps are six classes, not a fixed list.** Emit only those the pair actually
warrants:

| class | example |
|---|---|
| registration | complete or confirm SAM.gov / UEI |
| read & diarize | read the full solicitation; note the deadline |
| gap-closing | confirm CMMC posture; verify primary NAICS; commit the PI; **confirm match-funding capacity** |
| **upgrade** | pursue HUBZone / WOSB certification — unlocks reserved opportunities beyond this one |
| engage | contact the contracting officer *(revenue-mechanism opportunities)* |
| prepare | draft the technical volume against topic requirements |

The **upgrade** class is the most valuable of the six: it turns a single match into a
standing capability change.

Because the blocks derive from the evaluation, the output is *substantiated*: every claim
traces to a gate result or a source field.


## 8. Worked examples — the five standard test cases

Every submission is judged against the same five hypothetical startups. Each is
listed here as the profile the schema sees, plus the outcome a correct run should
produce. These double as acceptance criteria: if a case's expected shape doesn't
appear, the schema (or the data feeding it) is wrong.

**Standing caveat:** these five are authored alongside the schema and matched against mock
sites planted with the attributes the schema tests. They are demonstration scripts, not an
evaluation — the suite cannot fail. Treat a clean run as evidence the rules are internally
consistent, not that they are calibrated.

All five are Utah for-profit small businesses, so `state = UT`, `entity_type =
for-profit`, `is_small_business = true`, and `qualifiers = {small_business}` unless
the founder confirms more in the interview. Panel-8 set-aside status (woman /
veteran / HUBZone / 8(a) / native / rural) is **collected during onboarding**, not
assumed — where it would change the result, that's noted.

### Case 01 — AI Healthcare (nurse workflow software)
- **Profile:** concepts = {clinical documentation, nurse workflow, hospital operations, healthcare AI, workforce efficiency}; employees 15; intent = **capital** (non-dilutive R&D); need **$500K–$2M**.
- **Expected surface:** NIH/HHS and NSF grants, SBIR/STTR R&D, workforce-related programs.
- **Expected outcome:** GREEN on NSF America's Seed Fund and NIH SBIR (small-business eligible, domain strong, size fits); YELLOW on broader health-IT or workforce grants; agency precedent from HHS/NSF award history.
- **Watch:** concept expansion must reach *workforce development / labor productivity / health IT*, not just "AI healthcare" — the brief calls this out explicitly. GREEN here requires the PI and registration questions to be **answered** in onboarding; leave them unconfirmed and the cap (§2) correctly drops these to YELLOW. SBIR carries no cost share, so §2.1 shouldn't bite.

### Case 02 — Advanced Manufacturing (lightweight aerospace components)
- **Profile:** concepts = {additive manufacturing, aerospace components, lightweight materials, defense manufacturing}; employees 35; intent = **both** (R&D capital + procurement revenue); need **$2M–$5M**.
- **Expected surface:** DoD/NASA/DOE R&D, manufacturing programs, and **procurement** (government as customer).
- **Expected outcome:** GREEN on DoD/NASA SBIR Phase II (size fits a $2M+ need); GREEN/YELLOW on DoD contract opportunities via NAICS/PSC; revenue options ranked up because intent includes revenue.
- **Watch:** this is the clearest "revenue > funding rounds" case — contracts should be visible, not buried under grants. Contract response windows are often under 21 days; expect the new timing rule (§2.0) to hedge some of them to YELLOW, and check that it isn't hedging *all* of them.

### Case 03 — Climate / Water Technology (municipal water loss)
- **Profile:** concepts = {water loss, leak detection, municipal water infrastructure, environmental sensors, climate technology}; employees 10; intent = **both** (research capital + municipal pilots); need **$500K–$3M**.
- **Expected surface:** EPA/DOE programs, water & environmental grants, infrastructure funding, research, procurement/pilot opportunities.
- **Expected outcome:** GREEN on EPA and DOE grants (they dominate water-loss award history), YELLOW on NSF research and infrastructure programs; municipal procurement is ADJACENT (often local, not federal) — correctly ORANGE.
- **Watch:** USAspending keyword noise ("transepidermal water loss" in NIH grants) must be filtered by the eligibility gate + domain relevance, not surfaced as a match. **This is the cost-share case:** DOE programs routinely require a 20% match, which on a $2M award is $400K against $500K of revenue. Aquora's match capacity is unknown at onboarding, so those should cap at YELLOW with the affordability verify item — not present as Strong.

### Case 04 — Cybersecurity (AI threat detection) — hero demo
- **Profile:** concepts = {cybersecurity, threat detection, network intrusion, AI security}; employees 22; intent = **both**, revenue-leaning (R&D + federal expansion); need **$1M–$3M**.
- **Expected surface:** DoD/DHS opportunities, SBIR/STTR, federal procurement, historical cyber recipients.
- **Expected outcome:** GREEN on DoD/Air Force SBIR (domain strong, small-biz eligible) and DHS/CISA cyber contracts (revenue, small-business set-aside → passes); **RED on an 8(a)/SDVOSB set-aside the founder has confirmed the company does not hold** — a clean demonstration that gates disqualify on eligibility, not relevance.
- **Watch:** this is Cyberdriven, the storyboard walkthrough — set-aside gate behavior is the thing to show a judge. **The RED requires a confirmed negative, not silence** (§1, §2): if onboarding never asked about 8(a) ownership, the honest result is a capped YELLOW with a verify prompt. The demo therefore depends on onboarding capturing ownership status explicitly — otherwise the hero moment shows the wrong behavior. Also expect the agency cap (§6) to bite here: AFWERX alone can supply many matching topics.

### Case 05 — Consumer / Workforce (youth activity marketplace) — *intentionally harder*
- **Profile:** concepts = {youth activities marketplace, parenting, consumer app, enrichment programs}; employees 8; intent = **capital**; need **$250K–$1M**.
- **Expected surface:** workforce development, education, youth programs, small business, local/community development.
- **Expected outcome:** **Honest no-match.** Most fits are nonprofit-only education grants (eligibility gate → RED) or adjacent workforce programs (ORANGE). A generously-scored edtech SBIR may reach a weak YELLOW (~0.45–0.50). Best relevance stays below the 0.55 no-match ceiling → the portfolio heads the results with *"No strong match — the closest options are adjacent, not core."* — with those YELLOW/ORANGE cards still shown beneath it (§6.2).
- **Watch:** the brief explicitly rewards saying this rather than hallucinating a fit. This case is the schema's honesty test, not its coverage test. The funding headline (§6.1) matters most here: under the old sum-everything rule a no-match portfolio could still have announced eight figures of "potential funding" directly above the words *no strong match*.

### Expected tier distribution at a glance

| case | intent | strongest tier | revenue in play? | designed lesson |
|---|---|---|---|---|
| 01 Healthcare | capital | 🟢 green (SBIR/NSF) | no | concept expansion beyond buzzwords |
| 02 Manufacturing | both | 🟢 green (DoD/NASA + contracts) | yes | revenue surfaced alongside R&D |
| 03 Water | both | 🟢 green (EPA/DOE) | partial | keyword noise; cost-share affordability |
| 04 Cybersecurity | both | 🟢 green (SBIR + DHS) | yes | gates disqualify on eligibility |
| 05 Youth marketplace | capital | 🟡 weak yellow → **no-match** | no | honest "probably not a fit" |


## 9. Calibration — why the thresholds sit where they do

The four outcomes of a recommendation do **not** cost the same. This asymmetry, not
raw accuracy, is what sets the thresholds.

| | Good opportunity | Bad opportunity |
|---|---|---|
| **Served** | ✅ **True positive** — awarded, company grows. The win. | ⚠️ **False positive** — founder applies, gets rejected. Wasted weeks, lost trust, credibility spent. |
| **Not served** | ❌ **False negative** — money left on the table, struggle extended, *but the founder never knows.* | ✅ **True negative** — no cost, no damage, invisible. Safe. |

**The false positive is the expensive error.** A rejected "Likely Fit" spends the one
currency the product runs on — a time-poor founder's trust. A false negative costs the
founder money but is invisible to them, so it does no trust damage. The two errors are
not symmetric, and the schema is deliberately biased accordingly.

**Three consequences, already encoded in the rules above:**

1. **Green is precision-first.** GREEN is a promise, and a broken promise is the costly
   quadrant. The green threshold stays conservative and, when a gate is `unknown`, the
   tier is **downgraded to yellow** rather than risk a wrong green. Aim: zero
   indefensible greens.

2. **Yellow / orange buy recall without paying the false-positive cost.** A hedged lead
   that gets rejected is not a betrayal — it was labeled "verify eligibility" or
   "adjacent." This is why the *concerns* and *what-to-verify* blocks matter: they
   convert what would be a false positive into an honest lead. The label is the safety.

3. **Honest no-match is a true negative said out loud.** Declining (case 05) earns trust
   credit for restraint — cheaper and stronger than silent omission.

**Reachability is a false-positive surface too (0.3).** Every rule added in 0.3 closes a way
of making a promise we can't keep — an award the founder can't co-fund, a deadline they
can't meet, a status claim from stale data, a funding total no one will ever receive. None
of those are relevance failures; a perfectly-matched opportunity can fail all four. That's
why they became gates and header rules rather than facet penalties: relevance says *fit*,
gates say *reachable*, and only something both fitting and reachable earns a promise.

**Why the cap direction matters.** Reading `unknown` as a promotion (the 0.1 defect) looks
generous but breaks calibration in both directions at once: it inflates YELLOW with
0.25-grade leads, which cheapens the label the whole recall strategy depends on, and it
empties ORANGE, which is where honest adjacency is supposed to live. A ceiling preserves
both.

**How accurate must the system be?** Calibrated, not correct. The tiers turn a right/wrong
accuracy problem into an honest-confidence calibration problem. The bar:

- **~100% precision on green** — a wrong green is the only truly costly mistake.
- **Loose recall is acceptable — with one demo caveat.** In production a false negative is
  invisible. But judges hold an answer key: each test case's "WE WANT TO SEE" list. A
  missed item there is a *visible* false negative. So on the five cases, surface those
  items at least as YELLOW/ORANGE — coverage without loosening green.
- **Case 05 must decline** — the one case where forcing coverage is the failure.

**What is not yet calibrated, stated plainly.** The thresholds (0.70 / 0.45 / 0.25 / 0.55)
are reasoned, not fitted — no held-out set, no real result distribution behind them. The
honest claim is that the *structure* is calibrated: tiers, caps, and suppression make
confidence legible. The numbers are a starting position, and the first real pull of
opportunity data is what should move them.

Net rule of thumb: **conservative where we make a promise, generous-but-honest everywhere
else, unafraid to say no.**


## 10. Render rules — how each tier becomes a card

Detail scales **inversely** with confidence: a Strong card makes a promise; a Weak card
mostly explains itself and asks a question. The call-to-action is the safety mechanism —
we only push the founder to act where we're confident, so a false positive can't cost
them wasted effort (see §9).

Scoring defines the *shape* below; `rag_registry.md` defines the *wording* (§7). Where the
two ever disagree on shape, this section wins.

Every served card shares a spine — **agency / award name**, **$ amount / timing**,
**match %** (Strong/Maybe only), and a **Scorecard (SWOT)** — then diverges:

| tier | scorecard | extra sections | call to action |
|---|---|---|---|
| 🟢 **Strong match** | full SWOT (S·W·O·T) + **Alignment** | Next steps | **Begin Application** (commit) |
| 🟡 **Maybe match** | **diminished SWOT — Strengths + Weaknesses only** | **Missing** (requirement gap) · Next steps incl. an *upgrade* action | **Go** (explore) |
| 🟠 **Weak match** | full SWOT + hedge copy | *"This isn't a very strong match — here's why we still shared it."* · **criteria question** | *a question, not an apply button* |
| 🔴 **Poor / non-match** | — | — | **not served** |

**Diminished SWOT = Strengths and Weaknesses only.** Opportunities and Threats are dropped
on Maybe cards: speculating about upside before eligibility is confirmed is exactly the
overreach the yellow tier exists to prevent.

**Timing renders as a fact, not a warning banner.** Rolling reads as *"Rolling — no
deadline"*; a comfortable deadline as *"Open, closes Oct 1, 2026"*; a tight one as *"Open,
closes in 9 days"* with the feasibility question in the verify block. The gate already
capped the tier; the copy doesn't need to shout.

**Scorecard = SWOT.** Each served opportunity is explained as Strengths / Weaknesses /
Opportunities / Threats — a founder-native frame that carries the four explanation blocks
from §7 (why-fit → Strengths, concerns → Weaknesses/Threats, upside → Opportunities,
verify + next-steps below the card).

**The Weak card is interactive — and can promote itself.** Its "Do any of these criteria
pertain to you?" checklist is the **requirement gap for that opportunity** (§2.1), grouped
for legibility (*Ownership & team* · *Company credentials & location*), asked *in context*
exactly when it could change the outcome. If the founder confirms a qualifier, the
**relevant gates re-run** and a Weak or even suppressed match can climb tiers. The attribute
question is not separate onboarding — it lives on the card whose fate it can change.

**Poor / non-match is suppressed, not displayed.** This overrides the earlier "show the
disqualifying reason" default. Serving a bad opportunity costs trust even when labeled
(§9), so RED results are withheld from the founder entirely. Two distinct behaviors, kept
separate:

- **Per-opportunity:** RED cards are never rendered. Reasons are logged internally for
  tuning only.
- **Whole-portfolio:** when the no-match test fires (§6), the honest message heads the
  results and the surviving cards render beneath it. Only if nothing survives does the
  message stand alone. Suppress the bad ones; say so once at the top.


## 11. Record contract — what every scored pair must emit

The registry can only render what scoring hands it. These fields are required on every
`example_*.txt`-shaped record; a missing one is a schema violation, not a formatting nit.

| field | notes |
|---|---|
| `tier` | Strong · Maybe · Weak · Poor |
| `relevance` | 0–1, three decimals; `match_pct` = `round(relevance × 100)` |
| `mechanism` | **revenue · capital · capital_debt** (§5) — required; drives ranking, mechanism-fit, and header stats |
| `agency`, `award_name`, `amount_min`, `amount_max` | card spine |
| `timing` | `open` · `closed` · `rolling` · `unknown`, plus `close_date` and `days_remaining` where applicable |
| `cost_share_pct` + `match_obligation` | the % and the dollar figure computed against `award_min` (§2.1) |
| `retrieved_at` | source snapshot timestamp; drives the staleness rule (§2.0) |
| `source`, `source_id`, `url` | provenance — every served claim must be traceable to a record |
| `gates` | **all four** — `eligibility`, `set_aside`, `geography`, `timing` — each exactly `PASS` · `UNKNOWN` · `FAIL`; plus `unknowns: none\|present`. `unknowns: present` is set by **any** four-gate UNKNOWN **or any open `requirement_gap` item** (§2.1) — the four gates can all read PASS and the tier still cap |
| `facets` | per facet (`domain`, `mechanism`, `award_size`, `precedent`): `score` 0–1 **and** `reason` — a one-line evidence string |
| `swot` | neutral bullet facts only; no voiced sentences. Maybe records carry `strengths` + `weaknesses` only |
| `alignment` | Strong records only — label/status pairs |
| `requirement_gap` | Maybe + Weak — generated per §2.1, never a static list; includes cost-share affordability when applicable |
| `next_steps` | each tagged with its class (§7): registration · read · gap-closing · upgrade · engage · prepare |
| `cta` | `Begin Application` · `Go` · `none` |

Poor records carry the same shape with `render: none` and the failing gate recorded for
internal tuning.

**Portfolio record** (one per result set): `tier_counts`, `revenue_options`,
`capital_options`, `funding_in_reach` (top-3 sum, §6.1), `served_count`,
`agency_overflow` (per-agency counts collapsed by the cap), `honest_no_match` (bool).


## Tunable parameters (one place to adjust)

| parameter | default | effect |
|---|---:|---|
| relevance weights | 0.45 / 0.20 / 0.20 / 0.15 | balance of domain vs. mechanism vs. size vs. precedent |
| green threshold | 0.70 | how strong a match must be to be "Likely Fit" |
| yellow threshold | 0.45 | floor for "worth verifying" |
| orange threshold | 0.25 | floor for "adjacent" |
| no-match ceiling | 0.55 | below this (and no green) → honest "no strong match" heads the results |
| revenue bias | 0.5 | how hard to favor contracts for revenue-seeking founders; applies within a tier only |
| unknown handling | **cap (ceiling)** | unknowns demote toward yellow; they never promote |
| **`min_days_to_apply`** | **21** | below this many days to the deadline → timing UNKNOWN + feasibility verify item |
| **`max_data_age_days`** | **7** | snapshot older than this → timing UNKNOWN regardless of recorded status |
| **`agency_cap`** | **2** | maximum served results per agency; overflow collapses to "N more from this agency" |
| **`top_n_funding`** | **3** | how many top results the funding headline sums |


## How it maps to the rubric

| criterion | weight | where this schema earns it |
|---|---:|---|
| Usefulness | 30% | capital-vs-revenue framing; honest no-match; upgrade next-steps; reachability rules (affordability, deadline feasibility) and a funding headline a founder can act on |
| Quality of matching | 25% | gates + weighted relevance; distinguishes strong from weak; per-agency diversity so a portfolio isn't five clones |
| Intelligence & insight | 20% | set-aside gates, agency precedent surfaced as evidence, "verify" items, adjacency, cost-share as a first-class requirement |
| User experience | 15% | four clear explanation blocks, plain-language reasons, persona layer kept separate from data, timing stated as fact |
| Technical execution | 10% | deterministic, tunable, traceable; explicit record contract; provenance and freshness carried on every record |
