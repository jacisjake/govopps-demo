# Matching experiments — facets, slots, and Jev (v0.5 stage 04)

Scope: search facets + portfolio matching with Jev optimizations. **Studio is out
of scope.** Built from the design in `docs/jev-source-profiles.md` and the Paper
sheet "Agent Flow — v0.5 (fewer steps)".

## What was built

| File | Role |
|---|---|
| `code/assemble.py` | Deterministic slot assembly: `portfolio_state()` (mix / grant cap / contract cap from held + capacity), `unlock_signal()` (rule join on themes ∩, NAICS ∩, PSC ∩, agency), `assemble_plan()` (BUILD slot won by `relevance + 0.1×min(unlocks,3)`, SELL slots ≤ capacity headroom, `honest_no_match` blocks all slots) |
| `code/jev_match.py` | Jev layer: `unlock_judge()` (Noul per proposed pair), `revq_confirm()` (Choice per card, the ladder from the source profiles), `make_jev_reranker()` (blend hook passed into `assemble_plan(pair_reranker=…)`), `run_case()` end-to-end on the eval fixtures |
| `code/experiments/run_experiments.py` | E1–E4 below; `python3 -m code.experiments.run_experiments [--live]`; results → `code/experiments/results/experiments.json` |
| `code/tests/test_assemble.py` | 14 unittest invariants for the slot math |

Pipeline integration: nothing in `retrieve/score/rank_portfolio` changed — the
five-case eval and 64 existing+new unit tests pass untouched. `assemble` consumes
the same scored records `rank_portfolio` produces (cards get `themes/naics/psc`
by joining records back to their normalized rows via `join_cards`).

## Results (2026-10-02)

**E1 — Slot invariants: 10/10 PASS.** Caps respected, Poor never slots, the
grant that unlocks the most contracts beats a marginally-more-relevant grant for
the single BUILD slot, revenue-only collapses the build slot, concurrent
contracts reduce SELL headroom, plans are deterministic, and 05-rallytime's
honest-no-match banner now produces an empty plan instead of slotting Weak cards.

**E2 — Unlock pairing, 14 golden pairs (6 positive):**

| arm | precision | recall | notes |
|---|---|---|---|
| rule (themes/NAICS/PSC ∩) | 0.86 | 1.00 | one FP: NAICS-only coincidence (research platform grant vs IT helpdesk contract) — exactly the trap designed for it |
| Jev alone (p≥0.5, title-only state) | 0.62 | 0.83 | misses one true pair (0.48), and confabulates: same-agency traps fire (dining services p=0.68, head-start facilities p=0.63) |
| rule+jev blended (0.5) | 0.86 | 1.00 | preserves rule arms; Jev nudges scores without flipping slots |

**Finding:** with starved state (titles + agency only) Jev is *not* a better
pairer than the coded join — it substitutes agency vibes for capability
evidence. The lever is richer pair state: grant descriptions are already in the
classified corpus (`one_line`); contract scopes need the detail-fetch promised
in the source-profile doc (vec≈1 rows). Until then: rule = gate, Jev = tiebreak
+ disagreement reviewer, not oracle.

**E3 — Five-case plans** (fixtures + Jev-classified corpus as revenue pool):
cadence/aquora build their flagship SBIR/grant and have no sell pool (fixtures
carry no SAM rows — honest); ridgeline builds UT manufacturing grant and sells
two Navy SBIR DP2 topics; cyberdriven builds AFWERX and sells one classified
grants.gov row; rallytime correctly yields an empty plan. In 4/5 cases the flat
list's top card == the plan's BUILD card — assembly changes *structure*, not the
headline, unless unlock counts disagree with relevance (E1 proves the swap).

**E4 — Mechanism audit (222 non-Poor cards, Jev revq vs source-derived label):**
distribution — cost_or_no_margin 136, unclear 68, strings 10, profit-bearing 6,
**equity 2**. One hard flag: NASA aerospace-additive SBIR topic labeled capital
by source but profit-bearing by Jev (p 0.25, low confidence). The bigger finding
is upstream: four classified SBIR topic rows carry corpus `capital=0` →
`mechanism: revenue` (from `classify_corpus`'s Noul on title+400 chars) and are
now being SELL-slotted, while revq independently calls them
`profit_bearing_work` at p 0.79–0.86. Both judgments likely *wrong*: DoD
direct-to-Phase-II topics are R&D assistance with delivery pressure — a textual
gray zone. This is mislabeling born of thin state at classification time.

## Next levers (ordered)

1. **Reclassify the 2020-row corpus with the Topic Description / Purpose text**
   (not the 400-char blurb) before trusting `capital`/`revenue` for slot pools;
   add a `prize`/`dp2_topic` nuance to the revq option set.
2. **Pair state upgrade:** feed Jev grant-text + contract-scope excerpts instead
   of titles; re-run E2 — the rule gate is cheap enough to keep, so target
   Jev ≥ rule precision, not parity.
3. **PSC wiring:** `psc` is carried but inert in scoring; SAM snapshot has it —
   make it a first-class unlock key for procurement rows (NAICS-only traps were
   the rule's sole FP).
4. **Held-grants decay:** slotting currently caps by count; per the absorption
   memo, weight = relevance decay per existing concurrent grant of same theme.

## Repro

```
python3 -m unittest code.tests.test_assemble        # slot invariants
python3 code/eval.py                                 # five cases, unregressed
python3 -m code.experiments.run_experiments          # E1-E4 offline
python3 -m code.experiments.run_experiments --live   # + Jev arms (TYPESAFE_API_KEY)
python3 -m code.jev_match rerank 02-ridgeline        # per-case rule-vs-Jev plans
```
