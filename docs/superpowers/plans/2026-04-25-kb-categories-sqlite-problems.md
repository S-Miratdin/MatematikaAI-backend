# KB Categories, SQLite Migration, Interactive Problems — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add category/difficulty metadata + filter UI to the KB, migrate user-added topics to SQLite, then add interactive practice problems with client-side answer checking.

**Architecture:** Step A tags the static `kb_data.js` topics with three new metadata fields and adds a filter bar. Step B creates `db.py` (dual-mode SQLite/JSON layer) + `migrate.py` and refactors `server.py` routes. Step C adds a problems table in SQLite, two new API routes, and an accordion problem UI in the KB detail panel. `kb_data.js` stays static throughout — it provides SVG diagrams; SQLite handles user-added topics and problems.

**Tech Stack:** Python 3 / Flask, SQLite3 (stdlib), vanilla JS, existing CSS variables (`--accent`, `--green`, `--amber`, `--red`)

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `kb_data.js` | Modify | Add `school_section`, `task_type`, `grade`, `difficulty` to all 23 topics |
| `index.html` | Modify | Add filter bar HTML above `#kbNav` |
| `app.js` | Modify | Add filter state, `kbSetMetaFilter`, `kbToggleDiff`, `kbResetFilters`, update `kbRenderNav` and `kbNavItem`, add problems UI functions |
| `style.css` | Modify | Add `.kb2-filter-bar`, `.kb2-diff-badge`, `.kb2-problems-*` styles |
| `db.py` | Create | Dual-mode storage layer (SQLite when `matmozg.db` exists, JSON otherwise) |
| `migrate.py` | Create | One-time script: `knowledge_base.json` → `matmozg.db`, seeds sample problems |
| `server.py` | Modify | Import `db`, remove direct JSON I/O from KB routes, add `/api/kb/meta` and `/api/problems` routes |

---

## ═══ PHASE A: Metadata + Filter UI ═══

---

### Task A1: Add metadata fields to all 23 topics in kb_data.js

**Files:**
- Modify: `kb_data.js` (lines 80–695, one block per topic)

- [ ] **Step 1: Add the four fields to each topic**

  Open `kb_data.js`. For **every** topic entry add `school_section`, `task_type`, `grade`, `difficulty` immediately after `category_uz`. Use the mapping table below. Pattern (shown for `right_triangle`):

  ```js
  // Before
  id: 'right_triangle',
  category: 'Планиметрия',   category_uz: 'Planimetriya',
  title: 'Прямоугольный треугольник',

  // After
  id: 'right_triangle',
  category: 'Планиметрия',   category_uz: 'Planimetriya',
  school_section: 'Геометрия', task_type: 'Фигуры', grade: '7-8 класс', difficulty: 'easy',
  title: 'Прямоугольный треугольник',
  ```

  **Full mapping table:**

  | id | school_section | task_type | grade | difficulty |
  |----|---------------|-----------|-------|------------|
  | right_triangle | Геометрия | Фигуры | 7-8 класс | easy |
  | triangle | Геометрия | Фигуры | 7-8 класс | easy |
  | circle | Геометрия | Фигуры | 7-8 класс | easy |
  | rectangle | Геометрия | Фигуры | 5-6 класс | easy |
  | trapezoid | Геометрия | Фигуры | 7-8 класс | medium |
  | parallelogram | Геометрия | Фигуры | 7-8 класс | medium |
  | cube | Геометрия | Фигуры | 9-10 класс | medium |
  | cylinder | Геометрия | Фигуры | 9-10 класс | medium |
  | cone | Геометрия | Фигуры | 9-10 класс | medium |
  | sphere | Геометрия | Фигуры | 9-10 класс | hard |
  | linear_eq | Алгебра | Уравнения | 7-8 класс | easy |
  | quadratic_eq | Алгебра | Уравнения | 8-9 класс | medium |
  | arith_prog | Алгебра | Прогрессии | 9-10 класс | medium |
  | geom_prog | Алгебра | Прогрессии | 9-10 класс | medium |
  | powers | Алгебра | Степени | 7-8 класс | medium |
  | linear_fn | Алгебра | Функции | 7-8 класс | easy |
  | parabola | Алгебра | Функции | 9-10 класс | medium |
  | hyperbola | Алгебра | Функции | 9-10 класс | medium |
  | exp_fn | Алгебра | Функции | 11 класс | hard |
  | unit_circle | Тригонометрия | Функции | 9-10 класс | hard |
  | trig_values | Тригонометрия | Функции | 9-10 класс | medium |
  | trig_formulas | Тригонометрия | Уравнения | 11 класс | hard |
  | sin_cos_theorems | Тригонометрия | Фигуры | 9-10 класс | hard |

- [ ] **Step 2: Verify no typos** — open browser, check console for JS errors, confirm all 23 topics load in the KB panel.

