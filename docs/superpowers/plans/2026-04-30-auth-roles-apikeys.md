# Auth, Roles & API Key Protection — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Добавить регистрацию/вход с ролями admin/user, защитить config.json от публичного доступа, добавить панель администратора для управления пользователями и предложениями тем KB.

**Architecture:** Flask-Login управляет серверными сессиями. Новый `auth.py` Blueprint содержит модель User и эндпоинты аутентификации. `db.py` получает три новых таблицы: `users`, `user_history`, `kb_suggestions`. Фронтенд получает модальное окно входа, auth-aware шапку и вкладку "Admin". История решений привязывается к аккаунту для залогиненных пользователей.

**Tech Stack:** Python 3.14, Flask, Flask-Login 0.6.3, werkzeug (уже в Flask), SQLite, Vanilla JS

---

## File Map

| Файл | Действие | Ответственность |
|------|----------|-----------------|
| `requirements.txt` | Modify | Добавить flask-login |
| `db.py` | Modify | Новые таблицы + CRUD для users/history/suggestions |
| `auth.py` | Create | User model, Flask-Login setup, auth Blueprint |
| `create_admin.py` | Create | Одноразовый скрипт создания admin аккаунта |
| `server.py` | Modify | Интеграция auth, защита роутов, новые эндпоинты |
| `index.html` | Modify | Login modal, Admin tab, conditional KB UI |
| `app.js` | Modify | Auth state, login/register/logout, admin panel |
| `style.css` | Modify | Modal, user chip, admin panel стили |

---

## Task 1: Установить flask-login и расширить схему БД

**Files:**
- Modify: `requirements.txt`
- Modify: `db.py`

- [ ] **Шаг 1: Добавить flask-login в requirements.txt**

В конец файла `requirements.txt` добавить строку:
```
flask-login==0.6.3
```

- [ ] **Шаг 2: Установить зависимость**

```bash
pip install flask-login==0.6.3
```
Ожидаемый вывод: `Successfully installed flask-login-0.6.3`

- [ ] **Шаг 3: Обновить `_ensure_schema()` в db.py — добавить три новых таблицы**

Заменить всю функцию `_ensure_schema()` в `db.py`:

```python
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
        c.execute('''CREATE TABLE IF NOT EXISTS users (
            id            INTEGER PRIMARY KEY AUTOINCREMENT,
            username      TEXT NOT NULL UNIQUE,
            email         TEXT NOT NULL UNIQUE,
            password_hash TEXT NOT NULL,
            role          TEXT NOT NULL DEFAULT 'user' CHECK(role IN ('user','admin')),
            created_at    TEXT NOT NULL DEFAULT (datetime('now'))
        )''')
        c.execute('''CREATE TABLE IF NOT EXISTS user_history (
            id         INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id    INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
            problem    TEXT NOT NULL,
            solution   TEXT NOT NULL,
            created_at TEXT NOT NULL DEFAULT (datetime('now'))
        )''')
        c.execute('''CREATE TABLE IF NOT EXISTS kb_suggestions (
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
        )''')
```

- [ ] **Шаг 4: Всегда вызывать `_ensure_schema()` независимо от USE_SQLITE**

Найти в `db.py`:
```python
if USE_SQLITE:
    _ensure_schema()
```

Заменить на:
```python
_ensure_schema()
```

(Оставить `USE_SQLITE = os.path.exists(DB_FILE)` без изменений — он по-прежнему управляет операциями с topics/problems. Вызов `_ensure_schema()` всегда безопасен благодаря `CREATE TABLE IF NOT EXISTS` и тому, что `sqlite3.connect` создаёт файл БД если его нет.)

- [ ] **Шаг 5: Добавить CRUD функции для users в db.py**

Добавить после существующих функций в `db.py`:

