# Celery With Django: Step-by-Step Guide

**Celery** is a **distributed task queue** for Python. It lets your Django app hand slow or scheduled work to separate **worker** processes, so the web request can return right away.

Typical jobs you move to Celery:

| Job                                        | Why not do it inside the view?                                  |
|--------------------------------------------|-----------------------------------------------------------------|
| Sending emails (welcome, password reset…)  | SMTP can take seconds or fail — the user shouldn't wait         |
| Calling external APIs (payments, webhooks) | Slow, unreliable, need retries                                  |
| Image / video / PDF processing             | CPU-heavy, can take minutes                                     |
| Generating reports / CSV exports           | Long-running, the browser would time out                        |
| Cleanup jobs (delete old drafts, sessions) | Must run **on a schedule**, not when a user visits              |
| Sending notifications to many users        | Thousands of operations — split them up and run in parallel     |

What Celery gives you:

| Part                   | What it does                                                                 |
|------------------------|------------------------------------------------------------------------------|
| **Tasks**              | Normal Python functions marked with `@shared_task`                           |
| **Workers**            | Processes that pick tasks from a queue and run them                          |
| **Broker support**     | Uses Redis or RabbitMQ to pass messages between Django and workers           |
| **Results**            | Stores return values and states (PENDING, SUCCESS, FAILURE…)                 |
| **Retries**            | Automatic or manual retries with back-off                                    |
| **Time & rate limits** | Kill tasks that run too long; limit how often a task runs                    |
| **Canvas**             | Combine tasks: chains, groups, chords                                        |
| **Routing**            | Send different tasks to different queues / workers                          |
| **Beat**               | A scheduler for periodic tasks (cron-like)                                   |
| **Monitoring**         | `inspect` / `control` commands, events, and the Flower web dashboard         |

This guide continues the `blog` project from [django-setup-guide.md](django-setup-guide.md), [django-admin-guide.md](django-admin-guide.md) and [django-auth-guide.md](django-auth-guide.md). Every step builds on the previous one. Commands are for **Windows** (PowerShell); they are the same on macOS/Linux except where noted.

---

## Step 0: How Celery Works (Big Picture)

Before writing code, understand the moving parts:

```
 ┌──────────────┐  1. task.delay(...)   ┌──────────────┐  2. worker pulls   ┌──────────────┐
 │  Django      │ ───────────────────▶  │   BROKER     │ ─────────────────▶ │   WORKER     │
 │  (producer)  │   message: task name  │ (Redis /     │   the message      │ (celery      │
 │  view, admin │   + args + task id    │  RabbitMQ)   │                    │  worker)     │
 └──────┬───────┘                       └──────────────┘                    └──────┬───────┘
        │                                                                          │
        │ 4. AsyncResult(id).state / .get()      ┌──────────────────┐  3. stores    │
        └──────────────────────────────────────▶ │  RESULT BACKEND  │ ◀─────────────┘
                                                 │ (Redis / Django  │  state + return value
                                                 │  DB / none)      │
                                                 └──────────────────┘

 ┌──────────────┐
 │  BEAT        │  (optional) every N seconds / on a crontab, puts a task message on the broker
 │  (scheduler) │ ─────────────────────────────────────────────▶ BROKER
 └──────────────┘
```

**Words you will see a lot:**

| Term               | Meaning                                                                                     |
|--------------------|---------------------------------------------------------------------------------------------|
| **Task**           | A Python function registered with Celery                                                     |
| **Producer**       | Code that *sends* a task (your Django views, signals, shell)                                |
| **Message**        | What is actually sent: task name + arguments + id + options (serialized as JSON)            |
| **Broker**         | The message queue server that holds messages until a worker takes them (Redis, RabbitMQ)    |
| **Queue**          | A named line of messages inside the broker (default name: `celery`)                         |
| **Worker**         | A long-running process (`celery worker`) that executes tasks                                |
| **Pool**           | How a worker runs tasks in parallel: processes (prefork), threads, gevent, or solo           |
| **Concurrency**    | How many tasks one worker runs at the same time                                              |
| **Result backend** | Where task state and return values are stored (optional)                                    |
| **Task id**        | A UUID that identifies one run of a task                                                    |
| **Beat**           | The scheduler process that sends periodic tasks                                             |
| **Ack** (acknowledge) | Worker telling the broker "I have this message, delete it from the queue"               |

**What happens when you call `send_welcome_email.delay(user.id)`:**

1. Celery creates a **task id** (UUID).
2. It **serializes** the task name (`accounts.tasks.send_welcome_email`) and arguments (`[5]`) to JSON.
3. It **publishes** the message to the broker queue `celery`.
4. `.delay()` returns an `AsyncResult` **immediately** — your view continues and returns the response.
5. A worker that listens to the `celery` queue **receives** the message.
6. The worker looks up the task by name in its **registry**, deserializes the arguments, and **runs the function**.
7. The worker **acks** the message (by default *before* running — see Step 18).
8. If a result backend is configured, the worker **stores** the state (`SUCCESS` / `FAILURE`) and return value.
9. Any code holding the task id can now ask for the state/result.

**Important consequences** (keep them in mind for every step):

- The worker is a **separate process**. It does **not** share memory with `runserver`. It imports your Django project on its own.
- When you change task code you must **restart the worker** (it has no auto-reload by default — Step 8.2 shows a workaround).
- Arguments travel as **JSON**, so pass **ids and plain data**, never model instances, request objects or open files.
- If no worker is running, tasks just **wait in the broker** — nothing is lost (with Redis/RabbitMQ), nothing runs.

---

## Step 1: Prerequisites

You need:

- The `blog` project from the previous guides, with the virtual environment **activated**:
  ```powershell
  cd myproject
  venv\Scripts\Activate.ps1
  ```
- **Python 3.9+** (check the Celery release notes for the exact supported versions of the Celery version you install).
- **Django 4.2+**.
- A **broker** — this guide uses **Redis** (Step 2).

Current project layout (simplified):

```
myproject/
├── manage.py
├── db.sqlite3
├── config/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
├── accounts/        # custom user, sign-up (from the auth guide)
└── blog/            # Post model, views, templates
```

---

## Step 2: Install and Run a Broker (Redis)

Celery does not store messages itself — it needs a broker. Choices:

| Broker       | Pros                                                         | Cons                                                    |
|--------------|--------------------------------------------------------------|---------------------------------------------------------|
| **Redis**    | Simple, fast, also works as result backend and cache        | Messages live in memory (enable persistence for safety) |
| **RabbitMQ** | Most complete & reliable AMQP broker, best for big systems  | One more service to learn and run                       |
| Amazon SQS   | Managed, no server to run                                   | No result backend, some features missing               |

We use **Redis**. Redis does **not** officially support Windows, so pick **one** of these options:

### 2.1 Option A — Docker Desktop (recommended)

1. Install **Docker Desktop** from https://www.docker.com/products/docker-desktop/ and start it.
2. Run Redis:
   ```powershell
   docker run -d --name redis -p 6379:6379 redis:7
   ```
   | Part              | Meaning                                                    |
   |-------------------|------------------------------------------------------------|
   | `-d`              | Run in the background (detached)                           |
   | `--name redis`    | Name the container so you can start/stop it by name        |
   | `-p 6379:6379`    | Expose Redis' port 6379 on your machine                    |
   | `redis:7`         | The official Redis image, version 7                        |
3. Check it is running:
   ```powershell
   docker ps
   docker exec -it redis redis-cli ping
   ```
   Output: `PONG` ✅
4. Later sessions:
   ```powershell
   docker start redis     # start again after a reboot
   docker stop redis      # stop it
   ```

### 2.2 Option B — WSL (Windows Subsystem for Linux)

```powershell
wsl --install            # once, then reboot; installs Ubuntu
```

Inside the Ubuntu terminal:

```bash
sudo apt update
sudo apt install redis-server -y
sudo service redis-server start
redis-cli ping           # → PONG
```

Redis inside WSL is reachable from Windows at `localhost:6379`.

### 2.3 Option C — Memurai

**Memurai** is a Redis-compatible server for Windows (free developer edition): https://www.memurai.com/. Install it; it runs as a Windows service on port 6379.

### 2.4 macOS / Linux

```bash
# macOS
brew install redis
brew services start redis

# Ubuntu / Debian
sudo apt install redis-server
sudo systemctl enable --now redis-server

redis-cli ping           # → PONG
```

### 2.5 (Alternative) RabbitMQ

```powershell
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

- Broker URL: `amqp://guest:guest@localhost:5672//`
- Management UI: http://localhost:15672 (user `guest`, password `guest`)

The rest of the guide uses Redis, but everything works with RabbitMQ — only the broker URL changes.

### 2.6 Redis URL format

```
redis://[:password@]host:port/db_number
```

| Example                                  | Meaning                                    |
|------------------------------------------|--------------------------------------------|
| `redis://localhost:6379/0`               | Local Redis, database 0                    |
| `redis://localhost:6379/1`               | Same server, database 1 (keep apart data)  |
| `redis://:s3cret@redis.example.com:6379/0` | With password                            |
| `rediss://:s3cret@host:6380/0`           | TLS connection (note the double `s`)       |

Redis has 16 numbered databases (0–15) by default. A common layout: `0` = Celery broker, `1` = Celery results, `2` = Django cache.

---

## Step 3: Install Celery and Helper Packages

```powershell
pip install "celery[redis]"
```

`celery[redis]` installs Celery **plus** the `redis` Python client. (For RabbitMQ just `pip install celery` — the AMQP client comes with it.)

Also install the Django helper packages used later in this guide:

```powershell
pip install django-celery-results django-celery-beat flower
```

| Package                  | What it adds                                                          | Step  |
|--------------------------|-----------------------------------------------------------------------|-------|
| `celery`                 | The task queue, worker and beat                                       | all   |
| `redis`                  | Python client to talk to Redis                                        | all   |
| `django-celery-results`  | Stores task results in your Django database (+ admin pages)           | 12    |
| `django-celery-beat`     | Stores periodic schedules in the database, editable in the admin      | 23    |
| `flower`                 | Real-time web dashboard for workers and tasks                         | 24    |

Check the install:

```powershell
celery --version
```

Save dependencies:

```powershell
pip freeze > requirements.txt
```

---

## Step 4: Create the Celery App (`config/celery.py`)

Every project needs **one Celery application instance**. Put it next to `settings.py`.

Create `config/celery.py`:

```python
import os

from celery import Celery

# 1. Tell Celery which Django settings to use (same as manage.py does).
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')

# 2. Create the Celery application. The name is used as a prefix for auto-generated task names
#    in __main__ modules; by convention use the project package name.
app = Celery('config')

# 3. Read configuration from Django settings.
#    namespace='CELERY' means: every Celery setting must be written in settings.py
#    in UPPER CASE with a CELERY_ prefix, e.g. broker_url → CELERY_BROKER_URL.
app.config_from_object('django.conf:settings', namespace='CELERY')

# 4. Look for a tasks.py module in every app listed in INSTALLED_APPS
#    and register the tasks found there.
app.autodiscover_tasks()


# 5. A tiny task to check the setup. bind=True gives access to `self` (the task instance).
@app.task(bind=True, ignore_result=True)
def debug_task(self):
    print(f'Request: {self.request!r}')
```

### 4.1 What each line does

| Line                                              | Purpose                                                                                               |
|---------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| `os.environ.setdefault('DJANGO_SETTINGS_MODULE', …)` | The worker is started with the `celery` command, not `manage.py`, so it must be told where settings live. `setdefault` keeps any value you already exported (e.g. production settings). |
| `Celery('config')`                                | Creates the app. The first argument is the "main" module name.                                       |
| `config_from_object('django.conf:settings', namespace='CELERY')` | Loads every `CELERY_*` setting from `settings.py`. The string form means the settings are read lazily — the worker doesn't need them until it starts. |
| `autodiscover_tasks()`                            | Imports `<app>/tasks.py` for every installed app (lazily, after Django is ready).                    |
| `debug_task`                                      | Optional test task. `self.request` contains the task id, args, retries, hostname…                     |

