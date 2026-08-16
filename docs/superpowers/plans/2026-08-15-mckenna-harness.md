# McKenna Harness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** qwen runs McKenna’s conversation; rules only lock facts and compile search. The live “more details” ask is no longer discarded.

**Architecture:** Shrink `_system_prompt` to a ≤3000-char brief. `apply_model_turn` sends ticket + last 6 turns + hunt digest. `_question_for` prefers the model question and only drops exact repeats or locked-slot re-asks. Intake uses the same path. Hunt-on-every-response stays.

**Tech Stack:** Python 3 FastAPI in `.worktrees/ui-deploy/code`, unittest, existing `LiteLLMClient` / `qwen2.5-14b`. No new model. No frontend change required.

**Spec:** `docs/superpowers/specs/2026-08-15-mckenna-harness-design.md`  
Also: `spec/harness.md`, `spec/intake_translation_instruction.md`

## Global Constraints

- Stay on `LITELLM_MODEL=qwen2.5-14b`. Do not add a gateway model.
- Temperature stays 0. JSON shape stays `{ticket_patch, next_question, proposal}`.
- `sanitize_model_payload`, `compile_search_plan`, `patch_from_answer`, `_lock_held_gates` still run after every model turn.
- Hunt after pitch, every send, and package edit. Do not restore the search-ready 409.
- Do not dump `scoring_schema.md` or the full intake spec into the prompt.
- Do not push. Do not reload Caddy. Worktree: `.worktrees/ui-deploy`.
- Tests: `python3 -m unittest …` from the worktree root. Skip formatters / full-suite unless a task says otherwise.

## File map

| File | Role |
|---|---|
| `code/llm.py` | Brief, user payload, `apply_model_turn(ticket, raw_text, client, history=None, hunt=None)` |
| `code/api.py` | `_question_for`, hunt digest, intake model call, `session["last_question"]` |
| `code/interview.py` | Unchanged slot/infer helpers |
| `code/tests/test_llm.py` | Prompt + payload |
| `code/tests/test_api.py` | Question policy, intake model, last_question, digest |
| `code/tests/test_voice.py` | Replay: more-details kept; skip/NY still lock |

---

### Task 1: Short operating brief

**Files:**
- Modify: `code/llm.py` (`_system_prompt`, `_SPEC_FILES`)
- Test: `code/tests/test_llm.py`

**Interfaces:**
- Produces: `_system_prompt() -> str` ≤ 3000 chars; no full intake spec body
- Consumes: none

- [ ] **Step 1: Write the failing test**

In `code/tests/test_llm.py`, add:

```python
from llm import _system_prompt

class PromptTests(unittest.TestCase):
    def test_brief_is_short_and_operational(self):
        blob = _system_prompt()
        self.assertLessEqual(len(blob), 3000)
        lowered = blob.lower()
        self.assertIn("mckenna", lowered)
        self.assertIn("ticket_patch", lowered)
        self.assertIn("one question", lowered)
        self.assertNotIn("the llm gate at the **front** of the pipeline", lowered)
        self.assertNotIn("# govopps — response registry", lowered)
```

- [ ] **Step 2: Run it to make sure it fails**

Run: `python3 -m unittest code.tests.test_llm.PromptTests.test_brief_is_short_and_operational -v`  
cwd: `.worktrees/ui-deploy`  
Expected: FAIL — current prompt is ~22580 chars and contains the intake spec header.

- [ ] **Step 3: Write the brief**

Replace `_SPEC_FILES` / `_spec_pack()` usage in `_system_prompt()` with a fixed string. Keep `_spec_pack` deleted or unused.

Required lines (paraphrase ok, meaning not):

```
You are McKenna. One open question. Infer slots from the utterance. Do not run a form.
Respond with a single JSON object only. No markdown, no prose.
Shape: {"ticket_patch": {"profile": {}, "query": {}}, "next_question": string|null, "proposal": {}}.
ticket_patch is a partial merge. Never invent a URL or a tier. Never infer set-aside from a name.
no / nope / it is not on an open certs slot → not_held. skip / blank / idk → skipped.
Do not re-ask a recorded gate or a known state.
next_question is spoken McKenna, or null if another question would not change the hunt.
If a hunt digest is present, you may react to it. No invented award URLs.
```

