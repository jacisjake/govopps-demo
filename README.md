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

Self-score as of **2026-08-15**. **Now** = what a judge can actually run. **If spec ships** = the product this packet describes. We do not score the spec as if it were the demo.

| Criterion | Wt | Now | If spec ships | Why this number |
|---|---:|---:|---:|---|
| Usefulness | 30 | 8 | 26 | Thesis (studio, no-match, map-first) is right. Live URL is a placeholder. No founder can finish a packet yet. |
| Matching | 25 | 6 | 22 | `score_pair` exists and is tested. Retrieve (Grants.gov / SAM / SBIR) is not wired. The five cases cannot be walked end-to-end. |
| Intelligence | 20 | 3 | 17 | History-on-every-card is specified, not attached. No similar-company block in the UI. |
| UX | 15 | 4 | 13 | Placeholder is on-brand. Map, cards, McKenna interview are not served. |
| Technical | 10 | 5 | 8 | Scorer + session API + tests + this packet. Missing the four-source ingest the brief asked everyone to share. |
| **Total** | **100** | **26** | **86** | Gap is implementation, not an undecided product. |

Update this table when a row’s *Now* changes. Do not raise *Now* for work that only exists in markdown.

## Honesty — not production-ready

| Surface | Status | What you will actually see |
|---|---|---|
| https://goed.ut.gitsum.rest/govopps | Placeholder page | “McKenna’s still setting the table.” SPA+API can run locally; public URL still placeholder. |
| Studio (checklist / narrative / evidence / deadline) | Implemented locally | Accept + item writes exist in the API. Not on the public URL. |
| Intake / McKenna (qwen via LiteLLM) | Optional live path | Default `/turns` is passthrough. `live: true` needs LiteLLM. |
| Grants.gov `search2` | Transport stub + fixtures | Live HTTP only via `live_transport`. Tests never network. |
| SAM listings + opportunities | Snapshot script + retrieve stub | Queried only if `SAM_SNAPSHOT` is set. |
| SBIR.gov | Transport stub + fixtures | Same as Grants.gov. |
| USAspending history block | Attached on retrieve | Present on scored records locally. Not on the public URL. |
| Utah table (`source: utah`) | Seeded JSON (8 rows) | Theme-filtered on retrieve. |
| Harness eval (fixtures / `--live`) | Implemented locally | `python3 code/eval.py` — five cases pass on fixtures. |
| Scorer v0.3 | Implemented, tested | Not exposed through the public URL. |
| Session API | Implemented, tested | Private repo; not on the demo host. |
| Live SAM submit | Out of scope (brief) | Will not ship. |
| Mobile chat / `.gov` embed | Explicitly later | Not in this bounty slice. |

If a row above says spec only, treat a polished paragraph as intent, not evidence.
