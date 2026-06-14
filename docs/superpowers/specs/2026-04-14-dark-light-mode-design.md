# Dark / Light Mode — Design Spec

**Date:** 2026-04-14  
**Project:** MatematikaAI

---

## Summary

Add a user-toggleable dark/light theme to the existing MatematikaAI web app. The current UI is dark-only. The feature adds a light theme (VS Code Light / GitHub style) and a text button in the top bar to switch between them.

---

## Architecture

**Approach:** CSS `data-theme` attribute on `<html>` element.

- Dark theme: existing `:root` variables (no change)
- Light theme: `:root[data-theme="light"]` block with overridden variables
- Persistence: `localStorage` key `theme` (`"dark"` | `"light"`)
- Default: `"dark"` (preserves current behaviour for new users)

**Files changed:**
- `style.css` — add light theme variable block + button styles
- `index.html` — add toggle button to top bar
- `app.js` — add `initTheme()` and `toggleTheme()` functions

---

## CSS Changes

Add after the existing `:root` block in `style.css`:

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

The `anim-canvas` has a hardcoded `background: #0d1017` — replace with `var(--bg)` so it adapts to the active theme.

The grid background overlay (`body::before`) uses hardcoded `rgba(74,158,255,0.025)` — keep as-is since the opacity is very low and acceptable on both themes.

Accent colours (`--accent`, `--green`, `--red`, `--amber`) are unchanged — they read well on both backgrounds.

---

## HTML Changes

Add a theme toggle button to the right side of the top bar in `index.html`, inside `.top-bar`, after `.tab-nav`:

```html
<button class="btn-ghost theme-toggle" id="themeToggle" onclick="toggleTheme()">
  ☀️ Светлая
</button>
```

The button label and icon update dynamically via JS:
- Dark mode active → button shows `☀️ Светлая`
- Light mode active → button shows `🌙 Тёмная`

Add a small CSS rule to push the button to the right:

```css
.top-bar { justify-content: space-between; } /* or use margin-left: auto on .theme-toggle */
```

---

## JS Changes

Add to `app.js`:

```js
function initTheme() {
  const saved = localStorage.getItem('theme') || 'dark';
  applyTheme(saved);
}

function toggleTheme() {
  const current = document.documentElement.getAttribute('data-theme') || 'dark';
  applyTheme(current === 'dark' ? 'light' : 'dark');
}

function applyTheme(theme) {
  document.documentElement.setAttribute('data-theme', theme);
  localStorage.setItem('theme', theme);
  const btn = document.getElementById('themeToggle');
  if (btn) btn.textContent = theme === 'dark' ? '☀️ Светлая' : '🌙 Тёмная';
}
```

Call `initTheme()` on page load (at the bottom of `app.js` or in a `DOMContentLoaded` listener).

---

## Scope

- No changes to backend (`server.py`)
- No changes to `knowledge_base.json`, `config.json`, `history.json`
- No new dependencies

---

## Success Criteria

1. Clicking the button toggles between dark and light themes instantly
2. Chosen theme persists after page reload
3. All UI elements (cards, inputs, buttons, canvas, history list) render correctly in both themes
4. Default theme for new users is dark