---

### Task A2: Add filter bar to index.html

**Files:**
- Modify: `index.html` (inside `<div class="kb2-left">`, after `<div class="kb2-lang-row">`)

- [ ] **Step 1: Insert the filter bar HTML**

  Find this line in `index.html`:
  ```html
          <div class="kb2-nav" id="kbNav"></div>
  ```

  Insert **before** it:
  ```html
          <div class="kb2-filter-bar" id="kbFilterBar">
            <select class="kb2-filter-select" id="kbFilterSection" onchange="kbSetMetaFilter('school_section', this.value)">
              <option value="">Все разделы</option>
              <option>Арифметика</option>
              <option>Алгебра</option>
              <option>Геометрия</option>
              <option>Тригонометрия</option>
              <option>Анализ</option>
            </select>
            <select class="kb2-filter-select" id="kbFilterType" onchange="kbSetMetaFilter('task_type', this.value)">
              <option value="">Все типы</option>
              <option>Уравнения</option>
              <option>Фигуры</option>
              <option>Функции</option>
              <option>Прогрессии</option>
              <option>Проценты</option>
              <option>Степени</option>
            </select>
            <select class="kb2-filter-select" id="kbFilterGrade" onchange="kbSetMetaFilter('grade', this.value)">
              <option value="">Все классы</option>
              <option>5-6 класс</option>
              <option>7-8 класс</option>
              <option>8-9 класс</option>
              <option>9-10 класс</option>
              <option>11 класс</option>
            </select>
            <div class="kb2-diff-btns">
              <button class="kb2-diff-btn easy"   title="Легко"   onclick="kbToggleDiff('easy')"  >🟢</button>
              <button class="kb2-diff-btn medium" title="Средне" onclick="kbToggleDiff('medium')">🟡</button>
              <button class="kb2-diff-btn hard"   title="Сложно" onclick="kbToggleDiff('hard')"  >🔴</button>
            </div>
            <button class="btn-ghost-sm" onclick="kbResetFilters()">✕</button>
          </div>
  ```

---

### Task A3: Add filter styles to style.css

**Files:**
- Modify: `style.css` (append at end of file)

- [ ] **Step 1: Append these rules**

  ```css
  /* ── KB Filter Bar ──────────────────────────────────────────── */
  .kb2-filter-bar {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    padding: 8px 10px;
    border-bottom: 1px solid var(--border);
  }

  .kb2-filter-select {
    flex: 1;
    min-width: 0;
    background: var(--surface2);
    border: 1px solid var(--border);
    border-radius: var(--radius-xs);
    color: var(--text);
    font-family: var(--font-ui);
    font-size: 11px;
    padding: 4px 6px;
    cursor: pointer;
  }

  .kb2-filter-select:focus { outline: none; border-color: var(--accent); }

  .kb2-diff-btns {
    display: flex;
    gap: 4px;
  }

  .kb2-diff-btn {
    background: var(--surface2);
    border: 1px solid var(--border);
    border-radius: var(--radius-xs);
    cursor: pointer;
    font-size: 14px;
    padding: 3px 6px;
    line-height: 1;
    transition: border-color .15s;
  }

  .kb2-diff-btn.active { border-color: var(--accent); background: var(--accent-soft); }

  /* ── KB Difficulty Badge (in nav + detail) ──────────────────── */
  .kb2-diff-badge {
    display: inline-block;
    width: 7px;
    height: 7px;
    border-radius: 50%;
    margin-right: 5px;
    flex-shrink: 0;
    vertical-align: middle;
  }

  .kb2-diff-badge.easy   { background: var(--green); }
  .kb2-diff-badge.medium { background: var(--amber); }
  .kb2-diff-badge.hard   { background: var(--red);   }

  /* ── KB Problems Accordion ──────────────────────────────────── */
  .kb2-problems-panel {
    margin-top: 16px;
    border-top: 1px solid var(--border);
    padding-top: 12px;
  }

  .kb2-problems-toggle {
    background: none;
    border: 1px solid var(--border);
    border-radius: var(--radius-xs);
    color: var(--text2);
    cursor: pointer;
    font-family: var(--font-ui);
    font-size: 12px;
    padding: 5px 10px;
  }

  .kb2-problems-toggle:hover { border-color: var(--accent); color: var(--text); }

  .kb2-problems-content { padding-top: 12px; }

  .kb2-problem-counter {
    font-size: 11px;
    color: var(--text3);
    margin-bottom: 8px;
  }

  .kb2-problem-question {
    font-size: 14px;
    color: var(--text);
    margin-bottom: 12px;
    line-height: 1.5;
  }

  .kb2-problem-input-row {
    display: flex;
    gap: 8px;
    margin-bottom: 8px;
  }

  .kb2-problem-input {
    flex: 1;
    background: var(--surface2);
    border: 1px solid var(--border);
    border-radius: var(--radius-xs);
    color: var(--text);
    font-family: var(--font-mono);
    font-size: 13px;
    padding: 6px 10px;
  }

  .kb2-problem-input.correct { border-color: var(--green); }
  .kb2-problem-input.wrong   { border-color: var(--red);   }
  .kb2-problem-input:focus   { outline: none; border-color: var(--accent); }

  .kb2-problem-feedback {
    font-size: 13px;
    font-weight: 600;
    margin-bottom: 6px;
  }

  .kb2-problem-feedback.correct { color: var(--green); }
  .kb2-problem-feedback.wrong   { color: var(--red);   }

  .kb2-problem-hint {
    font-size: 12px;
    color: var(--text2);
    background: var(--surface2);
    border-radius: var(--radius-xs);
    padding: 6px 10px;
    margin-top: 6px;
  }

  .kb2-choice-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 6px;
    margin-bottom: 8px;
  }

  .kb2-choice-btn {
    background: var(--surface2);
    border: 1px solid var(--border);
    border-radius: var(--radius-xs);
    color: var(--text);
    cursor: pointer;
    font-family: var(--font-ui);
    font-size: 13px;
    padding: 7px 10px;
    text-align: left;
    transition: border-color .15s;
  }

  .kb2-choice-btn:hover:not(:disabled) { border-color: var(--accent); }
  .kb2-choice-btn.correct { border-color: var(--green); background: var(--green-soft); color: var(--green); }
  .kb2-choice-btn.wrong   { border-color: var(--red);   background: var(--red-soft);   color: var(--red);   }
  .kb2-choice-btn:disabled { cursor: default; }

  .kb2-next-btn { margin-top: 10px; }
  ```

