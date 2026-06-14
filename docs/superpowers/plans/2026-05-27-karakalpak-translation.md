# Karakalpak Auto-Translation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Auto-translate NER block in Solver and feedback text in Másele tab to Karakalpak via Tahrirchi API.

**Architecture:** Server generates Karakalpak NER by translating the raw problem text, then re-applying NER — included in the SSE `done` event. For Másele feedback, the frontend calls `/api/translate` after answer verification when `_tasksLang === 'kaa'`. Math symbols and numbers are never translated (protected by existing UUID masking in `tahrirchi_translate`).

**Tech Stack:** Python (Flask, SSE), Vanilla JS, Tahrirchi tilmoch API

---

## File Map

| File | Changes |
|------|---------|
| `server.py` | In `generate()` inside `api_solve`: after AI answer, translate problem to kaa, run NER on translated text, include `entities_kaa` in `done` event |
| `app.js` | Add `nerEntitiesKaaHTML` global; update `clearResult`, `showNERContainer`, SSE handler; add `translateToKaa` helper; update `checkTaskInput`, `checkTaskChoice` |

---

## Task 1: Server — Generate Karakalpak NER in solve endpoint

**Files:**
- Modify: `server.py:612-649` (inside `generate()` in `api_solve`)

### Context

`generate()` currently:
1. Line 566: `entities_html = extract_and_highlight_entities(problem)` → yields `{entities: ...}`
2. Lines 584-604: OpenRouter AI call
3. Line 611: yields `{ai_answer: ...}`
4. Lines 613-649: Formats + translates answer → yields `{done: True, answer: ..., recognized_entities: entities_html}`

We insert kaa NER generation between step 3 and step 4.

- [ ] **Step 1: Add kaa NER generation block after `yield ai_answer`**

Find the line in `server.py`:
```python
        yield f"data: {json.dumps({'ai_answer': ai_answer})}\n\n"
```
(currently line 611)

Insert immediately after it:
```python
        # ── Step 2b: Karakalpak NER (translate problem, re-apply NER) ──
        entities_html_kaa = ''
        if api_key.strip():
            try:
                problem_kaa = tahrirchi_translate(problem, 'rus_Cyrl', 'kaa_Latn', api_key, model)
                entities_html_kaa = extract_and_highlight_entities(problem_kaa)
            except Exception as _ner_kaa_err:
                print(f"[SOLVE] kaa NER failed: {_ner_kaa_err}")
```

- [ ] **Step 2: Add `entities_kaa` to the three `done` yield statements**

There are three yield points for `done` in `generate()`. Update ALL three to include `entities_kaa`:

**Yield 1** (early return when AI failed or no api_key, currently line 616):
```python
# OLD:
yield f"data: {json.dumps({'done': True, 'answer': formatted_answer, 'recognized_entities': entities_html})}\n\n"
# NEW:
yield f"data: {json.dumps({'done': True, 'answer': formatted_answer, 'recognized_entities': entities_html, 'entities_kaa': entities_html_kaa})}\n\n"
```

**Yield 2** (after successful translation, currently line 645):
```python
# OLD:
yield f"data: {json.dumps({'done': True, 'answer': translated_answer, 'recognized_entities': entities_html})}\n\n"
# NEW:
yield f"data: {json.dumps({'done': True, 'answer': translated_answer, 'recognized_entities': entities_html, 'entities_kaa': entities_html_kaa})}\n\n"
```

**Yield 3** (Russian, no translation, currently line 649):
```python
# OLD:
yield f"data: {json.dumps({'done': True, 'answer': formatted_answer, 'recognized_entities': entities_html})}\n\n"
# NEW:
yield f"data: {json.dumps({'done': True, 'answer': formatted_answer, 'recognized_entities': entities_html, 'entities_kaa': entities_html_kaa})}\n\n"
```

Note: `entities_html_kaa` is defined before the `if not is_success...` early return. Move the kaa NER block to BEFORE that early return check (line 613), so it runs even when AI fails.

Corrected insertion point — place the kaa NER block just after `yield ai_answer` (line 611), before `if not is_success or not api_key.strip():` (line 613):