```python
# ── Users ─────────────────────────────────────────────────────

def create_user(username: str, email: str, password_hash: str) -> int | None:
    try:
        with _conn() as c:
            c.execute(
                'INSERT INTO users (username, email, password_hash) VALUES (?,?,?)',
                (username, email, password_hash)
            )
            return c.lastrowid
    except sqlite3.IntegrityError:
        return None


def get_user_by_id(user_id: int) -> dict | None:
    with _conn() as c:
        row = c.execute('SELECT * FROM users WHERE id = ?', (user_id,)).fetchone()
        return dict(row) if row else None


def get_user_by_username(username: str) -> dict | None:
    with _conn() as c:
        row = c.execute('SELECT * FROM users WHERE username = ?', (username,)).fetchone()
        return dict(row) if row else None


def get_all_users() -> list:
    with _conn() as c:
        rows = c.execute(
            'SELECT id, username, email, role, created_at FROM users ORDER BY id'
        ).fetchall()
        return [dict(r) for r in rows]


def update_user_role(user_id: int, role: str):
    with _conn() as c:
        c.execute('UPDATE users SET role = ? WHERE id = ?', (role, user_id))


def delete_user(user_id: int):
    with _conn() as c:
        c.execute('DELETE FROM users WHERE id = ?', (user_id,))


# ── User History ───────────────────────────────────────────────

def get_user_history(user_id: int) -> list:
    with _conn() as c:
        rows = c.execute(
            'SELECT problem, solution FROM user_history WHERE user_id = ? ORDER BY created_at DESC LIMIT 100',
            (user_id,)
        ).fetchall()
        return [dict(r) for r in rows]


def add_user_history(user_id: int, problem: str, solution: str):
    with _conn() as c:
        c.execute(
            'INSERT INTO user_history (user_id, problem, solution) VALUES (?,?,?)',
            (user_id, problem, solution)
        )


def clear_user_history(user_id: int):
    with _conn() as c:
        c.execute('DELETE FROM user_history WHERE user_id = ?', (user_id,))


# ── KB Suggestions ────────────────────────────────────────────

def add_suggestion(user_id: int, entry: dict):
    with _conn() as c:
        c.execute(
            '''INSERT INTO kb_suggestions
               (user_id, title, content, diagram, school_section, task_type, grade, difficulty)
               VALUES (?,?,?,?,?,?,?,?)''',
            (user_id, entry['title'], entry['content'],
             entry.get('diagram', 'none'), entry.get('school_section'),
             entry.get('task_type'), entry.get('grade'),
             entry.get('difficulty', 'medium'))
        )


def get_suggestions(status: str = 'pending') -> list:
    with _conn() as c:
        rows = c.execute(
            '''SELECT s.*, u.username FROM kb_suggestions s
               JOIN users u ON u.id = s.user_id
               WHERE s.status = ? ORDER BY s.created_at DESC''',
            (status,)
        ).fetchall()
        return [dict(r) for r in rows]


def update_suggestion_status(suggestion_id: int, status: str):
    with _conn() as c:
        c.execute(
            'UPDATE kb_suggestions SET status = ? WHERE id = ?',
            (status, suggestion_id)
        )
        if status == 'approved':
            row = c.execute(
                'SELECT * FROM kb_suggestions WHERE id = ?', (suggestion_id,)
            ).fetchone()
            if row:
                d = dict(row)
                c.execute(
                    '''INSERT INTO topics
                       (title, content, diagram, school_section, task_type, grade, difficulty)
                       VALUES (?,?,?,?,?,?,?)''',
                    (d['title'], d['content'], d.get('diagram', 'none'),
                     d.get('school_section'), d.get('task_type'),
                     d.get('grade'), d.get('difficulty', 'medium'))
                )
```

- [ ] **Шаг 6: Проверить создание схемы**

```bash
python -c "import db; import sqlite3; c = sqlite3.connect('matmozg.db'); tables = [r[0] for r in c.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()]; print(tables); assert 'users' in tables and 'user_history' in tables and 'kb_suggestions' in tables, 'Missing tables!'; print('OK')"
```
Ожидаемый вывод: список таблиц включает `users`, `user_history`, `kb_suggestions`, затем `OK`

- [ ] **Шаг 7: Commit**

```bash
git add requirements.txt db.py
git commit -m "feat: add flask-login dep, users/user_history/kb_suggestions tables and CRUD"
```

---

## Task 2: Скрипт create_admin.py

**Files:**
- Create: `create_admin.py`

- [ ] **Шаг 1: Создать файл create_admin.py**

```python
#!/usr/bin/env python3
import getpass, sys, os

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

from werkzeug.security import generate_password_hash
import db as _db


def main():
    print("=== Создание Admin аккаунта ===")

    username = input("Логин: ").strip()
    if not username:
        print("Ошибка: логин не может быть пустым")
        sys.exit(1)

    if _db.get_user_by_username(username):
        print(f"Ошибка: пользователь '{username}' уже существует")
        sys.exit(1)

    email = input("Email: ").strip()
    if not email:
        print("Ошибка: email не может быть пустым")
        sys.exit(1)

    password = getpass.getpass("Пароль (мин. 6 символов): ")
    if len(password) < 6:
        print("Ошибка: пароль должен содержать минимум 6 символов")
        sys.exit(1)

    confirm = getpass.getpass("Подтвердите пароль: ")
    if password != confirm:
        print("Ошибка: пароли не совпадают")
        sys.exit(1)

    user_id = _db.create_user(username, email, generate_password_hash(password))
    if user_id is None:
        print("Ошибка: логин или email уже занят")
        sys.exit(1)

    _db.update_user_role(user_id, 'admin')
    print(f"\nAdmin аккаунт '{username}' успешно создан!")


if __name__ == '__main__':
    main()
```

- [ ] **Шаг 2: Запустить скрипт и создать ваш admin аккаунт**

```bash
python create_admin.py
```
Введите желаемый логин, email и пароль в ответ на запросы.

- [ ] **Шаг 3: Проверить создание admin**