---

### Task A4: Add filter logic to app.js

**Files:**
- Modify: `app.js`

- [ ] **Step 1: Add filter state variable after existing state vars**

  Find this block in `app.js`:
  ```js
  let _kbActiveId = null;
  let _kbFilter   = '';
  let _kbLang     = 'ru';
  ```

  Replace with:
  ```js
  let _kbActiveId   = null;
  let _kbFilter     = '';
  let _kbLang       = 'ru';
  let _kbMetaFilter = { school_section: '', task_type: '', grade: '', difficulty: '' };
  ```

- [ ] **Step 2: Replace `kbRenderNav` with filter-aware version**

  Find the full `kbRenderNav` function and replace it:

  ```js
  function kbRenderNav(q) {
    const nav = $('kbNav');
    if (!nav) return;

    let items = KB_DATA;

    const mf = _kbMetaFilter;
    if (mf.school_section) items = items.filter(t => t.school_section === mf.school_section);
    if (mf.task_type)      items = items.filter(t => t.task_type      === mf.task_type);
    if (mf.grade)          items = items.filter(t => t.grade          === mf.grade);
    if (mf.difficulty)     items = items.filter(t => t.difficulty     === mf.difficulty);

    if (q) {
      items = items.filter(t =>
        kbT(t).toLowerCase().includes(q) || kbCat(t).toLowerCase().includes(q)
      );
    }

    if (!items.length) {
      nav.innerHTML = '<div class="kb2-empty">Ничего не найдено</div>';
      return;
    }

    if (q) {
      nav.innerHTML = items.map(t => kbNavItem(t)).join('');
      return;
    }

    const categories = _kbLang === 'uz' ? KB_CATEGORIES_UZ : KB_CATEGORIES;
    let html = '';
    for (let i = 0; i < KB_CATEGORIES.length; i++) {
      const group = items.filter(t => t.category === KB_CATEGORIES[i]);
      if (!group.length) continue;
      html += `<div class="kb2-cat-header">${esc(categories[i])}</div>`;
      html += group.map(t => kbNavItem(t)).join('');
    }
    nav.innerHTML = html || '<div class="kb2-empty">Ничего не найдено</div>';
  }
  ```

- [ ] **Step 3: Update `kbNavItem` to show difficulty badge**

  Find `kbNavItem` and replace:
  ```js
  function kbNavItem(t) {
    const active = t.id === _kbActiveId ? ' active' : '';
    const badge  = t.difficulty
      ? `<span class="kb2-diff-badge ${t.difficulty}"></span>`
      : '';
    return `<div class="kb2-nav-item${active}" onclick="kbSelect('${t.id}')">${badge}${esc(kbT(t))}</div>`;
  }
  ```