> **Why the file is called `celery.py`:** inside the `config` package, `from celery import Celery` still imports the real library because Python 3 uses absolute imports. Do **not** name it `celery.py` at the project root (next to `manage.py`) — that *would* shadow the library.

---

## Step 5: Load the App When Django Starts (`config/__init__.py`)

Open `config/__init__.py` (it is empty) and add:

```python
# Make sure the Celery app is always imported when Django starts,
# so that @shared_task decorators use this app.
from .celery import app as celery_app

__all__ = ('celery_app',)
```

**Why this matters:** `@shared_task` (used in your apps) does not know about `config/celery.py`. It attaches tasks to the *current* app. Importing `celery_app` here makes it the current/default app as soon as Django loads `config`, both in `runserver` and in the worker.

---

## Step 6: Add the Celery Settings

Open `config/settings.py` and add at the bottom:

```python
# --- Celery ---
import os  # (already at the top of most settings files — don't import twice)

# Where messages go. Read from an environment variable, fall back to local Redis.
CELERY_BROKER_URL = os.environ.get('CELERY_BROKER_URL', 'redis://localhost:6379/0')

# Where results/states are stored (changed to 'django-db' in Step 12).
CELERY_RESULT_BACKEND = os.environ.get('CELERY_RESULT_BACKEND', 'redis://localhost:6379/1')

# Serialization — JSON only (safe; never accept pickle from the network).
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'

# Use the same time zone as Django (affects crontab schedules and ETA times).
CELERY_TIMEZONE = TIME_ZONE          # TIME_ZONE is defined earlier in settings.py
CELERY_ENABLE_UTC = True

# Report the STARTED state (otherwise a running task still says PENDING).
CELERY_TASK_TRACK_STARTED = True

# Global safety limits (seconds). A task may override them.
CELERY_TASK_TIME_LIMIT = 30 * 60        # hard limit: worker kills the task after 30 min
CELERY_TASK_SOFT_TIME_LIMIT = 25 * 60   # soft limit: raises SoftTimeLimitExceeded inside the task

# Delete stored results after 1 day.
CELERY_RESULT_EXPIRES = 60 * 60 * 24

# Keep retrying the broker connection when the worker starts (silences a deprecation warning).
CELERY_BROKER_CONNECTION_RETRY_ON_STARTUP = True
```

### 6.1 How setting names map

With `namespace='CELERY'`, take the official (lower-case) Celery setting name, upper-case it, and add `CELERY_`:

| Celery docs name            | In `settings.py`                     |
|-----------------------------|--------------------------------------|
| `broker_url`                | `CELERY_BROKER_URL`                  |
| `result_backend`            | `CELERY_RESULT_BACKEND`              |
| `task_serializer`           | `CELERY_TASK_SERIALIZER`             |
| `task_always_eager`         | `CELERY_TASK_ALWAYS_EAGER`           |
| `beat_schedule`             | `CELERY_BEAT_SCHEDULE`               |
| `worker_prefetch_multiplier`| `CELERY_WORKER_PREFETCH_MULTIPLIER`  |

> **Gotcha:** a typo such as `CELERY_BROKER = ...` is silently ignored — Celery then falls back to its default broker `amqp://guest@localhost//` and you'll see `Connection refused` errors mentioning port **5672**. If you see 5672 while using Redis, check the setting name.

### 6.2 Use environment variables (recommended)

Never hard-code passwords. In PowerShell (current terminal only):

```powershell
$env:CELERY_BROKER_URL = "redis://localhost:6379/0"
$env:CELERY_RESULT_BACKEND = "redis://localhost:6379/1"
```

macOS/Linux:

```bash
export CELERY_BROKER_URL="redis://localhost:6379/0"
```

Every process (runserver, worker, beat) must see the **same** values. Packages like `python-dotenv` or `django-environ` can load them from a `.env` file.

---

## Step 7: Write Your First Task

Create `blog/tasks.py`:

```python
import time

from celery import shared_task


@shared_task
def add(x, y):
    """A demo task that pretends to be slow."""
    time.sleep(5)
    return x + y
```

### 7.1 `@shared_task` vs `@app.task`

| Decorator        | Where to use it                         | Why                                                                     |
|------------------|-----------------------------------------|-------------------------------------------------------------------------|
| `@shared_task`   | **Inside Django apps** (`blog/tasks.py`) | Doesn't import `config.celery` → apps stay reusable, no circular imports |
| `@app.task`      | Inside `config/celery.py`               | Needs the concrete `app` object                                         |

Both accept the same options (`bind`, `name`, `max_retries`, …).

### 7.2 Task names

Celery registers the task under its **module path + function name**: `blog.tasks.add`. The worker finds the function by that name. You can choose a fixed name:

```python
@shared_task(name='blog.add_numbers')
def add(x, y):
    ...
```

> Keep imports consistent (`from blog.tasks import add`, not `from .tasks import add` in one place and `from myproject.blog.tasks import add` elsewhere), otherwise the auto-generated name can differ and the worker reports *"Received unregistered task"*.

---

## Step 8: Start the Worker

Open a **new terminal** (keep `runserver` in another one), activate the venv, go to the folder containing `manage.py`, and run:

```powershell
celery -A config worker --loglevel=info --pool=solo
```

| Part              | Meaning                                                                                       |
|-------------------|-----------------------------------------------------------------------------------------------|
| `celery`          | The Celery command-line program                                                               |
| `-A config`       | `--app`: where to find the Celery app → it imports `config` and finds `celery_app` / `app`    |
| `worker`          | Sub-command: start a worker                                                                   |
| `--loglevel=info` | Show received/succeeded tasks (`-l info` for short). Levels: debug, info, warning, error     |
| `--pool=solo`     | **Windows only:** run tasks one at a time in the main process (`-P solo`)                    |

> **Windows and pools:** Celery's default pool (**prefork**, uses `fork()`) does not work on Windows. Without `--pool=solo` you get errors like `ValueError: not enough values to unpack (expected 3, got 0)` and tasks never finish. Use:
> - `--pool=solo` — simplest, one task at a time (great for learning).
> - `--pool=threads --concurrency=4` — several tasks at once using threads (good for I/O tasks like email/HTTP).
> - Or run the worker in **WSL** / **Docker** to use prefork like production.
>
> On macOS/Linux simply run `celery -A config worker -l info`.

### 8.1 Read the startup banner

```
 -------------- celery@DESKTOP-ABC v5.x.x
--- ***** -----
-- ******* ---- Windows-11 2026-10-25 10:00:00
- *** --- * ---
- ** ---------- [config]
- ** ---------- .> app:         config:0x1f3a...
- ** ---------- .> transport:   redis://localhost:6379/0
- ** ---------- .> results:     redis://localhost:6379/1
- *** --- * --- .> concurrency: 1 (solo)
-- ******* ---- .> task events: OFF (enable -E to monitor tasks in this worker)
--- ***** -----
 -------------- [queues]
                .> celery           exchange=celery(direct) key=celery

[tasks]
  . blog.tasks.add
  . config.celery.debug_task

[2026-10-25 10:00:01,000: INFO/MainProcess] Connected to redis://localhost:6379/0
[2026-10-25 10:00:01,100: INFO/MainProcess] celery@DESKTOP-ABC ready.
```

Check each line:

| Line            | What to verify                                                                   |
|-----------------|----------------------------------------------------------------------------------|
| `transport`     | Your broker URL (if it says `amqp://guest@localhost` your settings weren't read) |
| `results`       | Your result backend (`disabled://` if none)                                      |
| `concurrency`   | Number of parallel tasks + pool type                                             |
| `[queues]`      | Queues this worker listens to (default `celery`)                                 |
| `[tasks]`       | **Every task the worker knows.** If your task is missing, it can't run it        |
| `ready.`        | Worker is connected and waiting                                                  |

Stop the worker with `Ctrl + C` (**warm shutdown**: finishes the current task first). Press `Ctrl + C` again to force a **cold shutdown**.

### 8.2 Restart after code changes

The worker loads your code **once**. After editing any task (or a model/function a task uses) stop it with `Ctrl + C` and start it again.

Optional auto-restart during development:

```powershell
pip install watchfiles
watchfiles --filter python "celery -A config worker -l info -P solo"
```

`watchfiles` restarts the command whenever a `.py` file changes.

---

## Step 9: Call the Task

Open a third terminal (venv active) and start the Django shell:

```powershell
python manage.py shell
```

```python
>>> from blog.tasks import add

>>> add(2, 3)                 # 1. A normal function call — runs HERE, synchronously, no worker
5

>>> result = add.delay(2, 3)  # 2. Send it to the worker — returns immediately
>>> result
<AsyncResult: 6f1c9a3e-8a59-4b0e-9d1a-2c1f0f4f7b21>

>>> result.id                 # the task id
'6f1c9a3e-8a59-4b0e-9d1a-2c1f0f4f7b21'

>>> result.ready()            # finished yet? (False during the 5-second sleep)
False

>>> result.get(timeout=10)    # wait up to 10 s for the result
5

>>> result.status
'SUCCESS'
```

In the **worker** terminal you'll see:

```
[... INFO/MainProcess] Task blog.tasks.add[6f1c9a3e-...] received
[... INFO/MainProcess] Task blog.tasks.add[6f1c9a3e-...] succeeded in 5.01s: 5
```

🎉 Celery is working.

### 9.1 The three ways to call a task

| Call                                          | Runs where        | Returns         | Use for                                    |
|-----------------------------------------------|-------------------|-----------------|--------------------------------------------|
| `add(2, 3)`                                   | Current process   | The real value  | Tests, or when you really want it inline   |
| `add.delay(2, 3)`                             | Worker            | `AsyncResult`   | Normal usage — shortcut for `apply_async`  |
| `add.apply_async(args=(2, 3), kwargs={}, **options)` | Worker     | `AsyncResult`   | When you need options (delay, queue…)     |

`delay(*args, **kwargs)` == `apply_async(args, kwargs)` — it just can't take execution options.

### 9.2 `apply_async` options

```python
from datetime import timedelta
from django.utils import timezone

add.apply_async((2, 3), countdown=10)                         # run in ~10 seconds
add.apply_async((2, 3), eta=timezone.now() + timedelta(hours=1))  # run at an exact time (aware datetime)
add.apply_async((2, 3), expires=60)                           # discard if not started within 60 s
add.apply_async((2, 3), queue='emails')                       # send to a specific queue (Step 21)
add.apply_async((2, 3), priority=5)                           # 0–9 (Step 21.4)
add.apply_async((2, 3), task_id='my-own-id-123')              # choose the id yourself
add.apply_async((2, 3), retry=True, retry_policy={            # retry *publishing* if broker is down
    'max_retries': 3, 'interval_start': 0, 'interval_step': 0.5, 'interval_max': 3,
})
add.apply_async((2, 3), link=add.s(10))                       # on success, call add(result, 10)
add.apply_async((2, 3), link_error=log_error.s())             # on failure, call an error handler
add.apply_async((2, 3), headers={'source': 'admin'})          # custom headers → self.request
add.apply_async((2, 3), ignore_result=True)                   # don't store the result
add.apply_async((2, 3), time_limit=60, soft_time_limit=50)    # per-call time limits
```

| Option             | Meaning                                                                              |
|--------------------|--------------------------------------------------------------------------------------|
| `countdown`        | Seconds to wait before running                                                       |
| `eta`              | Exact datetime to run (use timezone-aware datetimes)                                 |
| `expires`          | Seconds or datetime; if the worker gets it later, it's marked `REVOKED` and skipped  |
| `queue`            | Queue name to send to                                                                |
| `priority`         | Message priority (support depends on the broker)                                     |
| `task_id`          | Custom id (must be unique)                                                           |
| `retry` / `retry_policy` | Retry **sending** the message when the broker connection fails              |
| `link` / `link_error` | Callbacks on success / failure                                                    |
| `serializer`       | Override the serializer (keep `json`)                                                |
| `ignore_result`    | Don't store state/result                                                             |
| `time_limit` / `soft_time_limit` | Per-call limits overriding task/global ones                           |

> **ETA / countdown warning (Redis):** a task with an ETA is delivered to a worker immediately and **held in the worker's memory** until its time. If the ETA is later than the Redis `visibility_timeout` (default **1 hour**), Redis thinks the message was lost and **re-delivers** it → the task runs **twice**. For anything far in the future, use **beat** (Step 22) or a database field + periodic task instead of a long countdown. Or raise the timeout (Step 18.3).

### 9.3 Send a task by name (no import)

Useful when the task lives in another codebase:

```python
from config.celery import app

app.send_task('blog.tasks.add', args=[2, 3])
```

### 9.4 Call from the command line

```powershell
celery -A config call blog.tasks.add --args='[2, 3]'
celery -A config call blog.tasks.add --kwargs='{\"x\": 2, \"y\": 3}' --countdown=10
```

Prints the task id. (On PowerShell, escape inner double quotes as shown.)

---

## Step 10: Work With Results (`AsyncResult`)

### 10.1 Getting a result object later

You often store only the **id** (in a model, session or URL) and look it up later:

```python
from celery.result import AsyncResult

result = AsyncResult('6f1c9a3e-8a59-4b0e-9d1a-2c1f0f4f7b21')
# or: result = add.AsyncResult(task_id)
```

### 10.2 Attributes and methods

| Member                        | Meaning                                                                       |
|-------------------------------|-------------------------------------------------------------------------------|
| `result.id`                   | Task id                                                                       |
| `result.state` / `.status`    | Current state (see 10.3)                                                      |
| `result.ready()`              | `True` if finished (SUCCESS, FAILURE or REVOKED)                              |
| `result.successful()`         | `True` if SUCCESS                                                             |
| `result.failed()`             | `True` if FAILURE                                                             |
| `result.result` / `.info`     | Return value, **or** the exception instance on failure, **or** progress meta  |
| `result.get(timeout=10)`      | Wait and return the value; **re-raises** the task's exception on failure      |
| `result.get(propagate=False)` | Return the exception instead of raising it                                   |
| `result.traceback`            | Traceback string on failure                                                   |
| `result.date_done`            | When it finished                                                              |
| `result.name`, `.args`, `.kwargs` | Available when `CELERY_RESULT_EXTENDED = True`                           |
| `result.forget()`             | Delete the stored result from the backend                                     |
| `result.revoke()`             | Cancel the task (Step 10.5)                                                   |

### 10.3 Task states

| State      | Meaning                                                                                       |
|------------|-----------------------------------------------------------------------------------------------|
| `PENDING`  | Waiting **or unknown** — Celery returns PENDING for *any* id it has no record of (even a typo) |
| `RECEIVED` | A worker received it (only reported with events enabled)                                      |
| `STARTED`  | Running (only if `CELERY_TASK_TRACK_STARTED = True`)                                          |
| `RETRY`    | Failed and scheduled to retry                                                                  |
| `SUCCESS`  | Finished; `result.result` is the return value                                                  |
| `FAILURE`  | Raised an exception; `result.result` is the exception                                         |
| `REVOKED`  | Cancelled or expired                                                                           |
| *custom*   | Anything you set with `self.update_state(state='PROGRESS', ...)` (Step 16)                    |

### 10.4 Rules for `.get()`

1. **Never call `.get()` inside a task.** The worker would block waiting for another task that may need the same worker slot → deadlock. Celery raises `RuntimeError: Never call result.get() within a task!`. Use **chains/chords** (Step 19) instead.
2. **Avoid `.get()` in views** — it blocks the web request, defeating the purpose. Return the task id and poll (Step 16).
3. `.get()` needs a **result backend**; without one you get `NotImplementedError: No result backend is configured`.

### 10.5 Revoking (cancelling) tasks

```python
result = add.apply_async((2, 3), countdown=60)
result.revoke()                                   # won't run if not started yet

# Kill a task that is ALREADY running (prefork pool only, not solo/threads):
result.revoke(terminate=True, signal='SIGKILL')

# By id:
from config.celery import app
app.control.revoke('6f1c9a3e-...', terminate=True)
```

Workers keep the list of revoked ids **in memory**. If all workers restart, the list is lost. Keep it on disk with `--statedb`:

```powershell
celery -A config worker -l info -P solo --statedb=worker.state
```

### 10.6 Ignoring results

Most tasks (sending email, cleanup) don't need their result stored. Saving it costs a write per task.

```python
@shared_task(ignore_result=True)
def cleanup():
    ...
```

Or globally: `CELERY_TASK_IGNORE_RESULT = True` and opt-in per task with `ignore_result=False`.

---

## Step 11: Choose a Result Backend

| Backend value           | Stores in                    | Notes                                                       |
|-------------------------|------------------------------|-------------------------------------------------------------|
| `redis://…/1`           | Redis                        | Fast; supports chords; results expire automatically        |
| `django-db`             | Your Django database         | Via `django-celery-results`; viewable in the admin (Step 12) |
| `django-cache`          | Django's cache               | Via `django-celery-results`                                 |
| `rpc://`                | Sent back as AMQP messages   | RabbitMQ only; result can be read **once** by the caller; no chords |
| `db+postgresql://…`     | SQL database via SQLAlchemy  | Needs `sqlalchemy`                                          |
| *(not set)*             | Nowhere                      | `.get()` / states unavailable; fine if you never read results |

For learning we switch to `django-db` so you can **see** every task in the admin.

---

## Step 12: Store Results in the Django Database (`django-celery-results`)

### 12.1 Install and enable

Already installed in Step 3. Add it to `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django_celery_results',   # 👈 add
    'accounts',
    'blog',
]
```

### 12.2 Settings

Replace the result backend line:

```python
CELERY_RESULT_BACKEND = 'django-db'
CELERY_CACHE_BACKEND = 'django-cache'   # only used if you choose 'django-cache' somewhere
CELERY_RESULT_EXTENDED = True           # also store task name, args, kwargs, worker, retries
```

### 12.3 Create the tables

```powershell
python manage.py migrate django_celery_results
```

Tables created:

| Table                                   | Model         | Holds                                                     |
|-----------------------------------------|---------------|-----------------------------------------------------------|
| `django_celery_results_taskresult`      | `TaskResult`  | One row per task: id, name, status, result, traceback, dates |
| `django_celery_results_groupresult`     | `GroupResult` | Results of groups (Step 19)                               |
| `django_celery_results_chordcounter`    | `ChordCounter`| Counts finished chord members                             |

### 12.4 See it in the admin

Restart the worker (settings changed!), run `add.delay(4, 4)` in the shell, then open http://127.0.0.1:8000/admin/ → **Celery Results → Task results**. You'll see the task with status `SUCCESS`, result `8`, task name, args and the worker name.

### 12.5 Query results in code

```python
from django_celery_results.models import TaskResult

TaskResult.objects.filter(status='FAILURE').order_by('-date_done')[:10]
TaskResult.objects.get(task_id='6f1c9a3e-...').result      # JSON string of the return value
```

### 12.6 Cleaning old results

`CELERY_RESULT_EXPIRES` (Step 6) tells Celery to delete old database results. For database backends this is done by the built-in task **`celery.backend_cleanup`**, which **beat** runs automatically every day at 4:00 AM — only if beat is running (Step 22).

---

## Step 13: A Real Task — Send Email Without Blocking the Request

Let's send emails in the background. The auth guide configured the console email backend:

```python
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
DEFAULT_FROM_EMAIL = 'My Blog <noreply@myblog.local>'
```

> With the console backend, emails sent by a task are printed in the **worker** terminal — not the `runserver` one — because the worker is the process sending them.

### 13.1 Welcome email after sign-up

Create `accounts/tasks.py`:

```python
from celery import shared_task
from celery.utils.log import get_task_logger
from django.contrib.auth import get_user_model
from django.core.mail import send_mail

logger = get_task_logger(__name__)


@shared_task
def send_welcome_email(user_id):
    User = get_user_model()
    try:
        user = User.objects.get(pk=user_id)
    except User.DoesNotExist:
        logger.warning('User %s no longer exists, skipping welcome email', user_id)
        return 'user missing'

    if not user.email:
        return 'no email address'

    send_mail(
        subject='Welcome to My Blog!',
        message=f'Hi {user.get_username()},\n\nThanks for signing up.',
        from_email=None,                 # None → DEFAULT_FROM_EMAIL
        recipient_list=[user.email],
    )
    logger.info('Welcome email sent to %s', user.email)
    return f'sent to {user.email}'
```

Update `SignUpView.form_valid` in `accounts/views.py`:

```python
from django.db import transaction

from .tasks import send_welcome_email


class SignUpView(CreateView):
    ...

    def form_valid(self, form):
        response = super().form_valid(form)          # saves the user → self.object

        authors = Group.objects.filter(name='Authors').first()
        if authors:
            self.object.groups.add(authors)

        # Send the email only after the user row is really committed to the database.
        user_id = self.object.pk
        transaction.on_commit(lambda: send_welcome_email.delay(user_id))

        login(self.request, self.object, backend='django.contrib.auth.backends.ModelBackend')
        return response
```

### 13.2 Why pass the **id**, not the user object?

| Passing `user` (object)                                          | Passing `user.pk` (id)                                    |
|------------------------------------------------------------------|-----------------------------------------------------------|
| ❌ Not JSON serializable → `kombu.exceptions.EncodeError`          | ✅ An int — always serializable                            |
| ❌ Data may be outdated when the task runs (minutes later)         | ✅ The task loads fresh data                               |
| ❌ Large message size                                              | ✅ Tiny message                                            |

**Rule:** pass ids and simple values (`str`, `int`, `float`, `bool`, `list`, `dict`, `None`). Load objects inside the task.

### 13.3 Why `transaction.on_commit`?

If the view runs inside a database transaction (e.g. `ATOMIC_REQUESTS = True`, or inside `transaction.atomic()`), the new row isn't visible to other connections until **commit**. A fast worker could run the task **before** the commit and get `User.DoesNotExist`. If the transaction is rolled back, the email would be sent for a user that doesn't exist.

`transaction.on_commit(fn)` runs `fn` **only after a successful commit** (immediately if you're not in a transaction).

**Shortcut (Celery 5.4+):** tasks created with `@shared_task` in a Django project get extra methods:

```python
send_welcome_email.delay_on_commit(user_id)                  # same as on_commit(lambda: ....delay(user_id))
send_welcome_email.apply_async_on_commit((user_id,), countdown=5)
```

> These don't return an `AsyncResult` (the task isn't sent yet when they return). Use them when you don't need the id.

### 13.4 Notify the author when a post is published

Add to `blog/tasks.py`:

```python
from celery import shared_task
from celery.utils.log import get_task_logger
from django.core.mail import send_mail

from .models import Post

logger = get_task_logger(__name__)


@shared_task
def send_post_published_email(post_id):
    post = Post.objects.select_related('author').filter(pk=post_id).first()
    if post is None:
        return 'post deleted'
    if not post.author or not post.author.email:
        return 'author has no email'

    send_mail(
        subject=f'Your post "{post.title}" is live',
        message=f'Hi {post.author.get_username()},\n\n"{post.title}" has been published.',
        from_email=None,
        recipient_list=[post.author.email],
    )
    logger.info('Published email sent for post %s', post_id)
    return f'sent to {post.author.email}'
```

Update `post_publish` in `blog/views.py`:

```python
from .tasks import send_post_published_email


@login_required
@permission_required('blog.publish_post', raise_exception=True)
@require_POST
def post_publish(request, pk):
    post = get_object_or_404(Post, pk=pk)
    post.status = 'published'
    post.save(update_fields=['status', 'updated_at'])
    send_post_published_email.delay_on_commit(post.pk)        # 👈 background email
    messages.success(request, f'"{post.title}" is now published.')
    return redirect(post)
```

### 13.5 Try it

1. Terminal 1: `python manage.py runserver`
2. Terminal 2: `celery -A config worker -l info -P solo` (restart it — new tasks were added; check `[tasks]` lists `accounts.tasks.send_welcome_email` and `blog.tasks.send_post_published_email`).
3. Sign up a new user with an email at http://127.0.0.1:8000/accounts/signup/.
4. The page responds instantly; the **worker** terminal prints the email and `Task accounts.tasks.send_welcome_email[...] succeeded`.
5. Publish a post as an Editor → the author's email appears in the worker terminal.

---

## Step 14: Task Options and the `self` Argument (`bind=True`)

### 14.1 Bound tasks

`bind=True` passes the task instance as the first argument `self`:

```python
@shared_task(bind=True)
def show_request(self, x):
    print(self.request.id)          # this run's task id
    print(self.request.retries)     # how many times it has been retried
    print(self.request.hostname)    # worker name, e.g. celery@DESKTOP-ABC
    print(self.request.args, self.request.kwargs)
    return x
```

`self.request` fields:

| Field              | Meaning                                              |
|--------------------|------------------------------------------------------|
| `id`               | Task id                                              |
| `args`, `kwargs`   | Arguments                                            |
| `retries`          | Number of retries so far (0 on first run)            |
| `is_eager`         | `True` if run locally (eager mode / `apply()`)       |
| `eta`, `expires`   | Scheduling info                                      |
| `hostname`         | Worker node name                                     |
| `delivery_info`    | Queue / exchange / routing key info                  |
| `root_id`, `parent_id` | Ids in a workflow (chain/chord)                  |
| `headers`          | Custom headers passed to `apply_async`               |
| `called_directly`  | `True` if called as a normal function                |

You need `bind=True` for: `self.retry()`, `self.update_state()`, `self.request`.

### 14.2 All decorator options

```python
@shared_task(
    bind=True,
    name='blog.tasks.example',     # explicit task name
    ignore_result=False,           # store result
    max_retries=3,                 # for self.retry() / autoretry (None = forever)
    default_retry_delay=60,        # seconds between retries when using self.retry()
    autoretry_for=(Exception,),    # retry automatically on these exceptions
    retry_backoff=True,            # exponential back-off: 1s, 2s, 4s, 8s…
    retry_backoff_max=600,         # cap back-off at 10 minutes
    retry_jitter=True,             # randomize delays (avoid thundering herd)
    retry_kwargs={'max_retries': 5},
    rate_limit='10/m',             # max 10 per minute per worker
    time_limit=120,                # hard limit (seconds)
    soft_time_limit=100,           # soft limit (seconds)
    acks_late=True,                # ack after the task finishes (Step 18)
    reject_on_worker_lost=True,    # requeue if the worker process dies
    track_started=True,            # report STARTED
    queue='default',               # default queue for this task
    priority=5,                    # default priority
    expires=3600,                  # default expiry
    serializer='json',
    throws=(ValueError,),          # expected errors: logged as INFO, no traceback noise
    typing=True,                   # check arguments match the signature when calling
)
def example(self, x):
    ...
```

---

## Step 15: Retries — Handle Temporary Failures

Network calls fail. Celery can retry automatically.

### 15.1 Automatic retries (recommended)

Add to `blog/tasks.py`:

```python
import json
from urllib.error import URLError
from urllib.request import urlopen


@shared_task(
    autoretry_for=(URLError, TimeoutError, ConnectionError),
    retry_backoff=True,       # wait 1, 2, 4, 8, 16 s… between tries
    retry_backoff_max=300,    # never wait more than 5 minutes
    retry_jitter=True,        # add randomness
    retry_kwargs={'max_retries': 5},
)
def fetch_json(url):
    """Download JSON from an external API."""
    with urlopen(url, timeout=10) as response:
        return json.loads(response.read().decode())
```

If the request raises `URLError`, Celery schedules a retry. After 5 retries it gives up and the task ends in `FAILURE` with the last exception.

### 15.2 Manual retries with `self.retry()`

More control (decide per error, change countdown):

```python
from smtplib import SMTPException


@shared_task(bind=True, max_retries=3, default_retry_delay=30)
def send_newsletter(self, user_id):
    from django.contrib.auth import get_user_model
    from django.core.mail import send_mail

    user = get_user_model().objects.get(pk=user_id)
    try:
        send_mail('Newsletter', 'This week on My Blog…', None, [user.email])
    except SMTPException as exc:
        # raise is important: self.retry() raises a Retry exception that tells the worker
        # to stop this run and schedule a new one.
        raise self.retry(exc=exc, countdown=2 ** self.request.retries * 30)
```

| Argument            | Meaning                                                                  |
|---------------------|--------------------------------------------------------------------------|
| `exc`               | The original exception — re-raised when retries run out                  |
| `countdown` / `eta` | When to retry (default `default_retry_delay`, 180 s if not set)          |
| `max_retries`       | Override the task's limit for this call                                  |
| `args`, `kwargs`    | Retry with different arguments                                            |

When `max_retries` is exceeded, `self.retry(exc=exc)` raises `exc` (or `MaxRetriesExceededError` if no `exc` was given) → state `FAILURE`.

### 15.3 What to retry and what not to

| Retry ✅                                | Don't retry ❌                                         |
|-----------------------------------------|--------------------------------------------------------|
| Timeouts, connection errors             | `DoesNotExist` (the row is gone — retrying won't help) |
| HTTP 429 / 502 / 503 / 504              | HTTP 400 / 401 / 404 (your request is wrong)           |
| Database deadlocks / `OperationalError` | `ValueError`, `KeyError` — bugs in code/data           |

Don't use `autoretry_for=(Exception,)` blindly — a bug would be retried forever and spam logs.

---

## Step 16: Report Progress From a Long Task

### 16.1 The task

Add to `blog/tasks.py`:

```python
import time


@shared_task(bind=True)
def rebuild_search_index(self):
    post_ids = list(Post.objects.values_list('id', flat=True))
    total = len(post_ids)

    for i, post_id in enumerate(post_ids, start=1):
        time.sleep(1)                               # pretend to index one post
        self.update_state(
            state='PROGRESS',
            meta={'current': i, 'total': total},
        )

    return {'current': total, 'total': total, 'status': 'Index rebuilt'}
```

`update_state()` writes a custom state + `meta` dict to the **result backend**, so it needs a result backend (we have `django-db`).

### 16.2 The views

Add to `blog/views.py`:

```python
from celery.result import AsyncResult
from django.http import JsonResponse

from .tasks import rebuild_search_index


@login_required
@permission_required('blog.change_post', raise_exception=True)
def reindex_page(request):
    return render(request, 'blog/reindex.html')


@login_required
@permission_required('blog.change_post', raise_exception=True)
@require_POST
def reindex_start(request):
    result = rebuild_search_index.delay()
    return JsonResponse({'task_id': result.id}, status=202)       # 202 = Accepted


@login_required
def task_status(request, task_id):
    result = AsyncResult(task_id)
    data = {'task_id': task_id, 'state': result.state}

    if result.state == 'PROGRESS':
        data.update(result.info)                  # {'current': 3, 'total': 10}
    elif result.state == 'SUCCESS':
        data.update(result.result)
    elif result.state == 'FAILURE':
        data['error'] = str(result.result)        # the exception message — don't leak tracebacks
    return JsonResponse(data)
```

### 16.3 URLs

Add to `blog/urls.py`:

```python
    path('reindex/', views.reindex_page, name='reindex'),
    path('reindex/start/', views.reindex_start, name='reindex_start'),
    path('tasks/<str:task_id>/', views.task_status, name='task_status'),
```

### 16.4 Template with a progress bar

`blog/templates/blog/reindex.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Rebuild search index{% endblock %}

{% block content %}
    <h2>Rebuild search index</h2>

    <button id="start">Start</button>
    <progress id="bar" value="0" max="100" style="width: 300px;"></progress>
    <p id="status">Not started.</p>

    <script>
        const csrfToken = '{{ csrf_token }}';
        const statusUrl = '{% url "blog:task_status" "TASK_ID" %}';

        document.getElementById('start').addEventListener('click', async () => {
            const response = await fetch('{% url "blog:reindex_start" %}', {
                method: 'POST',
                headers: {'X-CSRFToken': csrfToken},
            });
            const {task_id} = await response.json();
            poll(task_id);
        });

        async function poll(taskId) {
            const response = await fetch(statusUrl.replace('TASK_ID', taskId));
            const data = await response.json();
            const bar = document.getElementById('bar');
            const status = document.getElementById('status');

            if (data.state === 'PROGRESS') {
                bar.max = data.total;
                bar.value = data.current;
                status.textContent = `Indexed ${data.current} of ${data.total} posts…`;
            } else if (data.state === 'SUCCESS') {
                bar.max = data.total || 1;
                bar.value = data.total || 1;
                status.textContent = data.status;
                return;                                   // stop polling
            } else if (data.state === 'FAILURE') {
                status.textContent = 'Failed: ' + data.error;
                return;
            } else {
                status.textContent = data.state + '…';    // PENDING / STARTED
            }
            setTimeout(() => poll(taskId), 1000);         // ask again in 1 second
        }
    </script>
{% endblock %}
```

### 16.5 Try it

Restart the worker, log in as an Editor, open http://127.0.0.1:8000/reindex/ and click **Start**. The bar fills as the task updates its state.

> **Security note:** any logged-in user who knows a task id can read its status with this view. For sensitive results, store which user started the task (e.g. in a model with the task id) and check it in `task_status`.

---

## Step 17: Time Limits and Rate Limits

### 17.1 Time limits

| Limit              | What happens when reached                                                                  |
|--------------------|---------------------------------------------------------------------------------------------|
| `soft_time_limit`  | `SoftTimeLimitExceeded` is raised **inside** the task → you can clean up                    |
| `time_limit`       | The worker **kills** the process running the task (no clean-up) → `TimeLimitExceeded`      |

```python
from celery.exceptions import SoftTimeLimitExceeded


@shared_task(soft_time_limit=60, time_limit=90)
def export_posts_csv():
    try:
        ...  # long work
    except SoftTimeLimitExceeded:
        # delete half-written files, log, etc.
        logger.warning('Export took too long, aborting')
        raise
```

> On **Windows** with `--pool=solo` or `threads`, time limits are **not enforced** (they rely on Unix signals / killing child processes). They work with prefork on Linux/macOS (WSL, Docker, production).

### 17.2 Rate limits

```python
@shared_task(rate_limit='10/m')      # '10/s', '100/m', '1000/h'
def call_partner_api(item_id):
    ...
```

- The limit is **per worker instance**, not global across all workers.
- Change it at runtime: `celery -A config control rate_limit blog.tasks.call_partner_api 5/m`.
- Disable all rate limits on a worker: `CELERY_WORKER_DISABLE_RATE_LIMITS = True`.

---

## Step 18: Reliability — Acks, Idempotency, Visibility Timeout

### 18.1 Early vs late acknowledgement

| Mode                       | When the message is deleted from the broker | If the worker crashes mid-task              |
|----------------------------|---------------------------------------------|---------------------------------------------|
| **Early ack** (default)    | When the worker **receives** it              | Task is **lost** (runs at most once)        |
| **Late ack** (`acks_late=True`) | After the task **finishes**             | Message is **redelivered** (runs at least once) |

```python
CELERY_TASK_ACKS_LATE = True                  # globally
CELERY_TASK_REJECT_ON_WORKER_LOST = True      # requeue if the worker process is killed (e.g. OOM)
CELERY_WORKER_PREFETCH_MULTIPLIER = 1         # each process reserves only 1 extra message
```

or per task: `@shared_task(acks_late=True, reject_on_worker_lost=True)`.

**Prefetch:** by default each worker process reserves `4` messages in advance (`worker_prefetch_multiplier = 4`). For long tasks set it to `1`, so one slow task doesn't hold messages that other idle workers could run.

### 18.2 Make tasks idempotent

With late acks (and with retries, and with Redis redelivery) a task **can run more than once**. Write tasks so running them twice is harmless:

```python
@shared_task(acks_late=True)
def mark_post_featured(post_id):
    # Idempotent: setting a flag twice gives the same result.
    Post.objects.filter(pk=post_id).update(status='published')


@shared_task(acks_late=True)
def charge_order(order_id):
    order = Order.objects.select_for_update().get(pk=order_id)   # example model
    if order.paid:                    # already done by a previous run → stop
        return 'already paid'
    ...                               # charge with an idempotency key at the payment provider
```

Tricks: check a "done" flag first; use `update_or_create`; use unique constraints; send an idempotency key to external APIs.

### 18.3 Redis `visibility_timeout`

Redis has no real acks; Celery emulates them. A message taken by a worker but not acked within `visibility_timeout` seconds is **given to another worker**. Default: 3600 s (1 hour).

```python
CELERY_BROKER_TRANSPORT_OPTIONS = {
    'visibility_timeout': 60 * 60 * 6,   # 6 hours — longer than your longest task or ETA
}
```

If tasks run (or wait with ETA/countdown) longer than this, they get executed twice. RabbitMQ doesn't have this problem.

---

## Step 19: Canvas — Combine Tasks Into Workflows

Add two helper tasks to `blog/tasks.py`:

```python
@shared_task
def mul(x, y):
    return x * y


@shared_task
def xsum(numbers):
    return sum(numbers)
```

### 19.1 Signatures — a task call you can pass around

```python
from blog.tasks import add, mul, xsum

sig = add.s(2, 3)          # signature: "add(2, 3)" — NOT sent yet
sig.delay()                # now send it
sig.apply_async(countdown=5)

add.s(2)                   # partial: missing argument filled in later by a chain
add.si(2, 3)               # immutable signature: ignores the parent's result
add.signature((2, 3), countdown=10)   # long form with options
```

### 19.2 Chain — run one after another, passing results

```python
from celery import chain

# ((2 + 2) + 4) * 10
result = chain(add.s(2, 2), add.s(4), mul.s(10))()
result.get()                     # 80

# Same with the | operator:
(add.s(2, 2) | add.s(4) | mul.s(10))()
```

Each task's return value becomes the **first argument** of the next. Use `.si()` when the next task should not receive it:

```python
(send_post_published_email.si(post.pk) | rebuild_search_index.si())()
```

If any task in the chain fails, the rest **don't run**.

### 19.3 Group — run in parallel

```python
from celery import group

job = group(add.s(i, i) for i in range(10))
result = job()                   # GroupResult
result.get()                     # [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
result.ready(), result.successful(), result.completed_count()
```

### 19.4 Chord — group + callback when all are done

```python
from celery import chord

result = chord(add.s(i, i) for i in range(10))(xsum.s())
result.get()                     # 90 — xsum received the list of all results
```

- A chord **requires a result backend** (Redis is best; `django-db` works; `rpc://` does not).
- If one member fails, the callback doesn't run and the chord fails.
- Don't build chords with thousands of members if you can batch instead.

### 19.5 map, starmap, chunks

```python
xsum.map([[1, 2], [3, 4]])                        # one task: [xsum([1,2]), xsum([3,4])]
add.starmap([(1, 2), (3, 4)])                     # one task: [add(1,2), add(3,4)]
add.chunks(zip(range(100), range(100)), 10)()     # 10 tasks, each doing 10 additions
```

`chunks` is great for "process 10,000 items": fewer messages than 10,000 separate tasks.

### 19.6 Error callbacks

```python
@shared_task
def on_error(request, exc, traceback):
    logger.error('Task %s failed: %r', request.id, exc)


add.apply_async((2, 'x'), link_error=on_error.s())
```

### 19.7 Real example — notify all subscribers

```python
from celery import group
from django.contrib.auth import get_user_model


@shared_task
def notify_all_users(post_id):
    user_ids = get_user_model().objects.filter(is_active=True).exclude(email='') \
        .values_list('id', flat=True)
    # one small task per user, run in parallel by all workers
    group(send_new_post_email.s(user_id, post_id) for user_id in user_ids).apply_async()


@shared_task(rate_limit='30/m', autoretry_for=(ConnectionError,), retry_backoff=True)
def send_new_post_email(user_id, post_id):
    ...
```

---

## Step 20: Logging in Tasks

### 20.1 Task logger

```python
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)


@shared_task
def something():
    logger.info('Starting')
    logger.warning('Careful')
    logger.error('Oops')
```

The task logger adds the **task name and id** to each line:

```
[2026-10-25 10:05:00,123: INFO/MainProcess] blog.tasks.something[0b7c...]: Starting
```

`print()` also works (worker redirects stdout to the log at WARNING level), but use the logger.

### 20.2 Log to a file

```powershell
celery -A config worker -l info -P solo --logfile=celery.log
```

### 20.3 Use Django's `LOGGING` setting instead

By default Celery **replaces** the root logger configuration. To keep Django's `LOGGING` config:

```python
CELERY_WORKER_HIJACK_ROOT_LOGGER = False
```

Or take over completely in `config/celery.py`:

```python
from celery.signals import setup_logging


@setup_logging.connect
def config_loggers(*args, **kwargs):
    from logging.config import dictConfig
    from django.conf import settings
    dictConfig(settings.LOGGING)
```

---

## Step 21: Queues and Routing

By default everything goes to the single queue `celery`. Problem: 10,000 newsletter emails block an urgent password-reset email. Solution: **separate queues** and dedicated workers.

### 21.1 Route tasks to queues

```python
CELERY_TASK_DEFAULT_QUEUE = 'default'

CELERY_TASK_ROUTES = {
    'accounts.tasks.send_welcome_email': {'queue': 'emails'},
    'blog.tasks.send_*': {'queue': 'emails'},              # glob patterns allowed
    'blog.tasks.rebuild_search_index': {'queue': 'heavy'},
}
```

Tasks not listed go to `default`.

You can also route per call: `add.apply_async((1, 2), queue='heavy')`, or per task: `@shared_task(queue='heavy')`.

### 21.2 Start workers for specific queues

```powershell
# Worker 1: everything quick
celery -A config worker -l info -P solo -Q default,emails -n worker1@%h

# Worker 2: only heavy jobs
celery -A config worker -l info -P solo -Q heavy -n heavy@%h
```

| Flag              | Meaning                                                                |
|-------------------|------------------------------------------------------------------------|
| `-Q a,b`          | Listen only to these queues                                            |
| `-n name@%h`      | Unique node name (`%h` = hostname). Required when running several workers on one machine |
| `-X queue`        | Listen to all queues **except** this one                               |

> **Gotcha:** after adding routes, a worker started without `-Q` listens only to `default` (the default queue). Tasks routed to `emails` will sit there forever until a worker consumes `emails`.

### 21.3 Check queue length (Redis)

```powershell
docker exec -it redis redis-cli -n 0 llen emails
docker exec -it redis redis-cli -n 0 llen default
```

### 21.4 Priorities

RabbitMQ supports real priorities (queue declared with `x-max-priority`). Redis emulates them:

```python
CELERY_BROKER_TRANSPORT_OPTIONS = {
    'visibility_timeout': 60 * 60 * 6,
    'priority_steps': list(range(10)),
    'sep': ':',
    'queue_order_strategy': 'priority',
}
```

With Redis, **0 is the highest priority**. Separate queues are usually simpler and more predictable than priorities.

---

## Step 22: Periodic Tasks With Celery Beat

**Beat** is a separate process that sends tasks on a schedule. Workers still run them.

### 22.1 Create the periodic tasks

Add to `blog/tasks.py`:

```python
from datetime import timedelta

from django.contrib.auth import get_user_model
from django.utils import timezone


@shared_task(ignore_result=True)
def delete_old_drafts(days=30):
    cutoff = timezone.now() - timedelta(days=days)
    deleted, _ = Post.objects.filter(status='draft', updated_at__lt=cutoff).delete()
    logger.info('Deleted %s old drafts', deleted)
    return deleted


@shared_task(ignore_result=True)
def send_weekly_digest():
    week_ago = timezone.now() - timedelta(days=7)
    posts = Post.objects.filter(status='published', created_at__gte=week_ago)
    if not posts:
        return 'nothing new'

    lines = '\n'.join(f'- {p.title}' for p in posts)
    emails = get_user_model().objects.filter(is_active=True).exclude(email='') \
        .values_list('email', flat=True)
    for email in emails:
        send_mail('This week on My Blog', f'New posts:\n{lines}', None, [email])
    return f'sent to {len(emails)} users'
```

### 22.2 Define the schedule in settings

```python
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    'delete-old-drafts-every-night': {
        'task': 'blog.tasks.delete_old_drafts',
        'schedule': crontab(hour=3, minute=0),           # every day at 03:00
        'kwargs': {'days': 30},
    },
    'weekly-digest-monday-morning': {
        'task': 'blog.tasks.send_weekly_digest',
        'schedule': crontab(hour=8, minute=0, day_of_week='mon'),
    },
    'demo-add-every-30-seconds': {
        'task': 'blog.tasks.add',
        'schedule': 30.0,                                # seconds (float/int or timedelta)
        'args': (16, 16),
        'options': {'expires': 25},                      # any apply_async option
    },
}
```

| Key         | Meaning                                                                  |
|-------------|--------------------------------------------------------------------------|
| key name    | Unique name of the entry (any string)                                    |
| `task`      | Registered task **name** (string, not the function)                      |
| `schedule`  | Seconds, `timedelta`, `crontab(...)` or `solar(...)`                     |
| `args`      | Positional arguments (tuple/list)                                         |
| `kwargs`    | Keyword arguments (dict)                                                  |
| `options`   | `apply_async` options: `queue`, `expires`, `priority`…                    |
| `relative`  | `True` → round the interval to the period (rarely needed)                 |

### 22.3 `crontab` cheat sheet

| Expression                                              | Runs                                          |
|---------------------------------------------------------|-----------------------------------------------|
| `crontab()`                                             | Every minute                                  |
| `crontab(minute='*/15')`                                | Every 15 minutes                              |
| `crontab(minute=0, hour='*/3')`                         | Every 3 hours on the hour                     |
| `crontab(minute=0, hour=0)`                             | Daily at midnight                             |
| `crontab(minute=30, hour=7, day_of_week='mon-fri')`     | Weekdays at 07:30                             |
| `crontab(minute=0, hour=9, day_of_week='sun')`          | Sundays at 09:00                              |
| `crontab(minute=0, hour=0, day_of_month='1')`           | First day of every month                      |
| `crontab(minute=0, hour=0, day_of_month='1', month_of_year='1,4,7,10')` | First day of each quarter     |
| `crontab(minute=0, hour='9-17')`                        | Every hour from 9:00 to 17:00                 |

Times use `CELERY_TIMEZONE`.

Solar schedules (sunrise/sunset at a location):

```python
from celery.schedules import solar

'schedule': solar('sunset', 27.7172, 85.3240)    # sunset in Kathmandu
```

### 22.4 Run beat

Open **another terminal**:

```powershell
celery -A config beat --loglevel=info
```

Output:

```
celery beat v5.x.x is starting.
LocalTime -> 2026-10-25 10:10:00
Configuration ->
    . broker -> redis://localhost:6379/0
    . loader -> celery.loaders.app.AppLoader
    . scheduler -> celery.beat.PersistentScheduler
    . db -> celerybeat-schedule
    . maxinterval -> 5.00 minutes (300s)
[... INFO/MainProcess] Scheduler: Sending due task demo-add-every-30-seconds (blog.tasks.add)
```

The worker terminal shows `blog.tasks.add` running every 30 seconds.

- The default scheduler saves last-run times in a local file `celerybeat-schedule` (add it to `.gitignore`).
- **Run exactly ONE beat process** for the whole system. Two beats = every periodic task runs twice.

### 22.5 Worker + beat in one process (development only)

```powershell
celery -A config worker -B -l info -P solo
```

`-B` embeds beat in the worker. Convenient while learning, but **never in production** with several workers (each would run its own beat → duplicates).

---

## Step 23: Manage Schedules in the Admin (`django-celery-beat`)

Hard-coded schedules need a deploy to change. `django-celery-beat` stores them in the database and lets admins edit them.

### 23.1 Enable it

```python
INSTALLED_APPS = [
    ...
    'django_celery_results',
    'django_celery_beat',      # 👈 add
    'accounts',
    'blog',
]

CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'
```

```powershell
python manage.py migrate django_celery_beat
```

Tables / models created:

| Model              | Holds                                                         |
|--------------------|---------------------------------------------------------------|
| `IntervalSchedule` | "every N seconds/minutes/hours/days"                          |
| `CrontabSchedule`  | minute, hour, day_of_week, day_of_month, month_of_year, timezone |
| `SolarSchedule`    | solar event + latitude/longitude                              |
| `ClockedSchedule`  | One exact datetime (run once)                                 |
| `PeriodicTask`     | The task name, args/kwargs (JSON), schedule, enabled, one_off, queue, expires, last_run_at, total_run_count |
| `PeriodicTasks`    | Internal: tracks changes so beat reloads the schedule         |

### 23.2 Run beat with the database scheduler

```powershell
celery -A config beat -l info
```

(It uses `CELERY_BEAT_SCHEDULER`. Without the setting, pass `--scheduler django_celery_beat.schedulers:DatabaseScheduler`.)

Entries in `CELERY_BEAT_SCHEDULE` are **copied into the database** on beat start-up, so you'll see them in the admin too.

### 23.3 Use the admin

Open http://127.0.0.1:8000/admin/ → **Periodic Tasks**:

1. **Crontabs → Add**: minute `0`, hour `6`, others `*`, timezone `UTC` → Save.
2. **Periodic tasks → Add**:
   - Name: `Morning cleanup`
   - Task (registered): pick `blog.tasks.delete_old_drafts` (the list comes from tasks Django can import)
   - Schedule: choose the crontab
   - Arguments: `[]`, Keyword arguments: `{"days": 14}` (must be **valid JSON** — double quotes)
   - Enabled ✅ → Save.
3. Beat notices the change (checks every few seconds) and schedules it — no restart needed.

Tick **One-off task** to run it only once; it's disabled after it runs. **Start datetime** delays the first run.

### 23.4 Create periodic tasks in code

```python
import json

from django_celery_beat.models import CrontabSchedule, IntervalSchedule, PeriodicTask

# every 10 minutes
schedule, _ = IntervalSchedule.objects.get_or_create(every=10, period=IntervalSchedule.MINUTES)
PeriodicTask.objects.update_or_create(
    name='Rebuild index every 10 minutes',
    defaults={
        'interval': schedule,
        'task': 'blog.tasks.rebuild_search_index',
        'args': json.dumps([]),
        'kwargs': json.dumps({}),
        'queue': 'heavy',
        'enabled': True,
    },
)

# crontab: weekdays at 18:30 in Kathmandu time
cron, _ = CrontabSchedule.objects.get_or_create(
    minute='30', hour='18', day_of_week='1-5', day_of_month='*', month_of_year='*',
    timezone='Asia/Kathmandu',
)
PeriodicTask.objects.update_or_create(
    name='Weekday evening digest',
    defaults={'crontab': cron, 'task': 'blog.tasks.send_weekly_digest'},
)

# disable / delete
PeriodicTask.objects.filter(name='Weekday evening digest').update(enabled=False)
```

`IntervalSchedule.period` choices: `DAYS`, `HOURS`, `MINUTES`, `SECONDS`, `MICROSECONDS`. A `PeriodicTask` must have **exactly one** of `interval`, `crontab`, `solar`, `clocked`.

> **Bulk updates:** `PeriodicTask.objects.update(...)` (queryset update) does not trigger the change signal; call `PeriodicTasks.update_changed()` afterwards so beat reloads.

> **Changed `TIME_ZONE`?** Reset the last-run times or tasks may be skipped:
> ```powershell
> python manage.py shell -c "from django_celery_beat.models import PeriodicTask; PeriodicTask.objects.update(last_run_at=None)"
> ```

---

## Step 24: Monitor and Control Workers

### 24.1 Inspect (read-only)

```powershell
celery -A config status                      # which workers are online
celery -A config inspect ping                # each worker answers pong
celery -A config inspect active              # tasks running right now
celery -A config inspect reserved            # received, waiting to run (prefetched)
celery -A config inspect scheduled           # tasks with ETA/countdown waiting
celery -A config inspect registered          # tasks each worker knows
celery -A config inspect stats               # pool, concurrency, totals, rusage
celery -A config inspect active_queues       # queues each worker consumes
celery -A config inspect revoked             # revoked task ids
celery -A config inspect conf                # worker configuration
celery -A config inspect query_task <id>     # info about one task
```

Add `-d worker1@HOSTNAME` to ask one worker only, `--timeout 5` to wait longer.

### 24.2 Control (changes things)

```powershell
celery -A config control enable_events               # turn on task events (for Flower/events)
celery -A config control disable_events
celery -A config control rate_limit blog.tasks.add 10/m
celery -A config control time_limit blog.tasks.add 60 120   # soft, hard
celery -A config control add_consumer emails         # start consuming a queue
celery -A config control cancel_consumer emails      # stop consuming a queue
celery -A config control pool_grow 2                 # +2 processes (prefork)
celery -A config control pool_shrink 1
celery -A config control revoke <task_id>
celery -A config control shutdown                    # warm shutdown of all workers
```

### 24.3 Same from Python

```python
from config.celery import app

i = app.control.inspect()
i.active()
i.registered()
i.stats()

app.control.ping(timeout=1)
app.control.rate_limit('blog.tasks.add', '10/m')
app.control.revoke(task_id, terminate=True)
app.control.broadcast('shutdown')
```

### 24.4 Purge — delete all waiting messages

```powershell
celery -A config purge                  # asks for confirmation; deletes ALL waiting tasks in default queues
celery -A config purge -Q emails -f     # one queue, no confirmation
```

⚠️ Irreversible. Never run it in production without thinking.

### 24.5 Events

Workers can send events (task received/started/succeeded/failed). Enable with `-E`:

```powershell
celery -A config worker -l info -P solo -E
celery -A config events                 # terminal monitor (curses UI; on Windows use Flower instead)
```

Settings equivalents: `CELERY_WORKER_SEND_TASK_EVENTS = True`, `CELERY_TASK_SEND_SENT_EVENT = True`.

### 24.6 Flower — web dashboard

```powershell
celery -A config flower --port=5555
```

Open http://localhost:5555. You can see:

- Workers: online/offline, pool, concurrency, processed count
- Tasks: name, args, state, runtime, result, traceback
- Broker: queue lengths
- Actions: revoke tasks, change rate limits, shut down/restart pools

Flower turns on events for workers automatically. Protect it in production:

```powershell
celery -A config flower --port=5555 --basic-auth=admin:Str0ngPassw0rd
```

Never expose Flower publicly without authentication — it can revoke tasks and shut down workers.

### 24.7 Other useful commands

```powershell
celery -A config report                 # versions + settings (for bug reports; hides secrets)
celery -A config shell                  # Python shell with the app loaded
celery -A config result <task_id>       # print a task's result
celery -A config list bindings          # queue/exchange bindings (RabbitMQ)
celery --help                           # all commands
celery -A config worker --help          # all worker options
```

---

## Step 25: Celery Signals

Celery has its own signals (separate from Django's). Useful for logging, metrics and alerts.

Create `blog/celery_signals.py`:

```python
from celery.signals import (
    task_failure,
    task_postrun,
    task_prerun,
    task_retry,
    worker_ready,
)
from celery.utils.log import get_task_logger
from django.core.mail import mail_admins

logger = get_task_logger(__name__)


@task_prerun.connect
def on_task_start(sender=None, task_id=None, task=None, args=None, kwargs=None, **extra):
    logger.info('START %s[%s]', task.name, task_id)


@task_postrun.connect
def on_task_end(sender=None, task_id=None, task=None, retval=None, state=None, **extra):
    logger.info('END %s[%s] -> %s', task.name, task_id, state)


@task_retry.connect
def on_task_retry(sender=None, request=None, reason=None, **extra):
    logger.warning('RETRY %s[%s]: %s', sender.name, request.id, reason)


@task_failure.connect
def on_task_failure(sender=None, task_id=None, exception=None, traceback=None, **extra):
    mail_admins(
        subject=f'Celery task failed: {sender.name}',
        message=f'Task id: {task_id}\nError: {exception!r}',
        fail_silently=True,
    )


@worker_ready.connect
def on_worker_ready(sender=None, **extra):
    logger.info('Worker %s is ready', sender.hostname)
```

Import it when Django loads the blog app — in `blog/apps.py`:

```python
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'

    def ready(self):
        from . import celery_signals  # noqa: F401  (connect Celery signal receivers)
```

| Signal                  | Fires                                                    |
|-------------------------|----------------------------------------------------------|
| `before_task_publish`   | In the producer, before a message is sent                |
| `after_task_publish`    | In the producer, after it was sent                       |
| `task_received`         | Worker received the message                              |
| `task_prerun`           | Just before the task runs                                |
| `task_postrun`          | Just after it runs (success or failure)                  |
| `task_success`          | Task succeeded                                           |
| `task_failure`          | Task raised an exception                                 |
| `task_retry`            | Task will be retried                                     |
| `task_revoked`          | Task was revoked                                         |
| `task_unknown`          | Worker got a message for an unregistered task            |
| `worker_init`           | Worker is starting                                       |
| `worker_ready`          | Worker is ready to receive tasks                         |
| `worker_process_init`   | Each prefork child process starts (e.g. open connections)|
| `worker_shutdown`       | Worker is shutting down                                  |
| `beat_init`             | Beat is starting                                         |
| `setup_logging`         | Override logging configuration (Step 20.3)               |

Always accept `**kwargs` in receivers — Celery may add arguments in new versions.

---

## Step 26: Concurrency, Pools and Worker Tuning

### 26.1 Pools

| Pool       | Flag                          | Works on      | Best for                                         |
|------------|-------------------------------|---------------|--------------------------------------------------|
| `prefork`  | default                       | Linux, macOS  | CPU-heavy work; production default               |
| `solo`     | `-P solo`                     | Everywhere    | Windows development, debugging (1 task at a time) |
| `threads`  | `-P threads`                  | Everywhere    | I/O work (HTTP, email); Windows with concurrency |
| `gevent`   | `-P gevent` (`pip install gevent`) | Everywhere | Thousands of concurrent I/O tasks             |
| `eventlet` | `-P eventlet` (`pip install eventlet`) | Everywhere | Same idea as gevent (less maintained)     |

### 26.2 Concurrency

```powershell
celery -A config worker -l info -P threads -c 8          # 8 threads
celery -A config worker -l info -c 4                     # (Linux) 4 processes
celery -A config worker -l info --autoscale=10,2         # (prefork) between 2 and 10 processes
```

- Default concurrency = number of CPU cores.
- CPU-bound → processes ≈ number of cores.
- I/O-bound → many more (threads/gevent: 50–500).

### 26.3 Prevent memory leaks

```python
CELERY_WORKER_MAX_TASKS_PER_CHILD = 100        # replace a process after 100 tasks
CELERY_WORKER_MAX_MEMORY_PER_CHILD = 200_000   # KB (~200 MB): replace after the task that exceeds it
```

(prefork only.) Same as CLI `--max-tasks-per-child=100`, `--max-memory-per-child=200000`.

### 26.4 Database connections in workers

Django closes database connections at the end of each **request**; a worker has no requests. Celery's Django integration closes stale connections before and after each task, respecting `CONN_MAX_AGE`. Things to remember:

- Don't keep querysets or model instances in module-level globals between tasks.
- With many worker processes × `CONN_MAX_AGE`, you can hit the database's connection limit — count them.
- **SQLite** is fine for learning, but with several workers you may see `database is locked`. Use PostgreSQL in production.

---

## Step 27: Test Your Tasks

### 27.1 Test the function directly (fastest, most common)

A task is still a normal function when called directly:

`blog/tests.py`:

```python
from datetime import timedelta
from unittest import mock

from django.contrib.auth import get_user_model
from django.core import mail
from django.test import TestCase
from django.urls import reverse
from django.utils import timezone

from .models import Post
from .tasks import add, delete_old_drafts, send_post_published_email

User = get_user_model()


class TaskFunctionTests(TestCase):
    def test_add(self):
        self.assertEqual(add(2, 3), 5)            # runs inline, no worker, no broker

    def test_published_email(self):
        author = User.objects.create_user('ana', 'ana@example.com', 'pass12345!')
        post = Post.objects.create(title='Hi', slug='hi', content='x', author=author)

        result = send_post_published_email(post.pk)

        self.assertEqual(result, 'sent to ana@example.com')
        self.assertEqual(len(mail.outbox), 1)     # Django's test email backend collects mail
        self.assertIn('Hi', mail.outbox[0].subject)

    def test_delete_old_drafts(self):
        old = Post.objects.create(title='Old', slug='old', content='x')
        Post.objects.filter(pk=old.pk).update(updated_at=timezone.now() - timedelta(days=40))
        Post.objects.create(title='New', slug='new', content='x')

        deleted = delete_old_drafts(days=30)

        self.assertEqual(deleted, 1)
        self.assertFalse(Post.objects.filter(slug='old').exists())
```

> `add` has `time.sleep(5)` — in real projects keep sleeps out of tasks, or patch `time.sleep` in tests.

### 27.2 Run through Celery machinery locally — `.apply()`

```python
    def test_apply(self):
        result = add.apply(args=(2, 3))           # runs synchronously, returns EagerResult
        self.assertTrue(result.successful())
        self.assertEqual(result.get(), 5)
```

`.apply()` runs retries, `bind`, `self.request` etc. like a worker would — in the test process.

### 27.3 Check that a view **sends** a task (mock it)

```python
class PublishViewTests(TestCase):
    def setUp(self):
        self.editor = User.objects.create_superuser('ed', 'ed@example.com', 'pass12345!')
        self.post = Post.objects.create(title='P', slug='p', content='x', author=self.editor)
        self.client.force_login(self.editor)

    @mock.patch('blog.views.send_post_published_email.delay_on_commit')
    def test_publish_queues_email(self, mocked_delay):
        self.client.post(reverse('blog:post_publish', args=[self.post.pk]))
        mocked_delay.assert_called_once_with(self.post.pk)
```

Patch where the name is **used** (`blog.views.…`), not where it's defined.

### 27.4 `on_commit` callbacks in tests

`TestCase` wraps each test in a transaction that is **rolled back**, so `on_commit` callbacks never run. Use `captureOnCommitCallbacks`:

```python
    @mock.patch('accounts.views.send_welcome_email.delay')
    def test_signup_sends_welcome_email(self, mocked_delay):
        with self.captureOnCommitCallbacks(execute=True):
            self.client.post(reverse('signup'), {
                'username': 'new', 'email': 'new@example.com',
                'password1': 'Sup3r-Secret!', 'password2': 'Sup3r-Secret!',
            })
        mocked_delay.assert_called_once()
```

(Use the field names of your `SignUpForm`.)

### 27.5 Eager mode (run every `.delay()` inline)

Create a test settings file `config/settings_test.py`:

```python
from .settings import *  # noqa

CELERY_TASK_ALWAYS_EAGER = True        # .delay() runs immediately in the same process
CELERY_TASK_EAGER_PROPAGATES = True    # exceptions in tasks are raised in the test
CELERY_BROKER_URL = 'memory://'        # no Redis needed
CELERY_RESULT_BACKEND = 'cache+memory://'
```

```powershell
python manage.py test --settings=config.settings_test
```

Eager mode is convenient but **hides real problems** (serialization errors, routing, timing). Prefer 27.1–27.4 and test the integration manually or with a real worker.

### 27.6 Real worker in tests (pytest)

With `pytest` + `pytest-django`, Celery provides fixtures (`celery_app`, `celery_worker`) via `celery.contrib.pytest` that start an in-process worker. Useful for testing chains/chords end-to-end. See https://docs.celeryq.dev/en/stable/userguide/testing.html.

---

## Step 28: Security

1. **JSON only.** Keep `CELERY_ACCEPT_CONTENT = ['json']`. Never accept `pickle` — a malicious message could execute code on your workers.
2. **Protect the broker.** Redis/RabbitMQ must not be reachable from the internet. Use a password and private network:
   ```python
   CELERY_BROKER_URL = 'redis://:LongRandomPassword@10.0.0.5:6379/0'
   ```
   Use `rediss://` (TLS) across untrusted networks:
   ```python
   import ssl
   CELERY_BROKER_USE_SSL = {'ssl_cert_reqs': ssl.CERT_REQUIRED}
   CELERY_REDIS_BACKEND_USE_SSL = {'ssl_cert_reqs': ssl.CERT_REQUIRED}
   ```
3. **Secrets in env vars**, not in code or git.
4. **Don't put sensitive data in task arguments** (passwords, card numbers) — they're stored in the broker, the result backend, logs and Flower. Pass an id and load the data in the task.
5. **Protect Flower** with auth and keep it off the public internet.
6. **Check permissions before queuing** — the worker trusts whatever arguments it gets.
7. **Validate task-status endpoints** — don't let users read other users' results (Step 16.5).

---

## Step 29: Run Everything With Docker Compose

A clean way to run Django + Redis + worker + beat + Flower with one command (and a Linux worker using prefork, even on Windows).

### 29.1 `Dockerfile`

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
```

### 29.2 `docker-compose.yml`

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    environment: &env
      DJANGO_SETTINGS_MODULE: config.settings
      CELERY_BROKER_URL: redis://redis:6379/0
    depends_on:
      - redis

  worker:
    build: .
    command: celery -A config worker -l info --concurrency=4
    volumes:
      - .:/app
    environment: *env
    depends_on:
      - redis

  beat:
    build: .
    command: celery -A config beat -l info
    volumes:
      - .:/app
    environment: *env
    depends_on:
      - redis

  flower:
    build: .
    command: celery -A config flower --port=5555
    ports:
      - "5555:5555"
    environment: *env
    depends_on:
      - redis
      - worker
```

Notes:

- Inside Docker, the host name of Redis is the **service name** `redis`, not `localhost`.
- `&env` / `*env` is a YAML anchor — define the environment once, reuse it.
- Stop the standalone `redis` container from Step 2 first (port 6379 conflict): `docker stop redis`.
- With SQLite, all containers share `db.sqlite3` through the `.:/app` volume — OK for learning; use a `postgres` service in real projects.

### 29.3 Run it

```powershell
docker compose up --build
docker compose exec web python manage.py migrate
docker compose exec web python manage.py createsuperuser
docker compose logs -f worker        # follow worker logs
docker compose restart worker        # after changing task code
docker compose down                  # stop everything
```

---

## Step 30: Deploy to a Linux Server (systemd)

In production, workers and beat run as **services** that start at boot and restart on crash.

### 30.1 Worker service — `/etc/systemd/system/celery-worker.service`

```ini
[Unit]
Description=Celery worker for myproject
After=network.target redis-server.service

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/srv/myproject
EnvironmentFile=/srv/myproject/.env
ExecStart=/srv/myproject/venv/bin/celery -A config worker \
    --loglevel=info \
    --concurrency=4 \
    --hostname=worker1@%%h \
    --max-tasks-per-child=200
Restart=always
RestartSec=5
# Give running tasks time to finish on stop/restart (warm shutdown)
KillSignal=SIGTERM
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target
```

(`%%h` because systemd itself uses `%`.)

### 30.2 Beat service — `/etc/systemd/system/celery-beat.service`

```ini
[Unit]
Description=Celery beat for myproject
After=network.target redis-server.service

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/srv/myproject
EnvironmentFile=/srv/myproject/.env
ExecStart=/srv/myproject/venv/bin/celery -A config beat --loglevel=info
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

(With the default file scheduler add `--schedule=/var/lib/celery/celerybeat-schedule` and make sure `www-data` can write there.)

### 30.3 Enable and manage

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now celery-worker celery-beat
sudo systemctl status celery-worker
sudo journalctl -u celery-worker -f        # live logs
sudo systemctl restart celery-worker       # after every deploy!
```

### 30.4 Deploy order

1. Pull the new code, install requirements.
2. `python manage.py migrate`.
3. Restart **web** (gunicorn/uwsgi), **worker** and **beat**.

> When a new version renames a task or changes its arguments, old messages still in the queue use the old name/arguments. Keep the old task around (or accept both signatures) until the queue is drained.

Alternatives: **supervisor**, `celery multi start` (built-in multi-worker launcher), Docker/Kubernetes, or a PaaS "worker" process type (e.g. a `Procfile` line `worker: celery -A config worker -l info`).

---

## Step 31: Production Checklist

- [ ] Broker (Redis/RabbitMQ) with password, private network, persistence (Redis AOF/RDB) or RabbitMQ durable queues.
- [ ] `CELERY_ACCEPT_CONTENT = ['json']` — no pickle.
- [ ] Broker/result URLs and secrets come from environment variables.
- [ ] Workers run as services (systemd/supervisor/Docker) with automatic restart.
- [ ] **Exactly one** beat process.
- [ ] Workers are restarted on every deploy.
- [ ] Global `CELERY_TASK_TIME_LIMIT` / `CELERY_TASK_SOFT_TIME_LIMIT` set.
- [ ] Tasks are idempotent; `acks_late` + `reject_on_worker_lost` where losing a task is unacceptable.
- [ ] `visibility_timeout` (Redis) longer than the longest task / ETA.
- [ ] `CELERY_WORKER_PREFETCH_MULTIPLIER = 1` for long tasks.
- [ ] Retries with back-off only for temporary errors.
- [ ] Separate queues for slow vs urgent tasks, with workers consuming each.
- [ ] `ignore_result=True` for tasks whose result nobody reads; `CELERY_RESULT_EXPIRES` set.
- [ ] Tasks queued with `transaction.on_commit` / `delay_on_commit` when they depend on new rows.
- [ ] Only ids/simple data in arguments; no sensitive data.
- [ ] `CELERY_WORKER_MAX_TASKS_PER_CHILD` / `MAX_MEMORY_PER_CHILD` against leaks.
- [ ] PostgreSQL (not SQLite) with enough connections for all worker processes.
- [ ] Monitoring: Flower behind auth, error tracking (e.g. Sentry has a Celery integration), alerts on queue length and failures.
- [ ] Logs collected from worker and beat.

---

## Reference: Common Settings

| Setting (`settings.py`)                      | Default                       | Meaning                                              |
|----------------------------------------------|-------------------------------|------------------------------------------------------|
| `CELERY_BROKER_URL`                          | `amqp://guest@localhost//`    | Broker connection                                     |
| `CELERY_RESULT_BACKEND`                      | none                          | Where results go                                      |
| `CELERY_ACCEPT_CONTENT`                      | `['json']`                    | Allowed incoming serializers                          |
| `CELERY_TASK_SERIALIZER`                     | `'json'`                      | Serializer for task messages                          |
| `CELERY_RESULT_SERIALIZER`                   | `'json'`                      | Serializer for results                                |
| `CELERY_TIMEZONE`                            | `'UTC'`                       | Time zone for schedules/ETA                           |
| `CELERY_ENABLE_UTC`                          | `True`                        | Use UTC internally                                    |
| `CELERY_TASK_TRACK_STARTED`                  | `False`                       | Report STARTED state                                  |
| `CELERY_TASK_TIME_LIMIT`                     | none                          | Hard time limit (s)                                   |
| `CELERY_TASK_SOFT_TIME_LIMIT`                | none                          | Soft time limit (s)                                   |
| `CELERY_TASK_ACKS_LATE`                      | `False`                       | Ack after the task finishes                           |
| `CELERY_TASK_REJECT_ON_WORKER_LOST`          | `False`                       | Requeue when the worker process dies                  |
| `CELERY_TASK_IGNORE_RESULT`                  | `False`                       | Don't store results by default                        |
| `CELERY_TASK_ALWAYS_EAGER`                   | `False`                       | Run tasks inline (tests only)                         |
| `CELERY_TASK_EAGER_PROPAGATES`               | `False`                       | Raise exceptions in eager mode                        |
| `CELERY_TASK_DEFAULT_QUEUE`                  | `'celery'`                    | Queue for unrouted tasks                              |
| `CELERY_TASK_ROUTES`                         | none                          | Task → queue mapping                                  |
| `CELERY_TASK_DEFAULT_RATE_LIMIT`             | none                          | Default rate limit for all tasks                      |
| `CELERY_RESULT_EXPIRES`                      | 1 day                         | Delete results after this time                        |
| `CELERY_RESULT_EXTENDED`                     | `False`                       | Store name, args, kwargs, worker with results         |
| `CELERY_BROKER_TRANSPORT_OPTIONS`            | `{}`                          | e.g. `visibility_timeout`, priorities                 |
| `CELERY_BROKER_CONNECTION_RETRY_ON_STARTUP`  | `True` (warns if unset)       | Retry broker connection at start-up                   |
| `CELERY_WORKER_CONCURRENCY`                  | CPU count                     | Parallel tasks per worker                             |
| `CELERY_WORKER_PREFETCH_MULTIPLIER`          | `4`                           | Messages reserved per process                         |
| `CELERY_WORKER_MAX_TASKS_PER_CHILD`          | none                          | Recycle process after N tasks                         |
| `CELERY_WORKER_MAX_MEMORY_PER_CHILD`         | none                          | Recycle process above N KB                            |
| `CELERY_WORKER_HIJACK_ROOT_LOGGER`           | `True`                        | Celery replaces logging config                        |
| `CELERY_WORKER_SEND_TASK_EVENTS`             | `False`                       | Same as `-E`                                          |
| `CELERY_BEAT_SCHEDULE`                       | `{}`                          | Periodic tasks                                        |
| `CELERY_BEAT_SCHEDULER`                      | `celery.beat:PersistentScheduler` | Scheduler class                                   |

Full list: https://docs.celeryq.dev/en/stable/userguide/configuration.html

## Reference: Worker Command-Line Options

| Option                          | Meaning                                            |
|---------------------------------|----------------------------------------------------|
| `-A config`                     | App location                                       |
| `-l info`                       | Log level                                          |
| `-P solo/threads/prefork/gevent`| Pool                                               |
| `-c 4`                          | Concurrency                                        |
| `-Q a,b`                        | Queues to consume                                  |
| `-X a`                          | Exclude queues                                     |
| `-n name@%h`                    | Node name                                          |
| `-E`                            | Send task events                                   |
| `-B`                            | Embed beat (dev only)                              |
| `--autoscale=max,min`           | Auto-scale processes                               |
| `--max-tasks-per-child=N`       | Recycle processes                                  |
| `--max-memory-per-child=KB`     | Recycle on memory                                  |
| `--logfile=path`                | Log file                                           |
| `--pidfile=path`                | PID file                                           |
| `--statedb=path`                | Persist revoked ids                                |
| `--without-heartbeat` / `--without-gossip` / `--without-mingle` | Reduce chatter between workers |

## Reference: Celery vs Django's Built-in Tasks Framework

Django 6.0 added `django.tasks` — a standard API (`@task`, `.enqueue()`) for background work. It ships only with development backends (`ImmediateBackend`, `DummyBackend`); a real background worker requires a third-party backend. Celery is a complete, mature system with its own workers, scheduling (beat), retries, routing, canvas and monitoring. For a new small project, `django.tasks` + a database-backed worker can be enough; for anything needing schedules, workflows, multiple queues or high volume, Celery is the established choice.

---

## Troubleshooting

| Problem                                                                    | Cause / Fix                                                                                     |
|----------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| `Error 10061 connecting to localhost:6379` / `Connection refused`          | Redis isn't running → `docker start redis`, check `redis-cli ping`                              |
| Error mentions `amqp://guest@localhost:5672`                               | Celery didn't read your settings: missing `namespace='CELERY'`, setting name typo, or wrong `-A` |
| Tasks stay `PENDING` forever                                               | No worker running; worker listens to another queue (`-Q`); wrong broker URL; or task id typo (unknown ids are PENDING) |
| Windows: `ValueError: not enough values to unpack (expected 3, got 0)`     | Prefork on Windows → add `--pool=solo` or `-P threads`                                          |
| `Received unregistered task of type 'blog.tasks.x'`                        | Worker not restarted after adding the task; app missing from `INSTALLED_APPS`; file not named `tasks.py`; inconsistent imports |
| `kombu.exceptions.EncodeError: Object of type User is not JSON serializable` | You passed a model instance → pass `user.pk`                                                   |
| Task runs with **old code**                                                | Restart the worker (or use `watchfiles`)                                                         |
| `DoesNotExist` inside a task right after `.create()`                       | Task ran before the transaction committed → `transaction.on_commit` / `delay_on_commit`          |
| `NotImplementedError: No result backend is configured`                     | Set `CELERY_RESULT_BACKEND`                                                                       |
| `RuntimeError: Never call result.get() within a task!`                     | Use chains/chords instead of waiting inside a task                                               |
| Periodic task runs twice                                                   | Two beat processes, or worker with `-B` plus a separate beat                                     |
| Periodic tasks not running                                                 | Beat not running; task disabled in admin; wrong task name; invalid JSON args; worker not consuming the queue |
| Task with long countdown/ETA runs twice (Redis)                            | ETA beyond `visibility_timeout` → raise it or use beat                                           |
| Time limits ignored                                                        | Using `solo`/`threads` pool (Windows) — limits need prefork                                     |
| Emails from tasks don't appear in `runserver` output                       | They're printed in the **worker** terminal (console backend)                                    |
| `database is locked`                                                       | SQLite with concurrent workers → PostgreSQL                                                      |
| `ImproperlyConfigured: Requested setting ..., but settings are not configured` | `DJANGO_SETTINGS_MODULE` not set in `config/celery.py` before creating the app              |
| `AppRegistryNotReady` when the worker starts                               | Importing models at the top of `config/celery.py` — import models only in `tasks.py`            |
| Chord callback never runs                                                  | No/unsupported result backend (e.g. `rpc://`), or a member failed                               |
| Worker uses lots of memory over time                                       | Set `CELERY_WORKER_MAX_TASKS_PER_CHILD` / `MAX_MEMORY_PER_CHILD`                                 |

---

## Final Project Structure

```
myproject/
├── manage.py
├── db.sqlite3
├── requirements.txt              # celery[redis], django-celery-results, django-celery-beat, flower
├── celerybeat-schedule           # (only with the default file scheduler — gitignore it)
├── Dockerfile
├── docker-compose.yml
├── config/
│   ├── __init__.py               # imports celery_app
│   ├── celery.py                 # Celery app: settings, autodiscover, debug_task
│   ├── settings.py               # CELERY_* settings, INSTALLED_APPS += results/beat
│   ├── settings_test.py          # eager mode for tests
│   └── urls.py
├── accounts/
│   ├── tasks.py                  # send_welcome_email
│   └── views.py                  # SignUpView queues the welcome email on commit
└── blog/
    ├── apps.py                   # ready() imports celery_signals
    ├── celery_signals.py         # task_prerun/postrun/failure/retry receivers
    ├── tasks.py                  # add, mul, xsum, fetch_json, send_post_published_email,
    │                             # rebuild_search_index, delete_old_drafts, send_weekly_digest
    ├── tests.py                  # task + view tests
    ├── urls.py                   # reindex, reindex_start, task_status
    ├── views.py                  # post_publish queues email; progress views
    └── templates/blog/
        └── reindex.html          # progress bar polling task_status
```

Terminals during development:

| Terminal | Command                                           |
|----------|---------------------------------------------------|
| 1        | `python manage.py runserver`                      |
| 2        | `celery -A config worker -l info -P solo`         |
| 3        | `celery -A config beat -l info` (if you use periodic tasks) |
| 4        | `celery -A config flower` (optional)              |
| —        | Redis running (`docker start redis`)              |

---

## Quick Reference

| Task                                   | Code / Command                                                      |
|----------------------------------------|---------------------------------------------------------------------|
| Install                                | `pip install "celery[redis]"`                                       |
| Start Redis (Docker)                   | `docker run -d --name redis -p 6379:6379 redis:7`                   |
| Celery app                             | `config/celery.py` + import in `config/__init__.py`                 |
| Define a task                          | `@shared_task` in `<app>/tasks.py`                                  |
| Start worker (Windows)                 | `celery -A config worker -l info -P solo`                           |
| Start worker (Linux/macOS)             | `celery -A config worker -l info`                                   |
| Send a task                            | `my_task.delay(arg)`                                                |
| Send with options                      | `my_task.apply_async((arg,), countdown=10, queue='emails')`         |
| Send after DB commit                   | `my_task.delay_on_commit(arg)` / `transaction.on_commit(lambda: ...)` |
| Get result                             | `AsyncResult(task_id).state` / `.get(timeout=10)`                   |
| Access task instance                   | `@shared_task(bind=True)` → `self.request`, `self.retry()`          |
| Auto retry                             | `autoretry_for=(Error,), retry_backoff=True, retry_kwargs={'max_retries': 5}` |
| Progress                               | `self.update_state(state='PROGRESS', meta={...})`                   |
| Cancel                                 | `AsyncResult(id).revoke()`                                          |
| Chain / group / chord                  | `(a.s() \| b.s())()` / `group(...)()` / `chord(group)(callback.s())` |
| Periodic tasks                         | `CELERY_BEAT_SCHEDULE` + `celery -A config beat -l info`            |
| Schedules in admin                     | `django_celery_beat` + `DatabaseScheduler`                          |
| Results in admin                       | `django_celery_results` + `CELERY_RESULT_BACKEND = 'django-db'`     |
| Who is online                          | `celery -A config status` / `inspect active`                        |
| Dashboard                              | `celery -A config flower --port=5555`                               |
| Delete waiting tasks                   | `celery -A config purge`                                            |
| Tests                                  | Call the function directly / `.apply()` / mock `.delay`             |

---

## Next Steps

- **Sentry** Celery integration — automatic error reports with task context.
- **django-celery-email** — send all Django emails through Celery automatically.
- **celery-once** / Redis locks — prevent the same task from running twice at the same time.
- **Kombu** docs — the messaging library under Celery (exchanges, queues, transports).
- **RabbitMQ** — when you need the most reliable broker and advanced routing.
- Official docs:
  - First steps with Django: https://docs.celeryq.dev/en/stable/django/first-steps-with-django.html
  - Tasks: https://docs.celeryq.dev/en/stable/userguide/tasks.html
  - Calling tasks: https://docs.celeryq.dev/en/stable/userguide/calling.html
  - Canvas: https://docs.celeryq.dev/en/stable/userguide/canvas.html
  - Workers: https://docs.celeryq.dev/en/stable/userguide/workers.html
  - Periodic tasks: https://docs.celeryq.dev/en/stable/userguide/periodic-tasks.html
  - Routing: https://docs.celeryq.dev/en/stable/userguide/routing.html
  - Configuration: https://docs.celeryq.dev/en/stable/userguide/configuration.html
  - django-celery-results: https://django-celery-results.readthedocs.io/
  - django-celery-beat: https://django-celery-beat.readthedocs.io/
  - Flower: https://flower.readthedocs.io/