```bash
python -c "import db; u = db.get_user_by_username('ВАШ_ЛОГИН'); print(u['role'] if u else 'not found')"
```
Ожидаемый вывод: `admin`

- [ ] **Шаг 4: Commit**

```bash
git add create_admin.py
git commit -m "feat: add create_admin.py one-time bootstrap script"
```

---

## Task 3: auth.py — User model и auth роуты

**Files:**
- Create: `auth.py`

- [ ] **Шаг 1: Создать auth.py**

```python
from flask import Blueprint, request, jsonify
from flask_login import (
    LoginManager, UserMixin, login_user, logout_user,
    current_user, login_required,
)
from werkzeug.security import generate_password_hash, check_password_hash
from functools import wraps
import db as _db

auth_bp      = Blueprint('auth', __name__)
login_manager = LoginManager()


class User(UserMixin):
    def __init__(self, row: dict):
        self.id       = str(row['id'])
        self.username = row['username']
        self.email    = row['email']
        self.role     = row['role']


def init_login_manager(app):
    login_manager.init_app(app)

    @login_manager.unauthorized_handler
    def unauthorized():
        return jsonify({'error': 'Authentication required'}), 401

    @login_manager.user_loader
    def load_user(user_id):
        row = _db.get_user_by_id(int(user_id))
        return User(row) if row else None


def admin_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if not current_user.is_authenticated:
            return jsonify({'error': 'Authentication required'}), 401
        if current_user.role != 'admin':
            return jsonify({'error': 'Admin only'}), 403
        return f(*args, **kwargs)
    return decorated


# ── Auth routes ───────────────────────────────────────────────

@auth_bp.route('/api/auth/register', methods=['POST'])
def register():
    data     = request.json or {}
    username = data.get('username', '').strip()
    email    = data.get('email', '').strip()
    password = data.get('password', '')
    if not username or not email or not password:
        return jsonify({'error': 'Нужны username, email и password'}), 400
    if len(password) < 6:
        return jsonify({'error': 'Пароль должен содержать минимум 6 символов'}), 400
    user_id = _db.create_user(username, email, generate_password_hash(password))
    if user_id is None:
        return jsonify({'error': 'Логин или email уже занят'}), 409
    row  = _db.get_user_by_id(user_id)
    user = User(row)
    login_user(user, remember=True)
    return jsonify({'ok': True, 'username': user.username, 'role': user.role})


@auth_bp.route('/api/auth/login', methods=['POST'])
def login():
    data     = request.json or {}
    username = data.get('username', '').strip()
    password = data.get('password', '')
    if not username or not password:
        return jsonify({'error': 'Нужны username и password'}), 400
    row = _db.get_user_by_username(username)
    if not row or not check_password_hash(row['password_hash'], password):
        return jsonify({'error': 'Неверный логин или пароль'}), 401
    user = User(row)
    login_user(user, remember=True)
    return jsonify({'ok': True, 'username': user.username, 'role': user.role})


@auth_bp.route('/api/auth/logout', methods=['POST'])
@login_required
def logout():
    logout_user()
    return jsonify({'ok': True})


@auth_bp.route('/api/auth/me', methods=['GET'])
def me():
    if not current_user.is_authenticated:
        return jsonify({'authenticated': False})
    return jsonify({
        'authenticated': True,
        'username': current_user.username,
        'role':     current_user.role,
    })
```

- [ ] **Шаг 2: Commit**

```bash
git add auth.py
git commit -m "feat: add auth.py with Flask-Login User model, admin_required decorator, auth routes"
```

---

## Task 4: server.py — интеграция auth, защита роутов, новые эндпоинты

**Files:**
- Modify: `server.py`

- [ ] **Шаг 1: Добавить импорты в начало server.py**

В блок импортов в начале `server.py` добавить:
```python
from auth import auth_bp, admin_required, init_login_manager
from flask_login import login_required, current_user
```

- [ ] **Шаг 2: Настроить Flask-Login и зарегистрировать Blueprint**

После строк `app = Flask(...)` и `app.config['MAX_CONTENT_LENGTH'] = ...` добавить:
```python
app.secret_key = os.environ.get('SECRET_KEY', os.urandom(24))
app.config['REMEMBER_COOKIE_DURATION'] = 60 * 60 * 24 * 30  # 30 дней

init_login_manager(app)
app.register_blueprint(auth_bp)
```

- [ ] **Шаг 3: Заблокировать статический доступ к config.json**

Добавить этот роут сразу после блока `# ─── ROUTES — Static ───`:
```python
@app.route('/config.json')
def block_config_json():
    return '', 403
```

- [ ] **Шаг 4: Защитить KB write роуты декоратором @admin_required**

Найти и заменить два роута:
```python
@app.route('/api/kb', methods=['POST'])
@admin_required
def add_kb():
    entry = request.json or {}
    if not entry.get('title') or not entry.get('content'):
        return jsonify({'error': 'Nom va mazmun kerak'}), 400
    _db.add_topic(entry)
    return jsonify({'ok': True})

@app.route('/api/kb/<int:idx>', methods=['DELETE'])
@admin_required
def delete_kb(idx):
    _db.delete_topic(idx)
    return jsonify({'ok': True})
```

