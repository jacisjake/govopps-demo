# McKenna Harness Design

Date: 2026-08-15
Worktree: `.worktrees/ui-deploy`
Status: ready for implementation plan

## Problem

Live McKenna is `qwen2.5-14b` at temperature 0. The gateway only exposes `qwen2.5-14b` and `medgemma`. The 14B is small, but the looping / not-abstracting is the harness.

Evidence (live, “I'm a racketeer”):

- System prompt is 22 580 chars (~5.6k tok): full `intake_translation_instruction.md` + `rag_registry.md`.
- User message is `utterance + ticket JSON`. No history. No hunt.
- qwen returned a usable open ask: “Can you provide more details about your business…”.
- `_census_question` matches the substring **“more details”** and `_question_for` replaces it with the slot “Who writes the check?”.
- Intake never calls the model (`_ticket_from_text` token scrape). Score is rules-only. The model never sees the hunt.

`spec/harness.md` already says: model extracts and proposes; compiler writes HTTP; McKenna voice comes **after** scoring from the record. Current code does the opposite.

## Goal

qwen runs the conversation. Rules lock facts and compile search. Hunt still runs after every utterance. Stay on `qwen2.5-14b` — no new LiteLLM model in this change.

## Non-goals

- Adding a larger model to the gateway.
- Changing retrieve / score math / card UI.
- Reverting hunt-on-every-response.
- Dumping `scoring_schema.md` or the full intake spec into the prompt.

## Approach

One conversational completer. Four inputs every call:

1. Short operating brief (not the spec corpus).
2. Compact working ticket.
3. Last 6 chat turns (founder + McKenna).
4. Hunt digest from the last portfolio, if any.

JSON out stays `{ticket_patch, next_question, proposal}`. `sanitize_model_payload` + `compile_search_plan` + `patch_from_answer` + `_lock_held_gates` still run after.

## Prompt

`_system_prompt()` becomes a fixed brief, target ≤ 3 000 chars. It must state:

- You are McKenna. One open question. Infer slots from the utterance. Do not run a form.
- `ticket_patch` is a partial merge. Never invent a URL or a tier. Never infer set-aside from a name.
- `no` / `nope` / `it is not` on an open certs slot → `not_held`. `skip` / `blank` / `idk` → `skipped`.
- Do not re-ask a recorded gate or a known state.
- `next_question` is spoken McKenna, or `null` if another question would not change the hunt.
- If a hunt digest is present, the question may react to it (why Weak, what would flip a card). No invented award URLs.

User payload, in this order:

```
## Ticket
<json ticket without search_plan>
## Hunt
<json digest or "none yet">
## Recent
<role: text, oldest first, max 6>
## Latest
<raw utterance>
```

`apply_model_turn` gains optional `history` and `hunt`. Existing callers that omit them still work.

## Question policy

Prefer the model’s `next_question` when it is a non-empty string and not `PARSE_FAIL_QUESTION`.

Drop it only when:

- it is an exact casefold match of the last McKenna line, or
- it re-asks 8(a)/SDVOSB after that key is `not_held` | `confirmed-held` | `skipped`, or
- it re-asks location after `profile.state` is set.

Delete the `_census_question` substring list (`more details`, `one-liner`, `certified in`, …). That list is what killed the live racketeer ask.

Slot `next_question(ticket)` is fallback only: parse fail, empty model question, or a dropped re-ask.

## Intake

After the token seed (`_ticket_from_text` or passthrough ticket), intake calls the same `apply_model_turn` path as a live turn, with empty history and no hunt. Fail-soft: keep the token ticket and the slot question.

Persist `session["last_question"]`. `_with_interview_state` returns that string when set; otherwise the slot.

## Hunt digest

Built from the last `session["portfolio"]` (not freeze):

```
{
  "served_count": int,
  "honest_no_match": bool,
  "cards": [{"agency", "award_name", "tier"}]  # top 5 served, skip Poor
}
```

Turns after the first score pass this digest in. Intake does not.

Hunt-on-every-response stays: pitch → score, every send → score, package edit → score.

## Files

- `code/llm.py` — brief, history/hunt payload, `apply_model_turn` extras.
- `code/api.py` — question policy, intake model call, last_question, hunt digest.
- `code/interview.py` — unchanged slot/infer helpers (buyers, cash, skip, states).
- Tests: `code/tests/test_llm.py`, `code/tests/test_api.py`, `code/tests/test_voice.py`.

## Success

Replay of the racketeer path:

1. Pitch “I'm a racketeer” → model question is **not** replaced because it contains “more details”.
2. “anybody … business burned” → buyers inferred; next ask is not the same check question.
3. “new york” → `state=NY`; no second state ask.
4. “skip” → certs `skipped`; next ask is hunt-aware or the hunt confirm line, not 8(a) again.

Unit: prompt < 3000 chars; history appears in `complete()` messages; census substring no longer drops an open ask.

## Out of this spec

LiteLLM model add. Caddy. Push to main.
