# Auth, Roles & API Key Protection — Design Spec
**Date:** 2026-04-30  
**Project:** MatematikaAI (d:\Project\Matmozg)  
**Stack:** Python Flask + SQLite + HTML/CSS/JS

---

## 1. Goals

1. User registration with two roles: `user` and `admin`
2. Public registration creates `user` accounts; admin promotes via admin panel
3. Block access to `config.json` static file (contains real API keys)
4. Admin is the sole owner of KB management and site configuration

---

## 2. Database Schema

Three new tables added to `matmozg.db` via `db.py`:

### `users`
```sql
CREATE TABLE IF NOT EXISTS users (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    username      TEXT NOT NULL UNIQUE,
    email         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role          TEXT NOT NULL DEFAULT 'user' CHECK(role IN ('user','admin')),
    created_at    TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### `user_history`
```sql
CREATE TABLE IF NOT EXISTS user_history (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id    INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    problem    TEXT NOT NULL,
    answer     TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### `kb_suggestions`
```sql
CREATE TABLE IF NOT EXISTS kb_suggestions (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id        INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title          TEXT NOT NULL,
    content        TEXT NOT NULL,
    diagram        TEXT DEFAULT 'none',
    school_section TEXT,
    task_type      TEXT,
    grade          TEXT,
    difficulty     TEXT DEFAULT 'medium',
    status         TEXT NOT NULL DEFAULT 'pending' CHECK(status IN ('pending','approved','rejected')),
    created_at     TEXT NOT NULL DEFAULT (datetime('now'))
);
```

---

## 3. Admin Bootstrap

- Script `create_admin.py` — run once from terminal
- Prompts for username and password interactively (no credentials stored in code)
- Hashes password with `werkzeug.security.generate_password_hash`
- Inserts row into `users` with `role='admin'`
- Exits with error if username already exists

---

## 4. Authentication

**New file:** `auth.py`

### User model
```python
class User(UserMixin):
    id, username, email, role
```
Loaded by `flask_login.LoginManager` user_loader from SQLite.

### Routes
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/auth/register` | Public registration → role='user' |
| POST | `/api/auth/login` | Login, sets session |
| POST | `/api/auth/logout` | Logout, clears session |
| GET  | `/api/auth/me` | Returns current user info (or 401) |

### Decorators
- `@login_required` — from flask_login, returns 401 JSON if not logged in
- `@admin_required` — custom, returns 403 JSON if role != 'admin'

### Session
- Flask server-side sessions via `flask_login`
- `app.secret_key` set from `SECRET_KEY` env var (fallback: random on startup — sessions reset on restart)
- Session lifetime: 30 days; `remember=True` is always passed to `login_user()` — no "Remember me" checkbox needed

---

## 5. API Key Protection

### Problem
`config.json` is served as a static file from `.` (project root). Anyone can access `/config.json` and read all API keys.

### Solution
Add explicit route in `server.py` that blocks the file:
```python
@app.route('/config.json')
def block_config_json():
    return '', 403
```

Additionally, `GET /api/config` already returns only boolean flags — this is correct and unchanged.

`POST /api/config` is protected with `@admin_required`.

---

## 6. Access Control Matrix

| Action | Anonymous | User | Admin |
|--------|-----------|------|-------|
| Solver, calculator (use) | ✅ | ✅ | ✅ |
| Knowledge base (read) | ✅ | ✅ | ✅ |
| History (browser localStorage) | ✅ | — | — |
| History (account, cross-device) | — | ✅ | ✅ |
| Suggest KB topic | — | ✅ | ✅ |
| Add / delete KB topic | — | — | ✅ |
| Approve / reject suggestions | — | — | ✅ |
| Manage site config (API keys) | — | — | ✅ |
| Manage users (promote/demote) | — | — | ✅ |

---

## 7. New API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/auth/register` | none | Register new user |
| POST | `/api/auth/login` | none | Login |
| POST | `/api/auth/logout` | login | Logout |
| GET  | `/api/auth/me` | login | Current user info |
| GET  | `/api/history` | login | User's account history |
| POST | `/api/history` | login | Save to account history |
| DELETE | `/api/history` | login | Clear account history |
| POST | `/api/kb/suggest` | login | Submit KB topic suggestion |
| GET  | `/api/kb/suggestions` | admin | List pending suggestions |
| POST | `/api/kb/suggestions/<id>/approve` | admin | Approve suggestion → add to KB |
| POST | `/api/kb/suggestions/<id>/reject` | admin | Reject suggestion |
| GET  | `/api/admin/users` | admin | List all users |
| PATCH | `/api/admin/users/<id>` | admin | Change user role |
| DELETE | `/api/admin/users/<id>` | admin | Delete user |

### Modified endpoints (add auth)
- `POST /api/kb` → `@admin_required`
- `DELETE /api/kb/<id>` → `@admin_required`
- `POST /api/config` → `@admin_required`

---

## 8. Frontend Changes

### Header
- Add **"Войти"** button (right side of top-bar) when anonymous
- When logged in: show `username` chip + **"Выйти"** button
- Admin sees additional **"Admin"** tab in navigation

### Login/Register Modal
- Overlay modal with two tabs: **Вход** / **Регистрация**
- Fields: username, email (register only), password
- On success: reload page state (no full page reload)

### Knowledge Base panel
- "Добавить тему" / "Удалить" buttons — hidden for anonymous and user, visible for admin only
- "Предложить тему" button — visible only for logged-in users

### Admin panel (new tab)
- **Users** section: table of users, role badge, promote/demote button, delete button
- **Suggestions** section: list of pending KB suggestions with Approve/Reject buttons

### History
- Anonymous: uses localStorage only (existing behavior); if `POST /api/history` returns 401, frontend silently ignores it
- Logged-in user: `POST /api/history` saves to account; `GET /api/history` loads from account (localStorage is ignored after login)
- No migration of localStorage history to account on login — they are separate stores

---

## 9. Dependencies

```
flask-login==0.6.3
```

Added to `requirements.txt`. `werkzeug` is already included with Flask.

---

## 10. Files Changed / Created

| File | Change |
|------|--------|
| `auth.py` | New — User model, auth routes, decorators |
| `create_admin.py` | New — one-time admin bootstrap script |
| `db.py` | Add users/user_history/kb_suggestions tables and CRUD |
| `server.py` | Register auth blueprint, block config.json, protect routes |
| `index.html` | Add login modal, admin tab, conditional UI elements |
| `app.js` | Auth state management, login/logout/register calls, admin panel logic |
| `style.css` | Modal styles, user chip, admin panel styles |
| `requirements.txt` | Add flask-login |