- [ ] **Шаг 5: Защитить POST /api/config декоратором @admin_required**

Найти и заменить:
```python
@app.route('/api/config', methods=['POST'])
@admin_required
def save_config():
    config     = load_json(CONFIG_FILE, {'api_key': '', 'model': 'tilmoch'})
    new_config = request.json or {}
    config.update(new_config)
    save_json(CONFIG_FILE, config)
    return jsonify({'ok': True})
```

- [ ] **Шаг 6: Обновить history роуты для поддержки аккаунтов**

Заменить все три существующих history роута:
```python
@app.route('/api/history', methods=['GET'])
def get_history():
    if current_user.is_authenticated:
        return jsonify(_db.get_user_history(int(current_user.id)))
    return jsonify(load_json(HISTORY_FILE, []))


@app.route('/api/history', methods=['POST'])
def add_history():
    entry = request.json or {}
    if current_user.is_authenticated:
        _db.add_user_history(
            int(current_user.id),
            entry.get('problem', ''),
            entry.get('solution', '') or entry.get('answer', ''),
        )
    else:
        history = load_json(HISTORY_FILE, [])
        history.append(entry)
        save_json(HISTORY_FILE, history)
    return jsonify({'ok': True})


@app.route('/api/history', methods=['DELETE'])
def clear_history():
    if current_user.is_authenticated:
        _db.clear_user_history(int(current_user.id))
    else:
        save_json(HISTORY_FILE, [])
    return jsonify({'ok': True})
```

- [ ] **Шаг 7: Добавить эндпоинты для предложений KB и управления пользователями**

Добавить перед `if __name__ == '__main__':`:
```python
# ─────────────────────────────────────────────
# ROUTES — KB Suggestions
# ─────────────────────────────────────────────
@app.route('/api/kb/suggest', methods=['POST'])
@login_required
def suggest_kb():
    entry = request.json or {}
    if not entry.get('title') or not entry.get('content'):
        return jsonify({'error': 'Нужны title и content'}), 400
    _db.add_suggestion(int(current_user.id), entry)
    return jsonify({'ok': True})


@app.route('/api/kb/suggestions', methods=['GET'])
@admin_required
def get_kb_suggestions():
    return jsonify(_db.get_suggestions('pending'))


@app.route('/api/kb/suggestions/<int:sid>/approve', methods=['POST'])
@admin_required
def approve_suggestion(sid):
    _db.update_suggestion_status(sid, 'approved')
    return jsonify({'ok': True})


@app.route('/api/kb/suggestions/<int:sid>/reject', methods=['POST'])
@admin_required
def reject_suggestion(sid):
    _db.update_suggestion_status(sid, 'rejected')
    return jsonify({'ok': True})


# ─────────────────────────────────────────────
# ROUTES — Admin: Users
# ─────────────────────────────────────────────
@app.route('/api/admin/users', methods=['GET'])
@admin_required
def admin_get_users():
    return jsonify(_db.get_all_users())


@app.route('/api/admin/users/<int:uid>', methods=['PATCH'])
@admin_required
def admin_update_user(uid):
    data = request.json or {}
    role = data.get('role')
    if role not in ('user', 'admin'):
        return jsonify({'error': "role должна быть 'user' или 'admin'"}), 400
    _db.update_user_role(uid, role)
    return jsonify({'ok': True})


@app.route('/api/admin/users/<int:uid>', methods=['DELETE'])
@admin_required
def admin_delete_user(uid):
    if uid == int(current_user.id):
        return jsonify({'error': 'Нельзя удалить свой аккаунт'}), 400
    _db.delete_user(uid)
    return jsonify({'ok': True})
```

- [ ] **Шаг 8: Проверить запуск сервера без ошибок**

```bash
python server.py
```
Ожидаемый вывод: сервер стартует на порту 10000 без ошибок импорта. Нажать Ctrl+C для остановки.

- [ ] **Шаг 9: Ручное тестирование эндпоинтов**

Запустить сервер, в другом терминале:
```bash
# Регистрация
curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:10000/api/auth/register \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"testuser\",\"email\":\"test@test.com\",\"password\":\"123456\"}"
# Ожидаемый ответ: {"ok":true,"role":"user","username":"testuser"}

# Проверить /me
curl -s -c cookies.txt -b cookies.txt http://localhost:10000/api/auth/me
# Ожидаемый ответ: {"authenticated":true,"role":"user","username":"testuser"}

# config.json заблокирован
curl -s -o /dev/null -w "%{http_code}" http://localhost:10000/config.json
# Ожидаемый ответ: 403

# KB write требует admin
curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:10000/api/kb \
  -H "Content-Type: application/json" \
  -d "{\"title\":\"test\",\"content\":\"test\"}"
# Ожидаемый ответ: {"error":"Admin only"}
```