Update `ApplyTurnTests.test_prompt_loads_current_spec_markdown`:

- Keep asserts for `ticket_patch`, `next_question`, `one question`, `not_held`.
- Drop `self.assertTrue("jess" in blob or "new girl" in blob)`.
- Keep the superseded-registry `assertNotIn`.

- [ ] **Step 4: Run the tests**

Run: `python3 -m unittest code.tests.test_llm -v`  
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add code/llm.py code/tests/test_llm.py
git commit -m "fix: shrink McKenna system prompt to an operating brief"
```

---

### Task 2: History and hunt in the completer payload

**Files:**
- Modify: `code/llm.py` (`apply_model_turn`)
- Test: `code/tests/test_llm.py`

**Interfaces:**
- Consumes: Task 1 brief
- Produces: `apply_model_turn(ticket, raw_text, client, history=None, hunt=None) -> dict`
- `history`: `list[dict]` with `role` in `{me,mckenna,user,assistant}` and `text`/`content`
- `hunt`: `dict | None`

- [ ] **Step 1: Write the failing test**

```python
class ApplyTurnTests(unittest.TestCase):
    def test_user_payload_includes_ticket_history_and_hunt(self):
        seen = {}

        class FakeClient:
            def complete(self, messages):
                seen["messages"] = messages
                return json.dumps(
                    {"ticket_patch": {}, "next_question": "Who pays when this works?", "proposal": {}}
                )

        apply_model_turn(
            {
                "profile": {"concepts": ["racketeer"]},
                "query": {"keywords_broad": ["racketeer"]},
                "search_plan": {"grants_gov": {"keyword": "racketeer"}},
            },
            "anybody who hates a fire",
            FakeClient(),
            history=[
                {"role": "me", "text": "I'm a racketeer"},
                {"role": "mckenna", "text": "Who writes the check?"},
            ],
            hunt={"served_count": 0, "honest_no_match": True, "cards": []},
        )
        user = seen["messages"][1]["content"]
        self.assertIn("## Ticket", user)
        self.assertIn("racketeer", user)
        self.assertNotIn("search_plan", user)
        self.assertIn("## Hunt", user)
        self.assertIn("honest_no_match", user)
        self.assertIn("## Recent", user)
        self.assertIn("I'm a racketeer", user)
        self.assertIn("## Latest", user)
        self.assertIn("anybody who hates a fire", user)
        self.assertEqual(len(seen["messages"]), 2)
```

Also add: more than 6 history items → only the last 6 appear.

```python
    def test_history_keeps_last_six_lines(self):
        seen = {}

        class FakeClient:
            def complete(self, messages):
                seen["user"] = messages[1]["content"]
                return json.dumps({"ticket_patch": {}, "next_question": None, "proposal": {}})

        history = [{"role": "me", "text": f"u{i}"} for i in range(8)]
        apply_model_turn({}, "now", FakeClient(), history=history)
        self.assertNotIn("u0", seen["user"])
        self.assertNotIn("u1", seen["user"])
        self.assertIn("u2", seen["user"])
        self.assertIn("u7", seen["user"])
```

- [ ] **Step 2: Run it to make sure it fails**

Run: `python3 -m unittest code.tests.test_llm.ApplyTurnTests.test_user_payload_includes_ticket_history_and_hunt code.tests.test_llm.ApplyTurnTests.test_history_keeps_last_six_lines -v`  
Expected: FAIL — current user content is `raw\n{ticket}` including `search_plan`.

- [ ] **Step 3: Implement payload**

In `apply_model_turn`:

```python
def apply_model_turn(ticket, raw_text, client, history=None, hunt=None):
    ticket = copy.deepcopy(ticket or {})
    shown = copy.deepcopy(ticket)
    shown.pop("search_plan", None)
    lines = []
    for item in list(history or [])[-6:]:
        if not isinstance(item, dict):
            continue
        role = item.get("role") or "me"
        text = item.get("text") or item.get("content") or ""
        if text:
            lines.append(f"{role}: {text}")
    user = (
        "## Ticket\n"
        f"{json.dumps(shown, default=str)}\n"
        "## Hunt\n"
        f"{json.dumps(hunt, default=str) if hunt is not None else 'none yet'}\n"
        "## Recent\n"
        f"{chr(10).join(lines) if lines else 'none'}\n"
        "## Latest\n"
        f"{raw_text}"
    )
    messages = [
        {"role": "system", "content": _system_prompt()},
        {"role": "user", "content": user},
    ]
    # existing parse / sanitize / compile unchanged
