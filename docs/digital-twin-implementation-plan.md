# Digital Twin — implementation plan (v0.6)

Supersedes the intake loop of v0.3 (frozen: Paper "GovOpps Blueprint — Frozen v0.3",
`design/opportunity-workflow.drawio`). Builds on v0.5 stage-04 assembly
(`docs/matching-experiments.md`, `docs/jev-source-profiles.md`). Studio and the
registry voice layer are unchanged.

## Invariants carried forward

1. Scoring stays deterministic. Jev and embeddings propose and verify; code decides.
2. The model never invents taxonomy values, tiers, or URLs — candidate sets are
   closed, claims are citation-linked or flagged attested.
3. Silence ≠ denial. `missing` is never inferred as `not-held`.
4. Nothing in `retrieve/score_pair/rank_portfolio` is rewritten; the twin feeds
   their existing inputs (`portfolio_state()` fields, `domain_score` hook).
5. Every acceptance criterion is a numbered AC with an executable test, a seeded
   mutation that must be caught, and a Jev semantic-coverage verdict (see
   "Falsifiable acceptance").

## Core data model: claims

```
claim = {
  id, kind, statement, level?,            # kind-specific payload
  state: derived_confirmed | verified | attested | contradicted | missing,
  disclosure: public | abstracted | registry | private,
  evidence: [ {doc_id, span} | {registry: "USPTO 11,234,567"} ],
  provenance: { source: "intake|upload|answer", embed_model, index_hash,
                jev: {question, prob, conf}, model }
}
kinds: project_tech · expertise(level: working|expert) · credential
     · ip · past_performance · capacity(funding_need_min/max, delivery_capacity)
     · identity(entity_type, is_small_business, state)
```

`contradicted` always routes to the founder review queue; it never auto-wins or
auto-suppresses. Abstracted claims are checked for inflation: one Noul per
abstraction — "does the abstract claim imply more than the verified detail
supports?" — disagreement sets `contradicted`.

## Gates

### Sufficiency gate (triggers the search)
`twin.search_ready()` replaces `interview.search_ready()` (interview.py:151):

- intent ∈ {capital, revenue, both}
- ≥1 project_tech or concepts claim present (any state ≥ attested)
- taxonomy closed: naics ∪ agencies ∪ themes non-empty and confirmed
- identity closed: entity_type, is_small_business answered
- credential block closed: every required cert ∈ {held, not-held, skipped}
- capacity: need_min present
- geography defaults UT (confirmed, not asked)

Pre-search UI is the gate's live checklist (each item names its repair action).
Cold-start fallback = demoted v0.3 interview asking exactly these questions.

### Twin gate (prices results, does not block search)
```
capital → PASS if ≥1 project_tech + expertise listed (no anchor needed)
revenue → PASS  if ≥1 of {credential, ip, expertise=expert, past_performance}
               in state verified (registry OR document; any disclosure)
               UNKNOWN if anchors exist but attested-only
               FAIL  if no anchor claim of any kind
contradicted claim on an opportunity's required item → cap tier at Weak
```
Enforced at: match time (frozen with twin version), `studio accept` (409 like
Weak today), claim-repair loop (attach/verify → re-gate → promote).

## Propose–Verify intake

```
text → embed (LiteLLM embedder) → ANN vs curated vocab index (top-k w/ definitions)
     → Jev fan-out: Choice(+no_match) per field, state = founder span + candidate texts
     → high conf: derived_confirmed · mid: founder review card · low: missing
```

Two thresholds required to confirm: cosine gates cost (skip calls), Jev gates
truth. Jev sees candidate label + definition + example — never similarity
scores. Every derived claim stores `embed_model + index_hash`; index rebuild
makes stale claims re-derivable by query.

Deletes: `translate.py::_RULES`, `interview.py` `_AGENCY_CUES/_STATES/_denial/
_SKIP/_GATE_CUES`, `search_plan.py::_THEME_ALIASES` cue vocabularies
(keep compile/validation structure). Qwen keeps one job: spoken McKenna.

