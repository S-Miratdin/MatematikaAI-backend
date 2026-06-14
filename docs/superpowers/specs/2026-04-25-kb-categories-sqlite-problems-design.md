# KB Categories, SQLite Migration, Interactive Problems — Design Spec

**Date:** 2026-04-25  
**Project:** MatematikaAI

---

## Summary

Three sequential improvements to the Knowledge Base:

- **A** — Add categories (school section, task type, grade) and difficulty levels to the existing JSON-based KB with UI filters
- **B** — Migrate storage to SQLite with a dual-mode layer (JSON fallback), hierarchy preserved, API interface unchanged
- **C** — Add interactive practice problems per topic with two answer formats (number input + multiple choice), checked client-side

---

## Step A — JSON Extension: Categories & Difficulty

### Data Schema

Each entry in `knowledge_base.json` gains four new optional fields:

```json
{
  "title": "Квадратное уравнение",
  "content": "...",
  "diagram": "parabola",
  "school_section": "Алгебра",
  "task_type": "Уравнения",
  "grade": "8-9 класс",
  "difficulty": "medium"
}
```

- `school_section`: one of `Арифметика`, `Алгебра`, `Геометрия`, `Тригонометрия`, `Анализ`
- `task_type`: one of `Уравнения`, `Фигуры`, `Функции`, `Прогрессии`, `Проценты`, `Дроби`, `Степени`
- `grade`: one of `5-6 класс`, `7-8 класс`, `9-10 класс`, `11 класс`
- `difficulty`: `easy` | `medium` | `hard` (default: `medium`)

Existing entries without these fields are treated as `difficulty: "medium"`, categories empty — no migration needed.

### API Changes

- `GET /api/kb` — unchanged, returns full list including new fields
- `POST /api/kb` — accepts new fields (all optional)
- `DELETE /api/kb/<idx>` — unchanged
- `GET /api/kb/meta` — NEW: returns all unique values for each category field, used by frontend to populate filter dropdowns dynamically

### UI Changes

**Filter bar** at the top of the Baza tab:
- Three `<select>` dropdowns: `Раздел` / `Тип задачи` / `Класс` — populated from `/api/kb/meta`
- Three difficulty toggle buttons: 🟢 Легко / 🟡 Средне / 🔴 Сложно (multi-select allowed)
- "Сбросить" button clears all filters
- Filtering is client-side only — no extra server requests

**Topic cards:**
- Colored difficulty badge next to the title: 🟢 / 🟡 / 🔴
- Cards without a difficulty field show no badge

**Add topic form:**
- Three new `<select>` fields for `school_section`, `task_type`, `grade` (all optional)
- Difficulty radio group: Легко / Средне / Сложно (default: Средне)

---

## Step B — SQLite Migration

### Database Schema (`matmozg.db`)

```sql
CREATE TABLE topics (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  title          TEXT NOT NULL,
  content        TEXT NOT NULL,
  diagram        TEXT DEFAULT 'none',
  school_section TEXT,
  task_type      TEXT,
  grade          TEXT,
  difficulty     TEXT DEFAULT 'medium',
  sort_order     INTEGER DEFAULT 0
);

CREATE TABLE problems (
  id        INTEGER PRIMARY KEY AUTOINCREMENT,
  topic_id  INTEGER NOT NULL REFERENCES topics(id) ON DELETE CASCADE,
  question  TEXT NOT NULL,
  type      TEXT NOT NULL CHECK(type IN ('input', 'choice')),
  answer    TEXT NOT NULL,
  choices   TEXT,   -- JSON array string, only for type='choice'
  hint      TEXT
);
```

### Dual-Mode Layer (`db.py`)

New file `db.py` contains all storage logic. At import time it checks:
- If `matmozg.db` exists → uses SQLite
- Otherwise → uses `knowledge_base.json`

Public functions (same signature regardless of backend):
- `get_topics() → list`
- `add_topic(entry: dict) → None`
- `delete_topic(idx_or_id: int) → None`
- `get_meta() → dict`
- `get_problems(topic_id: int) → list`
- `add_problem(entry: dict) → None`

`server.py` imports only from `db.py` — no direct JSON/SQLite calls in route handlers.

### Migration Script (`migrate.py`)

Standalone script, run once: `python migrate.py`

1. Reads `knowledge_base.json`
2. Creates `matmozg.db` with schema above
3. Inserts all topics, preserving all fields
4. Prints row count on success

`knowledge_base.json` is NOT deleted — kept as backup. After migration, server automatically switches to SQLite on next start.

### API Interface

All existing endpoints keep identical signatures. No frontend changes required for this step.

---

## Step C — Interactive Practice Problems

### Problem Data Format

**Input type** (numeric answer):
```json
{
  "topic_id": 1,
  "question": "Найди корни уравнения x² - 5x + 6 = 0",
  "type": "input",
  "answer": "2,3",
  "hint": "D = b² - 4ac = 25 - 24 = 1"
}
```

**Choice type** (multiple choice):
```json
{
  "topic_id": 1,
  "question": "Чему равен дискриминант для x² + 2x + 1 = 0?",
  "type": "choice",
  "answer": "0",
  "choices": ["1", "0", "-1", "4"],
  "hint": "D = b² - 4ac = 4 - 4 = 0"
}
```

For `input` type: answer may be multiple numbers separated by commas (e.g. `"2,3"`). Order is ignored. Comparison uses tolerance ±0.01.

### API

- `GET /api/problems/<topic_id>` — returns all problems for the topic
- `POST /api/problems` — add a problem (for seeding the database)

Answer validation happens **client-side** — answers are included in the data returned by `GET /api/problems/<topic_id>`. No separate validation endpoint needed.

### UI

Each topic card gains a **"Задачи (N)"** button showing problem count. Clicking it toggles an accordion panel below the card content.

Inside the panel — one problem at a time:

**Input type:**
- Question text
- Text input field
- "Проверить" button → green border + "✓ Правильно!" or red border + "✗ Неверно" + hint text revealed

**Choice type:**
- Question text
- 4 buttons with answer options
- On click: correct option turns green, selected wrong option turns red + hint revealed

Both types show a **"Следующая задача →"** button after answering (or if multiple problems exist). A counter shows `1 / 3` progress.

If a topic has no problems, the "Задачи" button is not shown.

---

## Implementation Order

1. **Step A** — Extend JSON schema + update `server.py` (`/api/kb/meta`) + update `app.js` (filters, badges) + update `index.html` (filter bar, form fields) + tag all 25 existing topics in `knowledge_base.json`
2. **Step B** — Write `db.py` + write `migrate.py` + update `server.py` to use `db.py` + run migration + verify
3. **Step C** — Add problems table queries to `db.py` + add `/api/problems` routes to `server.py` + add accordion UI to `app.js` + seed 2-3 problems per topic in `migrate.py` or separately

Each step is independently deployable and testable.