- [ ] **Шаг 10: Commit**

```bash
git add server.py
git commit -m "feat: integrate Flask-Login, protect routes, add admin and suggestion endpoints"
```

---

## Task 5: Frontend — Auth modal и управление состоянием

**Files:**
- Modify: `index.html`
- Modify: `app.js`
- Modify: `style.css`

- [ ] **Шаг 1: Добавить кнопку входа и admin вкладку в шапку index.html**

Найти `<nav class="tab-nav">` и добавить Admin кнопку внутрь nav (последней):
```html
      <button class="tab-btn" data-tab="admin" id="adminTabBtn" style="display:none">
        <svg viewBox="0 0 16 16" fill="none"><circle cx="8" cy="6" r="3" stroke="currentColor" stroke-width="1.3"/><path d="M2 14c0-3.314 2.686-6 6-6s6 2.686 6 6" stroke="currentColor" stroke-width="1.3" stroke-linecap="round"/></svg>
        Admin
      </button>
```

Между `</nav>` и кнопкой `theme-toggle` добавить:
```html
    <div class="auth-bar">
      <div id="userChip" class="user-chip" style="display:none">
        <span id="userChipName"></span>
        <button class="btn-ghost-sm" onclick="authLogout()">Выйти</button>
      </div>
      <button id="loginBtn" class="btn-primary btn-sm" onclick="openAuthModal('login')">Войти</button>
    </div>
```

- [ ] **Шаг 2: Добавить login/register модальное окно в index.html**

Добавить перед закрывающим `</body>`:
```html
<!-- ══════════════ AUTH MODAL ══════════════ -->
<div class="modal-overlay" id="authModal" style="display:none" onclick="closeAuthModal(event)">
  <div class="modal-box">
    <div class="modal-tabs">
      <button class="modal-tab active" id="modalTabLogin" onclick="switchModalTab('login')">Вход</button>
      <button class="modal-tab" id="modalTabRegister" onclick="switchModalTab('register')">Регистрация</button>
    </div>
    <form class="modal-form" id="loginForm" onsubmit="authLogin(event)">
      <label class="field-label">Логин</label>
      <input type="text" class="field-input" id="loginUsername" autocomplete="username" required>
      <label class="field-label">Пароль</label>
      <input type="password" class="field-input" id="loginPassword" autocomplete="current-password" required>
      <div class="modal-error" id="loginError"></div>
      <button type="submit" class="btn-primary w-full">Войти</button>
    </form>
    <form class="modal-form" id="registerForm" style="display:none" onsubmit="authRegister(event)">
      <label class="field-label">Логин</label>
      <input type="text" class="field-input" id="regUsername" autocomplete="username" required>
      <label class="field-label">Email</label>
      <input type="email" class="field-input" id="regEmail" autocomplete="email" required>
      <label class="field-label">Пароль (мин. 6 символов)</label>
      <input type="password" class="field-input" id="regPassword" autocomplete="new-password" minlength="6" required>
      <div class="modal-error" id="registerError"></div>
      <button type="submit" class="btn-primary w-full">Зарегистрироваться</button>
    </form>
  </div>
</div>

<!-- ══════════════ SUGGEST MODAL ══════════════ -->
<div class="modal-overlay" id="suggestModal" style="display:none" onclick="closeSuggestModal(event)">
  <div class="modal-box">
    <h3 class="modal-title">Предложить тему</h3>
    <form class="modal-form" onsubmit="submitSuggestion(event)">
      <label class="field-label">Название темы</label>
      <input type="text" class="field-input" id="suggestTitle" required>
      <label class="field-label">Содержание / формулы</label>
      <textarea class="field-textarea" id="suggestContent" rows="5" required></textarea>
      <div class="modal-error" id="suggestError"></div>
      <button type="submit" class="btn-primary w-full">Отправить на проверку</button>
    </form>
  </div>
</div>
```

- [ ] **Шаг 3: Добавить Admin панель секцию в index.html**

Внутри `<main class="panels">`, после существующих panels, добавить:
```html
    <!-- ━━━━━━━━━━━━ TAB 4: ADMIN ━━━━━━━━━━━━ -->
    <section class="panel" id="panel-admin">
      <div class="admin-layout">
        <div class="admin-section">
          <h3 class="admin-section-title">Пользователи</h3>
          <div id="adminUsersList" class="admin-list"></div>
        </div>
        <div class="admin-section">
          <h3 class="admin-section-title">
            Предложения тем
            <span id="suggestionsBadge" class="badge" style="display:none"></span>
          </h3>
          <div id="adminSuggestionsList" class="admin-list"></div>
        </div>
      </div>
    </section>
```

- [ ] **Шаг 4: Добавить кнопку "Предложить тему" в KB панель index.html**