```python
        yield f"data: {json.dumps({'ai_answer': ai_answer})}\n\n"

        # ── Step 2b: Karakalpak NER ──
        entities_html_kaa = ''
        if api_key.strip():
            try:
                problem_kaa = tahrirchi_translate(problem, 'rus_Cyrl', 'kaa_Latn', api_key, model)
                entities_html_kaa = extract_and_highlight_entities(problem_kaa)
            except Exception as _ner_kaa_err:
                print(f"[SOLVE] kaa NER failed: {_ner_kaa_err}")

        if not is_success or not api_key.strip():
            ...
```

- [ ] **Step 3: Verify server change manually**

Start the server: `python server.py`

POST a test request:
```bash
curl -s -X POST http://localhost:5000/api/solve \
  -H "Content-Type: application/json" \
  -d '{"problem":"Купили 5 кг яблок по 120 рублей. Сколько заплатили?","lang":"ru"}' | grep -o 'entities_kaa[^}]*'
```
Expected: output contains `entities_kaa` key with non-empty HTML string.

- [ ] **Step 4: Commit**
```bash
git add server.py
git commit -m "feat: add Karakalpak NER generation in solve endpoint"
```

---

## Task 2: Frontend — Store and display Karakalpak NER

**Files:**
- Modify: `app.js:26` (globals)
- Modify: `app.js:320-326` (clearResult)
- Modify: `app.js:335-374` (showNERContainer)
- Modify: `app.js:420-428` (SSE handler in solveClicked)

### Step 1: Add `nerEntitiesKaaHTML` global

- [ ] Find this line in `app.js` (line 26):
```js
let nerEntitiesHTML = ''; // NER results storage
```
Replace with:
```js
let nerEntitiesHTML    = ''; // NER results storage (Russian)
let nerEntitiesKaaHTML = ''; // NER results storage (Karakalpak)
```

### Step 2: Reset in `clearResult()`

- [ ] Find in `app.js` (around line 323):
```js
  nerEntitiesHTML = '';
```
Replace with:
```js
  nerEntitiesHTML    = '';
  nerEntitiesKaaHTML = '';
```

### Step 3: Update `showNERContainer()` to show kaa block

- [ ] Find `showNERContainer()` (lines 335-374). Replace the entire function with:

```js
function showNERContainer() {
  if (!nerEntitiesHTML && !nerEntitiesKaaHTML) return;

  const resultCard = $('resultCard');
  let nerContainer = resultCard.querySelector('.ner-container');
  if (nerContainer) nerContainer.remove();

  const lang = _kbLang || localStorage.getItem('lang') || 'ru';
  let nerTitle = 'Структурный анализ текста (NER):';
  if (lang === 'uz') nerTitle = 'Matnning strukturaviy tahlili (NER):';
  else if (lang === 'kaa') nerTitle = 'Mátnniń strukturaviy tahlili (NER):';

  nerContainer = document.createElement('div');
  nerContainer.className = 'ner-container notranslate';
  nerContainer.setAttribute('translate', 'no');

  const titleElement = document.createElement('div');
  titleElement.className = 'ner-title';
  titleElement.textContent = nerTitle;
  nerContainer.appendChild(titleElement);

  if (nerEntitiesHTML) {
    const contentEl = document.createElement('div');
    contentEl.className = 'ner-content';
    contentEl.innerHTML = nerEntitiesHTML;
    nerContainer.appendChild(contentEl);
  }

  if (nerEntitiesKaaHTML) {
    const kaaLabel = document.createElement('div');
    kaaLabel.className = 'ner-title';
    kaaLabel.style.marginTop = '8px';
    kaaLabel.textContent = 'Karakalpaqsha (NER):';
    nerContainer.appendChild(kaaLabel);

    const kaaContent = document.createElement('div');
    kaaContent.className = 'ner-content';
    kaaContent.innerHTML = nerEntitiesKaaHTML;
    nerContainer.appendChild(kaaContent);
  }

  resultCard.insertBefore(nerContainer, resultCard.firstChild);
}
```

### Step 4: Handle `evt.entities_kaa` in SSE event handler