```

- [ ] **Step 4: Run the tests**

Run: `python3 -m unittest code.tests.test_llm -v`  
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add code/llm.py code/tests/test_llm.py
git commit -m "feat: send ticket, history, and hunt digest to qwen"
```

---

### Task 3: Prefer the model question

**Files:**
- Modify: `code/api.py` (`_census_question`, `_question_for`)
- Test: `code/tests/test_api.py`

**Interfaces:**
- Consumes: model `next_question` string
- Produces: `_question_for(ticket, suggested=None, last_question=None) -> str`
- Drop only: empty / `PARSE_FAIL_QUESTION` / exact casefold repeat of `last_question` / `_reasks_set_aside` when that gate is recorded / `_reasks_known_state`

- [ ] **Step 1: Write the failing tests**

Add to `code/tests/test_api.py`:

```python
from api import _question_for, PARSE_FAIL_QUESTION

class QuestionPolicyTests(unittest.TestCase):
    def test_keeps_open_more_details_ask(self):
        ticket = {
            "profile": {"concepts": ["racketeer"]},
            "query": {"keywords_broad": ["racketeer"], "geography": ["UT"]},
        }
        asked = (
            "Can you provide more details about your business, such as "
            "the specific services or products you offer and your target market?"
        )
        self.assertEqual(_question_for(ticket, asked), asked)

    def test_drops_exact_repeat(self):
        ticket = {"profile": {"concepts": ["racketeer"]}}
        asked = "Who writes the check?"
        out = _question_for(ticket, asked, last_question=asked)
        self.assertNotEqual(out.casefold(), asked.casefold())

    def test_drops_8a_reask_after_skipped(self):
        ticket = {
            "profile": {
                "intent": "revenue",
                "attributes": {"set_asides": {"8a": "skipped", "sdvosb": "skipped"}},
            },
            "query": {
                "keywords_precise": ["youth"],
                "agencies": ["ED"],
                "mechanisms": ["contract"],
            },
        }
        out = _question_for(ticket, "Is the company 8(a) certified?")
        self.assertNotIn("8(a)", out.lower())
        self.assertNotIn("8a", out.lower())
```

- [ ] **Step 2: Run it to make sure it fails**

Run: `python3 -m unittest code.tests.test_api.QuestionPolicyTests -v`  
Expected: FAIL — `_census_question` matches “more details”; `_question_for` has no `last_question`.

- [ ] **Step 3: Implement policy**

Delete `_census_question` and every call.

```python
def _recorded_set_aside(ticket: dict[str, Any]) -> bool:
    bucket = ((ticket.get("profile") or {}).get("attributes") or {}).get("set_asides") or {}
    locked = {"not_held", "confirmed-held", "skipped"}
    return bucket.get("8a") in locked or bucket.get("sdvosb") in locked

def _question_for(ticket, suggested=None, last_question=None) -> str:
    slot = next_question(ticket)
    text = str(suggested or "").strip()
    if text and text != PARSE_FAIL_QUESTION:
        if last_question and text.casefold() == str(last_question).strip().casefold():
            text = ""
        elif _reasks_set_aside(text) and _recorded_set_aside(ticket):
            text = ""
        elif _reasks_known_state(text, ticket):
            text = ""
    if text and text != PARSE_FAIL_QUESTION:
        return text
    return slot or (_HUNT_FOLLOW_UP if search_ready(ticket) else _FOLLOW_UP)
```

