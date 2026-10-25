# Django Sessions (`django.contrib.sessions`): Step-by-Step Guide

HTTP is **stateless**: every request arrives at the server as a stranger, and the server forgets you the moment it sends the response. `django.contrib.sessions` is the built-in Django app that **remembers things about a visitor between requests**: "this browser is logged in as user 5", "this visitor has 3 posts in their reading list", "this visitor picked dark mode".

It gives you:

| Part                                   | What it does                                                                                     |
|----------------------------------------|--------------------------------------------------------------------------------------------------|
| **`SessionMiddleware`**                | Reads the session cookie on every request, attaches `request.session`, saves it on the response  |
| **`request.session`**                  | A dictionary-like object: `request.session['theme'] = 'dark'`                                    |
| **`SessionStore` (5 backends)**        | Where the data actually lives: database, cache, cache + database, files, or a signed cookie      |
| **`Session` model**                    | The `django_session` table used by the database backends                                         |
| **`AbstractBaseSession`**              | Base class to build **your own** session model with extra columns (user, IP, device…)            |
| **Expiry API**                         | `set_expiry()`, `get_expiry_age()`, `get_expiry_date()`, browser-close sessions                  |
| **Key management**                     | `cycle_key()` and `flush()` — protect against session fixation, log out cleanly                  |
| **Test cookie**                        | `set_test_cookie()` / `test_cookie_worked()` — check that the browser accepts cookies            |
| **Serializers**                        | Turn session data into text (JSON by default)                                                    |
| **`clearsessions`**                    | Management command that deletes expired sessions                                                 |
| **Async API (Django 5.0+)**            | `aget()`, `aset()`, `apop()`, `aflush()`, … for async views                                      |

Three words you will see a lot:

- **Session** = the data Django keeps for one visitor (one browser), e.g. `{'theme': 'dark', '_auth_user_id': '5'}`.
- **Session key** = a long random string (`'k3j9x0q…'`, 32 characters) that **identifies** one session. It is the only thing stored in the browser (except with the signed-cookie backend).
- **Session cookie** = the cookie called **`sessionid`** that carries the session key back and forth between browser and server.

This guide continues the `blog` project from [django-setup-guide.md](django-setup-guide.md), [django-admin-guide.md](django-admin-guide.md), [django-auth-guide.md](django-auth-guide.md) and [django-contenttypes-guide.md](django-contenttypes-guide.md) (project package `config`, apps `blog`, `accounts` and `social`, custom user `accounts.User`, `Post` with `author`, `slug` and `status`). Every step builds on the previous one. Commands are for **Windows** (PowerShell); they are the same on macOS/Linux except where noted.

---

## Prerequisites

- The project from the earlier guides, with the virtual environment **activated** (`(venv)` in your prompt).
- **Django 5.0+** recommended (needed for the async session API in Step 33). Check with:
  ```bash
  python -m django --version
  ```
- Migrations applied (`python manage.py migrate`) and a superuser created.
- A browser with **Developer Tools** (Chrome, Edge or Firefox — press `F12`).
- If you enabled `LoginRequiredMiddleware` in the auth guide (Step 21), every new **public** view in this guide needs the `@login_not_required` decorator, otherwise anonymous visitors get redirected to the login page. The examples below show it where it matters.

---

## Step 0: How Sessions Work (Big Picture)

### 0.1 The problem

```
Request 1:  GET /post/1/      →  server: "who are you?"  →  response
Request 2:  GET /post/2/      →  server: "who are you?"  →  response     (no memory of request 1)
```

HTTP has no built-in memory. If you want "the same visitor" across requests, the browser has to **send something back every time** that lets the server recognise it. That something is a **cookie**.

### 0.2 The solution — a key in a cookie, the data on the server

```
  BROWSER                                              DJANGO SERVER
  ───────                                              ─────────────
                                                       
  ① GET /session-demo/                                 
     (no cookie yet)          ───────────────────────► SessionMiddleware:
                                                         no 'sessionid' cookie
                                                         → request.session = empty SessionStore
                                                       view:
                                                         request.session['visits'] = 1
                                                       SessionMiddleware (response):
                                                         session was modified
                                                         → generate key 'k3j9x0q…'
                                                         → INSERT into django_session
                              ◄─────────────────────── Set-Cookie: sessionid=k3j9x0q…; HttpOnly; Path=/; …

  browser stores the cookie

  ② GET /session-demo/
     Cookie: sessionid=k3j9x0q…  ────────────────────► SessionMiddleware:
                                                         request.session = SessionStore('k3j9x0q…')
                                                       view:
                                                         request.session['visits']   ← SELECT … WHERE session_key='k3j9x0q…'
                                                         → 1, set it to 2
                                                       SessionMiddleware (response):
                                                         modified → UPDATE django_session
                              ◄─────────────────────── Set-Cookie: sessionid=k3j9x0q…  (expiry refreshed)
```

What is stored where (default **database** backend):

```
  Browser cookie jar                     Database: django_session
  ┌───────────────────────────┐          ┌──────────────────────┬───────────────────────────┬─────────────────────┐
  │ sessionid = k3j9x0q…      │ ───────► │ session_key (PK)     │ session_data              │ expire_date         │
  │ (only the KEY)            │          ├──────────────────────┼───────────────────────────┼─────────────────────┤
  └───────────────────────────┘          │ k3j9x0q…             │ eyJ2aXNpdHMiOjJ9:1t…:Xy…  │ 2026-11-08 10:00:00 │
                                         └──────────────────────┴───────────────────────────┴─────────────────────┘
                                                                  signed, base64-encoded JSON:
                                                                  {"visits": 2}
```

- The browser only holds a **random, unguessable key**. It cannot see or change the data.
- The server holds the **data**, signed with your `SECRET_KEY` so tampering is detected.
- If someone steals the key (the cookie), they **become** that visitor — that is why the cookie is `HttpOnly` and should be `Secure` in production (Step 34).

### 0.3 The full request/response cycle