- [ ] **Step 4: Add filter control functions after `kbSetLang`**

  ```js
  function kbSetMetaFilter(key, val) {
    _kbMetaFilter[key] = val;
    kbRenderNav(_kbFilter);
  }

  function kbToggleDiff(level) {
    _kbMetaFilter.difficulty = _kbMetaFilter.difficulty === level ? '' : level;
    document.querySelectorAll('.kb2-diff-btn').forEach(b => b.classList.remove('active'));
    if (_kbMetaFilter.difficulty) {
      document.querySelector(`.kb2-diff-btn.${_kbMetaFilter.difficulty}`)
              .classList.add('active');
    }
    kbRenderNav(_kbFilter);
  }

  function kbResetFilters() {
    _kbMetaFilter = { school_section: '', task_type: '', grade: '', difficulty: '' };
    const sel = id => { const el = $(id); if (el) el.value = ''; };
    sel('kbFilterSection'); sel('kbFilterType'); sel('kbFilterGrade');
    document.querySelectorAll('.kb2-diff-btn').forEach(b => b.classList.remove('active'));
    kbRenderNav(_kbFilter);
  }
  ```

- [ ] **Step 5: Update `kbSelect` to show difficulty badge in the detail header**

  Find this line inside `kbSelect`:
  ```js
        <h2 class="kb2-detail-title">${esc(kbT(topic))}</h2>
  ```

  Replace with:
  ```js
        <h2 class="kb2-detail-title">${topic.difficulty ? `<span class="kb2-diff-badge ${topic.difficulty}"></span>` : ''}${esc(kbT(topic))}</h2>
  ```

- [ ] **Step 6: Open browser, test each filter dropdown and difficulty button, reset button**

  Expected: selecting "Геометрия" → only geometry topics; 🟢 button → only easy topics; ✕ → all topics restored.

- [ ] **Step 7: Commit Phase A**

  ```bash
  git add kb_data.js index.html app.js style.css
  git commit -m "feat: add school_section/task_type/grade/difficulty metadata to KB topics + filter bar UI"
  ```

---

## ═══ PHASE B: SQLite Migration ═══

---

### Task B1: Create db.py (dual-mode storage layer)

**Files:**
- Create: `db.py`

- [ ] **Step 1: Create the file**

  ```python
  import json, os, sqlite3

  BASE_DIR = os.path.dirname(os.path.abspath(__file__))
  DB_FILE  = os.path.join(BASE_DIR, 'matmozg.db')
  KB_FILE  = os.path.join(BASE_DIR, 'knowledge_base.json')

  USE_SQLITE = os.path.exists(DB_FILE)


  def _conn():
      c = sqlite3.connect(DB_FILE)
      c.row_factory = sqlite3.Row
      c.execute('PRAGMA foreign_keys = ON')
      return c


  def _ensure_schema():
      with _conn() as c:
          c.execute('''CREATE TABLE IF NOT EXISTS topics (
              id             INTEGER PRIMARY KEY AUTOINCREMENT,
              title          TEXT NOT NULL,
              content        TEXT NOT NULL,
              diagram        TEXT DEFAULT 'none',
              school_section TEXT,
              task_type      TEXT,
              grade          TEXT,
              difficulty     TEXT DEFAULT 'medium',
              sort_order     INTEGER DEFAULT 0
          )''')
          c.execute('''CREATE TABLE IF NOT EXISTS problems (
              id        INTEGER PRIMARY KEY AUTOINCREMENT,
              topic_id  TEXT NOT NULL,
              question  TEXT NOT NULL,
              type      TEXT NOT NULL CHECK(type IN ('input','choice')),
              answer    TEXT NOT NULL,
              choices   TEXT,
              hint      TEXT
          )''')


  if USE_SQLITE:
      _ensure_schema()


  # ── JSON helpers ──────────────────────────────────────────────

  def _load_json():
      if not os.path.exists(KB_FILE):
          return []
      try:
          with open(KB_FILE, encoding='utf-8') as f:
              text = f.read().strip()
          return json.loads(text) if text else []
      except Exception:
          return []


  def _save_json(data):
      with open(KB_FILE, 'w', encoding='utf-8') as f:
          json.dump(data, f, indent=2, ensure_ascii=False)


  # ── Topics ────────────────────────────────────────────────────

  def get_topics():
      if USE_SQLITE:
          with _conn() as c:
              rows = c.execute(
                  'SELECT * FROM topics ORDER BY sort_order, id'
              ).fetchall()
              return [dict(r) for r in rows]
      return _load_json()


  def add_topic(entry: dict):
      if USE_SQLITE:
          with _conn() as c:
              c.execute(
                  '''INSERT INTO topics
                     (title, content, diagram, school_section, task_type, grade, difficulty)
                     VALUES (?,?,?,?,?,?,?)''',
                  (entry.get('title'), entry.get('content'),
                   entry.get('diagram', 'none'),
                   entry.get('school_section'), entry.get('task_type'),
                   entry.get('grade'), entry.get('difficulty', 'medium'))
              )
      else:
          kb = _load_json()
          kb.append(entry)
          _save_json(kb)


  def delete_topic(idx_or_id: int):
      if USE_SQLITE:
          with _conn() as c:
              c.execute('DELETE FROM topics WHERE id = ?', (idx_or_id,))
      else:
          kb = _load_json()
          if 0 <= idx_or_id < len(kb):
              kb.pop(idx_or_id)
              _save_json(kb)


  def get_meta():
      if USE_SQLITE:
          with _conn() as c:
              def vals(col):
                  return [r[0] for r in c.execute(
                      f'SELECT DISTINCT {col} FROM topics WHERE {col} IS NOT NULL'
                  ).fetchall()]
              return {
                  'school_sections': vals('school_section'),
                  'task_types':      vals('task_type'),
                  'grades':          vals('grade'),
              }
      kb = _load_json()
      return {
          'school_sections': list({e.get('school_section') for e in kb if e.get('school_section')}),
          'task_types':      list({e.get('task_type')      for e in kb if e.get('task_type')}),
          'grades':          list({e.get('grade')          for e in kb if e.get('grade')}),
      }


  # ── Problems ──────────────────────────────────────────────────

  def get_problems(topic_id: str):
      if not USE_SQLITE:
          return []
      with _conn() as c:
          rows = c.execute(
              'SELECT * FROM problems WHERE topic_id = ?', (topic_id,)
          ).fetchall()
          result = []
          for r in rows:
              d = dict(r)
              if d.get('choices'):
                  d['choices'] = json.loads(d['choices'])
              result.append(d)
          return result


  def add_problem(entry: dict):
      if not USE_SQLITE:
          return
      choices = entry.get('choices')
      if isinstance(choices, list):
          choices = json.dumps(choices, ensure_ascii=False)
      with _conn() as c:
          c.execute(
              '''INSERT INTO problems (topic_id, question, type, answer, choices, hint)
                 VALUES (?,?,?,?,?,?)''',
              (str(entry['topic_id']), entry['question'], entry['type'],
               entry['answer'], choices, entry.get('hint'))
          )
  ```