Update every `_question_for(...)` call site in `add_turn` to pass `session.get("last_question")`. After choosing the question, `session["last_question"] = turn["next_question"]`.

- [ ] **Step 4: Run the tests**

Run: `python3 -m unittest code.tests.test_api.QuestionPolicyTests code.tests.test_voice -v`  
Expected: PASS. If a voice test still asserts a census drop, change it to assert the model question is kept.

- [ ] **Step 5: Commit**

```bash
git add code/api.py code/tests/test_api.py
git commit -m "fix: keep qwen open questions; drop only locked re-asks"
```

---

### Task 4: Intake runs the model

**Files:**
- Modify: `code/api.py` (`intake`)
- Test: `code/tests/test_api.py`

**Interfaces:**
- Consumes: `apply_model_turn`, `_lock_held_gates`, `_question_for`
- Produces: intake `next_question` from the model when `live` is not required — always try, fail-soft to token ticket

- [ ] **Step 1: Write the failing test**

```python
    def test_intake_calls_model_and_returns_its_question(self):
        import api as api_mod

        class Chatty:
            def complete(self, messages):
                return json.dumps(
                    {
                        "ticket_patch": {
                            "query": {"buyers": ["people who hate fires"]}
                        },
                        "next_question": (
                            "Can you provide more details about what you actually sell?"
                        ),
                        "proposal": {},
                    }
                )

        previous = api_mod._LLM_CLIENT
        api_mod._LLM_CLIENT = Chatty()
        try:
            session_id = self._create()
            body = self.client.post(
                f"/api/sessions/{session_id}/intake",
                json={"text": "I'm a racketeer"},
            ).json()
            self.assertIn("more details", (body.get("next_question") or "").lower())
            buyers = ((body.get("ticket") or {}).get("query") or {}).get("buyers") or []
            self.assertTrue(buyers)
        finally:
            api_mod._LLM_CLIENT = previous
```

- [ ] **Step 2: Run it to make sure it fails**

Run: `python3 -m unittest code.tests.test_api.ApiTests.test_intake_calls_model_and_returns_its_question -v`  
Expected: FAIL — intake never calls the client; `_with_interview_state` overwrites with the slot.

- [ ] **Step 3: Implement**

After the token/passthrough ticket is built, if no explicit `profile`/`query`/`ticket` override from the client that is meant to skip LLM (passthrough `ticket` in body still skips — keep that), call:

```python
try:
    client = _LLM_CLIENT or default_client()
    if _LLM_CLIENT is None:
        _LLM_CLIENT = client
    result = apply_model_turn(session["ticket"], str(body.get("text") or ""), client)
    merged = patch_from_answer(result["ticket"], body.get("text"))
    session["ticket"] = _lock_held_gates(session.get("ticket"), merged)
    session["last_question"] = _question_for(
        session["ticket"], result.get("next_question")
    )
except Exception:
    session["last_question"] = _question_for(session["ticket"])
```

Passthrough: if `body` has `ticket` or `profile`/`query`, skip the model (tests that seed tickets must stay deterministic).

`_with_interview_state`:

```python
question = session.get("last_question") or next_question(ticket)
if question:
    out["next_question"] = question
```

- [ ] **Step 4: Run the tests**

Run: `python3 -m unittest code.tests.test_api code.tests.test_voice -v`  
Expected: PASS. Thin-pitch intake tests that only check a slot question still pass via fail-soft or last_question fallback.

- [ ] **Step 5: Commit**

```bash
git add code/api.py code/tests/test_api.py
git commit -m "feat: run qwen on intake, not only on turns"
```

---

### Task 5: Hunt digest on later turns

**Files:**
- Modify: `code/api.py` (`_hunt_digest`, `add_turn`)
- Test: `code/tests/test_api.py`

**Interfaces:**
- Produces: `_hunt_digest(session) -> dict | None`
- Shape: `{served_count: int, honest_no_match: bool, cards: [{agency, award_name, tier}]}` top 5 served, skip `Poor`
- `None` when `session["portfolio"]` is missing

