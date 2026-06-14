# Home Screen (Главное меню) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a start screen with 4 cards (Másele, Kalkulyator, Baza, Shınıǵıw) that appears on first visit; after a section is chosen, the normal tab UI is shown and the choice is remembered for future visits.

**Architecture:** A new `#panel-home` section (first in `.panels`) plus CSS rules driven by a `home-pending` class on `<html>` (set synchronously before paint, mirroring the existing theme script) and a `home-mode` class on `.app-shell`. `app.js` gains a shared `selectTab(tab)` function used by both `.tab-btn` clicks and new `.home-card` clicks, plus an `initHomeOrLastTab()` that decides on load whether to show the home screen or restore `localStorage.lastTab`.

**Tech Stack:** Plain HTML/CSS/JS (no build step, no JS test framework — verification is manual via a local static server).

---

## File Structure

- Modify `index.html`: add a pre-paint inline script for `home-pending`, add `#panel-home` markup (hero text + 4 `.home-card` buttons reusing existing tab SVG icons).
- Modify `style.css`: add a "HOME SCREEN" section with `home-pending`/`home-mode` visibility rules and `.home-cards`/`.home-card`/`.home-hero` styles.
- Modify `app.js`: refactor the tab-click handler into `selectTab(tab)`, wire up `.home-card` clicks, add `initHomeOrLastTab()` called from the `DOMContentLoaded` handler.

No new files. No backend changes.

---

### Task 1: Add `#panel-home` markup to `index.html`

**Files:**
- Modify: `index.html:12` (add pre-paint script)
- Modify: `index.html:58-60` (insert new panel before `panel-solver`)

- [ ] **Step 1: Add the `home-pending` pre-paint script**

In `index.html`, immediately after line 12
(`<script>document.documentElement.setAttribute('data-theme', localStorage.getItem('theme') || 'dark');</script>`),
add a new line:

```html
<script>if (!localStorage.getItem('lastTab')) document.documentElement.classList.add('home-pending');</script>
```

- [ ] **Step 2: Insert the `#panel-home` section**

In `index.html`, right after the `<main class="panels">` opening tag (line 58) and
*before* the `<!-- ━━━━━━━━━━━━ TAB 1: SOLVER ━━━━━━━━━━━━ -->` comment (line 60),
insert:

```html
    <!-- ━━━━━━━━━━━━ HOME: MAIN MENU ━━━━━━━━━━━━ -->
    <section class="panel" id="panel-home">
      <div class="home-hero">
        <h2>MatematikaAI</h2>
        <p>Bólimdi tańlań / Выберите раздел</p>
      </div>
      <div class="home-cards">
        <button class="home-card" data-tab="solver">
          <svg viewBox="0 0 16 16" fill="none"><path d="M2 4h12M2 8h8M2 12h10" stroke="currentColor" stroke-width="1.4" stroke-linecap="round"/></svg>
          <span class="home-card-title">Másele</span>
          <span class="home-card-desc">Máseleni sheshiw / Решить задачу</span>
        </button>
        <button class="home-card" data-tab="calc">
          <svg viewBox="0 0 16 16" fill="none"><rect x="2" y="2" width="12" height="12" rx="2" stroke="currentColor" stroke-width="1.3"/><path d="M5 6h2M9 6h2M5 10h2M9 10h2" stroke="currentColor" stroke-width="1.3" stroke-linecap="round"/></svg>
          <span class="home-card-title">Kalkulyator</span>
          <span class="home-card-desc">Esaplaw / Калькулятор</span>
        </button>
        <button class="home-card" data-tab="kb">
          <svg viewBox="0 0 16 16" fill="none"><path d="M3 2h10a1 1 0 011 1v10a1 1 0 01-1 1H3a1 1 0 01-1-1V3a1 1 0 011-1z" stroke="currentColor" stroke-width="1.3"/><path d="M5 6h6M5 9h4" stroke="currentColor" stroke-width="1.3" stroke-linecap="round"/></svg>
          <span class="home-card-title">Baza</span>
          <span class="home-card-desc">Bilim bazası / База знаний</span>
        </button>
        <button class="home-card" data-tab="tasks">
          <svg viewBox="0 0 16 16" fill="none"><path d="M2 3h12M2 8h8M13 11l-3 3-1.5-1.5" stroke="currentColor" stroke-width="1.4" stroke-linecap="round" stroke-linejoin="round"/></svg>
          <span class="home-card-title">Shınıǵıw</span>
          <span class="home-card-desc">Mashqlar / Тренажер</span>
        </button>
      </div>
    </section>

```