## Workstream dependency graph

```
W1 claim store ──┬── W2 vocab index + embeddings ── W3 propose-verify intake
                 ├── W4 Jev verify/citation layer ──┐
                 ├── W5 sufficiency gate ───────────── W7 twin gate + scoring integration
                 └── W6 corpus reclassify (Lever 1) ┘        │
                                                    W8 assembly wiring (BUILD/SELL)
                                                            │
                                                    W9 UI: ledger · review queue ·
                                                        repair checklist · twin editor
W10 evals: fixtures 06/07, golden pairs ≥60, ts_audit extension
W11 Jev CI: AC registry + mutation harness + coverage audit (runs from Phase 1)
```

W6 (reclassify 2,020 rows with Topic Description/Purpose text, add `prize`/
`dp2_topic` revq options) MUST precede W7 pool logic — E4 proved current
`capital` labels poison slot pools.

## Falsifiable acceptance (AC registry + Jev CI)

Doctrine — three independent verdicts per criterion:

1. **Test verdict** — deterministic, exit-code. Only code can say whether code
   passes. Jev never executes, never judges arithmetic, never overrides exit codes.
2. **Mutation verdict** — every AC ships with a seeded mutation in
   `code/tests/mutants/<ac_id>.patch`: a minimal break that *violates the
   criterion while (a lazy) test suite might still pass*. A surviving mutation
   falsifies the TEST, not the code — gate fails even with all tests green.
   This is what makes each AC honestly falsifiable.
3. **Jev verdict** — `code/jev_ci.py` asks, per AC, over state
   `{criterion, falsified_when, test source, mutation outcome}`:
   - Noul `detects`: "If `falsified_when` occurred, would this test suite fail?"
   - Choice `coverage`: `covers_criterion | partial | unrelated`
   - Noul `mutation_fair`: "Does this mutation actually violate the stated
     criterion?" (catches tests guarded by fake mutations)
   Confidence bands: `coverage=uncovered ∧ conf≥0.6` → blocking; mid-band →
   human-review ticket; a green tick is never auto-issued from low confidence.

**CI gate:** all tests pass ∧ zero surviving mutations ∧ no blocking Jev verdict
∧ all review tickets closed. Verdict table printed per AC; results cached per
`(test_hash, mutant_hash, jev_model_version)` so reruns are free when nothing changed.

`docs/ac.yaml` — machine-readable registry every row below feeds:
`{id, text, falsified_when, test, mutant, jev: bool}`

### AC matrix