- [ ] **Step 1: Write the failing tests**

```python
from api import _hunt_digest

class HuntDigestTests(unittest.TestCase):
    def test_digest_takes_top_five_non_poor(self):
        session = {
            "portfolio": {
                "served_count": 3,
                "honest_no_match": False,
                "served": [
                    {"agency": "AFWERX", "award_name": "A", "tier": "Strong"},
                    {"agency": "NIH", "award_name": "B", "tier": "Poor"},
                    {"agency": "DOE", "award_name": "C", "tier": "Weak"},
                ],
            }
        }
        digest = _hunt_digest(session)
        self.assertEqual(digest["served_count"], 3)
        self.assertEqual([row["award_name"] for row in digest["cards"]], ["A", "C"])

    def test_digest_none_before_score(self):
        self.assertIsNone(_hunt_digest({"ticket": {}}))
```

Plus an API test that a live turn after score passes hunt into the completer:

```python
    def test_live_turn_after_score_sends_hunt_digest(self):
        import api as api_mod
        seen = {}

        class Chatty:
            def complete(self, messages):
                seen["user"] = messages[1]["content"]
                return json.dumps(
                    {"ticket_patch": {}, "next_question": "What would make a Strong fit?", "proposal": {}}
                )

        previous = api_mod._LLM_CLIENT
        api_mod._LLM_CLIENT = Chatty()
        try:
            session_id = self._create()
            self.client.post(
                f"/api/sessions/{session_id}/intake",
                json={"text": "We're Cyberdriven. Our AI watches networks."},
            )
            self.client.post(
                f"/api/sessions/{session_id}/score",
                json={"opportunities": [_opportunity()], "today": TODAY.isoformat()},
            )
            self.client.post(
                f"/api/sessions/{session_id}/turns",
                json={"answer": "new york", "live": True},
            )
            self.assertIn("## Hunt", seen["user"])
            self.assertIn("AFWERX", seen["user"])
        finally:
            api_mod._LLM_CLIENT = previous
```

- [ ] **Step 2: Run it to make sure it fails**

Run: `python3 -m unittest code.tests.test_api.HuntDigestTests code.tests.test_api.ApiTests.test_live_turn_after_score_sends_hunt_digest -v`  
Expected: FAIL — `_hunt_digest` missing; turn payload has `Hunt: none yet`.

- [ ] **Step 3: Implement**

```python
def _hunt_digest(session: dict[str, Any]) -> dict[str, Any] | None:
    portfolio = session.get("portfolio")
    if not isinstance(portfolio, dict):
        return None
    cards = []
    for record in portfolio.get("served") or portfolio.get("records") or []:
        if not isinstance(record, dict) or record.get("tier") == "Poor":
            continue
        cards.append(
            {
                "agency": record.get("agency"),
                "award_name": record.get("award_name"),
                "tier": record.get("tier"),
            }
        )
        if len(cards) == 5:
            break
    return {
        "served_count": portfolio.get("served_count", len(cards)),
        "honest_no_match": bool(portfolio.get("honest_no_match")),
        "cards": cards,
    }
```

In `add_turn` live path:

```python
result = apply_model_turn(
    session.get("ticket") or {},
    body["answer"],
    client,
    history=session.get("turns") or [],
    hunt=_hunt_digest(session),
)
```

History mapping: existing turns store `{answer, next_question}`. Convert before send:

```python
def _chat_history(session: dict[str, Any]) -> list[dict[str, str]]:
    out: list[dict[str, str]] = []
    for turn in session.get("turns") or []:
        if not isinstance(turn, dict):
            continue
        if turn.get("answer"):
            out.append({"role": "me", "text": str(turn["answer"])})
        if turn.get("next_question"):
            out.append({"role": "mckenna", "text": str(turn["next_question"])})
    return out[-6:]
```

Pass `_chat_history(session)` as `history`.

- [ ] **Step 4: Run the tests**

