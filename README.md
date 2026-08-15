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
