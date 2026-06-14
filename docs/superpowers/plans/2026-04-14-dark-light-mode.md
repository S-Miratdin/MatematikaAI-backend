# Dark / Light Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a toggleable dark/light theme with a text button in the top bar, persisted in `localStorage`.

**Architecture:** CSS `data-theme` attribute on `<html>` controls the active theme. Light theme variables are defined in `:root[data-theme="light"]`. Three JS functions (`initTheme`, `toggleTheme`, `applyTheme`) handle switching and persistence. No new dependencies.

**Tech Stack:** Vanilla HTML/CSS/JS, CSS custom properties, `localStorage`

---

## File Map

| File | Action | What changes |
|------|--------|-------------|
| `style.css` | Modify | Add `:root[data-theme="light"]` block; fix hardcoded canvas bg; add `.theme-toggle` margin rule |
| `index.html` | Modify | Add theme toggle `<button>` to `.top-bar` |
| `app.js` | Modify | Add `initTheme()`, `toggleTheme()`, `applyTheme()`; call `initTheme()` in `DOMContentLoaded` |

---

## Task 1: Add light theme CSS variables

**Files:**
- Modify: `style.css` — after the closing `}` of the existing `:root { ... }` block (line 33)

- [ ] **Step 1: Add the light theme variable block**

In `style.css`, after the closing `}` of `:root { ... }` (after line 33), insert:

```css
:root[data-theme="light"] {
  --bg:          #f5f6f8;
  --surface:     #ffffff;
  --surface2:    #eef0f4;
  --surface3:    #e1e4ea;
  --border:      rgba(0,0,0,0.08);
  --border2:     rgba(74,158,255,0.35);
  --text:        #1a1d27;
  --text2:       #5a6070;
  --text3:       #9098b0;
}
```

- [ ] **Step 2: Fix hardcoded canvas background**

In `style.css`, find `.anim-canvas` (around line 664–670):

```css
.anim-canvas {
  flex: 1;
  width: 100%;
  border-radius: var(--radius-sm);
  background: #0d1017;   /* ← hardcoded dark colour */
  display: block;
}
```

Replace the `background` value:

```css
.anim-canvas {
  flex: 1;
  width: 100%;
  border-radius: var(--radius-sm);
  background: var(--bg);
  display: block;
}
```

- [ ] **Step 3: Add margin rule for the toggle button**

At the end of the `.tab-nav` section in `style.css` (after line 122), add:

```css
.theme-toggle {
  margin-left: auto;
}
```

- [ ] **Step 4: Commit**

```bash
git add style.css
git commit -m "feat: add light theme CSS variables and fix hardcoded canvas bg"
```

---

## Task 2: Add toggle button to top bar HTML

**Files:**
- Modify: `index.html` — inside `.top-bar`, after the closing `</nav>` of `.tab-nav` (line 41)

- [ ] **Step 1: Insert the button**

In `index.html`, after `</nav>` (line 41) and before `</header>` (line 42), add:

```html
    <button class="btn-ghost theme-toggle" id="themeToggle" onclick="toggleTheme()">☀️ Светлая</button>
```

The full `.top-bar` section should now look like:

```html
  <header class="top-bar">
    <div class="top-bar-brand">
      <span class="brand-dot"></span>
      <span class="brand-name">MatematikaAI</span>
      <span class="brand-sub">Qaraqalpaqsha</span>
    </div>
    <nav class="tab-nav">
      <!-- ... tab buttons unchanged ... -->
    </nav>
    <button class="btn-ghost theme-toggle" id="themeToggle" onclick="toggleTheme()">☀️ Светлая</button>
  </header>
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add theme toggle button to top bar"
```

---

## Task 3: Add theme JS logic

**Files:**
- Modify: `app.js` — add three functions before the `DOMContentLoaded` block (before line 77); call `initTheme()` inside the existing `DOMContentLoaded` handler

- [ ] **Step 1: Add the three theme functions**

In `app.js`, immediately before the `document.addEventListener('DOMContentLoaded', ...)` block (before line 77), insert:

```js
// ════════════════════════════════════════════════════════════
//  THEME
// ════════════════════════════════════════════════════════════
function applyTheme(theme) {
  document.documentElement.setAttribute('data-theme', theme);
  localStorage.setItem('theme', theme);
  const btn = $('themeToggle');
  if (btn) btn.textContent = theme === 'dark' ? '☀️ Светлая' : '🌙 Тёмная';
}

function toggleTheme() {
  const current = document.documentElement.getAttribute('data-theme') || 'dark';
  applyTheme(current === 'dark' ? 'light' : 'dark');
}

function initTheme() {
  applyTheme(localStorage.getItem('theme') || 'dark');
}
```

- [ ] **Step 2: Call `initTheme()` in `DOMContentLoaded`**

Find the existing `DOMContentLoaded` handler (line 77–83):

```js
document.addEventListener('DOMContentLoaded', async () => {
  await loadConfig();
  await loadHistory();
  await loadKB();
  startAnimLoop();
});
```

Add `initTheme()` as the **first** call inside it:

```js
document.addEventListener('DOMContentLoaded', async () => {
  initTheme();
  await loadConfig();
  await loadHistory();
  await loadKB();
  startAnimLoop();
});
```

- [ ] **Step 3: Commit**

```bash
git add app.js
git commit -m "feat: add theme switching logic with localStorage persistence"
```

---

## Task 4: Manual verification

- [ ] **Step 1: Start the server**

```bash
python server.py
```

Open `http://localhost:5000` (or whichever port server.py uses) in a browser.

- [ ] **Step 2: Verify dark mode (default)**

Expected:
- Page loads in dark theme
- Button in top bar shows `☀️ Светлая`
- All cards, inputs, buttons have dark backgrounds

- [ ] **Step 3: Switch to light mode**

Click `☀️ Светлая`.

Expected:
- Page switches to light gray theme instantly
- Button now shows `🌙 Тёмная`
- Top bar, cards, inputs, history list, calculator buttons all have light backgrounds
- Animation canvas background is light gray (not hardcoded dark)

- [ ] **Step 4: Verify persistence**

Reload the page.

Expected:
- Light theme is still active (button shows `🌙 Тёмная`)

- [ ] **Step 5: Switch back to dark**

Click `🌙 Тёмная`.

Expected:
- Page returns to dark theme
- Button shows `☀️ Светлая`
- Reload preserves dark theme

- [ ] **Step 6: Final commit (if any fixes were needed)**

```bash
git add -p
git commit -m "fix: theme switching visual corrections"
```