Note: `#panel-home` intentionally has no `active` class — `panel-solver` keeps its
existing `active` class as-is. Visibility for the first paint is handled entirely by
CSS via the `home-pending` class added in Step 1 (Task 2 adds those rules).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add home screen markup and pre-paint script"
```

---

### Task 2: Add CSS for the home screen

**Files:**
- Modify: `style.css` (new section after the existing PANELS section, i.e. after line 197)

- [ ] **Step 1: Add the HOME SCREEN CSS section**

In `style.css`, after the panel scrollbar rules block (ends at line 197 with
`.panel::-webkit-scrollbar-thumb { background: var(--surface3); border-radius: 3px; }`)
and before the `PANEL HEADER` section comment, insert:

```css
/* ─────────────────────────────────────────────
   HOME SCREEN
───────────────────────────────────────────── */
html.home-pending #panel-solver.panel.active { display: none; }
html.home-pending #panel-home { display: block; }
html.home-pending .tab-nav { display: none; }

.app-shell.home-mode .tab-nav { display: none; }

.home-hero {
  text-align: center;
  margin: 40px 0 28px;
}

.home-hero h2 {
  font-family: var(--font-title);
  font-size: 28px;
  font-weight: 800;
  color: var(--text);
}

.home-hero p {
  margin-top: 6px;
  color: var(--text2);
  font-size: 14px;
}

.home-cards {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 16px;
  max-width: 640px;
  margin: 0 auto;
}

.home-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 28px 16px;
  color: var(--text);
  font-family: var(--font-ui);
  cursor: pointer;
  transition: all 0.15s;
}

.home-card svg {
  width: 28px;
  height: 28px;
  color: var(--accent);
}

.home-card-title {
  font-family: var(--font-title);
  font-size: 16px;
  font-weight: 700;
}

.home-card-desc {
  font-size: 12px;
  color: var(--text2);
  text-align: center;
}

.home-card:hover {
  background: var(--surface2);
  border-color: var(--border2);
}