Run: `python3 -m unittest code.tests.test_api code.tests.test_llm -v`  
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add code/api.py code/tests/test_api.py
git commit -m "feat: ground McKenna turns on last hunt digest"
```

---

### Task 6: Racketeer replay locks

**Files:**
- Modify: only if a regression fails
- Test: `code/tests/test_voice.py`

**Interfaces:**
- Consumes: Tasks 3–5
- Produces: no new API

- [ ] **Step 1: Write the failing test**

```python
    def test_racketeer_more_details_is_not_replaced(self):
        import api as api_mod

        class Chatty:
            def complete(self, messages):
                return json.dumps(
                    {
                        "ticket_patch": {},
                        "next_question": (
                            "Can you provide more details about your business, "
                            "such as the specific services or products you offer "
                            "and your target market?"
                        ),
                        "proposal": {},
                    }
                )

        previous = api_mod._LLM_CLIENT
        api_mod._LLM_CLIENT = Chatty()
        try:
            session_id = self._create()
            body = self.client.post(
                f"/api/sessions/{session_id}/intake",
                json={"text": "I'm a racketeer"},
            ).json()
            q = body.get("next_question") or ""
            self.assertIn("more details", q.lower())
            self.assertNotIn("who writes the check", q.lower())
        finally:
            api_mod._LLM_CLIENT = previous
```

Keep existing `test_nope_then_unrelated_answer_does_not_reopen_set_aside` and `test_new_york` interview tests green.

- [ ] **Step 2: Run it**

Run: `python3 -m unittest code.tests.test_voice code.tests.test_interview -v`  
Expected: PASS after Tasks 3–4. If it fails, fix `_question_for` / intake, not the test.

- [ ] **Step 3: Commit if the test is new**

```bash
git add code/tests/test_voice.py
git commit -m "test: keep qwen more-details ask on racketeer intake"
```

---

### Task 7: Deploy and live smoke

**Files:** none in git. Deploy only.

- [ ] **Step 1: Unit gate**

Run: `python3 -m unittest code.tests.test_llm code.tests.test_api code.tests.test_voice code.tests.test_interview -v`  
cwd: `.worktrees/ui-deploy`  
Expected: PASS

- [ ] **Step 2: Deploy API only**

```bash
rsync -avz code/llm.py code/api.py ut.gitsum.rest:apps/govopps/code/
ssh -o BatchMode=yes ut.gitsum.rest bash /tmp/restart-govopps.sh
```

Do not rsync `web/dist` unless a frontend file changed (it should not).

- [ ] **Step 3: Live smoke**

Replay against `https://goed.ut.gitsum.rest/govopps/api`:

1. intake `"I'm a racketeer"` → `next_question` contains a real open ask, **not** “who writes the check” solely because the model said “more details”.
2. turn `"anybody who thinks it would be a shame if their business burned"` → buyers set; question ≠ previous.
3. turn `"new york"` → `profile.state == "NY"`; question does not ask state again.
4. turn `"skip"` → `set_asides.8a == skipped`; question does not mention 8(a).
5. each of 1–4 is followed by `POST …/score` 200 (UI already does this).

Print the four questions and the ticket fields above. Hard-reload the SPA and click through once.

- [ ] **Step 4: Do not commit deploy artifacts. Do not push.**

---

## Spec coverage

| Spec requirement | Task |
|---|---|
| Brief ≤ 3000 chars, no spec dump | 1 |
| Ticket + history + hunt in user payload | 2, 5 |
| Prefer model question; drop census list | 3 |
| Exact repeat / locked 8(a) / known state | 3 |
| Intake calls model, fail-soft | 4 |
| `last_question` wins over slot in session payload | 4 |
| Hunt digest top 5, none before score | 5 |
| Hunt-on-every-response unchanged | 7 (UI already); no 409 restore |
| skip / NY / nope still lock | 6 + existing interview tests |
| Stay on qwen2.5-14b | Global constraint |

## Placeholder scan

No TBD / “add tests later” / “similar to Task N”. Types match: `history` is `list[dict]`, `hunt` is `dict | None`, `_question_for(..., last_question=None)`.