---

### Task B2: Create migrate.py

**Files:**
- Create: `migrate.py`

- [ ] **Step 1: Create the file**

  ```python
  """
  Run once: python migrate.py
  Converts knowledge_base.json → matmozg.db and seeds sample practice problems.
  knowledge_base.json is kept as a backup — not deleted.
  """
  import json, os, sqlite3

  BASE_DIR = os.path.dirname(os.path.abspath(__file__))
  KB_FILE  = os.path.join(BASE_DIR, 'knowledge_base.json')
  DB_FILE  = os.path.join(BASE_DIR, 'matmozg.db')

  if os.path.exists(DB_FILE):
      print('matmozg.db already exists. Delete it first to re-run migration.')
      raise SystemExit(1)

  # ── Load source JSON ──────────────────────────────────────────
  topics = []
  if os.path.exists(KB_FILE):
      with open(KB_FILE, encoding='utf-8') as f:
          topics = json.load(f)

  # ── Create schema ─────────────────────────────────────────────
  conn = sqlite3.connect(DB_FILE)
  c = conn.cursor()

  c.execute('''CREATE TABLE topics (
      id             INTEGER PRIMARY KEY AUTOINCREMENT,
      title          TEXT NOT NULL,
      content        TEXT NOT NULL,
      diagram        TEXT DEFAULT 'none',
      school_section TEXT,
      task_type      TEXT,
      grade          TEXT,
      difficulty     TEXT DEFAULT 'medium',
      sort_order     INTEGER DEFAULT 0
  )''')

  c.execute('''CREATE TABLE problems (
      id        INTEGER PRIMARY KEY AUTOINCREMENT,
      topic_id  TEXT NOT NULL,
      question  TEXT NOT NULL,
      type      TEXT NOT NULL CHECK(type IN ('input','choice')),
      answer    TEXT NOT NULL,
      choices   TEXT,
      hint      TEXT
  )''')

  # ── Migrate topics ────────────────────────────────────────────
  for i, t in enumerate(topics):
      c.execute(
          '''INSERT INTO topics
             (title, content, diagram, school_section, task_type, grade, difficulty, sort_order)
             VALUES (?,?,?,?,?,?,?,?)''',
          (t.get('title', ''), t.get('content', ''),
           t.get('diagram', 'none'),
           t.get('school_section'), t.get('task_type'),
           t.get('grade'), t.get('difficulty', 'medium'), i)
      )

  # ── Seed sample problems ──────────────────────────────────────
  # These link to kb_data.js topic IDs (static JS), not the topics table.
  PROBLEMS = [
      # right_triangle
      {
          'topic_id': 'right_triangle',
          'question': 'Катеты треугольника a = 3, b = 4. Найди гипотенузу c.',
          'type': 'input', 'answer': '5',
          'hint': 'a² + b² = c²  →  9 + 16 = 25  →  c = √25 = 5',
      },
      {
          'topic_id': 'right_triangle',
          'question': 'Чему равен sin α, если противолежащий катет = 3, гипотенуза = 5?',
          'type': 'choice', 'answer': '0.6',
          'choices': ['0.8', '0.6', '0.75', '1.67'],
          'hint': 'sin α = противолежащий катет / гипотенуза = 3/5 = 0.6',
      },
      # quadratic_eq
      {
          'topic_id': 'quadratic_eq',
          'question': 'Найди корни уравнения x² − 5x + 6 = 0',
          'type': 'input', 'answer': '2,3',
          'hint': 'D = 25 − 24 = 1  →  x₁ = (5+1)/2 = 3,  x₂ = (5−1)/2 = 2',
      },
      {
          'topic_id': 'quadratic_eq',
          'question': 'Дискриминант уравнения x² + 2x + 1 = 0 равен?',
          'type': 'choice', 'answer': '0',
          'choices': ['4', '0', '-4', '2'],
          'hint': 'D = b² − 4ac = 4 − 4·1·1 = 0',
      },
      # linear_eq
      {
          'topic_id': 'linear_eq',
          'question': 'Реши уравнение 3x + 9 = 0',
          'type': 'input', 'answer': '-3',
          'hint': '3x = -9  →  x = -3',
      },
      # circle
      {
          'topic_id': 'circle',
          'question': 'Радиус круга r = 5. Найди площадь (π = 3.14, ответ округли до целых)',
          'type': 'input', 'answer': '78',
          'hint': 'S = π·r² = 3.14 × 25 = 78.5 ≈ 78',
      },
      {
          'topic_id': 'circle',
          'question': 'Длина окружности при r = 7 (π ≈ 3.14, округли до целых)?',
          'type': 'choice', 'answer': '44',
          'choices': ['22', '44', '154', '49'],
          'hint': 'C = 2πr = 2 × 3.14 × 7 = 43.96 ≈ 44',
      },
      # arith_prog
      {
          'topic_id': 'arith_prog',
          'question': 'a₁ = 2, d = 3. Найди 5-й член прогрессии.',
          'type': 'input', 'answer': '14',
          'hint': 'a₅ = a₁ + (5−1)·d = 2 + 4·3 = 14',
      },
  ]

  for p in PROBLEMS:
      choices = p.get('choices')
      if choices:
          choices = json.dumps(choices, ensure_ascii=False)
      c.execute(
          '''INSERT INTO problems (topic_id, question, type, answer, choices, hint)
             VALUES (?,?,?,?,?,?)''',
          (p['topic_id'], p['question'], p['type'],
           p['answer'], choices, p.get('hint'))
      )

  conn.commit()
  conn.close()

  print(f'Migration complete: {len(topics)} topics + {len(PROBLEMS)} sample problems → matmozg.db')
  ```

