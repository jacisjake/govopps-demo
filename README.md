# GovOpps — judging packet

GOED / StartupState bounty: [Government Opportunity Finder](https://startupstate-hackathon-brief.lovable.app/).

<p align="center">
  <a href="https://goed.ut.gitsum.rest/govopps"><strong>Open the working demo →</strong></a><br>
  <sub>https://goed.ut.gitsum.rest/govopps</sub>
</p>

**GovOpps** is a founder studio, not a grant search box. A founder tells **McKenna** about the company. She translates startup language into government facets, maps federal and Utah opportunities, and — on accept — opens a checklist toward a submission. Match is a door, not the win.

This repo is the **written packet**: scoring rules, harness, sources, card contract, and the five brief companies. The running product is the link above.

## What to score against the brief

| Brief (weight) | Where we spend it |
|---|---|
| Usefulness 30% | Map first; honest no-match; studio after Accept (narrative / evidence / deadline). No live SAM submit — brief says not to build a full application system. |
| Matching 25% | Intake translation → frozen facet ticket → deterministic search. Tiers from gates + weights, not the model. |
| Intelligence 20% | History on every served card (USAspending / SBIR). Adjacent stays Weak. Cost-share and tight deadlines are reachability, not “fit.” |
| UX 15% | McKenna voice; component registry; Poor never shown as a card. |
| Technical 10% | Four official sources + curated Utah table. Snapshots allowed. Eval is fixtures, `--live` opt-in. |

## The five cases

Same companies as the brief. Mock sites: [`mock_startup/`](mock_startup/). Expected shapes: [`spec/scoring_schema.md`](spec/scoring_schema.md) §8.

| # | Company | What should happen |
|---|---|---|
| 01 | [Cadence Health](mock_startup/cadencehealth.io.html) | NIH/HHS/NSF + SBIR on the map; workforce/health-IT expansion, not only “AI healthcare” |
| 02 | [Ridgeline Aero](mock_startup/ridgelineaero.com.html) | DoD/NASA/DOE **and** procurement, not grants only |
| 03 | [Aquora](mock_startup/aquora.io.html) | EPA/DOE; cost-share caps yellow until match capacity is confirmed |
| 04 | [Cyberdriven](mock_startup/cyberdriven.io.html) | Hero: greens on DoD/DHS + SBIR; **8(a)/SDVOSB suppressed** once the founder says they don’t hold it |
| 05 | [Rallytime](mock_startup/rallytime.co.html) | Honest no-match. Dressing this up as a federal win is the failure. |

## Packet contents

1. [Architecture](spec/architecture.md) — pipeline, what v1 will not build  
2. [Harness](spec/harness.md) + [diagram](docs/harness-c.html) — chat/artifacts → ticket → search. Model never tiers, never invents a URL.  
3. [Sources](spec/sources.md) — Grants.gov, SAM, SBIR, USAspending, Utah (`source: utah`)  
4. [Scoring schema v0.3](spec/scoring_schema.md) — **sole** scoring rules  
5. [Intake](spec/intake_translation_instruction.md) — profile + match query  
6. [Registry](spec/rag_registry.md) + [output format](spec/output_format.txt) — McKenna + card shapes  
7. [Studio](spec/studio.md) · [API](spec/api.md) · [components](spec/components.md)

`scoring_schema.md` wins if anything else disagrees about a tier.

## Internal rubric score

Self-score as of **2026-08-15**. **Now** = what a judge can actually run on https://goed.ut.gitsum.rest/govopps. **If spec ships** = the product this packet describes. We do not score the spec as if it were the demo.

Walk the five cases with **Test group → Confirm & start searching**. That path uses recorded fixtures + the case proposal. A free-text pitch hits live transports and is thinner (Grants.gov only; SAM/SBIR/USAspending/Utah stay empty without a snapshot or proposal).

| Criterion | Wt | Now | If spec ships | Why this number |
|---|---:|---:|---:|---|
| Usefulness | 30 | 18 | 26 | Live map-first SPA. Accept opens a studio shelf (narrative / evidence / deadline). Rallytime honesty fires on the Test-group walk. Packet tools are still thin. |
| Matching | 25 | 15 | 22 | Scorer v0.3 + freeze + compiler. Test-group `/score` walks 01–05 (Cadence NIH Strong, Aquora cost-share Maybe, Cyberdriven 8(a) suppressed, Rallytime no-match). Live SAM is empty, so Ridgeline has no procurement. Free-text retrieve is Grants.gov only. |
| Intelligence | 20 | 11 | 17 | History and cost-share gaps sit on the record and in the detail tray. Agency precedent is not surfaced as copy. Weak criteria are display-only (no promote). |
| UX | 15 | 11 | 13 | Confirm + editable search package. CTA rules hold; Poor is never a card. McKenna after score is still hardcoded, not registry-voiced. |
| Technical | 10 | 7 | 8 | Live API + SPA + `python3 code/eval.py` PASS 01–05. Utah is 8 themed rows. SAM needs `SAM_SNAPSHOT`. |
| **Total** | **100** | **62** | **86** | Founder-pitch Now. Test-group walkthrough is closer to **67**. Do not score the packet as 86. |

Update this table when a row’s *Now* changes. Do not raise *Now* for work that only exists in markdown.

## Honesty — not production-ready

| Surface | Status | What you will actually see |
|---|---|---|
| https://goed.ut.gitsum.rest/govopps | Live SPA+API | “Let’s find some money.” Same process serves `/api/*` and the Vite SPA. |
| Test group (01–05) | Implemented | Seeds the fixture ticket, opens Confirm, does **not** score until Confirm & start searching. |
| Opportunity Map | Live | Header: served, top-3 funding, closing-in-90, Federal / Utah. Cards from `components[]`. |
| Studio (checklist / narrative / evidence / deadline) | API + shelf UI | Accept Strong/Maybe. Item writes work. Deadline date lives on `timing.close_date`. |
| Intake / McKenna (qwen via LiteLLM) | Optional live path | `/turns` `live: true`. Fail-soft patches CMS/TEFCA/EHR → HHS. Confirm fields are editable. |
| Grants.gov `search2` | Live HTTP when `GOVOPPS_LIVE=1` | Free-text scores often return Grants.gov only. |
| SAM listings + opportunities | Snapshot only | Empty unless `SAM_SNAPSHOT` is set on the host. |
| SBIR.gov | Fixtures on Test group; live transport exists | Test group shows SBIR heroes. Free-text often returns 0. |
| USAspending history block | On scored records + tray | Shown when retrieve attached history. Empty history is omitted, not faked. |
| Utah table (`source: utah`) | 8 themed rows | Served only when `regional.themes` is non-empty. |
| Harness eval (fixtures / `--live`) | Implemented | `python3 code/eval.py` — 01–05 pass on fixtures. |
| Scorer v0.3 | Implemented, tested | Served through the public API. |
| Session API | Implemented, tested | On the demo host. |
| Live SAM submit | Out of scope (brief) | Will not ship. |
| Mobile chat / `.gov` embed | Explicitly later | Not in this bounty slice. |

If a row says snapshot-only or free-text-empty, treat a polished paragraph as intent, not evidence.