1. A request arrives.
2. **`SessionMiddleware.process_request`** reads the cookie named `SESSION_COOKIE_NAME` (`'sessionid'`) and creates `request.session = SessionStore(session_key)`. **Nothing is loaded yet** — it is lazy.
3. Other middleware runs. `AuthenticationMiddleware` sets up `request.user`, which (when used) reads `_auth_user_id` from the session → **this is the moment the session is loaded**.
4. Your view reads/writes `request.session` like a dict.
5. **`SessionMiddleware.process_response`** decides:
   - session **empty** and the browser sent a cookie → **delete the cookie**;
   - session **modified** (or `SESSION_SAVE_EVERY_REQUEST = True`) and not empty → **save** it and **send `Set-Cookie`**;
   - session was **accessed** at all → add `Vary: Cookie` to the response (so caches don't mix up users).
6. The response goes back to the browser.

### 0.4 Who uses sessions in your project already

| App / feature                          | What it stores in the session                                  |
|----------------------------------------|----------------------------------------------------------------|
| `django.contrib.auth` (login)          | `_auth_user_id`, `_auth_user_backend`, `_auth_user_hash`       |
| `django.contrib.admin`                 | Uses auth login → same keys                                    |
| `django.contrib.messages`              | `_messages` (when messages don't fit in the cookie)            |
| `PasswordResetConfirmView`             | `_password_reset_token` (hides the token from the URL)         |
| CSRF (only if `CSRF_USE_SESSIONS = True`) | `_csrftoken`                                                |
| Expiry API (`set_expiry()`)            | `_session_expiry`                                              |
| **You** (this guide)                   | `visits`, `recently_viewed`, `reading_list`, `theme`, `feedback`, … |

> **Rule:** keys starting with an underscore (`_`) are **reserved for Django**. Never use names like `_cart` for your own data.

---

## Step 1: Check That Sessions Are Enabled

Sessions are on by default in every new project. Open `config/settings.py` and confirm **two** things.

**1. `INSTALLED_APPS`**

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',          # 👈 provides the Session model + django_session table
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'accounts',
    'blog',
    'social',
]
```

The app is only **required** for the database-based backends (`db` and `cached_db`) because it provides the `Session` model. The cache, file and signed-cookie backends work without it — but leave it in; it is harmless and the default backend needs it.

**2. `MIDDLEWARE`** — order matters:

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',      # 👈 creates request.session
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',   # needs request.session
    'django.contrib.messages.middleware.MessageMiddleware',      # needs request.session
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

**This middleware is what actually makes sessions work.** Without it there is no `request.session` at all.

### 1.1 Middleware ordering rules

| Middleware                                 | Must be placed …                                   | Why                                                          |
|--------------------------------------------|----------------------------------------------------|--------------------------------------------------------------|
| `SessionMiddleware`                        | **before** `AuthenticationMiddleware`              | auth reads the user id from `request.session`                |
| `SessionMiddleware`                        | **before** `MessageMiddleware`                     | session-based message storage needs `request.session`        |
| `SessionMiddleware`                        | **before** `LocaleMiddleware` (if you use it)      | locale middleware may look at session/cookies                |
| `UpdateCacheMiddleware` (if you use it)    | **before** `SessionMiddleware` (i.e. higher)       | the session adds `Vary: Cookie`, which the cache must see    |
| `SecurityMiddleware`                       | above `SessionMiddleware` (first in the list)      | it can redirect HTTP → HTTPS before anything else runs       |

### 1.2 What breaks if you remove it

| You remove …                                 | What happens                                                                                                                |
|----------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| `SessionMiddleware`                          | `ImproperlyConfigured: The Django authentication middleware requires session middleware to be installed…`                     |
| `SessionMiddleware` (admin installed)        | System check error **`admin.E410`**: `'SessionMiddleware' must be in MIDDLEWARE in order to use the admin application.`      |
| `SessionMiddleware` (messages installed)     | `ImproperlyConfigured: The session-based temporary message storage requires session middleware to be installed…`             |
| `'django.contrib.sessions'` app (db backend) | `RuntimeError: Model class django.contrib.sessions.models.Session doesn't declare an explicit app_label and isn't in an application in INSTALLED_APPS.` |

### 1.3 Don't create your own app called `sessions`

The built-in app's **label** is `sessions`. If you run `python manage.py startapp sessions`, Django refuses to start:

```
django.core.exceptions.ImproperlyConfigured: Application labels aren't unique, duplicates: sessions
```

Pick another name (`usersessions`, `devices`, …). We will create `usersessions` in Step 30.

### 1.4 Verify

```bash
python manage.py check
```

```
System check identified no issues (0 silenced).
```

---

## Step 2: Know What the App Contains

The app lives inside Django's source code. Find it:

```bash
python -c "import django.contrib.sessions, os; print(os.path.dirname(django.contrib.sessions.__file__))"
```

```
C:\Users\<you>\...\myproject\venv\Lib\site-packages\django\contrib\sessions
```

```
django/contrib/sessions/
├── __init__.py
├── apps.py                 # SessionsConfig (label 'sessions', verbose name "Sessions")
├── backends/
│   ├── __init__.py
│   ├── base.py             # SessionBase — the dict-like API every backend shares; CreateError, UpdateError
│   ├── db.py               # SessionStore for SESSION_ENGINE = '...backends.db'        (default)
│   ├── cache.py            # SessionStore for '...backends.cache'
│   ├── cached_db.py        # SessionStore for '...backends.cached_db'
│   ├── file.py             # SessionStore for '...backends.file'
│   └── signed_cookies.py   # SessionStore for '...backends.signed_cookies'
├── base_session.py         # AbstractBaseSession, BaseSessionManager
├── exceptions.py           # InvalidSessionKey, SuspiciousSession, SessionInterrupted
├── management/
│   └── commands/
│       └── clearsessions.py
├── middleware.py           # SessionMiddleware
├── migrations/
│   └── 0001_initial.py     # creates django_session
├── models.py               # Session (the concrete model)
└── serializers.py          # JSONSerializer
```

The key idea: **every backend is a module that contains a class called `SessionStore`**. The `SESSION_ENGINE` setting is simply the dotted path of that module. The middleware does this at startup:

```python
# simplified from django/contrib/sessions/middleware.py
from importlib import import_module
from django.conf import settings

engine = import_module(settings.SESSION_ENGINE)    # e.g. django.contrib.sessions.backends.db
SessionStore = engine.SessionStore
```

and on each request:

```python
session_key = request.COOKIES.get(settings.SESSION_COOKIE_NAME)   # 'sessionid' cookie (or None)
request.session = SessionStore(session_key)
```

Class relationships:

```
                    SessionBase   (backends/base.py — dict-like API, expiry, keys, encode/decode)
                         ▲
     ┌──────────────┬────┴─────────┬──────────────────┬───────────────────────┐
     │              │              │                  │                       │
 db.SessionStore  cache.SessionStore  file.SessionStore  signed_cookies.SessionStore
     ▲
     │
 cached_db.SessionStore   (db + cache)

 AbstractBaseSession (models.Model, abstract)  ──►  Session  (table django_session)
```

---

## Step 3: Run Migrations and Look at the Table

### 3.1 The app's migrations

```bash
python manage.py showmigrations sessions
```

```
sessions
 [X] 0001_initial
```

There is only **one** migration. If you see `[ ]`, run `python manage.py migrate`.

### 3.2 See the SQL

```bash
python manage.py sqlmigrate sessions 0001
```

On SQLite:

```sql
BEGIN;
--
-- Create model Session
--
CREATE TABLE "django_session" ("session_key" varchar(40) NOT NULL PRIMARY KEY, "session_data" text NOT NULL, "expire_date" datetime NOT NULL);
CREATE INDEX "django_session_expire_date_a5c62663" ON "django_session" ("expire_date");
COMMIT;
```

### 3.3 The three columns

| Column         | Type                       | What it holds                                                                                    |
|----------------|----------------------------|--------------------------------------------------------------------------------------------------|
| `session_key`  | `varchar(40)`, primary key | The random key that is also in the cookie (Django generates **32** chars: `a-z` and `0-9`)       |
| `session_data` | `text`                     | Your data: serialized to JSON, optionally compressed, base64-encoded and **signed** with `SECRET_KEY` |
| `expire_date`  | `datetime`, **indexed**    | When this session stops being valid. Index makes `clearsessions` and lookups fast                |

The model behind it (simplified from `base_session.py` and `models.py`):

```python
class AbstractBaseSession(models.Model):
    session_key = models.CharField(_('session key'), max_length=40, primary_key=True)
    session_data = models.TextField(_('session data'))
    expire_date = models.DateTimeField(_('expire date'), db_index=True)

    objects = BaseSessionManager()

    class Meta:
        abstract = True
        verbose_name = _('session')
        verbose_name_plural = _('sessions')

    def __str__(self):
        return self.session_key

    @classmethod
    def get_session_store_class(cls):
        raise NotImplementedError

    def get_decoded(self):
        session_store_class = self.get_session_store_class()
        return session_store_class().decode(self.session_data)


class Session(AbstractBaseSession):
    objects = SessionManager()

    @classmethod
    def get_session_store_class(cls):
        from django.contrib.sessions.backends.db import SessionStore
        return SessionStore

    class Meta(AbstractBaseSession.Meta):
        db_table = 'django_session'
```

### 3.4 Look at the rows directly

Log in to the admin once (so at least one session exists), then:

```bash
python manage.py dbshell
```

```sql
.headers on
.mode column
SELECT session_key, expire_date, length(session_data) AS size FROM django_session;
.quit
```

```
session_key                       expire_date                 size
--------------------------------  --------------------------  ----
q8m2z7d5k1v9w3e6r4t0y2u8i5o1p7a3  2026-11-08 10:12:44.512318  164
```

> `dbshell` needs the `sqlite3` command-line program. If you get `CommandError: You appear not to have the 'sqlite3' program installed`, either install it (download "sqlite-tools" from sqlite.org and add it to `PATH`) or use the Django shell instead (Step 28).

---

## Step 4: Understand the Session Cookie

### 4.1 See it in the browser

1. Run the server: `python manage.py runserver`
2. Open `http://127.0.0.1:8000/admin/` and log in.
3. Press **F12** → **Application** tab (Chrome/Edge) or **Storage** tab (Firefox) → **Cookies** → `http://127.0.0.1:8000`.

You'll see:

| Name        | Value                              | Domain     | Path | Expires / Max-Age          | HttpOnly | Secure | SameSite |
|-------------|------------------------------------|------------|------|----------------------------|----------|--------|----------|
| `csrftoken` | `Xo1…`                             | 127.0.0.1  | /    | one year from now          |          |        | Lax      |
| `sessionid` | `q8m2z7d5k1v9w3e6r4t0y2u8i5o1p7a3` | 127.0.0.1  | /    | **2 weeks** from now       | ✓        |        | Lax      |

Compare the `sessionid` value with the `session_key` from Step 3.4 — **they are identical**.

### 4.2 See the raw HTTP headers

Windows 10/11 ship with `curl.exe` (type `curl.exe`, not `curl`, in PowerShell — `curl` is an alias for `Invoke-WebRequest` there). After we build the demo view in Step 5:

```bash
curl.exe -i http://127.0.0.1:8000/session-demo/
```

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Vary: Cookie
Set-Cookie: sessionid=n0cl6c1u5ib0kx5r7kh8d4q0l1a2b3c4; expires=Sun, 08 Nov 2026 10:20:31 GMT; HttpOnly; Max-Age=1209600; Path=/; SameSite=Lax
...
```

Keep the cookie between calls with a cookie jar file:

```bash
curl.exe -c cookies.txt -b cookies.txt http://127.0.0.1:8000/session-demo/
curl.exe -c cookies.txt -b cookies.txt http://127.0.0.1:8000/session-demo/
```

The visit counter goes up because the second call sends the cookie back.

### 4.3 Every cookie attribute explained

| Attribute in `Set-Cookie` | Comes from setting                      | Default       | Meaning                                                                                          |
|---------------------------|-----------------------------------------|---------------|--------------------------------------------------------------------------------------------------|
| `sessionid=…`             | `SESSION_COOKIE_NAME`                   | `'sessionid'` | Cookie name; value is the session key                                                            |
| `expires=…`, `Max-Age=…`  | `SESSION_COOKIE_AGE` / `set_expiry()`   | `1209600` (2 weeks) | When the browser throws the cookie away. **Missing** → deleted when the browser closes       |
| `Domain=…`                | `SESSION_COOKIE_DOMAIN`                 | `None`        | `None` = only the exact host that set it. `'.example.com'` = shared with all subdomains          |
| `Path=…`                  | `SESSION_COOKIE_PATH`                   | `'/'`         | Cookie is only sent for URLs under this path                                                     |
| `HttpOnly`                | `SESSION_COOKIE_HTTPONLY`               | `True`        | JavaScript (`document.cookie`) **cannot read it** → XSS can't steal the session                  |
| `Secure`                  | `SESSION_COOKIE_SECURE`                 | `False`       | Only sent over HTTPS. **Set `True` in production**                                               |
| `SameSite=…`              | `SESSION_COOKIE_SAMESITE`               | `'Lax'`       | Controls sending the cookie on cross-site requests (Step 34.4)                                   |

### 4.4 Cookie facts that surprise beginners

- **Cookies ignore the port.** `127.0.0.1:8000` and `127.0.0.1:8001` share cookies. Two Django projects running at the same time on different ports will **overwrite each other's `sessionid`** and keep logging you out. Fix: give each project a different `SESSION_COOKIE_NAME`.
- **`localhost` and `127.0.0.1` are different hosts.** Log in on one and you're not logged in on the other.
- **Browsers limit a cookie to about 4 KB** — irrelevant for the key, important for the signed-cookie backend (Step 26).
- **Django never puts the session key in the URL.** URL-based sessions leak through logs and the `Referer` header, so they are not supported.

---

## Step 5: Your First Session View — a Visit Counter

### 5.1 The view

Add to `blog/views.py`:

```python
from django.conf import settings
from django.contrib.auth.decorators import login_not_required   # only needed with LoginRequiredMiddleware
from django.http import Http404
from django.shortcuts import render


@login_not_required
def session_demo(request):
    if not settings.DEBUG:
        raise Http404   # this page shows internal session data — development only

    visits = request.session.get('visits', 0) + 1
    request.session['visits'] = visits

    context = {
        'visits': visits,
        'session_key': request.session.session_key,        # None on the very first visit!
        'cookie_value': request.COOKIES.get(settings.SESSION_COOKIE_NAME),
        'data': dict(request.session.items()),
        'expiry_age': request.session.get_expiry_age(),
        'expiry_date': request.session.get_expiry_date(),
        'browser_close': request.session.get_expire_at_browser_close(),
    }
    return render(request, 'blog/session_demo.html', context)
```

> If you did **not** enable `LoginRequiredMiddleware`, you can leave out the `login_not_required` import and decorator (it does nothing harmful either way; it exists since Django 5.1).

### 5.2 The URL

`blog/urls.py`:

```python
urlpatterns = [
    # ... existing patterns ...
    path('session-demo/', views.session_demo, name='session_demo'),
]
```

### 5.3 The template

`blog/templates/blog/session_demo.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Session demo{% endblock %}

{% block content %}
<h2>Session demo</h2>

<p>You have visited this page <strong>{{ visits }}</strong> time{{ visits|pluralize }}.</p>

<table border="1" cellpadding="4">
    <tr><th>Session key (server side)</th><td>{{ session_key|default:"None — not saved yet" }}</td></tr>
    <tr><th>Cookie sent by the browser</th><td>{{ cookie_value|default:"No cookie" }}</td></tr>
    <tr><th>Expires in (seconds)</th><td>{{ expiry_age }}</td></tr>
    <tr><th>Expires at</th><td>{{ expiry_date }}</td></tr>
    <tr><th>Expires when browser closes?</th><td>{{ browser_close|yesno }}</td></tr>
</table>

<h3>Everything in the session</h3>
<ul>
    {% for key, value in data.items %}
        <li><code>{{ key }}</code> = <code>{{ value }}</code></li>
    {% empty %}
        <li>(empty)</li>
    {% endfor %}
</ul>
{% endblock %}
```

### 5.4 Try it — and notice the key behaviour

1. Open a **private/incognito** window (no cookies) and go to `http://127.0.0.1:8000/session-demo/`.
   - Visits: **1**
   - Session key: **None — not saved yet**
   - Cookie: **No cookie**
2. Reload.
   - Visits: **2**
   - Session key: `n0cl6c1u…`
   - Cookie: `n0cl6c1u…` (same value)
3. Log in through `/admin/` in that window and come back: you'll now also see `_auth_user_id`, `_auth_user_backend` and `_auth_user_hash` in the list.

**Why is the key `None` on the first visit?** A new session gets its key only when it is **saved**, and saving happens in `SessionMiddleware.process_response` — **after** your view has run. If you need the key during the first request (e.g. to store it in another table), save explicitly:

```python
if not request.session.session_key:
    request.session.save()      # creates the row and assigns a key right now
key = request.session.session_key
```

### 5.5 Lazy loading

`request.session` exists on every request, but **the database is only queried the first time you touch it**:

```python
def cheap_view(request):
    return HttpResponse('No session query here')            # 0 session queries

def other_view(request):
    theme = request.session.get('theme')                    # 1 SELECT (first access)
    lang = request.session.get('lang')                      # 0 more — data is now cached on the object
    return HttpResponse(theme)
```

Note: using `request.user` (in the view **or** in a template, e.g. `{% if user.is_authenticated %}`) also loads the session, because the user id lives there.

---

## Step 6: The Complete Dictionary-Like API

`request.session` behaves like a Python `dict`. Open the shell to try everything without a browser:

```bash
python manage.py shell
```

```python
>>> from django.contrib.sessions.backends.db import SessionStore
>>> s = SessionStore()          # same class request.session uses
```

### 6.1 Write

```python
>>> s['theme'] = 'dark'                    # set one key
>>> s['visits'] = 1
>>> s.update({'lang': 'en', 'cart': []})   # set several
>>> s.setdefault('visits', 100)            # key exists → returns current value, no change
1
>>> s.setdefault('font', 'large')          # key missing → sets it and returns it
'large'
```

### 6.2 Read

```python
>>> s['theme']                             # KeyError if missing
'dark'
>>> s.get('missing')                       # None if missing
>>> s.get('missing', 'fallback')
'fallback'
>>> 'theme' in s
True
>>> list(s.keys())
['theme', 'visits', 'lang', 'cart', 'font']
>>> list(s.values())
['dark', 1, 'en', [], 'large']
>>> list(s.items())
[('theme', 'dark'), ('visits', 1), ('lang', 'en'), ('cart', []), ('font', 'large')]
```

### 6.3 Delete

```python
>>> del s['font']                          # KeyError if missing
>>> s.pop('lang')                          # remove and return
'en'
>>> s.pop('lang', None)                    # no error if missing
>>> s.clear()                              # remove ALL data, keep the same session key
>>> list(s.keys())
[]
```

### 6.4 Things a session is **not**

- It is **not** a real `dict` subclass: `isinstance(request.session, dict)` is `False`, and you can't `json.dumps(request.session)`. Use `dict(request.session.items())` if you need a plain dict.
- `request.session.has_key('key')` exists and does the same as `'key' in request.session` — prefer the `in` form, it's the normal Python way.
- `bool(request.session)` is `False` for an empty, unsaved session (it calls `is_empty()`), so `if request.session:` does **not** mean "has data in it" for a session that has a key but no data — check specific keys instead.
- Values must be **serializable** (JSON by default) — see Step 8.

### 6.5 Typical view patterns

```python
# Read with default
theme = request.session.get('theme', 'light')

# Counter
request.session['visits'] = request.session.get('visits', 0) + 1

# "Flash" value: read once, then remove
notice = request.session.pop('one_time_notice', None)

# Toggle
request.session['compact'] = not request.session.get('compact', False)

# Remove if present, no error
request.session.pop('wizard_data', None)
```

---

## Step 7: Understand When the Session Is Saved

Django saves the session **only if it was modified** (or `SESSION_SAVE_EVERY_REQUEST = True`). It knows it was modified through the flag `request.session.modified`, which the dict methods set for you.

### 7.1 What sets `modified = True`

| Code                                        | `accessed` | `modified`                        | Saved?                         |
|---------------------------------------------|:----------:|-----------------------------------|--------------------------------|
| `request.session['a'] = 1`                  | ✓          | ✓                                 | ✓                              |
| `del request.session['a']`                  | ✓          | ✓                                 | ✓                              |
| `request.session.update({...})`             | ✓          | ✓                                 | ✓                              |
| `request.session.clear()`                   | ✓          | ✓                                 | ✓ (as an empty session)        |
| `request.session.pop('a')`                  | ✓          | ✓ **only if `'a'` existed**       | only if `'a'` existed          |
| `request.session.setdefault('a', 1)`        | ✓          | ✓ **only if `'a'` was missing**   | only if `'a'` was missing      |
| `request.session.set_expiry(300)`           | ✓          | ✓ (writes `_session_expiry`)      | ✓                              |
| `request.session.get('a')` / `['a']`        | ✓          | ✗                                 | ✗                              |
| `'a' in request.session`, `.keys()`, `.items()` | ✓      | ✗                                 | ✗                              |
| `request.session['cart']['items'].append(5)`| ✓          | **✗ — nested change not detected**| **✗ — change is LOST**         |
| `request.session.modified = True`           | —          | ✓                                 | ✓                              |

### 7.2 The nested-mutation trap

```python
# Request 1
request.session['cart'] = {'items': []}            # saved ✓

# Request 2
request.session['cart']['items'].append(5)          # you changed a list INSIDE the session
# Django only sees a READ of request.session['cart'] — it doesn't know you changed it
# → nothing is saved → on request 3 the cart is still empty ❌
```

Two fixes:

```python
# Fix 1: tell Django
request.session['cart']['items'].append(5)
request.session.modified = True

# Fix 2 (cleaner): read → change → assign back
cart = request.session.get('cart', {'items': []})
cart['items'].append(5)
request.session['cart'] = cart                      # assignment sets modified = True
```

### 7.3 What `SessionMiddleware.process_response` does, exactly

Simplified from `django/contrib/sessions/middleware.py`:

```python
def process_response(self, request, response):
    accessed = request.session.accessed
    modified = request.session.modified
    empty = request.session.is_empty()

    # 1. Session is empty but the browser has a cookie → delete the cookie
    if settings.SESSION_COOKIE_NAME in request.COOKIES and empty:
        response.delete_cookie(settings.SESSION_COOKIE_NAME, path=..., domain=..., samesite=...)
        patch_vary_headers(response, ('Cookie',))
    else:
        # 2. Session was looked at → responses differ per user → Vary: Cookie
        if accessed:
            patch_vary_headers(response, ('Cookie',))

        # 3. Save if modified (or always, with SESSION_SAVE_EVERY_REQUEST), and not empty
        if (modified or settings.SESSION_SAVE_EVERY_REQUEST) and not empty:
            if request.session.get_expire_at_browser_close():
                max_age = None
                expires = None
            else:
                max_age = request.session.get_expiry_age()
                expires = http_date(time.time() + max_age)

            # 4. Never save on a server error (5xx) — the view may have crashed halfway
            if response.status_code < 500:
                try:
                    request.session.save()
                except UpdateError:
                    raise SessionInterrupted(
                        "The request's session was deleted before the request completed. "
                        "The user may have logged out in a concurrent request, for example."
                    )
                response.set_cookie(
                    settings.SESSION_COOKIE_NAME,
                    request.session.session_key,
                    max_age=max_age,
                    expires=expires,
                    domain=settings.SESSION_COOKIE_DOMAIN,
                    path=settings.SESSION_COOKIE_PATH,
                    secure=settings.SESSION_COOKIE_SECURE or None,
                    httponly=settings.SESSION_COOKIE_HTTPONLY or None,
                    samesite=settings.SESSION_COOKIE_SAMESITE,
                )
    return response
```

Consequences worth remembering:

1. **Empty sessions are never stored.** Anonymous visitors who never write anything don't create database rows.
2. **A 500 error discards session changes** made during that request.
3. **A 4xx response (404, 403, …) still saves** session changes.
4. If another request deleted the session while this one was running (e.g. the user clicked "log out" in another tab), you get **`SessionInterrupted`**, which Django turns into a **400 Bad Request**.
5. The cookie is **re-sent** on every save, which refreshes its expiry in the browser.

### 7.4 `SESSION_SAVE_EVERY_REQUEST`

```python
# config/settings.py
SESSION_SAVE_EVERY_REQUEST = True     # default False
```

| `SESSION_SAVE_EVERY_REQUEST` | Session saved when …          | Expiry is counted from …     | Cost                            |
|------------------------------|-------------------------------|------------------------------|---------------------------------|
| `False` (default)            | it was modified               | the last **modification**    | 1 write per modifying request   |
| `True`                       | **every** request (if not empty) | the last **request**      | 1 write **per request**         |

Turn it on when you want a **sliding expiration** ("log out after 30 minutes of doing nothing") — see Step 10.5. Empty sessions are still not created.

---

## Step 8: Understand What You Can Store (Serialization)

Session data must be turned into text to be stored. The **serializer** does that. The default is:

```python
SESSION_SERIALIZER = 'django.contrib.sessions.serializers.JSONSerializer'
```

### 8.1 What JSON can hold

| Python value                          | Can you store it?             | What you get back on the **next** request |
|---------------------------------------|-------------------------------|-------------------------------------------|
| `str`, `int`, `float`, `bool`, `None` | ✓                             | the same                                  |
| `list`                                | ✓                             | `list`                                    |
| `tuple` `(1, 2)`                      | ✓ (silently converted)        | **`list`** `[1, 2]`                       |
| `dict` with `str` keys                | ✓                             | `dict`                                    |
| `dict` with `int` keys `{5: 2}`       | ✓ (silently converted)        | **`{'5': 2}`** — keys become strings!     |
| `datetime`, `date`, `time`            | ✗ `TypeError`                 | —                                         |
| `Decimal`                             | ✗ `TypeError`                 | —                                         |
| `UUID`                                | ✗ `TypeError`                 | —                                         |
| `set`                                 | ✗ `TypeError`                 | —                                         |
| `bytes`                               | ✗ `TypeError`                 | —                                         |
| Model instance (`post`), `QuerySet`   | ✗ `TypeError`                 | —                                         |

### 8.2 The two sneaky bugs

**Bug 1 — the error shows up in the middleware, not your view:**

```python
request.session['last_seen'] = timezone.now()    # no error here!
return render(...)
# → 500 error when the middleware saves:
#   TypeError: Object of type datetime is not JSON serializable
#   (traceback points to django/contrib/sessions/... not to your line)
```

Data is only serialized at save time, so the traceback mentions `session.save()` / `signing.dumps()`. If you see this error, search your code for what you put in the session.

**Bug 2 — int keys become strings, but only from the next request:**

```python
# Request 1
request.session['qty'] = {5: 2}
request.session['qty'][5]          # 2   (still the original Python dict in memory)

# Request 2 (loaded from JSON)
request.session['qty'][5]          # KeyError: 5
request.session['qty']['5']        # 2
```

Fix: always use **string keys** for dicts you put in the session: `{str(post.pk): 2}`.

### 8.3 How to store the "impossible" types

| You want to store …  | Store this instead                     | Turn it back with …                                  |
|----------------------|----------------------------------------|------------------------------------------------------|
| A model instance     | `post.pk`                              | `Post.objects.filter(pk=pk).first()`                 |
| A list of objects    | `[p.pk for p in posts]`                | `Post.objects.filter(pk__in=pks)`                    |
| A `datetime`         | `dt.isoformat()` or `int(dt.timestamp())` | `datetime.fromisoformat(s)` / `datetime.fromtimestamp(n, tz=timezone.utc)` |
| A `Decimal` (money)  | `str(price)` or price in cents (`int`) | `Decimal(s)`                                         |
| A `UUID`             | `str(uuid)`                            | `uuid.UUID(s)`                                       |
| A `set`              | `sorted(my_set)` (list)                | `set(lst)`                                           |

```python
from datetime import datetime
from django.utils import timezone

request.session['last_seen'] = timezone.now().isoformat()            # '2026-10-25T10:12:44.512318+00:00'
last_seen = datetime.fromisoformat(request.session['last_seen'])     # back to an aware datetime
```

> Storing **ids instead of objects** is also the right design: the object in the database may change or be deleted; the session would hold a stale copy.

### 8.4 (Optional) A custom serializer

A serializer is any class with `dumps(obj) -> bytes` and `loads(data: bytes) -> obj`. Example that also accepts `datetime`, `Decimal` and `UUID` by writing them as strings (they come back as **strings**, not converted back):

```python
# config/session_serializers.py
import json

from django.core.serializers.json import DjangoJSONEncoder


class DjangoJSONSessionSerializer:
    def dumps(self, obj):
        return json.dumps(obj, separators=(',', ':'), cls=DjangoJSONEncoder).encode('latin-1')

    def loads(self, data):
        return json.loads(data.decode('latin-1'))
```

```python
# config/settings.py
SESSION_SERIALIZER = 'config.session_serializers.DjangoJSONSessionSerializer'
```

Notes:

- `json.dumps` escapes non-ASCII characters by default, so the output is pure ASCII and `latin-1` encoding is safe — this mirrors what Django's own `JSONSerializer` does.
- **Changing the serializer on a live site** can make existing sessions unreadable (users get logged out). Only do it when that's acceptable.
- **`PickleSerializer` no longer exists** (deprecated in Django 4.1, removed in 5.0). It could store any Python object, but if your `SECRET_KEY` ever leaked, an attacker could craft session data that **runs code on your server**. Don't recreate it.

---

## Step 9: Session Keys — Create, Rotate, Clear, Destroy

### 9.1 The key lifecycle methods

| Method / attribute              | Data         | Key                       | Stored row/entry          | Use it for                                     |
|---------------------------------|--------------|---------------------------|---------------------------|------------------------------------------------|
| `session.session_key`           | —            | read it (`None` if unsaved) | —                       | Identify the session                           |
| `session.save()`                | saved        | created if missing        | inserted / updated        | Force-save now (e.g. to get a key)             |
| `session.create()`              | saved        | **new** key               | new row                   | Low-level: start a brand-new session           |
| `session.clear()`               | **removed**  | kept                      | kept (saved empty)        | Wipe your data, keep the session               |
| `session.cycle_key()`           | **kept**     | **new** key               | old deleted, new created  | After privilege changes (login) — anti-fixation |
| `session.flush()`               | **removed**  | removed (`None`)          | **deleted**               | Logout / "forget me completely"                |
| `session.delete(key=None)`      | —            | —                         | deleted from storage      | Low-level delete (your own key, or another one) |
| `session.exists(key)`           | —            | —                         | checks storage            | "Is there a session with this key?"            |
| `session.is_empty()`            | —            | —                         | —                         | `True` if no key **and** no data               |
| `session.accessed` / `.modified`| —            | —                         | —                         | Flags read by the middleware                   |

### 9.2 How keys are generated

From `backends/base.py`:

```python
VALID_KEY_CHARS = string.ascii_lowercase + string.digits     # a-z0-9  (36 characters)

def _get_new_session_key(self):
    while True:
        session_key = get_random_string(32, VALID_KEY_CHARS)  # cryptographically secure random
        if not self.exists(session_key):
            return session_key
```

- 32 characters from 36 possibilities ≈ **165 bits of randomness** — impossible to guess.
- A key from the cookie shorter than **8** characters is treated as invalid and ignored.
- If the cookie contains a key that **doesn't exist** or is **expired**, Django silently starts a new, empty session (and deletes the old cookie if nothing is written).

### 9.3 See them in action — a demo "actions" view

Add to `blog/views.py`:

```python
from django.shortcuts import redirect
from django.views.decorators.http import require_POST


@login_not_required
@require_POST
def session_action(request):
    if not settings.DEBUG:
        raise Http404

    action = request.POST.get('action')
    if action == 'clear':
        request.session.clear()            # data gone, same key
    elif action == 'flush':
        request.session.flush()            # data gone, key gone, row deleted, cookie deleted
    elif action == 'cycle':
        request.session.cycle_key()        # same data, NEW key
    elif action == 'expire_browser':
        request.session.set_expiry(0)      # cookie dies when the browser closes
    elif action == 'expire_60':
        request.session.set_expiry(60)     # 60 seconds after the last modification
    elif action == 'expire_default':
        request.session.set_expiry(None)   # back to SESSION_COOKIE_AGE
    return redirect('blog:session_demo')
```

`blog/urls.py`:

```python
path('session-demo/action/', views.session_action, name='session_action'),
```

Add the buttons at the bottom of `session_demo.html` (inside `{% block content %}`):

```html
<h3>Try the API</h3>
<form method="post" action="{% url 'blog:session_action' %}">
    {% csrf_token %}
    <button name="action" value="clear">clear()</button>
    <button name="action" value="flush">flush()</button>
    <button name="action" value="cycle">cycle_key()</button>
    <button name="action" value="expire_browser">set_expiry(0)</button>
    <button name="action" value="expire_60">set_expiry(60)</button>
    <button name="action" value="expire_default">set_expiry(None)</button>
</form>
```

What you'll observe (keep DevTools → Cookies open):

| Button              | Visits counter after reload | Session key            | Cookie                         |
|---------------------|-----------------------------|------------------------|--------------------------------|
| `clear()`           | restarts at 1               | **same**               | same                           |
| `flush()`           | restarts at 1               | **new** (after reload) | deleted, then a new one is set |
| `cycle_key()`       | **keeps counting**          | **new**                | new value                      |
| `set_expiry(0)`     | keeps counting              | same                   | Expires: **"Session"**         |
| `set_expiry(60)`    | keeps counting (if within 60 s) | same               | Max-Age 60                     |
| `set_expiry(None)`  | keeps counting              | same                   | Max-Age 1209600 again          |

> If you are logged in, `clear()` and `flush()` **log you out** too — the `_auth_*` keys are part of the session data.

### 9.4 What `cycle_key()` and `flush()` do internally

```python
def cycle_key(self):
    data = self._session          # remember the data
    key = self.session_key
    self.create()                 # new key + new row
    self._session_cache = data    # put the data back
    if key:
        self.delete(key)          # remove the old row

def flush(self):
    self.clear()                  # empty the data
    self.delete()                 # remove the row from storage
    self._session_key = None      # forget the key
```

---

## Step 10: Session Expiry

### 10.1 The three ways a session can end

| Way                                   | Controlled by                                   | Default                 |
|---------------------------------------|-------------------------------------------------|-------------------------|
| After a fixed **age**                 | `SESSION_COOKIE_AGE` (seconds)                  | `1209600` = **2 weeks** |
| When the **browser closes**           | `SESSION_EXPIRE_AT_BROWSER_CLOSE = True`        | `False`                 |
| **Per session**, decided in code      | `request.session.set_expiry(value)`             | —                       |

```python
# config/settings.py
SESSION_COOKIE_AGE = 60 * 60 * 24 * 7          # 1 week
SESSION_EXPIRE_AT_BROWSER_CLOSE = False
```

### 10.2 `set_expiry(value)` — every accepted value

| `value`                        | Meaning                                                                                       |
|--------------------------------|-----------------------------------------------------------------------------------------------|
| `int` > 0, e.g. `300`          | Expire **300 seconds after the last modification**                                            |
| `0`                            | Cookie expires **when the browser closes**                                                    |
| `datetime`                     | Expire at that exact moment (stored as an ISO string, so it works with JSON)                  |
| `timedelta`                    | Converted to `now + timedelta`, then treated as a `datetime`                                  |
| `None`                         | Remove the custom expiry → fall back to `SESSION_COOKIE_AGE` / `SESSION_EXPIRE_AT_BROWSER_CLOSE` |

```python
from datetime import timedelta
from django.utils import timezone

request.session.set_expiry(300)                                   # 5 minutes
request.session.set_expiry(0)                                     # browser close
request.session.set_expiry(timedelta(hours=1))                    # 1 hour from now
request.session.set_expiry(timezone.now() + timedelta(days=30))   # fixed date
request.session.set_expiry(None)                                  # back to settings
```

Internally it just stores the value in the session under the reserved key `_session_expiry` — that's why it marks the session as modified.

### 10.3 Read the expiry

| Method                                  | Returns                                                                                       |
|-----------------------------------------|-----------------------------------------------------------------------------------------------|
| `get_expiry_age()`                      | Seconds until the session expires (counted from **now**)                                      |
| `get_expiry_date()`                     | The `datetime` when it expires                                                                |
| `get_expire_at_browser_close()`         | `True` if the cookie dies with the browser (custom `0`, or the global setting with no custom expiry) |
| `get_session_cookie_age()`              | `SESSION_COOKIE_AGE` (override it in a custom backend to compute the age dynamically)         |

Both `get_expiry_age()` and `get_expiry_date()` accept optional keyword arguments `modification=` (a `datetime`, default now) and `expiry=` (default: the session's stored expiry) — used by backends to compute expiry for data they've loaded.

### 10.4 Important details

1. **Reading does not extend a session.** Expiry is counted from the last time the session was **saved**. A user who only reads pages for two weeks will be logged out after two weeks, even if they were active (unless `SESSION_SAVE_EVERY_REQUEST = True`).
2. **"Browser close" is less reliable than it sounds.** Chrome/Edge/Firefox "Continue where you left off" restores session cookies after a restart. Don't rely on it for security.
3. **Browser-close sessions still have a server-side limit.** With `set_expiry(0)` the database row's `expire_date` is still `now + SESSION_COOKIE_AGE`, so the session dies on the server after 2 weeks even if the browser never closes.
4. **Two clocks expire a session**: the browser drops the cookie, *and* the server refuses keys whose `expire_date` has passed. Either one ends the session.
5. **Expired rows are not deleted automatically** with the `db` and `file` backends. Run `clearsessions` (Step 29).

### 10.5 Sliding expiration ("log out after 30 minutes of inactivity")

The simplest way — for **all** visitors:

```python
# config/settings.py
SESSION_COOKIE_AGE = 30 * 60              # 30 minutes
SESSION_SAVE_EVERY_REQUEST = True         # every request pushes the expiry forward
```

Each request saves the session, so the 30 minutes restart on every click. Cost: one database write per request. For a more selective version (only for logged-in users, with fewer writes), see the idle-timeout middleware in Step 16.

### 10.6 "Remember me" on the login page

The auth guide (Step 19.3) already built `RememberMeLoginView`. The session part of it is exactly this:

```python
if not self.request.POST.get('remember_me'):
    self.request.session.set_expiry(0)       # forget when the browser closes
# ticked → default SESSION_COOKIE_AGE (2 weeks)
```

A common variant — long sessions only when ticked:

```python
# config/settings.py
SESSION_COOKIE_AGE = 60 * 60 * 8                     # 8 hours by default

# accounts/views.py, inside RememberMeLoginView.form_valid, after super().form_valid(form)
if self.request.POST.get('remember_me'):
    self.request.session.set_expiry(60 * 60 * 24 * 30)  # 30 days
else:
    self.request.session.set_expiry(0)
```

---

## Step 11: All Session Settings Explained

Put any of these in `config/settings.py`. You only need to add the ones you want to change.

```python
# --- Storage ---
SESSION_ENGINE = 'django.contrib.sessions.backends.db'        # which backend (Step 21)
SESSION_CACHE_ALIAS = 'default'                               # which CACHES entry (cache/cached_db backends)
SESSION_FILE_PATH = None                                      # folder for the file backend (None = system temp dir)
SESSION_SERIALIZER = 'django.contrib.sessions.serializers.JSONSerializer'

# --- Cookie ---
SESSION_COOKIE_NAME = 'sessionid'
SESSION_COOKIE_AGE = 1209600                                  # seconds (2 weeks)
SESSION_COOKIE_DOMAIN = None                                  # e.g. '.example.com' to share across subdomains
SESSION_COOKIE_PATH = '/'
SESSION_COOKIE_SECURE = False                                 # True in production (HTTPS only)
SESSION_COOKIE_HTTPONLY = True                                # keep True
SESSION_COOKIE_SAMESITE = 'Lax'                               # 'Lax', 'Strict', 'None' or False

# --- Behaviour ---
SESSION_EXPIRE_AT_BROWSER_CLOSE = False
SESSION_SAVE_EVERY_REQUEST = False
```

| Setting                           | Default                                                   | When to change it                                                                                                   |
|-----------------------------------|-----------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| `SESSION_ENGINE`                  | `'django.contrib.sessions.backends.db'`                   | Faster sessions (cache/cached_db), no database (signed_cookies), custom model                                       |
| `SESSION_CACHE_ALIAS`             | `'default'`                                               | Keep sessions in a separate cache from your page cache                                                              |
| `SESSION_FILE_PATH`               | `None` (→ `tempfile.gettempdir()`, on Windows `C:\Users\<you>\AppData\Local\Temp`) | Using the file backend; point it at a private, persistent folder                    |
| `SESSION_SERIALIZER`              | `'django.contrib.sessions.serializers.JSONSerializer'`    | Rarely; see Step 8.4                                                                                                |
| `SESSION_COOKIE_NAME`             | `'sessionid'`                                             | Several Django projects on the same host; must differ from `CSRF_COOKIE_NAME` and `LANGUAGE_COOKIE_NAME`            |
| `SESSION_COOKIE_AGE`              | `1209600` (2 weeks)                                       | Banking-style short sessions, or long "stay logged in" sessions                                                     |
| `SESSION_COOKIE_DOMAIN`           | `None`                                                    | Log in once on `example.com`, stay logged in on `blog.example.com` → `'.example.com'`                               |
| `SESSION_COOKIE_PATH`             | `'/'`                                                     | Several Django apps on one domain under different paths (`/app1/`, `/app2/`)                                        |
| `SESSION_COOKIE_SECURE`           | `False`                                                   | **Always `True` in production** (HTTPS)                                                                             |
| `SESSION_COOKIE_HTTPONLY`         | `True`                                                    | Practically never. Your JavaScript has no reason to read the session key                                            |
| `SESSION_COOKIE_SAMESITE`         | `'Lax'`                                                   | `'Strict'` for extra CSRF protection; `'None'` (+ Secure) if your site runs inside a cross-site iframe              |
| `SESSION_EXPIRE_AT_BROWSER_CLOSE` | `False`                                                   | Shared/public computers                                                                                             |
| `SESSION_SAVE_EVERY_REQUEST`      | `False`                                                   | Sliding expiration (Step 10.5)                                                                                      |

Related settings from other apps:

| Setting                 | Default  | Relation to sessions                                                                                 |
|-------------------------|----------|------------------------------------------------------------------------------------------------------|
| `SECRET_KEY`            | —        | Signs session data. **Changing it invalidates every session** (everyone is logged out)              |
| `SECRET_KEY_FALLBACKS`  | `[]`     | Old keys still accepted for verifying — rotate `SECRET_KEY` without logging everyone out (Step 34.6) |
| `CSRF_USE_SESSIONS`     | `False`  | `True` stores the CSRF token in the session instead of the `csrftoken` cookie (Step 20.2)           |
| `MESSAGE_STORAGE`       | `FallbackStorage` | Messages go to a cookie first, then the session if too big (Step 20.1)                      |

> Changing **cookie** settings (`NAME`, `DOMAIN`, `PATH`) on a live site effectively logs everyone out — browsers keep sending the old cookie, which Django no longer looks at.

---

## Step 12: Example 1 — "Recently Viewed Posts"

Goal: remember the last 5 posts a visitor opened (works for anonymous visitors too) and show them on the post list.

### 12.1 Record views in `post_detail`

Update `post_detail` in `blog/views.py` (the version from the auth guide, Step 28):

```python
RECENTLY_VIEWED_KEY = 'recently_viewed'
RECENTLY_VIEWED_MAX = 5


def post_detail(request, pk):
    post = get_object_or_404(Post, pk=pk)
    is_owner = request.user.is_authenticated and post.author == request.user
    if post.status == 'draft' and not (is_owner or request.user.has_perm('blog.change_post')):
        return redirect('blog:post_list')

    if post.status == 'published':
        recent = request.session.get(RECENTLY_VIEWED_KEY, [])
        if post.pk in recent:
            recent.remove(post.pk)              # move it to the front
        recent.insert(0, post.pk)
        request.session[RECENTLY_VIEWED_KEY] = recent[:RECENTLY_VIEWED_MAX]   # assign back → saved

    return render(request, 'blog/post_detail.html', {'post': post, 'is_owner': is_owner})
```

Points to notice:

- We store **ids** (`int`), not `Post` objects — JSON-safe and never stale.
- We **assign the list back** to the session, so Django knows it changed (Step 7.2).
- Only **published** posts are remembered, so a draft can't leak through the list later.

### 12.2 A helper to turn ids back into posts (keeping the order)

```python
def posts_from_ids(ids):
    """Return published posts for the given ids, in the same order, skipping deleted ones."""
    posts_by_id = Post.objects.filter(pk__in=ids, status='published').in_bulk()
    return [posts_by_id[pk] for pk in ids if pk in posts_by_id]
```

`in_bulk()` returns `{pk: post}` in **one query**. A post that was deleted or unpublished since it was viewed is simply skipped.

### 12.3 Show them on the post list

```python
def post_list(request):
    posts = Post.objects.filter(status='published')
    if request.user.is_authenticated:
        posts = Post.objects.filter(Q(status='published') | Q(author=request.user))
    recently_viewed = posts_from_ids(request.session.get(RECENTLY_VIEWED_KEY, []))
    return render(request, 'blog/post_list.html', {'posts': posts, 'recently_viewed': recently_viewed})
```

`blog/templates/blog/post_list.html` — add:

```html
{% if recently_viewed %}
    <aside>
        <h3>Recently viewed</h3>
        <ul>
            {% for post in recently_viewed %}
                <li><a href="{% url 'blog:post_detail' post.pk %}">{{ post.title }}</a></li>
            {% endfor %}
        </ul>
    </aside>
{% endif %}
```

Test: open three posts, go back to the list → they appear newest first. Open a private window → the list is empty (different browser = different session).

---

## Step 13: Example 2 — A Reading List (Session "Cart")

Goal: an anonymous-friendly "save for later" list — exactly the same pattern as a shopping cart.

### 13.1 Keep the session logic in one place

Create `blog/reading_list.py`:

```python
SESSION_KEY = 'reading_list'


def get_ids(session):
    return session.get(SESSION_KEY, [])


def add(session, post_id):
    ids = get_ids(session)
    if post_id not in ids:
        ids.append(post_id)
        session[SESSION_KEY] = ids


def remove(session, post_id):
    ids = get_ids(session)
    if post_id in ids:
        ids.remove(post_id)
        session[SESSION_KEY] = ids


def clear(session):
    session.pop(SESSION_KEY, None)


def count(session):
    return len(get_ids(session))
```

Wrapping the session in small functions means the key name and data shape live in **one file**. If you later move the list to a database table for logged-in users, only this file changes.

### 13.2 Views

`blog/views.py`:

```python
from django.contrib import messages

from . import reading_list as rl


@login_not_required
def reading_list(request):
    posts = posts_from_ids(rl.get_ids(request.session))
    return render(request, 'blog/reading_list.html', {'posts': posts})


@login_not_required
@require_POST
def reading_list_add(request, pk):
    post = get_object_or_404(Post, pk=pk, status='published')
    rl.add(request.session, post.pk)
    messages.success(request, f'"{post.title}" was added to your reading list.')
    return redirect('blog:post_detail', pk=post.pk)


@login_not_required
@require_POST
def reading_list_remove(request, pk):
    rl.remove(request.session, pk)
    messages.info(request, 'Removed from your reading list.')
    return redirect('blog:reading_list')


@login_not_required
@require_POST
def reading_list_clear(request):
    rl.clear(request.session)
    return redirect('blog:reading_list')
```

**Why POST?** Changing state with a GET link is dangerous: link prefetchers, crawlers or a malicious `<img src="/reading-list/add/3/">` on another site could change the list. POST + `{% csrf_token %}` prevents that.

### 13.3 URLs

```python
path('reading-list/', views.reading_list, name='reading_list'),
path('reading-list/add/<int:pk>/', views.reading_list_add, name='reading_list_add'),
path('reading-list/remove/<int:pk>/', views.reading_list_remove, name='reading_list_remove'),
path('reading-list/clear/', views.reading_list_clear, name='reading_list_clear'),
```

### 13.4 Templates

Button in `blog/templates/blog/post_detail.html`:

```html
<form method="post" action="{% url 'blog:reading_list_add' post.pk %}">
    {% csrf_token %}
    <button type="submit">📌 Save for later</button>
</form>
```

`blog/templates/blog/reading_list.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Reading list{% endblock %}

{% block content %}
<h2>My reading list ({{ posts|length }})</h2>

{% for post in posts %}
    <p>
        <a href="{% url 'blog:post_detail' post.pk %}">{{ post.title }}</a>
        <form method="post" action="{% url 'blog:reading_list_remove' post.pk %}" style="display:inline">
            {% csrf_token %}
            <button type="submit">Remove</button>
        </form>
    </p>
{% empty %}
    <p>Nothing saved yet.</p>
{% endfor %}

{% if posts %}
    <form method="post" action="{% url 'blog:reading_list_clear' %}">
        {% csrf_token %}
        <button type="submit">Clear list</button>
    </form>
{% endif %}
{% endblock %}
```

### 13.5 What happens to the list on login and logout?

| Event                                    | What Django does to the session        | Reading list                       |
|------------------------------------------|----------------------------------------|------------------------------------|
| Anonymous visitor **logs in**            | `cycle_key()` — new key, **data kept** | ✓ kept                             |
| Logged-in user logs in **as someone else** | `flush()` — data wiped               | ✗ lost                             |
| User **logs out**                        | `flush()` — data wiped                 | ✗ lost                             |

To keep the list after logout, read it before `logout()` and put it back afterwards:

```python
# accounts/views.py
from django.contrib.auth import logout
from blog import reading_list as rl


@require_POST
def logout_keep_reading_list(request):
    ids = rl.get_ids(request.session)       # 1. save
    logout(request)                         # 2. flush (new empty session)
    if ids:
        request.session[rl.SESSION_KEY] = ids   # 3. restore into the NEW session
    return redirect('blog:post_list')
```

> For a list that follows a **user** across devices, store it in a database table (`ReadingListItem(user, post)`) and, on login, merge the session list into it with a `user_logged_in` signal. Sessions are per **browser**.

---

## Step 14: Use the Session in Templates (Context Processor)

### 14.1 Direct access

The `request` context processor is on by default, so every template can read the session:

```html
{% if request.session.theme == 'dark' %} … {% endif %}
{{ request.session.visits|default:0 }}
```

Check it's enabled in `config/settings.py`:

```python
TEMPLATES = [
    {
        # ...
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.request',     # 👈 provides {{ request }}
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

> Templates **cannot** read keys that start with an underscore (`{{ request.session._auth_user_id }}` raises a `TemplateSyntaxError`). That's fine — those are Django's.

### 14.2 A context processor for computed values

Templates should not know the internal shape of your session data. Create `blog/context_processors.py`:

```python
from . import reading_list as rl


def session_extras(request):
    return {
        'reading_list_count': rl.count(request.session),
        'theme': request.session.get('theme', 'light'),
    }
```

Register it:

```python
'context_processors': [
    'django.template.context_processors.request',
    'django.contrib.auth.context_processors.auth',
    'django.contrib.messages.context_processors.messages',
    'blog.context_processors.session_extras',          # 👈 add
],
```

Use it in `blog/templates/blog/base.html` (inside `<nav>`):

```html
<a href="{% url 'blog:reading_list' %}">Reading list ({{ reading_list_count }})</a> |
```

> A context processor runs for **every** template rendered with `render()`, so it loads the session on every page. That's normally fine (the auth context processor already does it), but keep context processors cheap.

---

## Step 15: Example 3 — Theme Preference (Light / Dark)

### 15.1 View

```python
from django.utils.http import url_has_allowed_host_and_scheme

THEMES = ('light', 'dark')


@login_not_required
@require_POST
def set_theme(request):
    theme = request.POST.get('theme')
    if theme in THEMES:                       # never store unvalidated input
        request.session['theme'] = theme

    next_url = request.POST.get('next', '/')
    if not url_has_allowed_host_and_scheme(next_url, allowed_hosts={request.get_host()},
                                           require_https=request.is_secure()):
        next_url = '/'                        # block open redirects to other sites
    return redirect(next_url)
```

```python
path('theme/', views.set_theme, name='set_theme'),
```

### 15.2 Template

In `blog/templates/blog/base.html`:

```html
<body class="theme-{{ theme }}">
    <style>
        .theme-dark { background: #111; color: #eee; }
        .theme-dark a { color: #8ab4f8; }
    </style>

    <form method="post" action="{% url 'blog:set_theme' %}" style="display:inline">
        {% csrf_token %}
        <input type="hidden" name="next" value="{{ request.get_full_path }}">
        {% if theme == 'dark' %}
            <button name="theme" value="light">☀ Light</button>
        {% else %}
            <button name="theme" value="dark">🌙 Dark</button>
        {% endif %}
    </form>
    ...
```

### 15.3 Session or plain cookie?

A theme is not secret and doesn't need the server to remember it. You could also store it in a normal cookie with `response.set_cookie('theme', 'dark')`. Compare:

| Store it in …          | Pros                                                         | Cons                                                         |
|------------------------|--------------------------------------------------------------|--------------------------------------------------------------|
| Session                | Server-controlled, can't be edited by the user, survives in one place with other data | Creates a session row even for anonymous visitors; lost on logout (`flush`) |
| Plain cookie           | No database row, readable by JavaScript, survives logout     | User can edit it (always validate), sent with every request  |
| User model / profile   | Follows the user to every device                             | Only for logged-in users                                     |

Rule of thumb: **sessions for things the server must trust or that belong to the current visit; plain cookies for harmless UI preferences; the database for things that belong to a user forever.**

---

## Step 16: Example 4 — Idle Timeout for Logged-In Users (Middleware)

Goal: log users out after 15 minutes without a request, but only write to the session **at most once a minute**.

Create `accounts/middleware.py`:

```python
import time

from django.conf import settings
from django.contrib import messages
from django.contrib.auth import logout
from django.shortcuts import redirect

LAST_ACTIVITY_KEY = 'last_activity'


class IdleTimeoutMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        self.timeout = getattr(settings, 'SESSION_IDLE_TIMEOUT', 15 * 60)

    def __call__(self, request):
        if request.user.is_authenticated:
            now = int(time.time())
            last = request.session.get(LAST_ACTIVITY_KEY, now)

            if now - last > self.timeout:
                logout(request)                                        # flushes the session
                messages.info(request, 'You were logged out after a period of inactivity.')
                return redirect(settings.LOGIN_URL)

            if now - last >= 60 or LAST_ACTIVITY_KEY not in request.session:
                request.session[LAST_ACTIVITY_KEY] = now               # write at most once a minute

        return self.get_response(request)
```

Register it **after** `AuthenticationMiddleware` (it needs `request.user`) and `MessageMiddleware` (it adds a message):

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'accounts.middleware.IdleTimeoutMiddleware',                # 👈 add
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

SESSION_IDLE_TIMEOUT = 15 * 60      # our own setting (seconds)
```

How it works:

- We store a **Unix timestamp** (`int`) — JSON-safe, unlike a `datetime`.
- Without the "once a minute" check, every single request would write to the session table.
- Because the session is saved at least every minute, its `expire_date` keeps moving forward too, so the normal `SESSION_COOKIE_AGE` doesn't cut an active user off.
- The message survives the logout because the default message storage puts small messages in a **cookie**, not the (now flushed) session.

Test quickly by setting `SESSION_IDLE_TIMEOUT = 10`, logging in, waiting 15 seconds and reloading.

---

## Step 17: Example 5 — A Multi-Step Form (Wizard)

Goal: a 3-step feedback form. Data from earlier steps is kept in the session until the last step.

### 17.1 Forms

`blog/forms.py`:

```python
from django import forms


class FeedbackContactForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()


class FeedbackMessageForm(forms.Form):
    TOPICS = [('bug', 'Bug report'), ('idea', 'Idea'), ('other', 'Other')]
    topic = forms.ChoiceField(choices=TOPICS)
    message = forms.CharField(widget=forms.Textarea)
```

### 17.2 View

```python
from .forms import FeedbackContactForm, FeedbackMessageForm

WIZARD_KEY = 'feedback_wizard'
WIZARD_FORMS = {1: FeedbackContactForm, 2: FeedbackMessageForm}


@login_not_required
def feedback(request, step=1):
    data = request.session.get(WIZARD_KEY, {})

    # Don't allow skipping steps: step N needs the data of step N-1
    if step > 1 and str(step - 1) not in data:
        return redirect('blog:feedback', step=1)

    if step == 3:                                           # confirmation step
        if request.method == 'POST':
            # ... save to a model or send an email here ...
            request.session.pop(WIZARD_KEY, None)           # done — remove wizard data
            messages.success(request, 'Thanks for your feedback!')
            return redirect('blog:post_list')
        summary = {**data['1'], **data['2']}
        return render(request, 'blog/feedback_confirm.html', {'summary': summary})

    form_class = WIZARD_FORMS[step]
    if request.method == 'POST':
        form = form_class(request.POST)
        if form.is_valid():
            data[str(step)] = form.cleaned_data             # str key: JSON turns int keys into str anyway
            request.session[WIZARD_KEY] = data              # assign back → saved
            return redirect('blog:feedback', step=step + 1)
    else:
        form = form_class(initial=data.get(str(step)))      # pre-fill when going back
    return render(request, 'blog/feedback_step.html', {'form': form, 'step': step})
```

```python
path('feedback/', views.feedback, name='feedback'),
path('feedback/<int:step>/', views.feedback, name='feedback'),
```

Notes:

- `cleaned_data` here only contains strings, so it's JSON-safe. A form with a `DateField`, `DecimalField` or `ModelChoiceField` would put `date`/`Decimal`/model objects in `cleaned_data` → convert them first (or store `request.POST` values) — see Step 8.
- The dict key is `str(step)` because JSON would turn `1` into `'1'` on the next request anyway (Step 8.2).
- `<int:step>` accepts any number; `WIZARD_FORMS[step]` raises `KeyError` for `step=7`. Add `if step not in (1, 2, 3): raise Http404` at the top in real code.

### 17.3 Templates

`blog/templates/blog/feedback_step.html`:

```html
{% extends 'blog/base.html' %}

{% block content %}
<h2>Feedback — step {{ step }} of 3</h2>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    {% if step > 1 %}<a href="{% url 'blog:feedback' step|add:'-1' %}">← Back</a>{% endif %}
    <button type="submit">Next →</button>
</form>
{% endblock %}
```

`blog/templates/blog/feedback_confirm.html`:

```html
{% extends 'blog/base.html' %}

{% block content %}
<h2>Feedback — confirm</h2>
<dl>
    {% for field, value in summary.items %}
        <dt>{{ field|capfirst }}</dt><dd>{{ value|linebreaksbr }}</dd>
    {% endfor %}
</dl>
<form method="post">
    {% csrf_token %}
    <a href="{% url 'blog:feedback' 2 %}">← Back</a>
    <button type="submit">Send</button>
</form>
{% endblock %}
```

> For complex wizards, the `django-formtools` package has a ready-made `SessionWizardView` that does exactly this.

---

## Step 18: Sessions and Authentication

Authentication is the biggest user of sessions. Here's exactly how they interact.

### 18.1 What `login()` does to the session

Simplified from `django/contrib/auth/__init__.py`:

```python
def login(request, user, backend=None):
    session_auth_hash = user.get_session_auth_hash()

    if SESSION_KEY in request.session:                       # someone is already logged in
        if (_get_user_session_key(request) != user.pk
                or not constant_time_compare(request.session.get(HASH_SESSION_KEY, ''), session_auth_hash)):
            request.session.flush()                          # different user → wipe everything
    else:
        request.session.cycle_key()                          # anonymous → new key, keep data

    request.session[SESSION_KEY] = user._meta.pk.value_to_string(user)     # '_auth_user_id'
    request.session[BACKEND_SESSION_KEY] = backend                         # '_auth_user_backend'
    request.session[HASH_SESSION_KEY] = session_auth_hash                  # '_auth_user_hash'
    request.user = user
    rotate_token(request)                                    # new CSRF token too
    user_logged_in.send(sender=user.__class__, request=request, user=user)
```

| Session key           | Example value                                    | Purpose                                                       |
|-----------------------|--------------------------------------------------|---------------------------------------------------------------|
| `_auth_user_id`       | `'5'` (always a string)                          | Which user this session belongs to                            |
| `_auth_user_backend`  | `'django.contrib.auth.backends.ModelBackend'`    | Which backend to ask to load the user                         |
| `_auth_user_hash`     | `'3f9a…'`                                        | HMAC of the user's password hash — detects password changes   |

### 18.2 What happens on every later request

`AuthenticationMiddleware` sets `request.user` lazily. When used, `get_user(request)`:

1. Reads `_auth_user_id` and `_auth_user_backend` from the session.
2. Asks the backend for the user (`SELECT … FROM accounts_user WHERE id = 5`).
3. Compares `_auth_user_hash` with `user.get_session_auth_hash()`.
4. Mismatch (password was changed elsewhere) → `request.session.flush()` → user is anonymous.

So a logged-in page costs **two queries**: one for the session, one for the user.

### 18.3 What `logout()` does

```python
def logout(request):
    user_logged_out.send(sender=..., request=request, user=request.user)
    request.session.flush()                  # all data gone, row deleted, new key next time
    request.user = AnonymousUser()
```

### 18.4 Session fixation — why the key changes on login

The attack it prevents:

```
1. Attacker visits your site, gets sessionid=AAA.
2. Attacker tricks the victim's browser into using sessionid=AAA
   (e.g. via a vulnerable subdomain that sets a cookie for .example.com).
3. Victim logs in. If the key stayed AAA, the attacker's browser — also using AAA — would now be logged in as the victim.
```

Because `login()` calls `cycle_key()`, the victim gets a **new** key `BBB` at login; the attacker's `AAA` is deleted and worthless.

**Rule for your own code:** whenever a user's privileges go **up** inside an existing session (log in, "sudo mode", switching to an admin role), call `request.session.cycle_key()`.

### 18.5 Password changes log out other devices

`_auth_user_hash` is derived from the password hash. When the password changes, every **other** session's stored hash no longer matches → those sessions are logged out on their next request. To keep the **current** session logged in after changing a password in your own view, call `update_session_auth_hash(request, user)` (auth guide, Step 31.2). The built-in `PasswordChangeView` does this for you.

---

## Step 19: The Test Cookie — Does the Browser Accept Cookies?

Sessions need cookies. If a browser blocks them, login silently fails (the user submits the form, gets "logged in", and the next page shows them anonymous). You can detect that with the **test cookie** API:

| Method                    | What it does                                                         |
|---------------------------|----------------------------------------------------------------------|
| `set_test_cookie()`       | Stores `'testcookie': 'worked'` in the session                       |
| `test_cookie_worked()`    | `True` if that value came back — i.e. the browser sent the cookie    |
| `delete_test_cookie()`    | Removes it again (clean up)                                          |

It needs **two requests**: set it while showing the form (GET), check it when the form is submitted (POST).

```python
# accounts/views.py
from django.contrib.auth import login
from django.contrib.auth.forms import AuthenticationForm


def cookie_checked_login(request):
    if request.method == 'POST':
        if not request.session.test_cookie_worked():
            return render(request, 'registration/login.html', {
                'form': AuthenticationForm(request),
                'cookie_error': 'Please enable cookies in your browser and try again.',
            })
        request.session.delete_test_cookie()

        form = AuthenticationForm(request, data=request.POST)
        if form.is_valid():
            login(request, form.get_user())
            return redirect(settings.LOGIN_REDIRECT_URL)
    else:
        form = AuthenticationForm(request)

    request.session.set_test_cookie()       # for the NEXT request (the POST)
    return render(request, 'registration/login.html', {'form': form})
```

In `templates/registration/login.html`:

```html
{% if cookie_error %}<p class="error">{{ cookie_error }}</p>{% endif %}
```

Test it: in Chrome go to Settings → Privacy → Third-party cookies → "Block all cookies" for `127.0.0.1`, then try to log in.

> Simpler alternative: the CSRF check already requires a cookie (`csrftoken`), so a browser with cookies off gets a **403 CSRF verification failed** page on any form POST. The test cookie lets you show a friendlier message.

---

## Step 20: Other Django Features That Use Sessions

### 20.1 Messages framework

```python
messages.success(request, 'Saved!')
```

The default `MESSAGE_STORAGE` is `'django.contrib.messages.storage.fallback.FallbackStorage'`:

1. Try to store messages in a signed **cookie** (`messages`).
2. If they're too big for a cookie, store the rest in the **session** under `_messages`.

Force session storage (e.g. to keep cookies small):

```python
MESSAGE_STORAGE = 'django.contrib.messages.storage.session.SessionStorage'
```

Note: with `SessionStorage`, messages added right **before** a `logout()` are lost (logout flushes the session). With the default `FallbackStorage`, small messages survive because they're in the cookie — that's what Step 16 relies on.

### 20.2 CSRF token in the session

```python
CSRF_USE_SESSIONS = True      # default False
```

- The CSRF token is stored as `_csrftoken` in the session instead of the `csrftoken` cookie.
- JavaScript can no longer read the token from a cookie; get it from the page (`{% csrf_token %}` hidden input or a `<meta>` tag).
- Every visitor who sees a form now gets a session row.
- Usually not needed — the cookie approach is safe. Use it if a security policy forbids non-HttpOnly cookies.

### 20.3 Password reset

`PasswordResetConfirmView` puts the token from the email link into the session as `_password_reset_token` and redirects to `/reset/<uidb64>/set-password/`, so the token doesn't leak through the `Referer` header to images or scripts on the "set new password" page.

### 20.4 Admin

The admin doesn't store anything special in the session; it relies on the `_auth_*` keys from login. Logging in to `/admin/` and to your site is the **same** session — log out of one and you're logged out of both.

---

## Step 21: Choose a Session Backend

`SESSION_ENGINE` decides **where the data lives**. Your views don't change — every backend offers the same `request.session` API.

| `SESSION_ENGINE`                                    | Data stored in              | Speed       | Survives restart? | Server-side logout? | Needs                         |
|-----------------------------------------------------|-----------------------------|-------------|-------------------|---------------------|-------------------------------|
| `django.contrib.sessions.backends.db` **(default)** | Database table              | OK (1 query)| ✓                 | ✓                   | sessions app + migrations     |
| `django.contrib.sessions.backends.cache`            | Cache only (Redis/Memcached)| Fastest     | ✗ (lost on cache flush/eviction) | ✓    | A shared cache                |
| `django.contrib.sessions.backends.cached_db`        | Cache **+** database        | Fast reads  | ✓                 | ✓                   | Cache + sessions app          |
| `django.contrib.sessions.backends.file`             | Files on disk               | OK          | ✓                 | ✓                   | A writable folder             |
| `django.contrib.sessions.backends.signed_cookies`   | The cookie itself (browser) | No storage  | ✓                 | **✗**               | Nothing                       |

Quick decision guide:

```
Is this a small/medium site and you just want it to work?        → db (default)
Do you already run Redis/Memcached and want speed + persistence?  → cached_db
Is losing all sessions on a cache restart acceptable?             → cache
No database at all, data tiny, no "log out everywhere" needed?   → signed_cookies
Single server, no database for sessions, quick experiments?      → file
```

> **Switching backends logs everyone out** — old sessions are in the old storage, which the new backend doesn't read. Plan it for a quiet moment.

---

## Step 22: The Database Backend in Detail (Default)

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.db'
```

### 22.1 What it does on each operation

| Operation                  | SQL (roughly)                                                                                  |
|----------------------------|------------------------------------------------------------------------------------------------|
| Load (first access)        | `SELECT … FROM django_session WHERE session_key = %s AND expire_date > now()`                  |
| Save new session           | `INSERT INTO django_session …` (inside a transaction; on key collision it retries with a new key) |
| Save existing session      | `UPDATE django_session SET session_data = …, expire_date = … WHERE session_key = %s`          |
| `flush()` / `delete()`     | `DELETE FROM django_session WHERE session_key = %s`                                            |
| `clearsessions`            | `DELETE FROM django_session WHERE expire_date < now()`                                         |

The **`expire_date > now()`** condition means an expired session is ignored immediately, even if its row still exists.

### 22.2 Using the `Session` model directly

```python
from django.contrib.sessions.models import Session
from django.utils import timezone

Session.objects.count()                                          # all rows (incl. expired)
Session.objects.filter(expire_date__gt=timezone.now()).count()   # active sessions

s = Session.objects.first()
s.session_key
s.expire_date
s.session_data                 # the signed, encoded string
s.get_decoded()                # {'visits': 3, '_auth_user_id': '1', ...}
```

`SessionManager` methods:

```python
Session.objects.encode({'a': 1})                       # → encoded string (what goes in session_data)
Session.objects.save('mykey123', {'a': 1}, expire_date)  # create/update a row; deletes it if the dict is empty
```

> To change a session's data, prefer `SessionStore(session_key=…)` over editing `Session` rows — the store handles encoding and caches (Step 27).

### 22.3 Multiple databases

To keep sessions in a separate database, route the `sessions` app with a database router:

```python
# config/routers.py
class SessionRouter:
    def db_for_read(self, model, **hints):
        if model._meta.app_label == 'sessions':
            return 'sessions'
        return None

    def db_for_write(self, model, **hints):
        if model._meta.app_label == 'sessions':
            return 'sessions'
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        if app_label == 'sessions':
            return db == 'sessions'
        return None
```

```python
# config/settings.py
DATABASES = {
    'default': {...},
    'sessions': {'ENGINE': 'django.db.backends.sqlite3', 'NAME': BASE_DIR / 'sessions.sqlite3'},
}
DATABASE_ROUTERS = ['config.routers.SessionRouter']
```

```bash
python manage.py migrate --database=sessions
```

---

## Step 23: The Cache Backend

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
```

Data lives **only** in the cache, under the key `django.contrib.sessions.cache<session_key>`, with a timeout equal to the session's expiry age. No database query at all.

### 23.1 You need a real, shared cache

The default cache (`LocMemCache`) is **per process**. With several server processes (Gunicorn workers, uWSGI) each one has its own memory, so a user's session exists in one worker and not in the next → random logouts. Also, every `runserver` restart (each code change!) wipes it. **Never use `LocMemCache` for sessions.**

Good choices:

| Cache                     | `BACKEND`                                               | Install                       |
|---------------------------|---------------------------------------------------------|-------------------------------|
| Redis (recommended)       | `django.core.cache.backends.redis.RedisCache`           | `pip install redis` + a Redis server |
| Memcached                 | `django.core.cache.backends.memcached.PyMemcacheCache`  | `pip install pymemcache` + Memcached |
| Database cache (for learning) | `django.core.cache.backends.db.DatabaseCache`       | `python manage.py createcachetable` |

### 23.2 Running Redis on Windows

Redis doesn't officially support Windows. Options:

- **Docker Desktop**: `docker run -d --name redis -p 6379:6379 redis:7`
- **WSL** (Ubuntu): `sudo apt install redis-server` then `sudo service redis-server start`
- **Memurai** — a Redis-compatible Windows service.

Check it's running:

```bash
docker exec -it redis redis-cli ping
```

```
PONG
```

### 23.3 Configure

```bash
pip install redis
```

```python
# config/settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/0',
    },
    'sessions': {                                     # separate cache just for sessions
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    },
}

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'sessions'
```

Why a separate cache? If you call `cache.clear()` on your page cache, or Redis evicts keys under memory pressure, you don't want to log everyone out along with it.

### 23.4 Check where the data went

```bash
python manage.py shell
```

```python
>>> from django.core.cache import caches
>>> from django.contrib.sessions.models import Session
>>> Session.objects.count()                     # nothing new in the database
0
>>> caches['sessions'].get('django.contrib.sessions.cache' + '<your sessionid cookie value>')
{'visits': 4, '_auth_user_id': '1', ...}
```

### 23.5 Trade-offs

- ✓ Very fast; expired sessions disappear automatically (cache timeout) — no `clearsessions` needed.
- ✗ A cache restart or eviction **logs users out**. Configure Redis persistence (AOF/RDB) and enough memory with an eviction policy like `volatile-lru` if you depend on it.
- ✗ You can't list/query sessions (no "show my devices" page).

---

## Step 24: The Cached Database Backend (Write-Through)

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
SESSION_CACHE_ALIAS = 'sessions'          # a real cache, as in Step 23
```

How it works:

```
  load():   cache hit?  ──yes──► use cached data                          (0 DB queries)
                │
                no
                ▼
            SELECT from django_session ──► put into cache ──► use it      (1 DB query)

  save():   UPDATE/INSERT django_session  ──►  update cache               (write-through)
  delete(): DELETE from django_session    ──►  delete from cache
```

- Cache key prefix: `django.contrib.sessions.cached_db`.
- Reads are as fast as the cache backend; data is as durable as the db backend.
- Writes still hit the database.
- Since **Django 5.1**, if the cache is down while saving, the error is **logged** to the `django.contrib.sessions` logger instead of crashing the request (the database copy is still saved).
- Needs the sessions app and `clearsessions` (for the database rows), like the db backend.

> ⚠️ If you **delete `Session` rows directly** (`Session.objects.filter(...).delete()`), the cached copies stay valid until the cache entry expires. Delete through the store instead: `SessionStore(session_key=key).delete()` (Step 30.6).

This is the best default for most production sites that already have Redis.

---

## Step 25: The File Backend

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.file'
SESSION_FILE_PATH = BASE_DIR / 'sessions'        # folder must exist and be writable
```

```bash
mkdir sessions
```

- One file per session, named `<SESSION_COOKIE_NAME><session_key>`, e.g. `sessions\sessionidq8m2z7d5k1v9w3e6r4t0y2u8i5o1p7a3`.
- The file contains the same signed, encoded data as the database column.
- Expiry: the file's modification time + age (or `_session_expiry` if set).
- If the folder doesn't exist you get `ImproperlyConfigured: The session storage path '...' doesn't exist. Please set your SESSION_FILE_PATH setting to an existing directory in which Django can store session data.`
- Run `clearsessions` to delete expired files.

Look at it:

```bash
dir sessions
type sessions\sessionid<your key>
```

When to use it: a single server where you don't want sessions in the database. With several servers, they'd all need the same shared folder — use db/cache instead.

> Add `sessions/` to `.gitignore`, and **never** set `SESSION_FILE_PATH` to a folder served as static/media files.

---

## Step 26: The Signed Cookie Backend

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.signed_cookies'
```

The **whole session data** is stored in the `sessionid` cookie, signed (not encrypted) with `SECRET_KEY`:

```
sessionid=eyJ2aXNpdHMiOjMsInRoZW1lIjoiZGFyayJ9:1tXk2a:Qm9vZ…
          └──── base64 JSON data ─────────────┘ └time┘ └signature┘
```

### 26.1 Prove the data is readable by the user

Copy the cookie value from DevTools, then:

```bash
python manage.py shell
```

```python
>>> import base64, json, zlib
>>> value = 'eyJ2aXNpdHMiOjMsInRoZW1lIjoiZGFyayJ9:1tXk2a:Qm9vZ...'
>>> payload = value.split(':')[0]
>>> compressed = payload.startswith('.')         # a leading '.' means zlib-compressed
>>> raw = base64.urlsafe_b64decode(payload.lstrip('.') + '=' * (-len(payload.lstrip('.')) % 4))
>>> json.loads(zlib.decompress(raw) if compressed else raw)
{'visits': 3, 'theme': 'dark'}
```

Anyone holding the cookie can **read** the data. They can't **change** it — any edit breaks the signature and Django ignores the cookie (starts a fresh session).

### 26.2 Limitations — read them all before choosing it

| Limitation                         | Consequence                                                                                       |
|------------------------------------|---------------------------------------------------------------------------------------------------|
| Data is **readable** by the user   | Never store secrets, internal flags, other users' data                                            |
| **~4 KB** cookie limit             | Browsers silently drop bigger cookies → the session "disappears". Keep data tiny                  |
| Sent with **every** request        | More upload bandwidth (images, CSS, JS requests too, unless on another domain)                    |
| **No server-side invalidation**    | `flush()`/`logout()` only remove the cookie from *this* browser. A **copy** of an old cookie (stolen, or saved before logout) stays valid until it expires — a **replay** risk |
| Can't list or delete sessions      | No "active devices", no "log out everywhere" except by changing `SECRET_KEY`                      |
| `SESSION_COOKIE_AGE` is enforced via the signature timestamp | Very long custom `set_expiry()` values don't extend the signature's max age     |
| Security depends on `SECRET_KEY`   | If it leaks, attackers can forge sessions for **any** user                                        |
| `SESSION_COOKIE_HTTPONLY` matters  | Keep it `True` so JavaScript (and XSS) can't read the data                                        |

When it's a good fit: very small data, no database available (e.g. a tiny service), and you accept the replay risk. For most sites, prefer `db` or `cached_db`.

---

## Step 27: Use Sessions Outside Views

### 27.1 With a specific backend

```python
>>> from django.contrib.sessions.backends.db import SessionStore
>>> s = SessionStore()
>>> s['last_login'] = 1761387164
>>> s.create()                          # or s.save()  — writes it and assigns a key
>>> s.session_key
'2b1189a188b44ad18c35e113ac6ceead'

>>> s2 = SessionStore(session_key='2b1189a188b44ad18c35e113ac6ceead')
>>> s2['last_login']
1761387164
>>> s2['extra'] = 'hi'
>>> s2.save()                           # outside a request there's no middleware — save yourself!
>>> s2.exists('2b1189a188b44ad18c35e113ac6ceead')
True
>>> s2.delete()
```

> Outside a request **nothing saves automatically**. Always call `.save()`.

### 27.2 With whatever backend the project is configured to use

Reusable code (management commands, Celery tasks, library code) shouldn't hard-code the backend:

```python
from importlib import import_module
from django.conf import settings

SessionStore = import_module(settings.SESSION_ENGINE).SessionStore

s = SessionStore(session_key=some_key)
s.get('cart')
```

### 27.3 Reading another user's session from a view

```python
def debug_other_session(request, key):
    other = SessionStore(session_key=key)
    data = dict(other.items())          # loads it
    # changing `other` does NOT touch request.session — they're separate objects
```

### 27.4 Encode and decode by hand

```python
>>> s = SessionStore()
>>> encoded = s.encode({'a': 1})
>>> encoded
'eyJhIjoxfQ:1tXk2a:bL3…'
>>> s.decode(encoded)
{'a': 1}
>>> s.decode('tampered' + encoded)      # bad signature → empty dict (and a warning is logged)
{}
```

Bad signatures are logged to the `django.security.SuspiciousSession` logger with the message `Session data corrupted`.

---

## Step 28: Inspect Sessions in the Shell and in the Admin

### 28.1 Shell recipes

```python
from django.contrib.auth import get_user_model
from django.contrib.sessions.models import Session
from django.utils import timezone

User = get_user_model()

# All active sessions with their user
for s in Session.objects.filter(expire_date__gt=timezone.now()):
    data = s.get_decoded()
    user_id = data.get('_auth_user_id')
    user = User.objects.filter(pk=user_id).first() if user_id else None
    print(s.session_key[:8], s.expire_date, user or 'anonymous', list(data.keys()))
```

```
q8m2z7d5 2026-11-08 10:12:44+00:00 admin ['_auth_user_id', '_auth_user_backend', '_auth_user_hash', 'visits']
n0cl6c1u 2026-11-08 10:20:31+00:00 anonymous ['visits', 'recently_viewed']
```

```python
# Log one user out of every device (db backend; see Step 30 for the efficient version)
for s in Session.objects.all():
    if s.get_decoded().get('_auth_user_id') == str(user.pk):
        s.delete()

# Log EVERYONE out
Session.objects.all().delete()
```

`get_decoded()` decodes in Python, so these loops read **every row** — fine for development, slow with millions of sessions.

### 28.2 Show sessions in the admin (read-only)

`accounts/admin.py` — add:

```python
from django.contrib.sessions.models import Session


@admin.register(Session)
class SessionAdmin(admin.ModelAdmin):
    list_display = ['session_key', 'user_display', 'expire_date', 'is_active']
    list_filter = ['expire_date']
    search_fields = ['session_key']
    date_hierarchy = 'expire_date'
    fields = ['session_key', 'expire_date', 'decoded_data']
    readonly_fields = fields

    @admin.display(description='User')
    def user_display(self, obj):
        user_id = obj.get_decoded().get('_auth_user_id')
        return user_id or 'anonymous'

    @admin.display(boolean=True, description='Active')
    def is_active(self, obj):
        return obj.expire_date > timezone.now()

    @admin.display(description='Data')
    def decoded_data(self, obj):
        return obj.get_decoded()

    def has_add_permission(self, request):
        return False

    def has_change_permission(self, request, obj=None):
        return False
```

(`from django.utils import timezone` at the top.) Deleting rows from this admin page logs those visitors out — handy for support.

> ⚠️ This page shows other people's session data. Only superusers should see it, and remember `_auth_user_hash` is in there. Don't keep it in production unless you need it.

---

## Step 29: Clear Expired Sessions

With the **db**, **cached_db** and **file** backends, expired sessions stay in storage until you delete them. The table grows forever otherwise.

### 29.1 The command

```bash
python manage.py clearsessions
```

- It calls `SessionStore.clear_expired()` of the configured backend.
- db / cached_db: `DELETE FROM django_session WHERE expire_date < now()`.
- file: deletes expired session files.
- cache: does nothing — the cache expires entries itself.
- signed_cookies: does nothing — nothing is stored on the server.
- A custom backend without `clear_expired()` → `Session engine '…' doesn't support clearing expired sessions.`

See it work:

```python
>>> from datetime import timedelta
>>> Session.objects.update(expire_date=timezone.now() - timedelta(days=1))   # expire everything (dev only!)
```

```bash
python manage.py clearsessions
python manage.py shell -c "from django.contrib.sessions.models import Session; print(Session.objects.count())"
```

```
0
```

### 29.2 Run it automatically

**Windows — Task Scheduler** (one line in PowerShell, adjust the paths):

```bash
schtasks /Create /SC DAILY /ST 03:00 /TN "Django clearsessions" /TR "\"C:\Users\<you>\myproject\venv\Scripts\python.exe\" \"C:\Users\<you>\myproject\manage.py\" clearsessions"
```

Check / run / remove it:

```bash
schtasks /Query /TN "Django clearsessions"
schtasks /Run /TN "Django clearsessions"
schtasks /Delete /TN "Django clearsessions" /F
```

**Linux — cron** (`crontab -e`):

```
0 3 * * * /home/app/myproject/venv/bin/python /home/app/myproject/manage.py clearsessions
```

**Hosting platforms** (Heroku Scheduler, Render cron jobs, Railway cron, Kubernetes CronJob…) — schedule the same command.

Daily is enough for most sites.

---

## Step 30: A Custom Session Model — "Active Devices" and "Log Out Other Devices"

The default table can't answer "which sessions belong to user 5?" without decoding every row. We'll build a session model with **extra columns** (`user`, `ip_address`, `user_agent`, `last_activity`) using `AbstractBaseSession`.

### 30.1 Create the app

```bash
python manage.py startapp usersessions
```

(Not `sessions` — that label is taken, Step 1.3.)

### 30.2 The model

`usersessions/models.py`:

```python
from django.conf import settings
from django.contrib.sessions.base_session import AbstractBaseSession
from django.db import models


class UserSession(AbstractBaseSession):
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        null=True, blank=True,
        on_delete=models.CASCADE,                 # deleting a user deletes their sessions
        related_name='sessions',
    )
    ip_address = models.GenericIPAddressField(null=True, blank=True)
    user_agent = models.CharField(max_length=300, blank=True)
    last_activity = models.DateTimeField(auto_now=True)

    class Meta(AbstractBaseSession.Meta):
        db_table = 'usersessions_session'

    @classmethod
    def get_session_store_class(cls):
        from .backends import SessionStore        # imported here to avoid a circular import
        return SessionStore
```

What we inherit: `session_key`, `session_data`, `expire_date`, `get_decoded()`, and the manager.

Why `last_activity` but no `created_at`? The store builds a **new model instance on every save** (`create_model_instance()`), so `auto_now_add=True` would be overwritten every time. `auto_now=True` ("last saved") is exactly what we want.

### 30.3 The session store (backend)

`usersessions/backends.py`:

```python
from django.contrib.sessions.backends.db import SessionStore as DBStore


class SessionStore(DBStore):
    @classmethod
    def get_model_class(cls):
        from .models import UserSession
        return UserSession

    def create_model_instance(self, data):
        obj = super().create_model_instance(data)      # sets session_key, session_data, expire_date
        try:
            obj.user_id = int(data.get('_auth_user_id'))
        except (TypeError, ValueError):
            obj.user_id = None                         # anonymous visitor
        obj.ip_address = data.get('ip_address')
        obj.user_agent = data.get('user_agent', '')[:300]
        return obj
```

`create_model_instance(data)` is called every time the session is saved; `data` is the session dict. We copy values **out of the session data into real columns**, which we can then filter on with normal queries.

> Using `cached_db` instead? Subclass `django.contrib.sessions.backends.cached_db.SessionStore` instead of the db one, and set a new `cache_key_prefix = 'usersessions.cached_db'` so old cache entries aren't confused with the new ones.

### 30.4 Put IP and device into the session at login

The store has no access to the request, so we record those values **in the session data** when a user logs in. `usersessions/signals.py`:

```python
from django.contrib.auth.signals import user_logged_in
from django.dispatch import receiver


@receiver(user_logged_in)
def record_session_device(sender, request, user, **kwargs):
    request.session['ip_address'] = request.META.get('REMOTE_ADDR')
    request.session['user_agent'] = request.META.get('HTTP_USER_AGENT', '')
```

> Behind a proxy or load balancer, `REMOTE_ADDR` is the proxy's IP. The real client IP is usually in `X-Forwarded-For` — only trust it if **your** proxy sets it.

`usersessions/apps.py`:

```python
from django.apps import AppConfig


class UsersessionsConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'usersessions'

    def ready(self):
        from . import signals  # noqa: F401  (connects the receivers)
```

### 30.5 Settings and migration

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    'django.contrib.sessions',     # may stay; its table is simply no longer used
    'usersessions',                # 👈 add
]

SESSION_ENGINE = 'usersessions.backends'      # the module that contains our SessionStore
```

```bash
python manage.py makemigrations usersessions
python manage.py migrate
```

> Switching the engine **logs everyone out** (old sessions are in `django_session`). Log in again.

Log in and check:

```python
>>> from usersessions.models import UserSession
>>> UserSession.objects.values('session_key', 'user__username', 'ip_address', 'user_agent', 'last_activity')
<QuerySet [{'session_key': 'v2…', 'user__username': 'admin', 'ip_address': '127.0.0.1', 'user_agent': 'Mozilla/5.0 …', 'last_activity': …}]>
>>> user.sessions.count()        # reverse relation from related_name='sessions'
1
```

### 30.6 "My active devices" page

`usersessions/views.py`:

```python
from django.contrib import messages
from django.contrib.auth.decorators import login_required
from django.shortcuts import redirect, render
from django.utils import timezone
from django.views.decorators.http import require_POST

from .backends import SessionStore
from .models import UserSession


def _active_sessions(user):
    return UserSession.objects.filter(user=user, expire_date__gt=timezone.now()).order_by('-last_activity')


@login_required
def session_list(request):
    return render(request, 'usersessions/session_list.html', {
        'sessions': _active_sessions(request.user),
        'current_key': request.session.session_key,
    })


@login_required
@require_POST
def session_end(request, session_key):
    session = _active_sessions(request.user).filter(session_key=session_key).first()  # only YOUR sessions
    if session and session_key != request.session.session_key:
        SessionStore(session_key=session_key).delete()      # through the store → also clears caches
        messages.success(request, 'That device has been logged out.')
    return redirect('usersessions:list')


@login_required
@require_POST
def session_end_others(request):
    others = _active_sessions(request.user).exclude(session_key=request.session.session_key)
    for key in others.values_list('session_key', flat=True):
        SessionStore(session_key=key).delete()
    messages.success(request, 'All other devices have been logged out.')
    return redirect('usersessions:list')
```

Security points:

- `session_end` filters by `user=request.user`, so nobody can end **someone else's** session by guessing a key.
- We delete through `SessionStore(...).delete()` instead of `queryset.delete()` so the code keeps working if you switch to a cached backend (Step 24 warning).
- Session keys appear in this page's HTML and URLs. They belong to the same user, but treat the page as sensitive (it's behind `login_required`, and keys are only shown to their owner). A stricter design shows a short id and looks the key up server-side.

`usersessions/urls.py`:

```python
from django.urls import path

from . import views

app_name = 'usersessions'

urlpatterns = [
    path('', views.session_list, name='list'),
    path('<str:session_key>/end/', views.session_end, name='end'),
    path('end-others/', views.session_end_others, name='end_others'),
]
```

`config/urls.py`:

```python
path('accounts/sessions/', include('usersessions.urls')),
```

`usersessions/templates/usersessions/session_list.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Active sessions{% endblock %}

{% block content %}
<h2>Where you're logged in</h2>

<table border="1" cellpadding="4">
    <tr><th>Device</th><th>IP</th><th>Last activity</th><th>Expires</th><th></th></tr>
    {% for s in sessions %}
        <tr>
            <td>{{ s.user_agent|truncatechars:60 }}</td>
            <td>{{ s.ip_address|default:"?" }}</td>
            <td>{{ s.last_activity|timesince }} ago</td>
            <td>{{ s.expire_date }}</td>
            <td>
                {% if s.session_key == current_key %}
                    <strong>This device</strong>
                {% else %}
                    <form method="post" action="{% url 'usersessions:end' s.session_key %}">
                        {% csrf_token %}
                        <button type="submit">Log out</button>
                    </form>
                {% endif %}
            </td>
        </tr>
    {% endfor %}
</table>

{% if sessions|length > 1 %}
    <form method="post" action="{% url 'usersessions:end_others' %}">
        {% csrf_token %}
        <button type="submit">Log out all other devices</button>
    </form>
{% endif %}
{% endblock %}
```

Add a link in the `base.html` nav for logged-in users:

```html
<a href="{% url 'usersessions:list' %}">Devices</a> |
```

Test: log in with two different browsers (e.g. Chrome and Edge), open the page in one, click **Log out** next to the other, and reload the other browser — it's logged out.

### 30.7 Admin for the custom model

`usersessions/admin.py`:

```python
from django.contrib import admin

from .models import UserSession


@admin.register(UserSession)
class UserSessionAdmin(admin.ModelAdmin):
    list_display = ['session_key', 'user', 'ip_address', 'last_activity', 'expire_date']
    list_filter = ['last_activity', 'expire_date']
    search_fields = ['session_key', 'user__username', 'ip_address']
    readonly_fields = ['session_key', 'user', 'ip_address', 'user_agent', 'last_activity', 'expire_date', 'decoded']
    exclude = ['session_data']

    @admin.display(description='Data')
    def decoded(self, obj):
        return obj.get_decoded()

    def has_add_permission(self, request):
        return False
```

Now remove the `SessionAdmin` from Step 28.2 (that table is no longer used). `clearsessions` keeps working because it uses `SESSION_ENGINE`'s store, whose `clear_expired()` uses `get_model_class()` → `UserSession`.

---

## Step 31: Write a Session Backend From Scratch (Understand the Interface)

You'll rarely need this, but it shows exactly what a backend must do. A backend is a module with a `SessionStore` class that subclasses `SessionBase` and implements six methods:

| Method                              | Must …                                                                                          |
|-------------------------------------|-------------------------------------------------------------------------------------------------|
| `exists(self, session_key)`         | Return `True` if a session with this key is stored                                              |
| `create(self)`                      | Get a new key (`self._get_new_session_key()`), save an empty/new session, set `self.modified = True`; on key collision (`CreateError`) try again |
| `save(self, must_create=False)`     | Store `self._get_session(no_load=must_create)` under `self.session_key`. If `must_create` and the key exists → raise `CreateError`. If updating a session that vanished → raise `UpdateError` |
| `delete(self, session_key=None)`    | Remove the given key (or `self.session_key` if `None`)                                          |
| `load(self)`                        | Return the session dict. If missing/expired/corrupt → set `self._session_key = None` and return `{}` |
| `clear_expired(cls)` (classmethod)  | Delete expired sessions (or do nothing if storage expires them itself)                          |

A tiny in-memory backend — **for learning only** (single process, lost on restart):

```python
# config/memory_sessions.py
from django.contrib.sessions.backends.base import CreateError, SessionBase, UpdateError
from django.utils import timezone

_STORE = {}   # {session_key: (encoded_data, expire_datetime)}


class SessionStore(SessionBase):
    def exists(self, session_key):
        return session_key in _STORE

    def create(self):
        while True:
            self._session_key = self._get_new_session_key()
            try:
                self.save(must_create=True)
            except CreateError:
                continue
            self.modified = True
            return

    def save(self, must_create=False):
        if self.session_key is None:
            return self.create()
        if must_create and self.exists(self.session_key):
            raise CreateError
        if not must_create and not self.exists(self.session_key):
            raise UpdateError
        data = self._get_session(no_load=must_create)
        _STORE[self.session_key] = (self.encode(data), self.get_expiry_date())

    def delete(self, session_key=None):
        key = session_key or self.session_key
        if key:
            _STORE.pop(key, None)

    def load(self):
        entry = _STORE.get(self.session_key)
        if entry is None or entry[1] <= timezone.now():
            self._session_key = None
            return {}
        return self.decode(entry[0])

    @classmethod
    def clear_expired(cls):
        now = timezone.now()
        for key in [k for k, (_, exp) in _STORE.items() if exp <= now]:
            _STORE.pop(key, None)
```

```python
SESSION_ENGINE = 'config.memory_sessions'
```

`SessionBase` gives you everything else for free: the dict API, `encode()`/`decode()` (signing + serializer), expiry methods, `cycle_key()`, `flush()`, the test cookie, and key generation. Switch back to the db backend when you're done experimenting.

---

## Step 32: Performance Tips

| Tip                                                                 | Why                                                                                     |
|---------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| Keep session data **small** (ids, flags, short strings)             | It's decoded on every request that touches it; signed cookies have a 4 KB limit         |
| Don't write on every request unless you need sliding expiry         | Each write is an `UPDATE`; `SESSION_SAVE_EVERY_REQUEST` doubles DB traffic              |
| Throttle "last seen" style writes (Step 16: once a minute)          | Same reason                                                                             |
| Don't touch the session in views that don't need it                 | It's lazy — untouched sessions cost 0 queries and keep responses cacheable              |
| Use `cached_db` (or `cache`) on busy sites                          | Removes the `SELECT` on every request                                                   |
| Run `clearsessions` regularly                                       | A huge `django_session` table slows inserts and backups                                 |
| Avoid `Session.objects.all()` + `get_decoded()` loops in production | They decode every row in Python — use a custom model with real columns (Step 30)        |
| Remember `Vary: Cookie`                                             | Any page that reads the session (incl. `{{ user }}` in templates) can't be shared-cached between users |

Count queries in development with the shell:

```python
>>> from django.db import connection, reset_queries
>>> from django.test import Client
>>> from django.test.utils import CaptureQueriesContext
>>> c = Client()
>>> with CaptureQueriesContext(connection) as ctx:
...     c.get('/session-demo/')
>>> [q['sql'][:60] for q in ctx.captured_queries]
```

(Or install **django-debug-toolbar**, which shows the session queries on every page.)

---

## Step 33: Sessions in Async Views (Django 5.0+)

In an `async def` view, the database backends can't run their synchronous queries directly. Use the **`a`-prefixed** methods:

```python
from django.http import JsonResponse


@login_not_required
async def async_counter(request):
    visits = await request.session.aget('async_visits', 0)
    await request.session.aset('async_visits', visits + 1)
    return JsonResponse({'visits': visits + 1})
```

| Sync                                  | Async                                      |
|---------------------------------------|--------------------------------------------|
| `request.session['k']` / `.get('k')`  | `await request.session.aget('k', default)` |
| `request.session['k'] = v`            | `await request.session.aset('k', v)`       |
| `.setdefault('k', v)`                 | `await request.session.asetdefault('k', v)`|
| `.pop('k', default)`                  | `await request.session.apop('k', default)` |
| `'k' in request.session`              | `await request.session.ahas_key('k')`      |
| `.keys()` / `.values()` / `.items()`  | `await .akeys()` / `.avalues()` / `.aitems()` |
| `.flush()` / `.cycle_key()`           | `await .aflush()` / `await .acycle_key()`  |
| `.set_expiry(v)` / `.get_expiry_age()`| `await .aset_expiry(v)` / `await .aget_expiry_age()` |
| `.save()` / `.delete()` / `.exists(k)`| `await .asave()` / `await .adelete()` / `await .aexists(k)` |
| test cookie methods                   | `aset_test_cookie()`, `atest_cookie_worked()`, `adelete_test_cookie()` |

- If you use the **sync** API in an async view before the session is loaded, you get `SynchronousOnlyOperation: You cannot call this from an async context - use a thread or sync_to_async.`
- For the user, use `user = await request.auser()` instead of `request.user`.
- Async login/logout: `await alogin(request, user)`, `await alogout(request)` (from `django.contrib.auth`).
- The middleware itself still saves the session at the end of the request, as usual.

---

## Step 34: Security — Everything You Need to Know

### 34.1 Threats and defences

| Threat                                | What it is                                                         | Defence                                                                                         |
|---------------------------------------|--------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| **Session hijacking (sniffing)**      | Someone on the same Wi-Fi reads the cookie from plain HTTP traffic | HTTPS everywhere + `SESSION_COOKIE_SECURE = True` + HSTS                                        |
| **Session hijacking (XSS)**           | Injected JavaScript reads `document.cookie`                        | `SESSION_COOKIE_HTTPONLY = True` (default) + auto-escaping templates                            |
| **Session fixation**                  | Attacker plants a known key before the victim logs in              | `login()` rotates the key; call `cycle_key()` yourself on privilege changes                     |
| **CSRF**                              | Another site makes the victim's browser submit a form to yours     | `CsrfViewMiddleware` + `{% csrf_token %}` + `SameSite=Lax`                                      |
| **Tampering with session data**       | User edits the stored data                                         | Data is signed with `SECRET_KEY`; db/cache/file data never leaves the server                    |
| **Key guessing**                      | Brute-forcing a valid key                                          | 32 random chars ≈ 165 bits — not feasible                                                       |
| **Stolen old cookie (replay)**        | Reusing a copied cookie after logout                               | Server-side backends delete the session on logout; avoid `signed_cookies` if this matters       |
| **Leaked `SECRET_KEY`**               | Attacker can forge signed data                                     | Keep it secret (env var), rotate with `SECRET_KEY_FALLBACKS`, never use pickle                   |
| **Long-lived sessions on shared PCs** | Next person at the computer is still logged in                     | Shorter `SESSION_COOKIE_AGE`, "remember me" opt-in, idle timeout                                |
| **Subdomain cookie injection**        | A compromised `evil.example.com` sets cookies for `.example.com`   | Leave `SESSION_COOKIE_DOMAIN = None` unless you really need sharing                             |

### 34.2 Don't put secrets in the session

Session data is **signed, not encrypted**. With server-side backends only people with database/cache/disk access can read it — but they can, trivially (Step 26.1). Never store passwords, full card numbers, API secrets or one-time codes in plain form.

### 34.3 Validate what you read back

Session data written by **your** code (server-side backends) can be trusted to be what you stored. But validate data **before** you store it (Step 15 checks `theme in THEMES`), and for the signed-cookie backend remember that the user can at least **read** everything.

### 34.4 `SameSite` options

| Value      | Cookie sent on…                                                                                 | Use when                                                     |
|------------|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| `'Lax'` (default) | same-site requests + top-level **GET** navigations from other sites (clicking a link)     | Almost always                                                |
| `'Strict'` | same-site requests only. Arriving from a link on another site → you appear logged out on that first page | High-security apps (banking)                      |
| `'None'`   | all requests, including cross-site iframes and `fetch()` — **requires `Secure`**                | Your site is embedded in another site's iframe, or a cross-site SPA uses cookie auth |
| `False`    | no `SameSite` attribute — the browser's default applies (modern browsers treat it as `Lax`)     | Rarely                                                        |

### 34.5 Production settings

```python
# config/settings.py (production)
SESSION_COOKIE_SECURE = True           # HTTPS only
SESSION_COOKIE_HTTPONLY = True         # default, keep it
SESSION_COOKIE_SAMESITE = 'Lax'        # default, or 'Strict'
SESSION_COOKIE_AGE = 60 * 60 * 24 * 7  # pick an age that fits your users
CSRF_COOKIE_SECURE = True
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000         # start small (e.g. 3600) and increase once HTTPS is confirmed
```

Check them:

```bash
python manage.py check --deploy
```

Session-related warnings you may see:

| ID              | Message (shortened)                                                                       | Fix                               |
|-----------------|-------------------------------------------------------------------------------------------|-----------------------------------|
| `security.W010` | `django.contrib.sessions` is in INSTALLED_APPS but `SESSION_COOKIE_SECURE` is not True    | `SESSION_COOKIE_SECURE = True`    |
| `security.W011` | `SessionMiddleware` is in MIDDLEWARE but `SESSION_COOKIE_SECURE` is not True              | same                              |
| `security.W012` | `SESSION_COOKIE_SECURE` is not set to True                                                | same                              |
| `security.W013` | sessions app installed but `SESSION_COOKIE_HTTPONLY` is not True                          | `SESSION_COOKIE_HTTPONLY = True`  |
| `security.W014` | `SessionMiddleware` installed but `SESSION_COOKIE_HTTPONLY` is not True                   | same                              |
| `security.W015` | `SESSION_COOKIE_HTTPONLY` is not set to True                                              | same                              |

> With `SESSION_COOKIE_SECURE = True` you **cannot log in over plain `http://127.0.0.1:8000`** — the browser won't send the cookie back. Only enable it in production settings (e.g. `SESSION_COOKIE_SECURE = not DEBUG`).

### 34.6 Rotating `SECRET_KEY` without logging everyone out

```python
SECRET_KEY = os.environ['DJANGO_SECRET_KEY']                     # the NEW key — signs new data
SECRET_KEY_FALLBACKS = [os.environ['DJANGO_OLD_SECRET_KEY']]     # old key — still accepted for verifying
```

Sessions signed with the old key keep working; they get re-signed with the new key the next time they're saved. After `SESSION_COOKIE_AGE` has passed, remove the old key from `SECRET_KEY_FALLBACKS`.

If the key **leaked**, don't use fallbacks — change it outright so all old sessions die immediately.

---

## Step 35: Test Sessions

### 35.1 With the test client

The test client keeps cookies between requests, just like a browser. `blog/tests.py`:

```python
from django.contrib.auth import get_user_model
from django.test import TestCase
from django.urls import reverse

from .models import Post

User = get_user_model()


class SessionFeatureTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user('writer', password='pass12345!')
        cls.post = Post.objects.create(title='Hello', slug='hello', content='...',
                                       author=cls.author, status='published')

    def test_visit_counter_increases(self):
        url = reverse('blog:session_demo')
        self.client.get(url)
        self.client.get(url)
        self.assertEqual(self.client.session['visits'], 2)

    def test_recently_viewed_is_recorded(self):
        self.client.get(reverse('blog:post_detail', args=[self.post.pk]))
        self.assertEqual(self.client.session['recently_viewed'], [self.post.pk])

    def test_add_to_reading_list(self):
        self.client.post(reverse('blog:reading_list_add', args=[self.post.pk]))
        self.assertIn(self.post.pk, self.client.session['reading_list'])

    def test_add_requires_post(self):
        response = self.client.get(reverse('blog:reading_list_add', args=[self.post.pk]))
        self.assertEqual(response.status_code, 405)
        self.assertNotIn('reading_list', self.client.session)

    def test_reading_list_survives_login(self):
        self.client.post(reverse('blog:reading_list_add', args=[self.post.pk]))
        key_before = self.client.session.session_key
        self.client.login(username='writer', password='pass12345!')
        self.assertNotEqual(self.client.session.session_key, key_before)   # key rotated
        self.assertEqual(self.client.session['reading_list'], [self.post.pk])  # data kept

    def test_logout_flushes_session(self):
        self.client.force_login(self.author)
        self.client.post(reverse('blog:reading_list_add', args=[self.post.pk]))
        self.client.post(reverse('logout'))
        self.assertNotIn('reading_list', self.client.session)
```

> `test_visit_counter_increases` requires `DEBUG = True`, but tests run with `DEBUG = False`. Decorate it with `@override_settings(DEBUG=True)` (`from django.test import override_settings`).

### 35.2 Put data into the session before a request

`self.client.session` returns a **new** `SessionStore` object each time you access it. Store it in a variable, change it, and **save** it:

```python
def test_reading_list_page_shows_saved_posts(self):
    session = self.client.session            # ✓ keep a reference
    session['reading_list'] = [self.post.pk]
    session.save()                           # ✓ required

    response = self.client.get(reverse('blog:reading_list'))
    self.assertContains(response, 'Hello')
```

The **wrong** way (silently does nothing):

```python
self.client.session['reading_list'] = [self.post.pk]   # ✗ modifies a temporary object that is never saved
```

With `AsyncClient` use `session = await self.async_client.asession()` (Django 5.0+).

### 35.3 With `RequestFactory` (no middleware)

`RequestFactory` doesn't run middleware, so there's no `request.session`. Add one:

```python
from django.contrib.sessions.middleware import SessionMiddleware
from django.test import RequestFactory, TestCase

from blog import reading_list as rl


class ReadingListUnitTests(TestCase):
    def make_request(self):
        request = RequestFactory().get('/')
        SessionMiddleware(lambda r: None).process_request(request)   # attaches request.session
        return request

    def test_add_is_idempotent(self):
        request = self.make_request()
        rl.add(request.session, 1)
        rl.add(request.session, 1)
        self.assertEqual(rl.get_ids(request.session), [1])
        self.assertTrue(request.session.modified)
```

(Or simply `request.session = SessionStore()` with an imported store.)

### 35.4 Test expiry and the custom model

```python
from usersessions.models import UserSession


class UserSessionTests(TestCase):
    def test_login_records_device(self):
        user = User.objects.create_user('u', password='pass12345!')
        self.client.post(reverse('login'), {'username': 'u', 'password': 'pass12345!'},
                         HTTP_USER_AGENT='TestBrowser/1.0')
        s = UserSession.objects.get(user=user)
        self.assertEqual(s.user_agent, 'TestBrowser/1.0')
        self.assertEqual(s.ip_address, '127.0.0.1')

    def test_set_expiry_zero_means_browser_close(self):
        session = self.client.session
        session.set_expiry(0)
        self.assertTrue(session.get_expire_at_browser_close())
```

Run:

```bash
python manage.py test blog usersessions
```

---

## Step 36: Production Checklist

- [ ] `SessionMiddleware` in `MIDDLEWARE`, before `AuthenticationMiddleware` and `MessageMiddleware`.
- [ ] A **real backend**: `db` (default) or `cached_db` with Redis. Never `cache` + `LocMemCache`.
- [ ] `SESSION_COOKIE_SECURE = True` and HTTPS (with `SECURE_SSL_REDIRECT` and HSTS).
- [ ] `SESSION_COOKIE_HTTPONLY = True` (default).
- [ ] `SESSION_COOKIE_SAMESITE = 'Lax'` or `'Strict'`.
- [ ] A sensible `SESSION_COOKIE_AGE` (2 weeks default; shorter for sensitive apps).
- [ ] `SECRET_KEY` from an environment variable, never in Git; rotation plan with `SECRET_KEY_FALLBACKS`.
- [ ] `clearsessions` scheduled daily (db / cached_db / file backends).
- [ ] No secrets or large objects in session data; only JSON-safe values.
- [ ] `cycle_key()` after any privilege change you implement yourself.
- [ ] Unique `SESSION_COOKIE_NAME` if several Django apps share a domain.
- [ ] `python manage.py check --deploy` shows no session warnings.
- [ ] Debug-only views (like `/session-demo/`) disabled or removed.

---

## Reference: `request.session` API

| Method / attribute                         | Async version (5.0+)              | Description                                                      |
|--------------------------------------------|-----------------------------------|------------------------------------------------------------------|
| `session[key]`                             | —                                 | Get value; `KeyError` if missing                                 |
| `session[key] = value`                     | `aset(key, value)`                | Set value (marks modified)                                       |
| `del session[key]`                         | —                                 | Delete; `KeyError` if missing                                    |
| `key in session`                           | `ahas_key(key)`                   | Membership test                                                  |
| `get(key, default=None)`                   | `aget(key, default=None)`         | Get with default                                                 |
| `pop(key, default)`                        | `apop(key, default)`              | Remove and return                                                |
| `setdefault(key, value)`                   | `asetdefault(key, value)`         | Set if missing, return value                                     |
| `update(dict)`                             | `aupdate(dict)`                   | Set several keys                                                 |
| `keys()`, `values()`, `items()`            | `akeys()`, `avalues()`, `aitems()`| Like dict                                                        |
| `clear()`                                  | —                                 | Remove all data, keep key                                        |
| `flush()`                                  | `aflush()`                        | Delete data **and** the stored session; new key next save        |
| `cycle_key()`                              | `acycle_key()`                    | New key, keep data                                               |
| `set_expiry(value)`                        | `aset_expiry(value)`              | `int` seconds, `0`, `datetime`, `timedelta`, `None`              |
| `get_expiry_age(**kw)`                     | `aget_expiry_age(**kw)`           | Seconds until expiry                                             |
| `get_expiry_date(**kw)`                    | `aget_expiry_date(**kw)`          | `datetime` of expiry                                             |
| `get_expire_at_browser_close()`            | `aget_expire_at_browser_close()`  | `True` / `False`                                                 |
| `get_session_cookie_age()`                 | —                                 | `SESSION_COOKIE_AGE` (overridable)                               |
| `set_test_cookie()`                        | `aset_test_cookie()`              | Store the test value                                             |
| `test_cookie_worked()`                     | `atest_cookie_worked()`           | Did it come back?                                                |
| `delete_test_cookie()`                     | `adelete_test_cookie()`           | Remove the test value                                            |
| `clear_expired()` (classmethod)            | `aclear_expired()`                | Delete expired sessions from storage                             |
| `save(must_create=False)`                  | `asave(must_create=False)`        | Write to storage                                                 |
| `create()`                                 | `acreate()`                       | New key + write                                                  |
| `delete(session_key=None)`                 | `adelete(session_key=None)`       | Remove from storage                                              |
| `exists(session_key)`                      | `aexists(session_key)`            | Key stored?                                                      |
| `load()`                                   | `aload()`                         | Read data from storage                                           |
| `encode(dict)` / `decode(str)`             | —                                 | Sign+serialize / verify+deserialize                              |
| `is_empty()`                               | —                                 | No key and no data                                               |
| `session_key`                              | —                                 | The key (`None` until saved)                                     |
| `modified`, `accessed`                     | —                                 | Flags used by the middleware                                     |

## Reference: Backends

| `SESSION_ENGINE`                                   | Store class module                                    | Cache key prefix                       | `clearsessions` |
|----------------------------------------------------|-------------------------------------------------------|----------------------------------------|-----------------|
| `django.contrib.sessions.backends.db`              | `db.SessionStore`                                     | —                                      | ✓               |
| `django.contrib.sessions.backends.cache`           | `cache.SessionStore`                                  | `django.contrib.sessions.cache`        | not needed      |
| `django.contrib.sessions.backends.cached_db`       | `cached_db.SessionStore`                              | `django.contrib.sessions.cached_db`    | ✓               |
| `django.contrib.sessions.backends.file`            | `file.SessionStore`                                   | —                                      | ✓               |
| `django.contrib.sessions.backends.signed_cookies`  | `signed_cookies.SessionStore`                         | —                                      | not possible    |

## Reference: Exceptions

| Exception                                                | Raised when                                                                    |
|----------------------------------------------------------|--------------------------------------------------------------------------------|
| `django.contrib.sessions.backends.base.CreateError`      | Saving a new session whose key already exists (backend retries with a new key) |
| `django.contrib.sessions.backends.base.UpdateError`      | Updating a session that no longer exists in storage                            |
| `django.contrib.sessions.exceptions.SessionInterrupted`  | Middleware got `UpdateError` (concurrent logout) → **400 Bad Request**         |
| `django.contrib.sessions.exceptions.SuspiciousSession`   | Suspicious session data (logged to `django.security.SuspiciousSession`)        |
| `django.contrib.sessions.exceptions.InvalidSessionKey`   | Invalid session key characters (e.g. file backend)                             |

## Reference: Reserved Session Keys

| Key                       | Set by                                    |
|---------------------------|-------------------------------------------|
| `_auth_user_id`           | `django.contrib.auth.login()`             |
| `_auth_user_backend`      | `django.contrib.auth.login()`             |
| `_auth_user_hash`         | `django.contrib.auth.login()`             |
| `_session_expiry`         | `set_expiry()`                            |
| `_messages`               | messages `SessionStorage`                 |
| `_password_reset_token`   | `PasswordResetConfirmView`                |
| `_csrftoken`              | CSRF, only when `CSRF_USE_SESSIONS = True`|
| `testcookie`              | `set_test_cookie()`                       |

## Reference: Management Commands

| Command                                | What it does                                     |
|----------------------------------------|--------------------------------------------------|
| `python manage.py clearsessions`       | Delete expired sessions (db, cached_db, file)    |
| `python manage.py migrate sessions`    | Create the `django_session` table                |
| `python manage.py sqlmigrate sessions 0001` | Show the table's SQL                        |
| `python manage.py createcachetable`    | Create the table for `DatabaseCache` (Step 23)   |
| `python manage.py check --deploy`      | Includes session cookie security checks          |

---

## Troubleshooting

| Problem                                                                                  | Cause / fix                                                                                                     |
|------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| `AttributeError: 'WSGIRequest' object has no attribute 'session'`                        | `SessionMiddleware` missing from `MIDDLEWARE`, or you're using `RequestFactory` (Step 35.3)                      |
| `ImproperlyConfigured: The Django authentication middleware requires session middleware…`| Put `SessionMiddleware` **above** `AuthenticationMiddleware`                                                    |
| `no such table: django_session`                                                          | Run `python manage.py migrate`                                                                                  |
| `RuntimeError: Model class django.contrib.sessions.models.Session doesn't declare an explicit app_label…` | Add `'django.contrib.sessions'` to `INSTALLED_APPS` (db / cached_db backends)                 |
| `Application labels aren't unique, duplicates: sessions`                                 | You created an app named `sessions`. Rename it (Step 1.3)                                                       |
| `TypeError: Object of type datetime/Decimal/… is not JSON serializable`                  | You stored a non-JSON value; store strings/ints instead (Step 8)                                                |
| Data I changed in a list/dict inside the session is lost                                 | Nested mutation; set `request.session.modified = True` or assign back (Step 7.2)                               |
| Dict keys changed from `5` to `'5'`                                                      | JSON only has string keys; use string keys (Step 8.2)                                                           |
| `session_key` is `None`                                                                  | New session not saved yet; call `request.session.save()` first (Step 5.4)                                       |
| Logged out randomly / sessions vanish                                                    | `LocMemCache` with several processes, cache eviction, `cache.clear()`, or two projects sharing the `sessionid` cookie on different ports (Step 4.4) |
| Can't stay logged in locally after a settings change                                     | `SESSION_COOKIE_SECURE = True` over `http://` — only enable in production (Step 34.5)                          |
| Everyone was logged out after deploy                                                     | `SECRET_KEY`, `SESSION_ENGINE`, serializer or cookie name/domain changed                                        |
| Login works on `localhost` but not on `127.0.0.1` (or vice versa)                        | Different hosts = different cookies; use one consistently                                                       |
| Login works in one subdomain only                                                        | Set `SESSION_COOKIE_DOMAIN = '.example.com'` (and the same for `CSRF_COOKIE_DOMAIN`)                            |
| `400 Bad Request` with `SessionInterrupted` in the logs                                  | The session was deleted by a concurrent request (e.g. logout in another tab). Usually harmless                  |
| Session data "disappears" with `signed_cookies`                                          | Cookie over ~4 KB — browsers drop it. Store less, or use a server-side backend                                  |
| `self.client.session['x'] = 1` in tests has no effect                                    | Store `session = self.client.session`, modify, then `session.save()` (Step 35.2)                                |
| `SynchronousOnlyOperation` in an async view                                              | Use `await request.session.aget()` / `aset()` (Step 33)                                                         |
| `django_session` table is huge                                                           | Schedule `clearsessions` (Step 29)                                                                              |
| Template error accessing `request.session._auth_user_id`                                 | Templates can't access names starting with `_`; use `user` instead                                              |

---

## Final Project Structure

```
myproject/
├── manage.py
├── db.sqlite3                     # contains django_session (and usersessions_session after Step 30)
├── config/
│   ├── settings.py                # SESSION_* settings, MIDDLEWARE order, 'usersessions' + context processor
│   ├── urls.py                    # includes usersessions.urls at accounts/sessions/
│   ├── routers.py                 # (optional) SessionRouter — Step 22.3
│   ├── session_serializers.py     # (optional) custom serializer — Step 8.4
│   └── memory_sessions.py         # (learning only) backend from scratch — Step 31
├── accounts/
│   ├── admin.py                   # (optional) read-only SessionAdmin — Step 28.2
│   ├── middleware.py              # IdleTimeoutMiddleware — Step 16
│   └── views.py                   # RememberMeLoginView, cookie_checked_login, logout_keep_reading_list
├── blog/
│   ├── context_processors.py      # session_extras: reading_list_count, theme
│   ├── forms.py                   # FeedbackContactForm, FeedbackMessageForm
│   ├── reading_list.py            # get_ids / add / remove / clear / count
│   ├── tests.py                   # SessionFeatureTests, ReadingListUnitTests
│   ├── urls.py                    # session_demo, session_action, reading_list*, set_theme, feedback
│   ├── views.py                   # session_demo, post_detail (recently viewed), reading list, theme, wizard
│   └── templates/blog/
│       ├── base.html              # theme class, reading list link, theme toggle, Devices link
│       ├── session_demo.html
│       ├── reading_list.html
│       ├── feedback_step.html
│       └── feedback_confirm.html
├── social/                        # from the contenttypes guide
├── usersessions/
│   ├── admin.py                   # UserSessionAdmin
│   ├── apps.py                    # ready() imports signals
│   ├── backends.py                # SessionStore(DBStore) — SESSION_ENGINE = 'usersessions.backends'
│   ├── migrations/
│   ├── models.py                  # UserSession(AbstractBaseSession): user, ip_address, user_agent, last_activity
│   ├── signals.py                 # user_logged_in → store IP and user agent in the session
│   ├── tests.py
│   ├── urls.py                    # list, end, end_others
│   ├── views.py                   # active devices, log out one / all others
│   └── templates/usersessions/
│       └── session_list.html
└── sessions/                      # (only if you tried the file backend — add to .gitignore)
```

---

## Quick Reference

```python
# settings.py
INSTALLED_APPS = [..., 'django.contrib.sessions', ...]
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    ...,
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    ...,
]
SESSION_ENGINE = 'django.contrib.sessions.backends.db'     # or cached_db / cache / file / signed_cookies
SESSION_COOKIE_AGE = 1209600
SESSION_COOKIE_SECURE = True                               # production
SESSION_EXPIRE_AT_BROWSER_CLOSE = False
SESSION_SAVE_EVERY_REQUEST = False

# views
request.session['key'] = 'value'                 # write (JSON-safe values only)
request.session.get('key', 'default')            # read
request.session.pop('key', None)                 # remove
'key' in request.session                         # check
request.session['cart'] = cart                   # assign back after changing nested data
request.session.modified = True                  # ...or force a save
request.session.set_expiry(0)                    # until browser closes (or seconds / datetime / timedelta / None)
request.session.cycle_key()                      # new key, same data (after privilege changes)
request.session.flush()                          # destroy completely (logout does this)
request.session.session_key                      # None until saved → request.session.save()

# async views (Django 5.0+)
value = await request.session.aget('key')
await request.session.aset('key', value)

# outside views
from importlib import import_module
from django.conf import settings
SessionStore = import_module(settings.SESSION_ENGINE).SessionStore
s = SessionStore(session_key=key); s['x'] = 1; s.save()

# model
from django.contrib.sessions.models import Session
Session.objects.filter(expire_date__gt=timezone.now())
session_obj.get_decoded()

# tests
session = self.client.session; session['x'] = 1; session.save()
```

```bash
python manage.py migrate
python manage.py clearsessions
python manage.py check --deploy
curl.exe -i -c cookies.txt -b cookies.txt http://127.0.0.1:8000/session-demo/
```

---

## Next Steps

- **django-redis** — a feature-rich Redis cache backend (alternative to Django's built-in `RedisCache`).
- **django-user-sessions** — a ready-made version of Step 30 (sessions per user, devices list, admin).
- **django-formtools** — `SessionWizardView` for multi-step forms (Step 17).
- **django-debug-toolbar** — see the session queries and session data on every page.
- **Django REST Framework** `SessionAuthentication` — use the same session login for your API (CSRF required for unsafe methods).
- Official docs:
  - How to use sessions: https://docs.djangoproject.com/en/stable/topics/http/sessions/
  - Session settings: https://docs.djangoproject.com/en/stable/ref/settings/#sessions
  - Security in Django: https://docs.djangoproject.com/en/stable/topics/security/
  - Cryptographic signing: https://docs.djangoproject.com/en/stable/topics/signing/
