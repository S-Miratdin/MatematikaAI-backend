# Tasks Tab (Mashq) — Design Spec
**Date:** 2026-05-04
**Project:** MatematikaAI (d:\Project\Matmozg)
**Stack:** Python Flask + SQLite + HTML/CSS/JS

---

## 1. Goals

Add a dedicated **Mashq** tab for practising mathematical problems. Currently problems are buried inside the Baza tab (collapsible section at the bottom of a topic detail view). The new tab makes them prominent and adds a proper training flow with progress tracking.

---

## 2. User Flow

```
[Topic list] → [Practice session] → [Results screen]
      ↑___________________________________|
                 (Retry or back)
```

Three screens inside a single `#panel-tasks` section; no page reloads.

---

## 3. Screen 1 — Topic List

### Layout
- Vertical scrollable list grouped by category headers (Планиметрия, Алгебра, etc.)
- Only topics that have at least one problem in the DB are shown
- Topics without problems are silently skipped

### Each topic row
```
[difficulty dot]  Topic title            X/N   [Button]
                  [████████░░░░░░] 57%
```

| Element | Detail |
|---|---|
| Difficulty dot | 🟢 easy · 🟡 medium · 🔴 hard (from `topic.difficulty`) |
| Progress bar | filled portion = correct / total |
| Counter | "X/N" — solved correctly out of total |
| Button label | "Boshlash" when progress < 100%, "✓ Pройдено" when all correct |

### Progress source
Stored in `localStorage` under key `mashq_progress`:
```json
{
  "topic-id-1": { "correct": 3, "total": 7 },
  "topic-id-2": { "correct": 7, "total": 7 }
}
```
Progress is only updated if the new result is better than the stored one (correct count improves).

---

## 4. Screen 2 — Practice Session

### Header bar
```
[← Назад]   Topic name   [● ● ● ○ ○ ○ ○]   3 / 7
```
- Back arrow returns to topic list (progress already saved)
- Dots = one per problem; filled = answered, empty = remaining; clickable to jump

### Two-column layout
| Left column | Right column |
|---|---|
| Problem counter (N / Total) | Formula box from topic (`topic.formula`) |
| Question text | Hint (hidden by default, shown after wrong answer) |
| Answer area (input or choice buttons) | |
| "Проверить" / "Следующая →" button | |

### Answer types (reuse existing logic from Baza tab)
- **input** — text field, normalize + numeric tolerance check (same as `kbCheckInput`)
- **choice** — grid of buttons, highlight correct green / wrong red on click (same as `kbCheckChoice`)

### Feedback
- Correct: field/button turns green, "✓ Правильно!" message appears, "Следующая →" button appended
- Wrong: field/button turns red, "✗ Неверно" message, hint revealed from right column
- After last problem: auto-advance to Results screen

### Mobile
Right column (formula + hint) stacks below the question (single column).

---

## 5. Screen 3 — Results

```
★ ★ ★ ☆ ☆

Topic name

  5 из 7 правильных

[████████████░░░░] 71%

[  Ещё раз  ]  [  Другая тема  ]
```

### Star rating
| Correct | Stars |
|---|---|
| All correct, no mistakes | ★★★★★ |
| ≥ 85% | ★★★★☆ |
| ≥ 70% | ★★★☆☆ |
| ≥ 50% | ★★☆☆☆ |
| < 50% | ★☆☆☆☆ |

### Actions
- **Ещё раз** — shuffle problems array, restart session for same topic
- **Другая тема** — return to topic list (Screen 1)

### Progress update rule
`localStorage` entry for this topic is updated only if `correct` count is strictly greater than the stored value.

---

## 6. Data Flow

```
loadTasksTab()
  └─ fetch /api/kb/meta          → category list
  └─ fetch /api/topics           → all topics (with difficulty, formula)
  └─ read localStorage           → progress map
  └─ render topic list

selectTopic(topicId)
  └─ fetch /api/problems/:id     → problems array
  └─ shuffle problems
  └─ render practice screen

finishSession(correct, total)
  └─ update localStorage if improved
  └─ render results screen
```

No new backend endpoints required. All existing APIs (`/api/topics`, `/api/problems/:id`, `/api/kb/meta`) are already available.

---

## 7. Files Changed

| File | Change |
|---|---|
| `index.html` | Add `<button data-tab="tasks">` in `.tab-nav`; add `<section id="panel-tasks">` |
| `app.js` | Add `loadTasksTab`, `selectTopic`, `buildPracticeScreen`, `buildProblem`, `checkTaskInput`, `checkTaskChoice`, `showTaskNext`, `finishSession`, `renderTaskResults`, progress helpers |
| `style.css` | Add `.tasks-*`, `.practice-*`, `.results-*` classes; mobile overrides |

---

## 8. CSS Design Tokens (follow existing patterns)

- Background: `var(--surface)` / `var(--surface2)`
- Accent: `var(--accent)` (#4a9eff)
- Correct: `var(--green)`
- Wrong: `var(--red)`
- Font UI: `var(--font-ui)` (Manrope)
- Font mono: `var(--font-mono)` (DM Mono) — for formula box
- Border radius: `var(--radius-sm)`

---

## 9. Out of Scope

- Server-side progress storage (localStorage only)
- Adding or editing problems from this tab (admin flow stays in Baza tab)
- Timed challenges or leaderboards
