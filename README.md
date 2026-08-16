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
| Usefulness 30% | Map first; honest no-match; studio after Accept (narrative / evidence / deadline). Hunt re-runs after every utterance. No live SAM submit — brief says not to build a full application system. |
| Matching 25% | Intake + interview mutate a working ticket; score freezes it and retrieves. Tiers from gates + weights, not the model. |
| Intelligence 20% | History on served records (USAspending / SBIR when the transport returns it). Adjacent stays Weak. Cost-share and tight deadlines are reachability, not “fit.” |
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

Org-box on the demo seeds these as test groups (`POST /api/sessions/{id}/group`).

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

Self-score as of **2026-08-16**. **Now** = what a judge can actually run at `goed.ut.gitsum.rest/govopps`. **If spec ships** = the product this packet describes.

| Criterion | Wt | Now | If spec ships | Why this number |
|---|---:|---:|---:|---|
| Usefulness | 30 | 20 | 26 | Live map-first SPA. Accept opens studio (narrative / evidence / deadline). Hunt after every send. Packet tools are still thin. |
| Matching | 25 | 18 | 22 | Scorer v0.3 + freeze + compiler. Test-group walk of 01–05. Live retrieve is Grants.gov; SAM needs `SAM_SNAPSHOT`. A tight compile can empty the map. |
| Intelligence | 20 | 14 | 17 | History and cost-share sit on the record and in the detail tray. Weak criteria are display-only. |
| UX | 15 | 12 | 13 | Confirm + editable search package + McKenna rail on map/no-match. 14B often returns `next_question: null`, so a slot fallback still talks. |
| Technical | 10 | 8 | 8 | Live API + SPA + `python3 code/eval.py` on fixtures. Utah is themed rows. LiteLLM is `qwen2.5-14b` at `ai.gitsum.rest`. |
| **Total** | **100** | **72** | **86** | Demo is runnable. Remaining gap is SAM live + a bigger model, not a missing product. |

Update this table when a row’s *Now* changes. Do not raise *Now* for work that only exists in markdown.

## Honesty — not production-ready

| Surface | Status | What you will actually see |
|---|---|---|
| https://goed.ut.gitsum.rest/govopps | Live SPA + API | Pitch → McKenna → map or honest no-match. Hard-reload after deploys. |
| Studio (checklist / narrative / evidence / deadline) | Live | Accept Strong/Maybe. Weak cannot accept (409). |
| Intake / McKenna (qwen via LiteLLM) | Live, fail-soft | `POST …/intake` and `POST …/turns` `{live: true}`. Model JSON is sanitized; rules lock `not_held` / `skipped`. Empty model question falls back to a slot. |
| Hunt | After every utterance | UI scores on pitch, every send, and package edits. Thin tickets no longer 409. |
| Grants.gov `search2` | Live when `GOVOPPS_LIVE=1` | Tests never network. |
| SAM listings + opportunities | Snapshot | Queried only if `SAM_SNAPSHOT` is set. Live SAM is usually empty. |
| SBIR.gov | Transport + fixtures | Live when `GOVOPPS_LIVE=1`. |
| USAspending history block | Attached on retrieve | Present when that source returns rows. |
| Utah table (`source: utah`) | Seeded JSON | Theme-filtered; no themes → no Utah rows. |
| Harness eval (fixtures / `--live`) | Implemented | `python3 code/eval.py` — five cases on fixtures. |
| Scorer v0.3 | Live through `/score` | Off-model. |
| Session API | Live under `/govopps/api` | FastAPI, `root_path=/govopps`. |
| Live SAM submit | Out of scope (brief) | Will not ship. |
| Mobile chat / `.gov` embed | Explicitly later | Not in this bounty slice. |

If a row above says spec only, treat a polished paragraph as intent, not evidence.