- [ ] Find in `solveClicked()` (around line 420-428):
```js
          if (evt.entities)  { nerEntitiesHTML = evt.entities; showNERContainer(); }
```
Add after it:
```js
          if (evt.entities_kaa) { nerEntitiesKaaHTML = evt.entities_kaa; }
```

Then find where `evt.recognized_entities` is handled (around line 424-427):
```js
            if (evt.recognized_entities) { 
              nerEntitiesHTML = evt.recognized_entities; 
              showNERContainer(); 
            }
```
Replace with:
```js
            if (evt.recognized_entities) { 
              nerEntitiesHTML = evt.recognized_entities;
            }
            if (evt.entities_kaa !== undefined) {
              nerEntitiesKaaHTML = evt.entities_kaa || '';
            }
            if (evt.recognized_entities || evt.entities_kaa !== undefined) {
              showNERContainer();
            }
```

- [ ] **Verify in browser:** Solve a problem → wait for result → both Russian NER block and Karakalpak NER block appear under "Структурный анализ текста (NER):".

- [ ] **Commit:**
```bash
git add app.js
git commit -m "feat: display Karakalpak NER block in solver results"
```

---

## Task 3: Frontend — `translateToKaa` helper

**Files:**
- Modify: `app.js` — add helper after `translatorCopy()` function (around line 1153)

- [ ] **Add `translateToKaa` function** after `translatorCopy()` closing `}`:

```js
async function translateToKaa(texts) {
  try {
    const r = await fetch('/api/translate', {
      method:  'POST',
      headers: { 'Content-Type': 'application/json' },
      body:    JSON.stringify({ texts, source_lang: 'rus_Cyrl', target_lang: 'kaa_Latn', model: 'tilmoch' }),
    });
    if (!r.ok) return null;
    const data = await r.json();
    return Array.isArray(data.translations) ? data.translations : null;
  } catch (_) {
    return null;
  }
}
```

- [ ] **Commit:**
```bash
git add app.js
git commit -m "feat: add translateToKaa helper in app.js"
```

---

## Task 4: Frontend — Karakalpak feedback in `checkTaskInput`

**Files:**
- Modify: `app.js:1165-1195` (`checkTaskInput` function)

### Context

Currently `checkTaskInput` (line 1165):
```js
  feedback.textContent = correct
    ? (_tasksLang === 'uz' ? '✓ Toʻgʻri!' : '✓ Правильно!')
    : (_tasksLang === 'uz' ? '✗ Notoʻgʻri' : '✗ Неверно');
```
And hint (line 1193):
```js
    if (hint) { hint.textContent = _taskFieldText(p, 'hint'); hint.style.display = 'block'; }
```

- [ ] **Replace the feedback + hint section in `checkTaskInput`**

Find this block (lines 1180-1194):
```js
  input.className    = 'practice-input ' + (correct ? 'correct' : 'wrong');
  feedback.style.display = 'block';
  feedback.textContent   = correct
    ? (_tasksLang === 'uz' ? '✓ Toʻgʻri!' : '✓ Правильно!')
    : (_tasksLang === 'uz' ? '✗ Notoʻgʻri' : '✗ Неверно');
  feedback.className     = 'practice-feedback ' + (correct ? 'correct' : 'wrong');

  if (correct) {
    _taskCorrect++;
    _taskAnswered.add(_taskProbIdx);
    showTaskNext();
  } else if (p.hint) {
    const hint = $('practiceHint');
    if (hint) { hint.textContent = _taskFieldText(p, 'hint'); hint.style.display = 'block'; }
  }
```