В секции `#panel-kb`, в `.kb2-left`, после `.kb2-filter-bar` добавить:
```html
          <div class="kb2-auth-actions">
            <button class="btn-ghost-sm" id="kbSuggestBtn" style="display:none" onclick="openSuggestModal()">+ Предложить тему</button>
          </div>
```

- [ ] **Шаг 5: Добавить CSS в style.css**

Добавить в конец `style.css`:
```css
/* ── Auth bar ──────────────────────────────────────────── */
.auth-bar {
  display: flex;
  align-items: center;
  gap: .5rem;
}

.user-chip {
  display: flex;
  align-items: center;
  gap: .5rem;
  padding: .25rem .75rem;
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 999px;
  font-size: .8rem;
  color: var(--text);
}

.btn-sm {
  padding: .3rem .75rem;
  font-size: .8rem;
}

/* ── Modal ─────────────────────────────────────────────── */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,.6);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-box {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 1.5rem;
  width: 100%;
  max-width: 380px;
  box-shadow: 0 8px 32px rgba(0,0,0,.4);
}

.modal-title {
  font-size: 1rem;
  font-weight: 600;
  margin: 0 0 1rem;
  color: var(--text);
}

.modal-tabs {
  display: flex;
  gap: .5rem;
  margin-bottom: 1.25rem;
}

.modal-tab {
  flex: 1;
  padding: .5rem;
  background: none;
  border: 1px solid var(--border);
  border-radius: 8px;
  color: var(--text-muted);
  cursor: pointer;
  font-size: .875rem;
  transition: all .2s;
}

.modal-tab.active {
  background: var(--accent);
  border-color: var(--accent);
  color: #fff;
}

.modal-form {
  display: flex;
  flex-direction: column;
  gap: .75rem;
}

.field-input {
  width: 100%;
  padding: .6rem .8rem;
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: 8px;
  color: var(--text);
  font-size: .875rem;
  outline: none;
  box-sizing: border-box;
}

.field-input:focus {
  border-color: var(--accent);
}

.modal-error {
  color: #e74c3c;
  font-size: .8rem;
  min-height: 1rem;
}

.w-full { width: 100%; }

/* ── KB auth actions ───────────────────────────────────── */
.kb2-auth-actions {
  padding: .5rem 0;
  border-top: 1px solid var(--border);
  margin-top: .25rem;
}

/* ── Admin panel ───────────────────────────────────────── */
.admin-layout {
  display: flex;
  flex-direction: column;
  gap: 2rem;
  padding: 1.5rem;
  max-width: 800px;
  margin: 0 auto;
}

.admin-section {
  display: flex;
  flex-direction: column;
  gap: .5rem;
}

.admin-section-title {
  font-size: .95rem;
  font-weight: 600;
  color: var(--text);
  margin: 0 0 .5rem;
  display: flex;
  align-items: center;
  gap: .5rem;
}

.admin-list {
  display: flex;
  flex-direction: column;
  gap: .5rem;
}

.admin-row {
  display: flex;
  align-items: center;
  gap: .75rem;
  padding: .65rem 1rem;
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 8px;
  font-size: .85rem;
  color: var(--text);
}

.admin-row-name { flex: 1; font-weight: 500; }

.role-badge {
  padding: .15rem .5rem;
  border-radius: 4px;
  font-size: .75rem;
  font-weight: 600;
}
.role-badge.admin { background: #6c47ff22; color: #6c47ff; }
.role-badge.user  { background: #00b89422; color: #00b894; }

.badge {
  background: #e74c3c;
  color: #fff;
  font-size: .7rem;
  font-weight: 700;
  padding: .1rem .45rem;
  border-radius: 999px;
}

.suggestion-meta {
  font-size: .78rem;
  color: var(--text-muted);
}

.suggestion-content {
  font-size: .8rem;
  color: var(--text-muted);
  white-space: pre-wrap;
  max-height: 60px;
  overflow: hidden;
  margin-top: .25rem;
}

.admin-row-actions {
  display: flex;
  gap: .4rem;
  margin-left: auto;
}
```

- [ ] **Шаг 6: Добавить auth state и функции в app.js**

После строки `let config = { api_key: '', model: 'tilmoch' };` добавить:
```javascript
let currentUser = null; // { username, role } или null
```