---

### Task B3: Refactor server.py KB routes to use db.py

**Files:**
- Modify: `server.py`

- [ ] **Step 1: Add import at the top of server.py, after existing imports**

  Find:
  ```python
  app = Flask(__name__, static_folder='.', static_url_path='')
  ```

  Insert **before** it:
  ```python
  import db as _db
  ```

- [ ] **Step 2: Replace the three KB routes**

  Find and delete the entire block:
  ```python
  @app.route('/api/kb', methods=['GET'])
  def get_kb():
      kb = load_json(KB_FILE, DEFAULT_KB)
      if _sync_arith_tables(kb):
          save_json(KB_FILE, kb)
      return jsonify(kb)

  @app.route('/api/kb', methods=['POST'])
  def add_kb():
      kb    = load_json(KB_FILE, DEFAULT_KB)
      entry = request.json or {}
      if not entry.get('title') or not entry.get('content'):
          return jsonify({'error': 'Nom va mazmun kerak'}), 400
      kb.append({'title': entry['title'], 'content': entry['content']})
      save_json(KB_FILE, kb)
      return jsonify({'ok': True})

  @app.route('/api/kb/<int:idx>', methods=['DELETE'])
  def delete_kb(idx):
      kb = load_json(KB_FILE, DEFAULT_KB)
      if 0 <= idx < len(kb):
          kb.pop(idx)
          save_json(KB_FILE, kb)
      return jsonify({'ok': True})
  ```

  Replace with:
  ```python
  @app.route('/api/kb', methods=['GET'])
  def get_kb():
      return jsonify(_db.get_topics())

  @app.route('/api/kb/meta', methods=['GET'])
  def get_kb_meta():
      return jsonify(_db.get_meta())

  @app.route('/api/kb', methods=['POST'])
  def add_kb():
      entry = request.json or {}
      if not entry.get('title') or not entry.get('content'):
          return jsonify({'error': 'Nom va mazmun kerak'}), 400
      _db.add_topic(entry)
      return jsonify({'ok': True})

  @app.route('/api/kb/<int:idx>', methods=['DELETE'])
  def delete_kb(idx):
      _db.delete_topic(idx)
      return jsonify({'ok': True})
  ```

