# Interview Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Landing no longer scores immediately. Founder lands on Confirm, talks to McKenna via `/turns`, then searches.

**Architecture:** `App` step becomes landing → confirm → working → map|nomatch → studio. Confirm hydrates from `session.ticket`. McKenna send POSTs `/turns` `{answer, live:true}` and appends `next_question`. Confirm & start searching is the existing `score({})` path. Org-box fixtures still skip to score.

**Tech Stack:** React 19, Vitest, existing FastAPI `/govopps/api` + `apply_model_turn`.

**Spec:** `spec/intake_translation_instruction.md`, `spec/api.md`, `docs/superpowers/specs/2026-08-15-ui-mockups-design.md`

## Global Constraints

- No Cyberdriven/Rallytime copy as App defaults.
- Missing McKenna prop → omit bubble.
- `/turns` `live:true` is the LLM path; UI still works if it fails (answer stored, generic follow-up).
- Score still freezes ticket. Do not score until Confirm CTA or org-box.
- Rail stays `max-height: 100vh`. McKenna always has a real textarea.
- Do not push. Do not reload Caddy.

---

### Task 1: `turns` client

**Files:**
- Modify: `web/src/api.ts`
- Test: `web/src/api.test.ts`

**Interfaces:**
- Produces: `turns(id: string, body: { answer: string; live?: boolean }): Promise<TurnResult>`
- `TurnResult = { ticket: Record<string, unknown>; turns: Array<Record<string, unknown>> }`

- [ ] **Step 1: Write the failing test**

```ts
it("POSTs a live turn answer", async () => {
  const payload = {
    ticket: { query: { keywords_broad: ["water"] } },
    turns: [{ answer: "we cut water loss", next_question: "Small business?" }],
  };
  vi.stubGlobal("fetch", vi.fn().mockResolvedValue(json(payload)));
  const result = await turns("ses_1", { answer: "we cut water loss", live: true });
  expect(result.turns[0].next_question).toBe("Small business?");
  expect(fetchMock).toHaveBeenCalledWith(
    "/govopps/api/sessions/ses_1/turns",
    expect.objectContaining({
      method: "POST",
      body: JSON.stringify({ answer: "we cut water loss", live: true }),
    }),
  );
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/api.test.ts -t "live turn"`
Expected: FAIL — `turns` is not exported

- [ ] **Step 3: Write minimal implementation**

Add `turns()` next to `intake()` using `request`.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/api.test.ts`
Expected: PASS

---

### Task 2: Confirm hydrates from ticket and sends turns

**Files:**
- Modify: `web/src/screens/Confirm.tsx`
- Modify: `web/src/components/Rail.tsx` (optional `onSend`)
- Test: `web/src/flow.test.tsx`

**Interfaces:**
- Consumes: `ticket`, `pitch`, `messages`, `onSend(text)`, `onConfirm()`
- Produces: visible keywords from `ticket.query.keywords_broad`; Confirm CTA calls `onConfirm`

- [ ] **Step 1: Write the failing test**

Landing submit must not call `/score`. Page shows "Your opportunity map builds here" and a McKenna textbox. Send posts `/turns`. Confirm & start searching then posts `/score`.

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/flow.test.tsx -t "confirm"`
Expected: FAIL — App still scores on submit

- [ ] **Step 3: Write minimal implementation**

`Confirm` takes props. Package rail lists ticket keywords/geography. Rail `onSend` enabled. Footer button calls `onConfirm`.

- [ ] **Step 4: Run tests**

Run: `npx vitest run src/flow.test.tsx`
Expected: PASS (after Task 3)

---

### Task 3: App flow landing → confirm → score

**Files:**
- Modify: `web/src/App.tsx`
- Modify: `web/src/components/Rail.tsx`

**Interfaces:**
- `Step` adds `"confirm"`
- `onSubmit` = createSession + intake, then `setStep("confirm")`
- `onSend` = `turns(id, {answer, live:true})`, append founder + next_question
- `onConfirm` = existing score path (`working` spinner)

- [ ] **Step 1: Extend flow tests for send + confirm CTA**
- [ ] **Step 2: Watch fail**
- [ ] **Step 3: Wire App + Rail onSend**
- [ ] **Step 4: Full vitest pass**

---

### Task 4: Thin URL/domain intake seeds real keywords

**Files:**
- Modify: `code/api.py` `_ticket_from_text`
- Test: `code/tests/test_api.py`

**Interfaces:**
- If intake text contains a host (`uioli.co`), fetch `https://{host}/` (timeout 8s) and take title + meta description tokens ≥4 letters into `keywords_broad` / `concepts`.
- Fetch failure → keep host token; do not invent product copy.

- [ ] **Step 1: Failing test** — `intake({"text":"uioli.co is my product"})` with a stubbed fetch includes `substantiation` or page title tokens, not only `uioli`/`product`
- [ ] **Step 2: Watch fail**
- [ ] **Step 3: Implement fetch+tokenize**
- [ ] **Step 4: `python3 -m unittest code.tests.test_api`**

---

### Task 5: Deploy and judge against spec

**Files:** none new

- [ ] Build `web/` and rsync `web/dist` + `code/api.py` to Blanco
- [ ] Restart only if `api.py` changed
- [ ] Browser: submit `uioli.co is my product` → Confirm, not NIH cards
- [ ] Send one McKenna line → bubble + follow-up
- [ ] Confirm & start searching → score
- [ ] Judge: spec loop is confirm-then-ask-then-search. Org-box may still skip.

---

## Spec coverage

- Confirm-accuracy gate → Task 2–3
- One question at a time via McKenna → Task 3 (`next_question`)
- `/turns` resource → Task 1
- Translate startup language → Task 4 (URL) + live turns when LiteLLM has a key
- Re-retrieve after each answer → not this pass (score only on Confirm CTA)
- Sufficiency exit → founder presses Confirm; no auto-score