Добавить новый раздел AUTH в `app.js` (перед разделом HISTORY):
```javascript
// ════════════════════════════════════════════════════════════
//  AUTH
// ════════════════════════════════════════════════════════════

async function loadAuth() {
  try {
    const r    = await fetch('/api/auth/me');
    const data = await r.json();
    currentUser = data.authenticated ? { username: data.username, role: data.role } : null;
  } catch (_) {
    currentUser = null;
  }
  updateAuthUI();
}

function updateAuthUI() {
  const chip     = $('userChip');
  const loginBtn = $('loginBtn');
  const adminTab = $('adminTabBtn');
  const chipName = $('userChipName');
  const suggestBtn = $('kbSuggestBtn');

  if (currentUser) {
    chip.style.display     = 'flex';
    loginBtn.style.display = 'none';
    chipName.textContent   = currentUser.username;
    if (adminTab)    adminTab.style.display    = currentUser.role === 'admin' ? '' : 'none';
    if (suggestBtn)  suggestBtn.style.display  = '';
  } else {
    chip.style.display     = 'none';
    loginBtn.style.display = '';
    if (adminTab)   adminTab.style.display   = 'none';
    if (suggestBtn) suggestBtn.style.display = 'none';
  }
}

function openAuthModal(tab = 'login') {
  $('authModal').style.display = 'flex';
  switchModalTab(tab);
}

function closeAuthModal(e) {
  if (e.target === $('authModal')) $('authModal').style.display = 'none';
}

function switchModalTab(tab) {
  $('loginForm').style.display    = tab === 'login'    ? 'flex' : 'none';
  $('registerForm').style.display = tab === 'register' ? 'flex' : 'none';
  $('modalTabLogin').classList.toggle('active',    tab === 'login');
  $('modalTabRegister').classList.toggle('active', tab === 'register');
  $('loginError').textContent    = '';
  $('registerError').textContent = '';
}

async function authLogin(e) {
  e.preventDefault();
  const username = $('loginUsername').value.trim();
  const password = $('loginPassword').value;
  const r = await fetch('/api/auth/login', {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({ username, password }),
  });
  const data = await r.json();
  if (!r.ok) { $('loginError').textContent = data.error; return; }
  currentUser = { username: data.username, role: data.role };
  $('authModal').style.display = 'none';
  updateAuthUI();
  await loadHistory();
}

async function authRegister(e) {
  e.preventDefault();
  const username = $('regUsername').value.trim();
  const email    = $('regEmail').value.trim();
  const password = $('regPassword').value;
  const r = await fetch('/api/auth/register', {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({ username, email, password }),
  });
  const data = await r.json();
  if (!r.ok) { $('registerError').textContent = data.error; return; }
  currentUser = { username: data.username, role: data.role };
  $('authModal').style.display = 'none';
  updateAuthUI();
  await loadHistory();
}

async function authLogout() {
  await fetch('/api/auth/logout', { method: 'POST' });
  currentUser = null;
  updateAuthUI();
  await loadHistory();
}

// ════════════════════════════════════════════════════════════
//  KB SUGGEST
// ════════════════════════════════════════════════════════════

function openSuggestModal() {
  $('suggestError').textContent = '';
  $('suggestTitle').value       = '';
  $('suggestContent').value     = '';
  $('suggestModal').style.display = 'flex';
}

function closeSuggestModal(e) {
  if (e.target === $('suggestModal')) $('suggestModal').style.display = 'none';
}

async function submitSuggestion(e) {
  e.preventDefault();
  const title   = $('suggestTitle').value.trim();
  const content = $('suggestContent').value.trim();
  const r = await fetch('/api/kb/suggest', {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({ title, content }),
  });
  const data = await r.json();
  if (!r.ok) { $('suggestError').textContent = data.error; return; }
  $('suggestModal').style.display = 'none';
  alert('Предложение отправлено! Администратор рассмотрит его.');
}
```

- [ ] **Шаг 7: Обновить DOMContentLoaded в app.js**

Найти:
```javascript
document.addEventListener('DOMContentLoaded', async () => {
  initTheme();
  await loadConfig();
  await loadHistory();
  kbInit();
});
```

Заменить на:
```javascript
document.addEventListener('DOMContentLoaded', async () => {
  initTheme();
  await loadAuth();
  await loadConfig();
  await loadHistory();
  kbInit();
});
```

- [ ] **Шаг 8: Добавить загрузку admin панели при переключении вкладки**

Найти код переключения вкладок:
```javascript
document.querySelectorAll('.tab-btn').forEach(btn => {
  btn.addEventListener('click', () => {
    const tab = btn.dataset.tab;
    document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
    document.querySelectorAll('.panel').forEach(p => p.classList.remove('active'));
    btn.classList.add('active');
    $('panel-' + tab).classList.add('active');
  });
});
```

Заменить на:
```javascript
document.querySelectorAll('.tab-btn').forEach(btn => {
  btn.addEventListener('click', () => {
    const tab = btn.dataset.tab;
    document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
    document.querySelectorAll('.panel').forEach(p => p.classList.remove('active'));
    btn.classList.add('active');
    $('panel-' + tab).classList.add('active');
    if (tab === 'admin' && currentUser?.role === 'admin') loadAdminPanel();
  });
});
```

- [ ] **Шаг 9: Commit**

```bash
git add index.html app.js style.css
git commit -m "feat: auth modal, login/register/logout UI, auth state management, suggest modal"
```

---

## Task 6: Frontend — Admin панель

**Files:**
- Modify: `app.js`

- [ ] **Шаг 1: Добавить функции admin панели в app.js**