- [ ] **Step 3: Run the migration and restart server**

  ```bash
  python migrate.py
  python server.py
  ```

  Expected output of migrate.py:
  ```
  Migration complete: 28 topics + 8 sample problems → matmozg.db
  ```
  (number of topics will vary based on knowledge_base.json content)

- [ ] **Step 4: Verify GET /api/kb returns JSON, server starts without errors**

  ```bash
  curl http://localhost:10000/api/kb
  curl http://localhost:10000/api/kb/meta
  ```

  Both should return valid JSON (not errors).

- [ ] **Step 5: Commit Phase B**

  ```bash
  git add db.py migrate.py server.py
  git commit -m "feat: add SQLite dual-mode storage layer via db.py, migrate knowledge_base.json to matmozg.db"
  ```

---

## ═══ PHASE C: Interactive Practice Problems ═══

---

### Task C1: Add problems API routes to server.py

**Files:**
- Modify: `server.py` (after the KB routes block)

- [ ] **Step 1: Add two new routes**

  ```python
  # ─────────────────────────────────────────────
  # ROUTES — Problems
  # ─────────────────────────────────────────────
  @app.route('/api/problems/<string:topic_id>', methods=['GET'])
  def get_problems(topic_id):
      return jsonify(_db.get_problems(topic_id))

  @app.route('/api/problems', methods=['POST'])
  def add_problem():
      entry = request.json or {}
      required = ('topic_id', 'question', 'type', 'answer')
      if not all(entry.get(k) for k in required):
          return jsonify({'error': 'topic_id, question, type, answer required'}), 400
      if entry['type'] not in ('input', 'choice'):
          return jsonify({'error': "type must be 'input' or 'choice'"}), 400
      _db.add_problem(entry)
      return jsonify({'ok': True})
  ```

- [ ] **Step 2: Verify**

  ```bash
  curl http://localhost:10000/api/problems/right_triangle
  ```

  Expected: JSON array with 2 problems seeded by migrate.py.

---

### Task C2: Add problem UI logic to app.js

**Files:**
- Modify: `app.js` (add a new section after `kbSelect` function)

- [ ] **Step 1: Add problems state variables** in the KB state block

  Find:
  ```js
  let _kbMetaFilter = { school_section: '', task_type: '', grade: '', difficulty: '' };
  ```

  Add after it:
  ```js
  let _kbProblems   = [];
  let _kbProbIdx    = 0;
  ```

- [ ] **Step 2: Update `kbSelect` to add `#kbProblemsSection` placeholder and trigger load**

  At the **end** of the `detail.innerHTML = \`...\`` template string inside `kbSelect`, before the closing backtick, add inside `.kb2-detail-inner`:

  Find:
  ```js
        ${topic.svg ? `<div class="kb2-diagram">${topic.svg}</div>` : ''}
      </div>
    `;
  ```

  Replace with:
  ```js
        ${topic.svg ? `<div class="kb2-diagram">${topic.svg}</div>` : ''}
        <div id="kbProblemsSection"></div>
      </div>
    `;

    _kbProblems = [];
    _kbProbIdx  = 0;
    kbLoadProblems(id);
  ```