Replace with:
```js
  input.className        = 'practice-input ' + (correct ? 'correct' : 'wrong');
  feedback.style.display = 'block';
  const feedbackRu       = correct
    ? (_tasksLang === 'uz' ? '✓ Toʻgʻri!' : '✓ Правильно!')
    : (_tasksLang === 'uz' ? '✗ Notoʻgʻri' : '✗ Неверно');
  feedback.textContent = feedbackRu;
  feedback.className   = 'practice-feedback ' + (correct ? 'correct' : 'wrong');

  if (correct) {
    _taskCorrect++;
    _taskAnswered.add(_taskProbIdx);
    showTaskNext();
  } else if (p.hint) {
    const hint = $('practiceHint');
    if (hint) { hint.textContent = _taskFieldText(p, 'hint'); hint.style.display = 'block'; }
  }

  if (_tasksLang === 'kaa') {
    const hintText  = (!correct && p.hint) ? _taskFieldText(p, 'hint') : '';
    const toTranslate = hintText ? [feedbackRu, hintText] : [feedbackRu];
    translateToKaa(toTranslate).then(translations => {
      if (!translations) return;
      feedback.textContent = translations[0] || feedbackRu;
      if (hintText && translations[1]) {
        const hint = $('practiceHint');
        if (hint) hint.textContent = translations[1];
      }
    });
  }
```

- [ ] **Verify in browser:**
  1. Switch language to kaa in Másele tab (button "Qar.")
  2. Select a topic → enter a correct answer
  3. Feedback shows Russian "✓ Правильно!" first, then switches to Karakalpak (1-2 sec)
  4. Enter wrong answer with a hint → hint text also appears in Karakalpak

- [ ] **Commit:**
```bash
git add app.js
git commit -m "feat: translate checkTaskInput feedback to Karakalpak"
```

---

## Task 5: Frontend — Karakalpak hint in `checkTaskChoice`

**Files:**
- Modify: `app.js:1197-1215` (`checkTaskChoice` function)

### Context

Currently `checkTaskChoice` (line 1211-1213):
```js
  } else if (p.hint) {
    const hint = $('practiceHint');
    if (hint) { hint.textContent = _taskFieldText(p, 'hint'); hint.style.display = 'block'; }
  }
```

`checkTaskChoice` has no text feedback for correct/wrong — only CSS classes on buttons. So we only need to translate the hint.

- [ ] **Replace the hint section in `checkTaskChoice`**

Find (lines 1207-1214):
```js
  if (correct) {
    _taskCorrect++;
    _taskAnswered.add(_taskProbIdx);
    showTaskNext();
  } else if (p.hint) {
    const hint = $('practiceHint');
    if (hint) { hint.textContent = _taskFieldText(p, 'hint'); hint.style.display = 'block'; }
  }
```

Replace with:
```js
  if (correct) {
    _taskCorrect++;
    _taskAnswered.add(_taskProbIdx);
    showTaskNext();
  } else if (p.hint) {
    const hint = $('practiceHint');
    const hintText = _taskFieldText(p, 'hint');
    if (hint) { hint.textContent = hintText; hint.style.display = 'block'; }
    if (_tasksLang === 'kaa') {
      translateToKaa([hintText]).then(translations => {
        if (translations && translations[0] && hint) hint.textContent = translations[0];
      });
    }
  }
```

- [ ] **Verify in browser:**
  1. Switch language to kaa in Másele tab
  2. Select a topic → choose a wrong answer on a multiple-choice question that has a hint
  3. Hint appears in Russian, then switches to Karakalpak (1-2 sec)
  4. Choose correct answer → no hint, no crash

- [ ] **Commit:**
```bash
git add app.js
git commit -m "feat: translate checkTaskChoice hint to Karakalpak"
```

---

## Final Verification Checklist

- [ ] Open browser DevTools → Console — no JS errors
- [ ] Solver: type `"Купили 5 кг яблок по 120 рублей"` → solve → Russian NER appears, then Karakalpak NER below it
- [ ] Numbers (`5`, `120`) and units (`кг`, `рублей`) appear in NER tags in both blocks, NOT translated
- [ ] Másele (kaa mode): correct answer → feedback switches to Karakalpak in ~1-2 sec
- [ ] Másele (kaa mode): wrong answer + hint → hint switches to Karakalpak in ~1-2 sec
- [ ] Másele (kaa mode): wrong answer without hint → no crash
- [ ] Másele (ru mode): feedback stays in Russian (kaa translation NOT triggered)
- [ ] No API key configured → kaa NER is empty, only Russian NER shown (no crash)
- [ ] Tahrirchi unavailable → Russian feedback stays, no crash (null check in `.then()`)
