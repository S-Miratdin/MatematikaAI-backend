# Karakalpak Auto-Translation Design

## Summary

Two features that use Tilmoch Tahrirchi to auto-translate text to Karakalpak. Math symbols, numbers, and signs are never translated — protected by the existing UUID masking in `tahrirchi_translate`.

---

## Feature 1: Solver tab — NER block auto-translated to Karakalpak

### Flow

1. User types a math problem and clicks "Решить"
2. AI streams the solution (existing behavior, unchanged)
3. NER analysis block appears with colored entity spans (existing behavior)
4. **NEW:** The NER text is immediately translated to Karakalpak and shown as a second block below the Russian NER block

### How it works

**Server-side (`server.py`, solve endpoint):**
- After `extract_and_highlight_entities(problem)` produces `ner_html_ru` (existing)
- Call `tahrirchi_translate(problem, 'rus_Cyrl', 'kaa_Latn', api_key, 'tilmoch')` to get `problem_kaa`
- Call `extract_and_highlight_entities(problem_kaa)` to produce `ner_html_kaa`
- Send `ner_html_kaa` in the SSE stream alongside existing `ner_html_ru`

**Frontend (`app.js`, `showNERContainer`):**
- Receives `evt.entities` (Russian NER HTML) and `evt.entities_kaa` (Karakalpak NER HTML)
- Displays Russian NER block (existing)
- Displays Karakalpak NER block below with title `'Karakalpaqsha NER:'`

### Translation rules
- Only the surrounding text is translated
- VALUE spans (numbers), UNIT spans (units), OBJ spans (names) — preserved as-is from NER re-application on translated text
- Math symbols, formulas, decimal numbers — protected by UUID masking in `tahrirchi_translate`

### Files changed
- `server.py` — solve endpoint (add kaa NER generation)
- `app.js` — `showNERContainer()` function (add kaa NER block rendering)

---

## Feature 2: Másele tab — answer feedback translated to Karakalpak

### Flow

1. Task is shown in Russian/Uzbek (original data, no change)
2. User types or selects an answer
3. Feedback (✓ correct / ✗ wrong + hint) appears in Russian first as a placeholder
4. **NEW (only when `_tasksLang === 'kaa'`):** Feedback text is immediately sent to `/api/translate` → replaced with Karakalpak translation

### What is translated
- Correct label: e.g. `✓ Правильно!` → Karakalpak
- Wrong label: e.g. `✗ Неверно` → Karakalpak  
- Hint text: `_taskFieldText(p, 'hint')` value → Karakalpak (if hint exists and answer was wrong)

### What is NOT translated
- The question text and answer choices (remain in Russian/Uzbek)
- Numbers, math symbols, signs — protected by Tahrirchi masking

### Translation call
- Single batch `/api/translate` call with `texts: [feedbackLabel, hintText]`, `source_lang: 'rus_Cyrl'`, `target_lang: 'kaa_Latn'`, `model: 'tilmoch'`
- On success: update `#practiceFeedback` and `#practiceHint` with translated strings
- On error: keep Russian text as fallback (no crash)

### Helper function
Add `translateToKaa(texts)` in `app.js` — thin wrapper over `/api/translate` returning a Promise of translated strings array.

### Files changed
- `app.js` — `checkTaskInput()` and `checkTaskChoice()` (add kaa translation branch)
- `app.js` — add `translateToKaa()` helper

---

## Architecture notes

- No new API endpoints needed — both features use existing `/api/translate`
- `tahrirchi_translate` math protection (UUID masking) is reused as-is — handles both features
- Error handling: translation errors are non-fatal; Russian text remains as fallback
- Latency: ~1-2 sec for translation API call; acceptable since it runs after the answer is already shown

---

## Testing checklist

1. **Solver NER:**
   - Solve a problem → Russian NER block appears with colored tags
   - Karakalpak NER block appears below → text is in Karakalpak, colored tags present
   - Numbers and math symbols are unchanged in both blocks

2. **Másele feedback:**
   - Switch to Karakalpak language in Másele tab
   - Select a topic → task shown in Russian
   - Answer correctly → feedback shows Karakalpak "correct" text
   - Answer wrong → feedback shows Karakalpak "wrong" text + hint in Karakalpak
   - Answer wrong with no hint → only "wrong" label translated, no crash

3. **Fallback:**
   - Disconnect network → answer in Másele → Russian text stays, no crash
   - Solver with failed translation → only Russian NER shown, no crash