- [ ] **Step 3: Add the four problem functions after `kbSelect`**

  ```js
  async function kbLoadProblems(topicId) {
    try {
      const r = await fetch(`/api/problems/${topicId}`);
      const problems = await r.json();
      if (!problems.length) return;
      _kbProblems = problems;
      _kbProbIdx  = 0;
      const sec = $('kbProblemsSection');
      if (!sec) return;
      sec.innerHTML = `
        <div class="kb2-problems-panel">
          <button class="kb2-problems-toggle" onclick="kbToggleProblems(this)">
            Задачи (${problems.length}) ▾
          </button>
          <div class="kb2-problems-content" id="kbProblemsContent" style="display:none"></div>
        </div>`;
    } catch (_) {}
  }

  function kbToggleProblems(btn) {
    const content = $('kbProblemsContent');
    if (!content) return;
    const visible = content.style.display !== 'none';
    content.style.display = visible ? 'none' : 'block';
    btn.textContent = `Задачи (${_kbProblems.length}) ${visible ? '▾' : '▸'}`;
    if (!visible && !content.innerHTML) {
      content.innerHTML = kbBuildProblem(_kbProbIdx);
    }
  }

  function kbBuildProblem(idx) {
    const p = _kbProblems[idx];
    const counter = `<div class="kb2-problem-counter">${idx + 1} / ${_kbProblems.length}</div>`;
    const hint    = `<div class="kb2-problem-hint" id="probHint" style="display:none">${esc(p.hint || '')}</div>`;

    if (p.type === 'input') {
      return `
        ${counter}
        <div class="kb2-problem-question">${esc(p.question)}</div>
        <div class="kb2-problem-input-row">
          <input type="text" class="kb2-problem-input" id="probInput" placeholder="Ответ...">
          <button class="btn-primary" style="white-space:nowrap" onclick="kbCheckInput()">Проверить</button>
        </div>
        <div class="kb2-problem-feedback" id="probFeedback" style="display:none"></div>
        ${hint}
      `;
    }

    const opts = p.choices.map(ch =>
      `<button class="kb2-choice-btn" onclick="kbCheckChoice(this,'${esc(ch)}')">${esc(ch)}</button>`
    ).join('');
    return `
      ${counter}
      <div class="kb2-problem-question">${esc(p.question)}</div>
      <div class="kb2-choice-grid">${opts}</div>
      ${hint}
    `;
  }

  function kbCheckInput() {
    const p       = _kbProblems[_kbProbIdx];
    const input   = $('probInput');
    const feedback = $('probFeedback');
    if (!input || !feedback) return;

    const normalize = s => s.trim().toLowerCase()
      .split(',').map(x => x.trim()).filter(Boolean).sort();
    const given    = normalize(input.value);
    const expected = normalize(p.answer);

    const correct = given.length === expected.length && given.every((v, i) => {
      const a = parseFloat(v), b = parseFloat(expected[i]);
      return !isNaN(a) && !isNaN(b) ? Math.abs(a - b) < 0.01 : v === expected[i];
    });

    input.className = 'kb2-problem-input ' + (correct ? 'correct' : 'wrong');
    feedback.style.display = 'block';
    feedback.textContent   = correct ? '✓ Правильно!' : '✗ Неверно';
    feedback.className     = 'kb2-problem-feedback ' + (correct ? 'correct' : 'wrong');
    if (!correct && p.hint) $('probHint').style.display = 'block';
    if (correct) kbShowNext();
  }

  function kbCheckChoice(btn, chosen) {
    const p = _kbProblems[_kbProbIdx];
    const correct = chosen === p.answer;
    btn.closest('.kb2-choice-grid').querySelectorAll('.kb2-choice-btn').forEach(b => {
      b.disabled = true;
      if (b.textContent.trim() === p.answer) b.classList.add('correct');
    });
    if (!correct) btn.classList.add('wrong');
    if (p.hint) $('probHint').style.display = 'block';
    if (correct) kbShowNext();
  }

  function kbShowNext() {
    const content = $('kbProblemsContent');
    if (!content || _kbProbIdx >= _kbProblems.length - 1) return;
    const btn = document.createElement('button');
    btn.className   = 'btn-ghost-sm kb2-next-btn';
    btn.textContent = 'Следующая задача →';
    btn.onclick = () => {
      _kbProbIdx++;
      content.innerHTML = kbBuildProblem(_kbProbIdx);
    };
    content.appendChild(btn);
  }
  ```

- [ ] **Step 4: Open browser, select a topic that has problems (e.g. "Прямоугольный треугольник"), click "Задачи (2)" button**

  Expected:
  - Panel expands showing question 1 of 2
  - Input type: type wrong answer → red border + hint shown; type correct answer → green + "Следующая задача →" appears
  - Choice type: click wrong option → red + correct goes green; click correct → green

- [ ] **Step 5: Test a topic with no problems (e.g. "Гипербола") — "Задачи" button should not appear**

- [ ] **Step 6: Commit Phase C**

  ```bash
  git add server.py app.js
  git commit -m "feat: add interactive practice problems — accordion UI with input/choice answer checking"
  ```

---

## Self-Review Checklist

- [x] All 23 kb_data.js topics tagged with metadata (Task A1)
- [x] Filter bar HTML matches `kbSetMetaFilter` / `kbToggleDiff` / `kbResetFilters` function signatures
- [x] `_kbMetaFilter` state variable declared before functions that use it
- [x] `kbNavItem` uses `t.id` (not `t.title`) for `onclick` — matches existing pattern
- [x] `db.py` `_ensure_schema()` uses `CREATE TABLE IF NOT EXISTS` — idempotent
- [x] `delete_topic` in SQLite mode uses row `id` (integer PK), not list index
- [x] `get_problems` in JSON mode returns `[]` — correct, problems require SQLite
- [x] Problem `topic_id` is TEXT matching kb_data.js string IDs (`'right_triangle'`, etc.)
- [x] `kbCheckInput` handles comma-separated answers with ±0.01 tolerance
- [x] `kbCheckChoice` disables all buttons after selection to prevent double-click
- [x] `kbShowNext` only appends button when there ARE more problems