| AC | Phase | Falsified when… | Test | Seeded mutation |
|---|---|---|---|---|
| AC-1.1 | P1 | any path writes an illegal state transition (e.g. verified→attested silently, contradicted auto-cleared) | state-machine property loop over all kind×state pairs | permit verified→attested without provenance entry |
| AC-1.2 | P1 | twin contains a credential `not_held` after an unanswered turn | replay all-`idk`/deflection transcript (intake-eval u2/u3 corpus) | default unanswered cert → not_held |
| AC-1.3 | P1 | two runs on same twin version + corpus produce different gate verdicts | freeze-replay across process restart | read live `date` inside timing gate |
| AC-1.4 | P1 | `search_ready()` True with any required item unmet | inject each of the 7 gate items off, one at a time | drop `is_small_business` from predicate |
| AC-2.1 | P2 | any derived claim value not present in the vocab index | assert membership over all 5-fixture runs; includes `no_match` never stored as a value | let no_match fallback string leak into claims |
| AC-2.2 | P2 | a taxonomy value derived by deleted v0.3 rules is absent from twin claims | set-compare rules output ⊆ derived output, 5 fixtures | top-k retrieval = 1 |
| AC-2.3 | P2 | <80% of 10 labeled ambiguous utterances land in review, not confirmed | labeled ambiguity set (built from u3/u4 failure modes) | widen confirm threshold |
| AC-2.4 | P2 | changing embedder leaves claims indistinguishable from fresh ones | stale-by-index_hash query after simulated swap | stop storing index_hash |
| AC-3.1 | P3 | fixture `06-contract-no-anchor` serves any contract card at Strong/Maybe | card-tier assertion | remove gate call from tier assignment |
| AC-3.2 | P3 | fixture `07-sbir-junior` suppresses any capital opportunity for missing anchors | SBIR cards present and promotable | apply revenue anchor rule to capital branch |
| AC-3.3 | P3 | registry-only IP claim (synthetic USPTO handle, zero content) fails the anchor rule | gate PASS assertion + disclosure==registry | require doc-evidence for verified |
| AC-3.4 | P3 | any Strong/Maybe card has a contradicted claim on its required items | walk served slate × required-claim join | cap only when disclosure==public |
| AC-3.5 | P3 | any suppressed card lacks a machine-readable reason | enumeration over suppressed∪served; every suppression has {gate, claim, repair} | suppress on low Jev conf with empty reason |
| AC-3.6 | P3 | post-W6 golden set (≥60 pairs) rule precision − blended precision < 0 | re-run E2 arms with upgraded pair state | revert to title-only state (must fail loudly, i.e. test pins state fields) |
| AC-3.7 | P3 | <90% of 10 seeded inflation pairs (vague-over-thin) judged contradicted at p≥0.6 | labeled inflation corpus | drop inflation Noul from fan-out |
| AC-4.1 | P4 | any gate denial payload has empty/null repair action | UI-contract test on serialized payloads for every gate branch | emit null repair for one branch |
| AC-4.2 | P4 | ledger counts ≠ claim-table counts | property test over randomized twins | count contradicted under attested |

Per-phase exit = its AC rows all green under the three-way CI gate.

## `code/jev_ci.py` interface

```
python3 -m code.jev_ci run --phase 1      # full three-way verdict table
python3 -m code.jev_ci coverage           # AC↔test↔criterion audit only (needs key)
python3 -m code.jev_ci mutate --ac AC-1.2 # apply mutant, run suite, expect RED
python3 -m unittest code.tests.test_jev_ci  # offline arm: stub responses, protocol shape
```
Offline mode (fixture-cached Jev answers) keeps CI runnable without a key;
`--live` refreshes coverage audit on registry or test-source changes only.

## Eval & operations

- Extend `ts_audit` to claims: judgment-vs-labeled-span agreement per kind
- `run_experiments` gains E5 (intake derivation) / E6 (gate behavior); per-experiment
  cost ledger (calls, tokens, cents, p50 ms) — reclassification ≈ 202 calls, cents
- Determinism note: rule path deterministic; live Jev plans reproducible per
  (model version, cached responses)
- Latency budget: Jev fan-out only at intake-commit / upload / Confirm events,
  never per keystroke; hunt stays offline-scored
- CI verdict caching + tickets reviewed weekly; Jev model-version bump reruns
  coverage audit wholesale (cheap: one fan-out per AC)

## Risks

| Risk | Mitigation |
|---|---|
| Jev misses explicit denials (intake eval u4) | regex veto-only on downgrades; generous review band; AC-2.3 pins the floor |
| Twin inflation via vague language | AC-3.7 measures the inflation Noul directly |
| Vocab index too narrow → no_match overfires | monitor no_match rates per field; index additions are the fix, no code change |
| Embedder swap invalidates claims | AC-2.4 makes staleness queryable, not tribal knowledge |
| Cold start, zero docs | attested floor + demoted interview; gate explains itself |
| Cert/IP rule locks out no-cert small firms | past_performance included in anchor set (policy flag raised, unresolved) |
| Jev blessing weak tests | mutation arm — surviving mutant fails CI regardless of Jev verdict |
| AC registry drifts from tests | AC-ids in test docstrings + `jev_ci coverage` rerun on any test diff |

## Out of scope (v0.6)

Auto-filing; NDA-gated document sharing with agencies; agentic Jev (level 10);
context compaction (level 7); anything touching registry persona copy.