Добавить в конец `app.js`:
```javascript
// ════════════════════════════════════════════════════════════
//  ADMIN PANEL
// ════════════════════════════════════════════════════════════

async function loadAdminPanel() {
  await Promise.all([loadAdminUsers(), loadAdminSuggestions()]);
}

async function loadAdminUsers() {
  const r = await fetch('/api/admin/users');
  if (!r.ok) return;
  const users = await r.json();
  const list  = $('adminUsersList');
  if (!list) return;

  if (!users.length) {
    list.innerHTML = '<div style="color:var(--text-muted);font-size:.85rem">Нет пользователей</div>';
    return;
  }

  list.innerHTML = users.map(u => `
    <div class="admin-row">
      <span class="admin-row-name">${esc(u.username)}</span>
      <span class="suggestion-meta">${esc(u.email)}</span>
      <span class="role-badge ${u.role}">${u.role}</span>
      <div class="admin-row-actions">
        ${u.role === 'user'
          ? `<button class="btn-ghost-sm" onclick="adminSetRole(${u.id},'admin')">→ Admin</button>`
          : `<button class="btn-ghost-sm" onclick="adminSetRole(${u.id},'user')">→ User</button>`
        }
        <button class="btn-danger btn-sm" onclick="adminDeleteUser(${u.id},'${esc(u.username)}')">✕</button>
      </div>
    </div>
  `).join('');
}

async function adminSetRole(uid, role) {
  const r = await fetch(`/api/admin/users/${uid}`, {
    method:  'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({ role }),
  });
  if (r.ok) loadAdminUsers();
}

async function adminDeleteUser(uid, username) {
  if (!confirm(`Удалить пользователя "${username}"?`)) return;
  const r    = await fetch(`/api/admin/users/${uid}`, { method: 'DELETE' });
  const data = await r.json();
  if (!r.ok) { alert(data.error); return; }
  loadAdminUsers();
}

async function loadAdminSuggestions() {
  const r = await fetch('/api/kb/suggestions');
  if (!r.ok) return;
  const suggestions = await r.json();
  const list  = $('adminSuggestionsList');
  const badge = $('suggestionsBadge');
  if (!list) return;

  if (badge) {
    badge.textContent   = suggestions.length || '';
    badge.style.display = suggestions.length ? '' : 'none';
  }

  if (!suggestions.length) {
    list.innerHTML = '<div style="color:var(--text-muted);font-size:.85rem">Нет предложений</div>';
    return;
  }

  list.innerHTML = suggestions.map(s => `
    <div class="admin-row" style="flex-direction:column;align-items:flex-start;gap:.4rem">
      <div style="display:flex;align-items:center;gap:.6rem;width:100%">
        <span class="admin-row-name">${esc(s.title)}</span>
        <span class="suggestion-meta">от ${esc(s.username)}</span>
        <div class="admin-row-actions">
          <button class="btn-primary btn-sm" onclick="adminApproveSuggestion(${s.id})">Принять</button>
          <button class="btn-danger btn-sm"  onclick="adminRejectSuggestion(${s.id})">Отклонить</button>
        </div>
      </div>
      <div class="suggestion-content">${esc(s.content)}</div>
    </div>
  `).join('');
}

async function adminApproveSuggestion(sid) {
  const r = await fetch(`/api/kb/suggestions/${sid}/approve`, { method: 'POST' });
  if (r.ok) loadAdminSuggestions();
}

async function adminRejectSuggestion(sid) {
  const r = await fetch(`/api/kb/suggestions/${sid}/reject`, { method: 'POST' });
  if (r.ok) loadAdminSuggestions();
}
```

- [ ] **Шаг 2: Commit**

```bash
git add app.js
git commit -m "feat: add admin panel — users management, role promotion, suggestions approval"
```

---

## Task 7: Финальная проверка

- [ ] **Шаг 1: Запустить сервер**

```bash
python server.py
```

- [ ] **Шаг 2: Проверить всё в браузере**

Открыть `http://localhost:10000` и проверить:

1. Перейти на `http://localhost:10000/config.json` — должен вернуть пустую страницу (403)
2. В шапке видна кнопка "Войти"
3. Нажать "Войти" → модальное окно с вкладками Вход / Регистрация открывается
4. Зарегистрировать нового пользователя → в шапке появляется chip с именем
5. Открыть раздел "База" → видна кнопка "Предложить тему", не видны кнопки Add/Delete KB
6. Нажать "Предложить тему" → ввести название и содержание → отправить → подтверждение
7. Выйти из аккаунта → кнопка "Войти" вернулась
8. Войти как admin (логин из create_admin.py)
9. В навигации появилась вкладка "Admin"
10. Нажать "Admin" → видны: список пользователей и предложение от тестового пользователя
11. Нажать "Принять" на предложении → предложение исчезает из списка
12. Решить задачу → история сохраняется → перезагрузить страницу → история восстановилась (привязана к аккаунту)

- [ ] **Шаг 3: Финальный commit**

```bash
git add .
git commit -m "feat: complete auth/roles/API key protection — registration, admin panel, suggestions"
```