@media (max-width: 520px) {
  .home-cards {
    grid-template-columns: 1fr;
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add style.css
git commit -m "Add home screen styles"
```

---

### Task 3: Wire up home screen logic in `app.js`

**Files:**
- Modify: `app.js:54-67` (TAB SWITCHING section)
- Modify: `app.js:92-98` (INIT section)

- [ ] **Step 1: Replace the TAB SWITCHING section**

In `app.js`, replace the existing block (lines 54-67):

```js
// ════════════════════════════════════════════════════════════
//  TAB SWITCHING
// ════════════════════════════════════════════════════════════
document.querySelectorAll('.tab-btn').forEach(btn => {
  btn.addEventListener('click', () => {
    const tab = btn.dataset.tab;
    document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
    document.querySelectorAll('.panel').forEach(p => p.classList.remove('active'));
    btn.classList.add('active');
    $('panel-' + tab).classList.add('active');
    if (tab === 'admin' && currentUser?.role === 'admin') loadAdminPanel();
    if (tab === 'tasks') loadTasksTab();
  });
});
```

with:

```js
// ════════════════════════════════════════════════════════════
//  TAB SWITCHING
// ════════════════════════════════════════════════════════════
function selectTab(tab) {
  document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
  document.querySelectorAll('.panel').forEach(p => p.classList.remove('active'));
  const btn = document.querySelector(`.tab-btn[data-tab="${tab}"]`);
  if (btn) btn.classList.add('active');
  $('panel-' + tab).classList.add('active');
  document.querySelector('.app-shell').classList.remove('home-mode');
  localStorage.setItem('lastTab', tab);
  if (tab === 'admin' && currentUser?.role === 'admin') loadAdminPanel();
  if (tab === 'tasks') loadTasksTab();
}

document.querySelectorAll('.tab-btn').forEach(btn => {
  btn.addEventListener('click', () => selectTab(btn.dataset.tab));
});

document.querySelectorAll('.home-card').forEach(card => {
  card.addEventListener('click', () => selectTab(card.dataset.tab));
});

function initHomeOrLastTab() {
  const lastTab   = localStorage.getItem('lastTab');
  const validTabs = ['solver', 'calc', 'kb', 'tasks', 'admin'];
  const tabValid  = lastTab
    && validTabs.includes(lastTab)
    && (lastTab !== 'admin' || currentUser?.role === 'admin');

  if (tabValid) {
    selectTab(lastTab);
  } else {
    document.querySelectorAll('.panel').forEach(p => p.classList.remove('active'));
    $('panel-home').classList.add('active');
    document.querySelector('.app-shell').classList.add('home-mode');
  }
  document.documentElement.classList.remove('home-pending');
}
```

- [ ] **Step 2: Call `initHomeOrLastTab()` from the INIT section**

In `app.js`, replace the `DOMContentLoaded` handler (lines 92-98):

```js
document.addEventListener('DOMContentLoaded', async () => {
  initTheme();
  await loadAuth();
  await loadConfig();
  await loadHistory();
  kbInit();
});
```

with:

```js
document.addEventListener('DOMContentLoaded', async () => {
  initTheme();
  await loadAuth();
  initHomeOrLastTab();
  await loadConfig();
  await loadHistory();
  kbInit();
});
```

`initHomeOrLastTab()` runs after `loadAuth()` so `currentUser.role` is known before
deciding whether a saved `admin` tab is still valid.

- [ ] **Step 3: Commit**

```bash
git add app.js
git commit -m "Add home screen tab selection and init logic"
```

---

### Task 4: Manual verification

**Files:** none (browser-only check)

- [ ] **Step 1: Serve the site locally**

```bash
cd /d/Project/Matmozg && python -m http.server 8000
```

Open `http://localhost:8000/index.html` in a browser.

- [ ] **Step 2: Verify first-visit home screen**

In the browser devtools console, run `localStorage.clear()`, then reload the page.

Expected:
- The tab row (Másele / Kalkulyator / Baza / Shınıǵıw / Admin) is **not** visible.
- A "MatematikaAI — Bólimdi tańlań / Выберите раздел" heading and 4 cards
  (Másele, Kalkulyator, Baza, Shınıǵıw) are shown.

- [ ] **Step 3: Verify card click enters normal mode**

Click the "Másele" card.

Expected:
- The tab row appears, with "Másele" highlighted as active.
- The Másele panel (problem input) is shown.
- In devtools console, `localStorage.getItem('lastTab')` returns `"solver"`.

- [ ] **Step 4: Verify reload restores the chosen section**

Reload the page (without clearing `localStorage`).

Expected:
- The home screen is **not** shown.
- The tab row is visible with "Másele" active, and the Másele panel is shown directly.

- [ ] **Step 5: Verify clearing the saved tab returns to the home screen**

In devtools console, run `localStorage.removeItem('lastTab')`, then reload.

Expected:
- The home screen with 4 cards is shown again (same as Step 2).

- [ ] **Step 6: Verify the other three cards**

Repeat Steps 2-4 for the "Kalkulyator", "Baza", and "Shınıǵıw" cards, confirming
each opens its corresponding panel and is restored correctly on reload. For
"Shınıǵıw", also confirm the trainer topic list loads (this exercises the
`loadTasksTab()` call inside `selectTab`).

---

## Self-Review Notes

- Spec coverage: first-visit home screen ✓ (Tasks 1-3), card → normal mode + save
  choice ✓ (Task 3 `selectTab`), repeat-visit restore ✓ (Task 3
  `initHomeOrLastTab`), no "На главную" button ✓ (not implemented, per
  clarified scope), admin card excluded from home screen ✓ (only 4 cards in
  Task 1), invalid/admin-without-role fallback to home ✓ (`tabValid` check in
  `initHomeOrLastTab`).
- No placeholders; all CSS/HTML/JS snippets are complete and ready to paste.
- `selectTab` is the single source of truth for tab activation, used by both
  `.tab-btn` and `.home-card` click handlers and by `initHomeOrLastTab` —
  consistent naming throughout.
