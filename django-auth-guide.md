# Django Authentication (`django.contrib.auth`): Step-by-Step Guide

`django.contrib.auth` is Django's built-in **authentication and authorization** system. It gives you:

| Part                  | What it does                                                                 |
|-----------------------|------------------------------------------------------------------------------|
| **Users**             | A `User` model to store accounts (username, password, email, flags…)         |
| **Passwords**         | Secure hashing, password validation rules, change and reset flows            |
| **Login / logout**    | Functions, forms, views and templates to sign users in and out               |
| **Sessions link**     | Remembers who is logged in across requests (`request.user`)                  |
| **Permissions**       | Yes/no flags like "can add post", per user or per group                      |
| **Groups**            | Bundles of permissions you assign to many users at once                      |
| **Access control**    | Decorators, mixins and middleware that protect views                         |
| **Backends**          | Pluggable ways to check credentials (username, email, LDAP, …)               |
| **Signals**           | Hooks fired on login, logout and failed login                                |

Two words you will see a lot:

- **Authentication** = *who are you?* (checking username + password, logging in)
- **Authorization** = *what are you allowed to do?* (permissions, groups, `is_staff`)

This guide continues the `blog` project from [django-setup-guide.md](django-setup-guide.md) and [django-admin-guide.md](django-admin-guide.md). Every step builds on the previous one. Commands are for **Windows** (PowerShell); they are the same on macOS/Linux except where noted.

---

## Step 0: How Authentication Works (Big Picture)

Before writing code, understand what happens on every request:

```
Browser request (with "sessionid" cookie)
        │
        ▼
SessionMiddleware          → loads the session data from the database using the cookie
        │
        ▼
AuthenticationMiddleware   → sets request.user (lazily, only when you first use it):
        │                      1. reads _auth_user_id, _auth_user_backend, _auth_user_hash from the session
        │                      2. asks that backend to load the user by id
        │                      3. compares _auth_user_hash with user.get_session_auth_hash()
        │                         (if the password changed, the hash differs → user is logged out)
        │                      4. if anything fails → request.user = AnonymousUser()
        ▼
Your view                  → checks request.user.is_authenticated, permissions, etc.
```

**When a user logs in** (`login(request, user)`):

1. The session key is rotated (`cycle_key()`) to prevent *session fixation* attacks.
2. `_auth_user_id`, `_auth_user_backend` and `_auth_user_hash` are saved in the session.
3. `request.user` is set to the user.
4. The `user_logged_in` signal fires → Django updates `user.last_login`.

**When a user logs out** (`logout(request)`):

1. The `user_logged_out` signal fires.
2. The whole session is flushed (deleted) and a new empty one is created.
3. `request.user` becomes `AnonymousUser`.

Keep this picture in mind — every step below is a piece of it.

---

## Step 1: Check That Auth Is Enabled

A project made with `startproject` already has auth switched on. Open `config/settings.py` and confirm each part.

**1. `INSTALLED_APPS`**

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',          # the auth framework (User, Group, Permission)
    'django.contrib.contenttypes',  # REQUIRED by auth — permissions point to content types
    'django.contrib.sessions',      # REQUIRED to stay logged in between requests
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog',
]
```

- `django.contrib.auth` provides the models, views, forms, backends and management commands.
- `django.contrib.contenttypes` records every model in your project; each `Permission` is linked to one content type (e.g. "blog | post").

**2. `MIDDLEWARE`** — order matters:

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',      # 1st: loads the session
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',                 # protects login/logout forms
    'django.contrib.auth.middleware.AuthenticationMiddleware',   # 2nd: sets request.user (needs session)
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

`AuthenticationMiddleware` **must come after** `SessionMiddleware`, because it reads the user id from the session.

**3. `TEMPLATES` → `context_processors`**

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',   # adds {{ user }} and {{ perms }} to every template
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

The `auth` context processor puts two variables in every template:

| Variable   | What it is                                                      |
|------------|-----------------------------------------------------------------|
| `user`     | The current user (`User` instance or `AnonymousUser`)           |
| `perms`    | A helper to check permissions: `{% if perms.blog.add_post %}`   |

**4. `AUTH_PASSWORD_VALIDATORS`** — rules for new passwords (covered in Step 8):

```python
AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator'},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]
```

---

## Step 2: Understand the Tables Auth Creates

When you run `python manage.py migrate`, auth creates these tables:

| Table                         | Holds                                                     |
|-------------------------------|-----------------------------------------------------------|
| `auth_user`                   | User accounts (only if you use the **default** User model) |
| `auth_group`                  | Groups (e.g. "Editors")                                   |
| `auth_permission`             | Every permission (e.g. `blog.add_post`)                   |
| `auth_group_permissions`      | Which permissions each group has (many-to-many)           |
| `auth_user_groups`            | Which groups each user belongs to (many-to-many)          |
| `auth_user_user_permissions`  | Permissions given directly to a user (many-to-many)       |
| `django_content_type`         | One row per model (from `contenttypes`)                   |
| `django_session`              | Session data (from `sessions`), incl. the logged-in user id |

**Relationships:**

```
User ──< user_groups >── Group ──< group_permissions >── Permission ── ContentType
  │                                                          ▲
  └─────────────< user_user_permissions >────────────────────┘
```

> Permissions are **not** created by migrations directly. After `migrate` finishes, a `post_migrate` signal creates the four default permissions (`add`, `change`, `delete`, `view`) for every model. This matters in Step 26.

---

## Step 3: Decide — Default or Custom User Model

This is the **most important decision** in auth, and you must make it **before the first `migrate`** of a real project.

| Option                         | When to use                                                                 |
|--------------------------------|-----------------------------------------------------------------------------|
| Default `User`                 | Quick experiments only                                                      |
| `AbstractUser` subclass        | **Recommended for every new project.** Same as default, but you can add fields later |
| `AbstractBaseUser` subclass    | You need a completely different user (e.g. email instead of username)        |

> **Why always use a custom model?** Switching from the default `User` to a custom one *after* you have data and migrations is very hard. Starting with a custom model that's identical to the default costs nothing and lets you add fields (phone, avatar, bio) later with a simple migration.

We'll put everything account-related (custom user, sign-up, profile, templates) in a new app called `accounts`.

### 3.1 Create the `accounts` app

```bash
python manage.py startapp accounts
```

Add it to `INSTALLED_APPS` in `config/settings.py`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'accounts',
    'blog',
]
```

### 3.2 Create the custom user model (`AbstractUser`)

Edit `accounts/models.py`:

```python
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    # AbstractUser already has: username, first_name, last_name, email, password,
    # is_staff, is_active, is_superuser, last_login, date_joined, groups, user_permissions

    email = models.EmailField('email address', unique=True)  # make email required + unique
    bio = models.TextField(blank=True)

    def __str__(self):
        return self.username
```

### 3.3 Tell Django to use it

Add this line anywhere in `config/settings.py`:

```python
AUTH_USER_MODEL = 'accounts.User'   # format: '<app_label>.<ModelName>'
```

### 3.4 Reset the database (learning projects only)

Your `blog` project already ran `migrate` with the default user, so the database must be rebuilt:

1. Stop the server.
2. Delete `db.sqlite3`.
3. Delete every file in `blog/migrations/` **except** `__init__.py`.
4. Run:

```bash
python manage.py makemigrations accounts blog
python manage.py migrate
python manage.py createsuperuser
```

Now the users table is `accounts_user` instead of `auth_user`.

> In a real project with production data you can't just delete the database. Migrating to a custom user mid-project needs careful manual migrations — which is exactly why you should do this step on day one.

### 3.5 Register the custom user in the admin

The default `User` is auto-registered in the admin; your custom one isn't. Edit `accounts/admin.py`:

```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin

from .models import User


@admin.register(User)
class CustomUserAdmin(UserAdmin):
    # Edit page: the default sections + our extra field
    fieldsets = UserAdmin.fieldsets + (
        ('Profile', {'fields': ('bio',)}),
    )
    # "Add user" page: the default fields + email and bio
    add_fieldsets = UserAdmin.add_fieldsets + (
        ('Profile', {'fields': ('email', 'bio')}),
    )
    list_display = ('username', 'email', 'first_name', 'last_name', 'is_staff', 'is_active')
```

`UserAdmin` already gives you the password-change link, the groups/permissions widgets and the filters.

### 3.6 Golden rules for referencing the user model

Never import `User` directly from `django.contrib.auth.models` in your code — it would point to the wrong model.

| Where                               | Use                                    |
|-------------------------------------|----------------------------------------|
| In a model's `ForeignKey`/`OneToOne`| `settings.AUTH_USER_MODEL` (a string)  |
| Everywhere else (views, forms, tests, shell) | `get_user_model()` (returns the class) |

```python
# models.py
from django.conf import settings
from django.db import models

class Post(models.Model):
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)

# views.py / forms.py / tests.py
from django.contrib.auth import get_user_model

User = get_user_model()
```

Your `blog/models.py` from the admin guide already uses `settings.AUTH_USER_MODEL`, so it works with the custom model.

### 3.7 (Alternative) Email instead of username — `AbstractBaseUser`

Skip this unless you want users to log in with **email only** and have **no username field**. This guide continues with the `AbstractUser` model above. (Step 29 shows an easier way to allow email login while keeping usernames.)

```python
# accounts/models.py  (ALTERNATIVE version)
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models
from django.utils import timezone


class UserManager(BaseUserManager):
    use_in_migrations = True

    def _create_user(self, email, password, **extra_fields):
        if not email:
            raise ValueError('The email must be set')
        email = self.normalize_email(email)       # lowercases the domain part
        user = self.model(email=email, **extra_fields)
        user.set_password(password)               # hashes the password
        user.save(using=self._db)
        return user

    def create_user(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', False)
        extra_fields.setdefault('is_superuser', False)
        return self._create_user(email, password, **extra_fields)

    def create_superuser(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        if extra_fields.get('is_staff') is not True:
            raise ValueError('Superuser must have is_staff=True.')
        if extra_fields.get('is_superuser') is not True:
            raise ValueError('Superuser must have is_superuser=True.')
        return self._create_user(email, password, **extra_fields)


class User(AbstractBaseUser, PermissionsMixin):
    # AbstractBaseUser gives: password, last_login, set_password(), check_password(), ...
    # PermissionsMixin gives: is_superuser, groups, user_permissions, has_perm(), ...
    email = models.EmailField(unique=True)
    first_name = models.CharField(max_length=150, blank=True)
    last_name = models.CharField(max_length=150, blank=True)
    is_staff = models.BooleanField(default=False)
    is_active = models.BooleanField(default=True)
    date_joined = models.DateTimeField(default=timezone.now)

    objects = UserManager()

    USERNAME_FIELD = 'email'     # the field used to log in (must be unique)
    EMAIL_FIELD = 'email'        # used by password reset
    REQUIRED_FIELDS = []         # extra fields createsuperuser asks for (never include USERNAME_FIELD or password)

    def __str__(self):
        return self.email

    def get_full_name(self):
        return f'{self.first_name} {self.last_name}'.strip()

    def get_short_name(self):
        return self.first_name
```

With this model, `UserAdmin` must be adjusted because it expects `username`:

```python
# accounts/admin.py  (ALTERNATIVE version)
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from django.contrib.auth.forms import UserChangeForm, UserCreationForm

from .models import User


class CustomUserCreationForm(UserCreationForm):
    class Meta:
        model = User
        fields = ('email',)


class CustomUserChangeForm(UserChangeForm):
    class Meta:
        model = User
        fields = '__all__'


@admin.register(User)
class CustomUserAdmin(UserAdmin):
    add_form = CustomUserCreationForm
    form = CustomUserChangeForm
    ordering = ('email',)
    list_display = ('email', 'first_name', 'last_name', 'is_staff')
    search_fields = ('email', 'first_name', 'last_name')
    fieldsets = (
        (None, {'fields': ('email', 'password')}),
        ('Personal info', {'fields': ('first_name', 'last_name')}),
        ('Permissions', {'fields': ('is_active', 'is_staff', 'is_superuser', 'groups', 'user_permissions')}),
        ('Important dates', {'fields': ('last_login', 'date_joined')}),
    )
    add_fieldsets = (
        (None, {'classes': ('wide',), 'fields': ('email', 'password1', 'password2')}),
    )
```

**Requirements for any `AbstractBaseUser` model:**

- `USERNAME_FIELD` must point to a **unique** field.
- `REQUIRED_FIELDS` lists fields `createsuperuser` prompts for.
- `is_active` attribute (defaults to `True` on `AbstractBaseUser`).
- A manager with `create_user()` and `create_superuser()`.
- Add `PermissionsMixin` if you want Django's groups/permissions (and the admin).

---

## Step 4: Know the User Model Inside Out

Everything below applies to the default `User` and to any `AbstractUser` subclass.

### 4.1 Fields

| Field              | Type                 | Details                                                                 |
|--------------------|----------------------|-------------------------------------------------------------------------|
| `id`               | Auto primary key     |                                                                         |
| `username`         | `CharField(150)`     | Required, unique. Letters, digits and `@ . + - _` only                  |
| `first_name`       | `CharField(150)`     | Optional                                                                |
| `last_name`        | `CharField(150)`     | Optional                                                                |
| `email`            | `EmailField`         | Optional by default (we made it required + unique)                      |
| `password`         | `CharField(128)`     | **A hash, never the real password** (see Step 7)                        |
| `groups`           | `ManyToManyField(Group)` | Groups the user belongs to                                          |
| `user_permissions` | `ManyToManyField(Permission)` | Permissions given directly to this user                        |
| `is_staff`         | `BooleanField`       | Can log in to the **admin site**                                        |
| `is_active`        | `BooleanField`       | Account enabled. Set `False` instead of deleting users                  |
| `is_superuser`     | `BooleanField`       | Has **all** permissions without being given them                        |
| `last_login`       | `DateTimeField`      | Updated on each login; `NULL` if never logged in                        |
| `date_joined`      | `DateTimeField`      | Set when the account is created                                         |

### 4.2 Attributes

| Attribute            | Value on a real `User`        | Value on `AnonymousUser` |
|----------------------|-------------------------------|--------------------------|
| `is_authenticated`   | Always `True`                 | `False`                  |
| `is_anonymous`       | Always `False`                | `True`                   |
| `USERNAME_FIELD`     | `'username'`                  | —                        |
| `EMAIL_FIELD`        | `'email'`                     | —                        |
| `REQUIRED_FIELDS`    | `['email']`                   | —                        |

> `is_authenticated` is a **property**, not a method. Write `user.is_authenticated`, not `user.is_authenticated()`. It doesn't mean "active" — check `is_active` separately if needed.

### 4.3 Methods

| Method                                 | What it does                                                          |
|----------------------------------------|-----------------------------------------------------------------------|
| `get_username()`                       | Returns the value of `USERNAME_FIELD` (use this instead of `.username` in reusable code) |
| `get_full_name()`                      | `first_name + ' ' + last_name`                                        |
| `get_short_name()`                     | `first_name`                                                          |
| `set_password(raw)`                    | Hashes and stores the password. **Does not save** — call `save()`     |
| `check_password(raw)`                  | `True` if `raw` matches the stored hash                               |
| `set_unusable_password()`              | Marks the user as having no password (e.g. social-login-only users)   |
| `has_usable_password()`                | `False` after `set_unusable_password()`                               |
| `get_user_permissions()`               | Set of permission strings given directly to the user                  |
| `get_group_permissions()`              | Set of permission strings from the user's groups                      |
| `get_all_permissions()`                | Both sets combined                                                    |
| `has_perm('app.codename')`             | `True` if the user has that permission                                |
| `has_perms(['app.a', 'app.b'])`        | `True` only if the user has **all** of them                           |
| `has_module_perms('app_label')`        | `True` if the user has **any** permission in that app                 |
| `email_user(subject, message, from_email=None)` | Sends an email to `user.email`                               |
| `get_session_auth_hash()`              | HMAC of the password hash, stored in the session (see Step 0)         |

**Permission rules to remember:**

- An **inactive** user (`is_active=False`) has **no** permissions — `has_perm()` always returns `False`.
- An **active superuser** has **every** permission — `has_perm()` always returns `True`, even for permissions that don't exist.

### 4.4 Manager methods (`User.objects`)

| Method                                            | What it does                                               |
|---------------------------------------------------|------------------------------------------------------------|
| `create_user(username, email=None, password=None, **extra)` | Creates a normal user with a **hashed** password |
| `create_superuser(username, email=None, password=None, **extra)` | Same, with `is_staff=True`, `is_superuser=True` |
| `with_perm('app.codename', is_active=True, include_superusers=True)` | Users who have that permission         |
| `get_by_natural_key(username)`                    | Get a user by `USERNAME_FIELD`                             |
| `normalize_email(email)`                          | Lowercases the domain part of an email                     |

> **Never** create users with `User.objects.create(username='x', password='secret')` — the password would be stored as plain text and the user couldn't log in. Always use `create_user()` or call `set_password()`.

### 4.5 `AnonymousUser`

When nobody is logged in, `request.user` is an `AnonymousUser` object. It looks like a `User` but:

| Attribute / method                       | Value                         |
|------------------------------------------|-------------------------------|
| `id`, `pk`                               | `None`                        |
| `username`                               | `''`                          |
| `is_staff`, `is_superuser`, `is_active`  | `False`                       |
| `is_authenticated` / `is_anonymous`      | `False` / `True`              |
| `groups`, `user_permissions`             | Always empty                  |
| `has_perm()`                             | `False` (unless a custom backend allows anonymous perms) |
| `save()`, `delete()`, `set_password()`   | Raise `NotImplementedError`   |

This is why you can always write `request.user.is_authenticated` without first checking for `None`.

---

## Step 5: Work With Users in the Shell

Open the Django shell:

```bash
python manage.py shell
```

**Create users:**

```python
from django.contrib.auth import get_user_model
User = get_user_model()

alice = User.objects.create_user(username='alice', email='alice@example.com', password='S3cret-pass!')
bob = User.objects.create_user('bob', 'bob@example.com', 'An0ther-pass!', first_name='Bob')
admin = User.objects.create_superuser('boss', 'boss@example.com', 'Sup3r-pass!')
```

**Look at the stored password (it's a hash):**

```python
alice.password
# 'pbkdf2_sha256$1000000$Xb2...$k3J...='
```

**Check and change passwords:**

```python
alice.check_password('S3cret-pass!')   # True
alice.check_password('wrong')          # False

alice.set_password('N3w-pass-123!')
alice.save()                           # set_password() doesn't save by itself!
```

**Update fields:**

```python
alice.first_name = 'Alice'
alice.last_name = 'Smith'
alice.save()
alice.get_full_name()                  # 'Alice Smith'
```

**Authenticate (check credentials without a request):**

```python
from django.contrib.auth import authenticate

authenticate(username='alice', password='N3w-pass-123!')   # <User: alice>
authenticate(username='alice', password='wrong')           # None
```

**Deactivate instead of delete:**

```python
bob.is_active = False
bob.save()
authenticate(username='bob', password='An0ther-pass!')     # None — inactive users can't log in
```

**Query users:**

```python
User.objects.filter(is_staff=True)
User.objects.filter(email__iexact='ALICE@example.com')
User.objects.filter(last_login__isnull=True)    # never logged in
User.objects.with_perm('blog.add_post')         # users who can add posts
```

**Groups and permissions (more in Steps 23–26):**

```python
from django.contrib.auth.models import Group, Permission

editors, created = Group.objects.get_or_create(name='Editors')
perm = Permission.objects.get(codename='add_post', content_type__app_label='blog')
editors.permissions.add(perm)
alice.groups.add(editors)

alice = User.objects.get(pk=alice.pk)   # re-fetch (permissions are cached — see Step 23)
alice.has_perm('blog.add_post')         # True
```

Type `exit()` to leave the shell.

---

## Step 6: Use the Auth Management Commands

| Command                                         | What it does                                       |
|-------------------------------------------------|----------------------------------------------------|
| `python manage.py createsuperuser`              | Interactively creates a superuser                  |
| `python manage.py changepassword <username>`    | Interactively sets a new password for a user       |
| `python manage.py migrate`                      | Creates auth tables and default permissions        |

**`createsuperuser` options:**

```bash
python manage.py createsuperuser --username boss --email boss@example.com
```

**Non-interactive (for scripts/CI/Docker)** — set the password with an environment variable:

```powershell
# PowerShell
$env:DJANGO_SUPERUSER_PASSWORD = "Sup3r-pass!"
python manage.py createsuperuser --noinput --username boss --email boss@example.com
```

```bash
# macOS/Linux
DJANGO_SUPERUSER_PASSWORD='Sup3r-pass!' python manage.py createsuperuser --noinput --username boss --email boss@example.com
```

Every field in `REQUIRED_FIELDS` can also come from `DJANGO_SUPERUSER_<FIELD>` (e.g. `DJANGO_SUPERUSER_EMAIL`).

> `createsuperuser` runs your password validators and warns if the password is weak. You can type `y` to bypass them — only do this in development.

---

## Step 7: Understand How Passwords Are Stored

Django **never** stores the real password. It stores a one-way hash in this format:

```
<algorithm>$<iterations>$<salt>$<hash>
pbkdf2_sha256$1000000$Xb2pQ8...$k3J9vT...=
```

| Part         | Meaning                                                                       |
|--------------|-------------------------------------------------------------------------------|
| `algorithm`  | Which hasher was used (`pbkdf2_sha256`, `argon2`, `bcrypt_sha256`, `scrypt`)  |
| `iterations` | How many times the hash was repeated (makes brute force slow)                 |
| `salt`       | Random value per password, so identical passwords get different hashes       |
| `hash`       | The final result                                                              |

A password starting with `!` means **unusable** (from `set_unusable_password()`).

### 7.1 Choosing hashers — `PASSWORD_HASHERS`

The default (you don't need to write this — it's what Django uses already):

```python
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',       # used for NEW passwords
    'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',   # the rest can still VERIFY old hashes
    'django.contrib.auth.hashers.Argon2PasswordHasher',
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
    'django.contrib.auth.hashers.ScryptPasswordHasher',
]
```

- The **first** hasher is used to hash new passwords.
- All others are used only to **check** existing hashes.
- When a user logs in with a hash made by an older hasher (or fewer iterations), Django **automatically re-hashes** it with the first one.

**Switch to Argon2 (recommended by security experts):**

```bash
pip install "django[argon2]"
```

```python
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.Argon2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
    'django.contrib.auth.hashers.ScryptPasswordHasher',
]
```

(For bcrypt: `pip install "django[bcrypt]"`.)

### 7.2 Hashing functions you can call yourself

```python
from django.contrib.auth.hashers import make_password, check_password, is_password_usable

hashed = make_password('my-password')         # 'pbkdf2_sha256$1000000$...'
check_password('my-password', hashed)         # True
is_password_usable(hashed)                    # True
make_password(None)                           # an unusable password ('!...')
```

You rarely need these — `user.set_password()` and `user.check_password()` use them for you.

### 7.3 Faster tests

Hashing is deliberately slow. In a test-only settings file you can use a fast (insecure) hasher:

```python
# config/settings_test.py — NEVER use in production
from .settings import *

PASSWORD_HASHERS = ['django.contrib.auth.hashers.MD5PasswordHasher']
```

Run tests with `python manage.py test --settings=config.settings_test`.

---

## Step 8: Configure Password Validation

Validators stop users choosing weak passwords. They run in:

- `UserCreationForm`, `SetPasswordForm`, `PasswordChangeForm` (sign-up, reset, change)
- `createsuperuser` and `changepassword` commands

They do **not** run in `create_user()` or `set_password()` — those trust whatever you give them.

### 8.1 Built-in validators

| Validator                           | Rejects                                                 | Options                                      |
|-------------------------------------|---------------------------------------------------------|----------------------------------------------|
| `UserAttributeSimilarityValidator`  | Passwords too similar to username, names or email       | `user_attributes`, `max_similarity` (0.7)    |
| `MinimumLengthValidator`            | Passwords shorter than N characters                     | `min_length` (8)                             |
| `CommonPasswordValidator`           | ~20,000 common passwords (`password123`, `qwerty`…)     | `password_list_path`                         |
| `NumericPasswordValidator`          | Passwords made only of digits                           | —                                            |

**Customize with `OPTIONS`:**

```python
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
        'OPTIONS': {'user_attributes': ('username', 'email', 'first_name', 'last_name'), 'max_similarity': 0.7},
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {'min_length': 10},
    },
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
    {'NAME': 'accounts.validators.SymbolValidator'},   # our custom one, below
]
```

### 8.2 Write a custom validator

Create `accounts/validators.py`:

```python
import re

from django.core.exceptions import ValidationError


class SymbolValidator:
    """Require at least one symbol such as !@#$%."""

    def __init__(self, symbols='!@#$%^&*()-_=+[]{};:,.<>?/'):
        self.symbols = symbols

    def validate(self, password, user=None):
        if not any(char in self.symbols for char in password):
            raise ValidationError(
                'Your password must contain at least one symbol: %(symbols)s',
                code='password_no_symbol',
                params={'symbols': self.symbols},
            )

    def get_help_text(self):
        return f'Your password must contain at least one symbol: {self.symbols}'
```

A validator needs two methods:

- `validate(password, user=None)` — raise `ValidationError` if invalid, return `None` if fine.
- `get_help_text()` — text shown under the password field in forms.

### 8.3 Run validators in your own code

```python
from django.contrib.auth.password_validation import (
    validate_password,
    password_validators_help_texts,
    password_changed,
)
from django.core.exceptions import ValidationError

try:
    validate_password('abc', user=alice)
except ValidationError as e:
    print(e.messages)   # ['This password is too short...', 'This password is too common.', ...]

password_validators_help_texts()   # list of all help texts
password_changed('new-password', user=alice)   # tell validators a password was changed (for history-type validators)
```

---

## Step 9: Add the Auth Settings

Add these to `config/settings.py`:

```python
# --- Authentication ---
AUTH_USER_MODEL = 'accounts.User'           # from Step 3

LOGIN_URL = 'login'                         # where @login_required sends anonymous users (URL name or path)
LOGIN_REDIRECT_URL = 'blog:post_list'       # where to go after login if there's no ?next=
LOGOUT_REDIRECT_URL = 'blog:post_list'      # where to go after logout

PASSWORD_RESET_TIMEOUT = 60 * 60 * 24       # reset links expire after 1 day (default: 3 days)

# --- Email (needed for password reset) ---
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'   # prints emails in the terminal
DEFAULT_FROM_EMAIL = 'My Blog <noreply@myblog.local>'
```

**All auth-related settings:**

| Setting                    | Default                                      | Purpose                                              |
|----------------------------|----------------------------------------------|------------------------------------------------------|
| `AUTH_USER_MODEL`          | `'auth.User'`                                | Which model represents users                         |
| `AUTHENTICATION_BACKENDS`  | `['django.contrib.auth.backends.ModelBackend']` | How credentials are checked (Step 29)             |
| `LOGIN_URL`                | `'/accounts/login/'`                         | Login page for protected views                       |
| `LOGIN_REDIRECT_URL`       | `'/accounts/profile/'`                       | After login with no `next`                           |
| `LOGOUT_REDIRECT_URL`      | `None` (shows `logged_out.html`)             | After logout                                         |
| `PASSWORD_RESET_TIMEOUT`   | `259200` (3 days, in seconds)                | How long reset links work                            |
| `PASSWORD_HASHERS`         | PBKDF2 first                                 | Step 7                                               |
| `AUTH_PASSWORD_VALIDATORS` | 4 validators (in new projects)               | Step 8                                               |
| `SESSION_COOKIE_AGE`       | `1209600` (2 weeks)                          | How long a login lasts                               |
| `SESSION_EXPIRE_AT_BROWSER_CLOSE` | `False`                               | Log out when the browser closes                      |
| `SESSION_COOKIE_SECURE`    | `False`                                      | Send session cookie only over HTTPS (set `True` in production) |
| `CSRF_COOKIE_SECURE`       | `False`                                      | Same for the CSRF cookie                             |

> The default `LOGIN_REDIRECT_URL` is `/accounts/profile/`, which doesn't exist unless you create it. That's why you almost always set it.

**Production email (SMTP) example** — use instead of the console backend when going live:

```python
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = 'you@gmail.com'
EMAIL_HOST_PASSWORD = os.environ['EMAIL_HOST_PASSWORD']   # never hard-code secrets (needs `import os`)
```

---

## Step 10: Set Up a Project-Level Templates Folder

Auth views look for templates in a folder called `registration/`. Put them in a **project-level** `templates/` folder.

> **Why not inside an app?** `django.contrib.admin` ships its own `registration/password_reset_form.html`, `registration/logged_out.html`, etc. App template folders are searched in `INSTALLED_APPS` order, and `admin` is listed first — so the admin's versions would win over yours. Folders listed in `DIRS` are searched **before** any app folder, so your templates always win.

Create the folders next to `manage.py`:

```
myproject/
├── manage.py
├── config/
├── accounts/
├── blog/
└── templates/
    └── registration/
```

Tell Django about it in `config/settings.py`:

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],     # ← add this
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

---

## Step 11: Add the Built-in Auth URLs

Django ships ready-made views for login, logout, password change and password reset. Include them in `config/urls.py`:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('django.contrib.auth.urls')),   # login, logout, password change/reset
    path('accounts/', include('accounts.urls')),              # our own: signup, profile (Step 17)
    path('', include('blog.urls')),
]
```

Create an empty `accounts/urls.py` for now (we'll fill it in Step 17):

```python
from django.urls import path

from . import views

app_name = 'accounts'

urlpatterns = []
```

**What `django.contrib.auth.urls` gives you:**

| URL                                   | URL name                    | View                          | Template                                         |
|---------------------------------------|-----------------------------|-------------------------------|--------------------------------------------------|
| `accounts/login/`                     | `login`                     | `LoginView`                   | `registration/login.html`                        |
| `accounts/logout/`                    | `logout`                    | `LogoutView`                  | `registration/logged_out.html`                   |
| `accounts/password_change/`           | `password_change`           | `PasswordChangeView`          | `registration/password_change_form.html`         |
| `accounts/password_change/done/`      | `password_change_done`      | `PasswordChangeDoneView`      | `registration/password_change_done.html`         |
| `accounts/password_reset/`            | `password_reset`            | `PasswordResetView`           | `registration/password_reset_form.html` (+ email templates) |
| `accounts/password_reset/done/`       | `password_reset_done`       | `PasswordResetDoneView`       | `registration/password_reset_done.html`          |
| `accounts/reset/<uidb64>/<token>/`    | `password_reset_confirm`    | `PasswordResetConfirmView`    | `registration/password_reset_confirm.html`       |
| `accounts/reset/done/`                | `password_reset_complete`   | `PasswordResetCompleteView`   | `registration/password_reset_complete.html`      |

> These URL names have **no namespace**: use `{% url 'login' %}`, not `{% url 'accounts:login' %}`.

You can also wire the views individually if you want different paths or options:

```python
from django.contrib.auth import views as auth_views

urlpatterns = [
    path('login/', auth_views.LoginView.as_view(template_name='registration/login.html',
                                                redirect_authenticated_user=True), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
    # ... and so on for the others
]
```

---

## Step 12: Build the Login Page

### 12.1 Template

Create `templates/registration/login.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Log in{% endblock %}

{% block content %}
    <h2>Log in</h2>

    {% if next %}
        {% if user.is_authenticated %}
            <p>Your account doesn't have access to this page. Log in with an account that does.</p>
        {% else %}
            <p>Please log in to see this page.</p>
        {% endif %}
    {% endif %}

    <form method="post" action="{% url 'login' %}">
        {% csrf_token %}

        {% if form.non_field_errors %}
            <div class="error">{{ form.non_field_errors }}</div>
        {% endif %}

        <p>
            {{ form.username.label_tag }}
            {{ form.username }}
            {{ form.username.errors }}
        </p>
        <p>
            {{ form.password.label_tag }}
            {{ form.password }}
            {{ form.password.errors }}
        </p>

        <input type="hidden" name="next" value="{{ next }}">
        <button type="submit">Log in</button>
    </form>

    <p><a href="{% url 'password_reset' %}">Forgot your password?</a></p>
    <p>No account? <a href="{% url 'accounts:signup' %}">Sign up</a></p>
{% endblock %}
```

> The sign-up link uses a URL we create in Step 17. Until then, remove that line or you'll get a `NoReverseMatch` error.

### 12.2 What each part does

| Part                            | Purpose                                                                     |
|---------------------------------|-----------------------------------------------------------------------------|
| `method="post"`                 | Credentials must never go in the URL                                        |
| `{% csrf_token %}`              | Required on every POST form — protects against cross-site request forgery   |
| `form`                          | An `AuthenticationForm` with `username` and `password` fields               |
| `form.non_field_errors`         | Shows "Please enter a correct username and password" or "This account is inactive" |
| `next`                          | The page the user was trying to reach; `LoginView` redirects there after login |

### 12.3 Context `LoginView` gives the template

| Variable        | Value                                                    |
|-----------------|----------------------------------------------------------|
| `form`          | The `AuthenticationForm`                                 |
| `next`          | The URL to redirect to after login                       |
| `site`, `site_name` | Current site/domain                                  |

### 12.4 How `LoginView` decides where to go after login

1. The `next` value from POST or GET — **only if it's a safe URL on your own host** (Django blocks open redirects to other sites).
2. Otherwise `LOGIN_REDIRECT_URL`.

### 12.5 `LoginView` options

| Attribute                    | Default                       | Purpose                                                  |
|------------------------------|-------------------------------|----------------------------------------------------------|
| `template_name`              | `'registration/login.html'`   | Template to render                                        |
| `redirect_authenticated_user`| `False`                       | If `True`, logged-in users visiting the page are redirected away |
| `authentication_form`        | `AuthenticationForm`          | Use your own form class                                  |
| `redirect_field_name`        | `'next'`                      | Name of the "go here after" parameter                    |
| `next_page`                  | `None`                        | Fixed URL after login (overrides `LOGIN_REDIRECT_URL`)   |
| `extra_context`              | `None`                        | Extra template variables                                 |
| `success_url_allowed_hosts`  | `set()`                       | Other hosts allowed in `next`                            |

### 12.6 Try it

Run the server and go to **http://127.0.0.1:8000/accounts/login/**. Log in with your superuser — you're redirected to the post list.

---

## Step 13: Add Logout

Since Django 5.0, **logout must be a POST request** (a GET link to `/accounts/logout/` returns *405 Method Not Allowed*). This stops other websites from logging your users out with a hidden image or link.

Use a small form wherever you want a logout button:

```html
<form method="post" action="{% url 'logout' %}">
    {% csrf_token %}
    <button type="submit">Log out</button>
</form>
```

After logout, `LogoutView` redirects to (in order):

1. `next_page` attribute, if set
2. `next` in POST/GET, if safe
3. `LOGOUT_REDIRECT_URL`, if set
4. Otherwise it renders `registration/logged_out.html`

Because we set `LOGOUT_REDIRECT_URL` in Step 9, users go back to the post list. If you prefer a "You've been logged out" page, remove that setting and create `templates/registration/logged_out.html`:

```html
{% extends 'blog/base.html' %}

{% block content %}
    <h2>Logged out</h2>
    <p>You have been logged out. <a href="{% url 'login' %}">Log in again</a>.</p>
{% endblock %}
```

---

## Step 14: Show Login Status on Every Page

Update `blog/templates/blog/base.html` so every page shows who's logged in:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}My Blog{% endblock %}</title>
</head>
<body>
    <header>
        <h1><a href="{% url 'blog:post_list' %}">My Blog</a></h1>

        <nav>
            {% if user.is_authenticated %}
                Hello, <strong>{{ user.get_full_name|default:user.username }}</strong> |
                <a href="{% url 'accounts:profile' %}">Profile</a> |
                <a href="{% url 'password_change' %}">Change password</a> |
                {% if perms.blog.add_post %}
                    <a href="{% url 'blog:post_create' %}">New post</a> |
                {% endif %}
                {% if user.is_staff %}
                    <a href="{% url 'admin:index' %}">Admin</a> |
                {% endif %}
                <form method="post" action="{% url 'logout' %}" style="display:inline">
                    {% csrf_token %}
                    <button type="submit">Log out</button>
                </form>
            {% else %}
                <a href="{% url 'login' %}?next={{ request.path|urlencode }}">Log in</a> |
                <a href="{% url 'accounts:signup' %}">Sign up</a>
            {% endif %}
        </nav>
    </header>
    <hr>

    {% if messages %}
        <ul class="messages">
            {% for message in messages %}
                <li class="{{ message.tags }}">{{ message }}</li>
            {% endfor %}
        </ul>
    {% endif %}

    {% block content %}{% endblock %}
</body>
</html>
```

- `user` and `perms` come from the `auth` context processor (Step 1).
- `?next={{ request.path|urlencode }}` sends the user back to the current page after login.
- `accounts:profile`, `accounts:signup` and `blog:post_create` are created in Steps 17, 18 and 28. Until then, temporarily remove those links to avoid `NoReverseMatch` errors.

---

## Step 15: Password Change

Logged-in users change their password by entering the old one and the new one twice. `PasswordChangeView` is already wired at `accounts/password_change/` and **requires login** automatically.

Create `templates/registration/password_change_form.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Change password{% endblock %}

{% block content %}
    <h2>Change password</h2>
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Change my password</button>
    </form>
{% endblock %}
```

Create `templates/registration/password_change_done.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Password changed{% endblock %}

{% block content %}
    <h2>Password changed</h2>
    <p>Your password was changed successfully.</p>
    <p><a href="{% url 'blog:post_list' %}">Back to the blog</a></p>
{% endblock %}
```

**What happens behind the scenes:**

1. `PasswordChangeForm` checks the old password, then that the two new ones match, then runs your password validators.
2. The new password is hashed and saved.
3. `update_session_auth_hash(request, user)` is called, so the user **stays logged in** in this browser. All their **other** sessions (other devices) are logged out, because the session hash no longer matches (see Step 0).

**Options:**

| View                     | Attribute        | Default                                        |
|--------------------------|------------------|------------------------------------------------|
| `PasswordChangeView`     | `template_name`  | `'registration/password_change_form.html'`     |
|                          | `success_url`    | `reverse_lazy('password_change_done')`         |
|                          | `form_class`     | `PasswordChangeForm`                           |
| `PasswordChangeDoneView` | `template_name`  | `'registration/password_change_done.html'`     |

---

## Step 16: Password Reset (Forgot Password)

The reset flow has **four pages and one email**:

```
1. password_reset          → user enters their email
        │  (Django emails a one-time link if an active user with a usable password has that email)
        ▼
2. password_reset_done     → "Check your email"
        │
        ▼  user clicks the link in the email
3. password_reset_confirm  → user enters a new password twice
        │
        ▼
4. password_reset_complete → "Your password has been set. Log in."
```

You need **six templates**. Create them all in `templates/registration/`.

### 16.1 `password_reset_form.html` — ask for email

```html
{% extends 'blog/base.html' %}

{% block title %}Reset password{% endblock %}

{% block content %}
    <h2>Forgot your password?</h2>
    <p>Enter your email address and we'll send you a link to set a new one.</p>
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Send reset link</button>
    </form>
{% endblock %}
```

### 16.2 `password_reset_done.html` — "check your email"

```html
{% extends 'blog/base.html' %}

{% block title %}Check your email{% endblock %}

{% block content %}
    <h2>Check your email</h2>
    <p>If an account exists with the email you entered, you'll receive a reset link shortly.</p>
    <p>If you don't get an email, check your spam folder and make sure you used the address you signed up with.</p>
{% endblock %}
```

> Django shows this page **even if the email doesn't exist**, so attackers can't use the form to find out who has an account.

### 16.3 `password_reset_email.html` — the email body (plain text)

```
{% autoescape off %}
Hi {{ user.get_username }},

You asked to reset your password on {{ site_name }}.

Click the link below to choose a new password:
{{ protocol }}://{{ domain }}{% url 'password_reset_confirm' uidb64=uid token=token %}

This link works once and expires in 24 hours.

If you didn't ask for this, you can ignore this email.

— The {{ site_name }} team
{% endautoescape %}
```

**Variables available in the email templates:**

| Variable     | Value                                                          |
|--------------|----------------------------------------------------------------|
| `email`      | The address the email is sent to                               |
| `user`       | The user object                                                |
| `uid`        | The user's id, base64-encoded                                  |
| `token`      | A one-time token (becomes invalid after the password changes or after `PASSWORD_RESET_TIMEOUT`) |
| `domain`     | e.g. `127.0.0.1:8000`                                          |
| `site_name`  | Same as domain unless you use the sites framework              |
| `protocol`   | `http` or `https`                                              |

### 16.4 `password_reset_subject.txt` — the email subject

```
Reset your password on {{ site_name }}
```

Keep it on **one line** (newlines are removed anyway).

### 16.5 `password_reset_confirm.html` — choose new password

```html
{% extends 'blog/base.html' %}

{% block title %}Set a new password{% endblock %}

{% block content %}
    {% if validlink %}
        <h2>Set a new password</h2>
        <form method="post">
            {% csrf_token %}
            {{ form.as_p }}
            <button type="submit">Change my password</button>
        </form>
    {% else %}
        <h2>Link invalid</h2>
        <p>This reset link is invalid, possibly because it has already been used or has expired.</p>
        <p><a href="{% url 'password_reset' %}">Request a new one</a>.</p>
    {% endif %}
{% endblock %}
```

`validlink` is `False` if the token was already used, expired, or tampered with.

### 16.6 `password_reset_complete.html` — done

```html
{% extends 'blog/base.html' %}

{% block title %}Password set{% endblock %}

{% block content %}
    <h2>Password set</h2>
    <p>Your new password has been saved.</p>
    <p><a href="{% url 'login' %}">Log in</a></p>
{% endblock %}
```

### 16.7 Test the flow

1. Make sure your user has an email (edit it in the admin if not).
2. Log out, go to **http://127.0.0.1:8000/accounts/password_reset/** and enter that email.
3. Look at the **terminal running `runserver`** — the console email backend prints the email there.
4. Copy the link into the browser, set a new password, and log in with it.

### 16.8 How the token works (no database table needed)

The token is a hash of: the user's id, their **current password hash**, their **last login time**, their email, and a timestamp — signed with your `SECRET_KEY`. Therefore:

- Once the password changes, the old token no longer matches → **one-time use**.
- Once the user logs in again, `last_login` changes → old links stop working.
- After `PASSWORD_RESET_TIMEOUT` seconds the timestamp is too old → **expired**.
- Changing `SECRET_KEY` invalidates all outstanding links.

When the link is opened, `PasswordResetConfirmView` checks the token, stores it in the session and redirects to `.../reset/<uidb64>/set-password/`, so the real token doesn't leak through the browser's `Referer` header.

### 16.9 Reset view options

| View                          | Attribute                  | Default / purpose                                        |
|-------------------------------|----------------------------|----------------------------------------------------------|
| `PasswordResetView`           | `template_name`            | `'registration/password_reset_form.html'`                |
|                               | `email_template_name`      | `'registration/password_reset_email.html'`               |
|                               | `html_email_template_name` | `None` — set it to also send an HTML version             |
|                               | `subject_template_name`    | `'registration/password_reset_subject.txt'`              |
|                               | `from_email`               | `None` → uses `DEFAULT_FROM_EMAIL`                       |
|                               | `extra_email_context`      | Extra variables for the email templates                  |
|                               | `form_class`               | `PasswordResetForm`                                      |
|                               | `token_generator`          | `default_token_generator`                                |
|                               | `success_url`              | `reverse_lazy('password_reset_done')`                    |
| `PasswordResetConfirmView`    | `template_name`            | `'registration/password_reset_confirm.html'`             |
|                               | `form_class`               | `SetPasswordForm`                                        |
|                               | `post_reset_login`         | `False` — set `True` to log the user in automatically     |
|                               | `post_reset_login_backend` | Backend path to use when several are configured          |
|                               | `reset_url_token`          | `'set-password'`                                         |
|                               | `success_url`              | `reverse_lazy('password_reset_complete')`                |
| `PasswordResetDoneView`       | `template_name`            | `'registration/password_reset_done.html'`                |
| `PasswordResetCompleteView`   | `template_name`            | `'registration/password_reset_complete.html'`            |

---

## Step 17: Build a Sign-Up Page

Django has no sign-up view, but it has the form: `UserCreationForm`. With a custom user model you must subclass it so it uses your model.

### 17.1 Forms

Create `accounts/forms.py`:

```python
from django import forms
from django.contrib.auth import get_user_model
from django.contrib.auth.forms import UserCreationForm

User = get_user_model()


class SignUpForm(UserCreationForm):
    class Meta(UserCreationForm.Meta):
        model = User
        fields = ('username', 'email', 'first_name', 'last_name')
        # password1 and password2 are added automatically by UserCreationForm


class ProfileForm(forms.ModelForm):
    class Meta:
        model = User
        fields = ('first_name', 'last_name', 'email', 'bio')
```

**What `UserCreationForm` does for you:**

- Adds `password1` and `password2` fields.
- Checks the two passwords match.
- Runs `AUTH_PASSWORD_VALIDATORS` and shows their help text.
- Checks the username isn't taken (case-insensitive).
- On `save()`, calls `set_password()` so the password is hashed.

### 17.2 View

Edit `accounts/views.py`:

```python
from django.contrib.auth import login
from django.contrib.auth.models import Group
from django.shortcuts import redirect
from django.urls import reverse_lazy
from django.views.generic import CreateView

from .forms import SignUpForm


class SignUpView(CreateView):
    form_class = SignUpForm
    template_name = 'registration/signup.html'
    success_url = reverse_lazy('blog:post_list')

    def dispatch(self, request, *args, **kwargs):
        # Logged-in users don't need to sign up again
        if request.user.is_authenticated:
            return redirect(self.success_url)
        return super().dispatch(request, *args, **kwargs)

    def form_valid(self, form):
        response = super().form_valid(form)          # saves the user → self.object

        # Optional: put every new user in the "Authors" group (created in Step 26)
        authors = Group.objects.filter(name='Authors').first()
        if authors:
            self.object.groups.add(authors)

        # Log the new user in straight away.
        # 'backend' is required once you have more than one backend (Step 29).
        login(self.request, self.object, backend='django.contrib.auth.backends.ModelBackend')
        return response
```

### 17.3 URL

Edit `accounts/urls.py`:

```python
from django.urls import path

from . import views

app_name = 'accounts'

urlpatterns = [
    path('signup/', views.SignUpView.as_view(), name='signup'),
]
```

### 17.4 Template

Create `templates/registration/signup.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Sign up{% endblock %}

{% block content %}
    <h2>Create an account</h2>
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Sign up</button>
    </form>
    <p>Already have an account? <a href="{% url 'login' %}">Log in</a></p>
{% endblock %}
```

Visit **http://127.0.0.1:8000/accounts/signup/**, create an account, and you're logged in immediately.

> **Email verification:** Django doesn't verify emails out of the box. For "click the link to activate your account", either set `is_active=False` on sign-up and email a token link (built with `default_token_generator`, like password reset), or use **django-allauth**, which handles verification, social login and more.

---

## Step 18: Build a Profile Page

### 18.1 Views

Add to `accounts/views.py`:

```python
from django.contrib import messages
from django.contrib.auth.decorators import login_required
from django.shortcuts import redirect, render

from .forms import ProfileForm, SignUpForm


@login_required
def profile(request):
    return render(request, 'accounts/profile.html')


@login_required
def profile_edit(request):
    if request.method == 'POST':
        form = ProfileForm(request.POST, instance=request.user)
        if form.is_valid():
            form.save()
            messages.success(request, 'Your profile was updated.')
            return redirect('accounts:profile')
    else:
        form = ProfileForm(instance=request.user)
    return render(request, 'accounts/profile_edit.html', {'form': form})
```

(Merge the imports with the ones already at the top of the file.)

### 18.2 URLs

```python
urlpatterns = [
    path('signup/', views.SignUpView.as_view(), name='signup'),
    path('profile/', views.profile, name='profile'),
    path('profile/edit/', views.profile_edit, name='profile_edit'),
]
```

### 18.3 Templates

`templates/accounts/profile.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}My profile{% endblock %}

{% block content %}
    <h2>{{ user.get_full_name|default:user.username }}</h2>
    <ul>
        <li>Username: {{ user.username }}</li>
        <li>Email: {{ user.email }}</li>
        <li>Joined: {{ user.date_joined|date:"d M Y" }}</li>
        <li>Last login: {{ user.last_login|date:"d M Y, H:i"|default:"never" }}</li>
        <li>Groups: {% for group in user.groups.all %}{{ group.name }}{% if not forloop.last %}, {% endif %}{% empty %}none{% endfor %}</li>
    </ul>
    {% if user.bio %}<p>{{ user.bio|linebreaks }}</p>{% endif %}
    <p><a href="{% url 'accounts:profile_edit' %}">Edit profile</a></p>
{% endblock %}
```

`templates/accounts/profile_edit.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Edit profile{% endblock %}

{% block content %}
    <h2>Edit profile</h2>
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Save</button>
    </form>
{% endblock %}
```

> Never put `is_staff`, `is_superuser`, `groups` or `user_permissions` in a form that normal users can submit — they could give themselves admin rights.

---

## Step 19: Write Your Own Login and Logout Views (Manual Way)

The built-in views cover most needs, but you should understand the three core functions:

| Function                                  | What it does                                                            |
|-------------------------------------------|-------------------------------------------------------------------------|
| `authenticate(request, **credentials)`    | Asks each backend to verify credentials. Returns a `User` or `None`. **Doesn't log in.** |
| `login(request, user, backend=None)`      | Saves the user in the session (logs them in)                            |
| `logout(request)`                         | Flushes the session (logs them out). Safe to call even if not logged in |

### 19.1 Fully manual login view

```python
# accounts/views.py
from django.conf import settings
from django.contrib.auth import authenticate, login, logout
from django.shortcuts import redirect, render
from django.utils.http import url_has_allowed_host_and_scheme
from django.views.decorators.http import require_POST


def manual_login(request):
    error = None
    if request.method == 'POST':
        username = request.POST.get('username')
        password = request.POST.get('password')
        user = authenticate(request, username=username, password=password)

        if user is not None:
            login(request, user)

            # Only follow ?next= if it points to our own site (prevents open redirects)
            next_url = request.POST.get('next') or request.GET.get('next')
            if next_url and url_has_allowed_host_and_scheme(
                next_url,
                allowed_hosts={request.get_host()},
                require_https=request.is_secure(),
            ):
                return redirect(next_url)
            return redirect(settings.LOGIN_REDIRECT_URL)

        error = 'Invalid username or password.'

    return render(request, 'accounts/manual_login.html', {
        'error': error,
        'next': request.GET.get('next', ''),
    })


@require_POST
def manual_logout(request):
    logout(request)
    return redirect('blog:post_list')
```

**Important details:**

- `authenticate()` returns `None` for wrong passwords **and** for inactive users (with the default `ModelBackend`).
- Always pass `request` to `authenticate()` — some backends and the `user_login_failed` signal use it.
- Never write `redirect(request.GET['next'])` without checking it — attackers could send users to a fake site.

### 19.2 Same thing using `AuthenticationForm` (recommended)

The form handles validation, error messages and the inactive-user check for you:

```python
from django.contrib.auth.forms import AuthenticationForm


def form_login(request):
    if request.method == 'POST':
        form = AuthenticationForm(request, data=request.POST)   # note: request is the FIRST argument
        if form.is_valid():
            login(request, form.get_user())
            return redirect(settings.LOGIN_REDIRECT_URL)
    else:
        form = AuthenticationForm(request)
    return render(request, 'registration/login.html', {'form': form})
```

### 19.3 "Remember me" checkbox

Extend the built-in `LoginView` so that without the box ticked, the login ends when the browser closes:

```python
from django.contrib.auth.views import LoginView


class RememberMeLoginView(LoginView):
    redirect_authenticated_user = True

    def form_valid(self, form):
        response = super().form_valid(form)             # logs the user in
        if not self.request.POST.get('remember_me'):
            self.request.session.set_expiry(0)          # 0 = expire when browser closes
        # else: uses SESSION_COOKIE_AGE (2 weeks by default)
        return response
```

Use it **instead of** the built-in login URL (put it before the `include('django.contrib.auth.urls')` line so it matches first):

```python
# config/urls.py
from accounts.views import RememberMeLoginView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/login/', RememberMeLoginView.as_view(), name='login'),
    path('accounts/', include('django.contrib.auth.urls')),
    path('accounts/', include('accounts.urls')),
    path('', include('blog.urls')),
]
```

Add the checkbox to `templates/registration/login.html`, inside the form:

```html
<p>
    <label><input type="checkbox" name="remember_me"> Remember me</label>
</p>
```

`request.session.set_expiry(value)` accepts:

| Value               | Meaning                                         |
|---------------------|-------------------------------------------------|
| `0`                 | Expire when the browser closes                  |
| An integer `n`      | Expire after `n` seconds of inactivity          |
| `datetime`/`timedelta` | Expire at that time                          |
| `None`              | Use the global `SESSION_COOKIE_AGE`             |

---

## Step 20: Use `request.user` in Views

In every view, `request.user` is either a `User` or an `AnonymousUser`:

```python
def dashboard(request):
    if request.user.is_authenticated:
        posts = request.user.post_set.all()          # posts written by this user
        greeting = f'Welcome back, {request.user.get_username()}'
    else:
        posts = []
        greeting = 'Welcome, guest'
    return render(request, 'blog/dashboard.html', {'posts': posts, 'greeting': greeting})
```

**Common checks:**

```python
request.user.is_authenticated        # logged in?
request.user.is_staff                # can use the admin?
request.user.is_superuser            # all permissions?
request.user.has_perm('blog.add_post')
request.user.groups.filter(name='Editors').exists()
request.user == post.author          # ownership
```

**Getting the user outside a view** (e.g. a helper that has the request):

```python
from django.contrib.auth import get_user

user = get_user(request)    # same as request.user but bypasses the lazy wrapper
```

> `request.user` is **lazy** — the database is only queried when you first touch it, and the result is cached for the rest of that request.

---

## Step 21: Restrict Views to Logged-In Users

### 21.1 Function views — `@login_required`

```python
from django.contrib.auth.decorators import login_required


@login_required
def my_drafts(request):
    drafts = Post.objects.filter(author=request.user, status='draft')
    return render(request, 'blog/my_drafts.html', {'posts': drafts})
```

An anonymous visitor is redirected to `LOGIN_URL?next=/the/page/they/wanted/`. After logging in, `LoginView` sends them back.

**Options:**

```python
@login_required(login_url='/members/login/', redirect_field_name='go_to')
def my_view(request):
    ...
```

| Argument              | Default            | Purpose                           |
|-----------------------|--------------------|-----------------------------------|
| `login_url`           | `settings.LOGIN_URL` | Where to send anonymous users   |
| `redirect_field_name` | `'next'`           | Query parameter name              |

### 21.2 Class-based views — `LoginRequiredMixin`

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import ListView


class MyDraftsView(LoginRequiredMixin, ListView):
    template_name = 'blog/my_drafts.html'
    context_object_name = 'posts'
    # optional:
    login_url = 'login'
    redirect_field_name = 'next'

    def get_queryset(self):
        return Post.objects.filter(author=self.request.user, status='draft')
```

> **Mixin order matters:** auth mixins must be **left-most** (before `ListView`, `CreateView`, etc.), so the check runs before the view does anything.

### 21.3 Protect a URL without touching the view

```python
from django.contrib.auth.decorators import login_required

urlpatterns = [
    path('drafts/', login_required(views.MyDraftsView.as_view()), name='my_drafts'),
]
```

### 21.4 Protect the whole site — `LoginRequiredMiddleware` (Django 5.1+)

For internal tools where **every** page needs login, add the middleware **after** `AuthenticationMiddleware`:

```python
MIDDLEWARE = [
    # ...
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.auth.middleware.LoginRequiredMiddleware',   # ← add
    # ...
]
```

Now everything requires login, **except** views you mark as public:

```python
from django.contrib.auth.decorators import login_not_required
from django.utils.decorators import method_decorator


@login_not_required
def post_list(request):
    ...


@method_decorator(login_not_required, name='dispatch')
class SignUpView(CreateView):
    ...
```

The built-in `LoginView`, password reset views and the admin login are already marked `login_not_required`.

### 21.5 Restrict to staff — `@staff_member_required`

```python
from django.contrib.admin.views.decorators import staff_member_required


@staff_member_required
def stats(request):
    ...
```

Redirects non-staff users to the **admin** login page. (It lives in `django.contrib.admin`, not auth.)

---

## Step 22: Custom Access Rules (`user_passes_test` and `UserPassesTestMixin`)

When the rule is more than "logged in" — e.g. "email must end with @company.com" or "must be the post's author".

### 22.1 Function views — `@user_passes_test`

```python
from django.contrib.auth.decorators import user_passes_test


def is_company_member(user):
    return user.is_authenticated and user.email.endswith('@company.com')


@user_passes_test(is_company_member, login_url='login')
def internal_page(request):
    ...
```

The test function gets the **user** and returns `True`/`False`. Failing users are redirected to `login_url`. (`@user_passes_test` doesn't automatically check login — include `is_authenticated` in your test, or stack `@login_required` on top.)

### 22.2 Class-based views — `UserPassesTestMixin`

Here the test can see the object being edited, so it's great for **ownership** checks:

```python
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin
from django.views.generic import UpdateView


class PostUpdateView(LoginRequiredMixin, UserPassesTestMixin, UpdateView):
    model = Post
    fields = ['title', 'slug', 'content']
    template_name = 'blog/post_form.html'

    def test_func(self):
        post = self.get_object()
        return post.author == self.request.user or self.request.user.has_perm('blog.change_post')
```

### 22.3 What happens when access is denied (all mixins)

All auth mixins share the same behavior (from `AccessMixin`):

| Situation                          | Result                                              |
|------------------------------------|-----------------------------------------------------|
| User is **not logged in**          | Redirect to `LOGIN_URL?next=...`                    |
| User **is logged in** but fails    | **403 Forbidden** (`PermissionDenied`)              |
| `raise_exception = True`           | Always 403, even for anonymous users                |

**Mixin attributes you can set:**

| Attribute             | Purpose                                                |
|-----------------------|--------------------------------------------------------|
| `login_url`           | Override `LOGIN_URL`                                   |
| `redirect_field_name` | Override `'next'`                                      |
| `raise_exception`     | `True` → always raise 403 instead of redirecting       |
| `permission_denied_message` | Message attached to the `PermissionDenied` exception |

Override `handle_no_permission()` for full control, e.g. show a message and redirect:

```python
from django.contrib import messages
from django.shortcuts import redirect

class PostUpdateView(LoginRequiredMixin, UserPassesTestMixin, UpdateView):
    ...
    def handle_no_permission(self):
        if self.request.user.is_authenticated:
            messages.error(self.request, "You can only edit your own posts.")
            return redirect('blog:post_list')
        return super().handle_no_permission()
```

---

## Step 23: Understand Permissions

A **permission** is a named yes/no flag stored in the `auth_permission` table.

### 23.1 The default four

For **every model**, Django automatically creates four permissions after `migrate`:

| Codename        | Name shown in admin    | Full string used in code |
|-----------------|------------------------|--------------------------|
| `add_post`      | Can add post           | `'blog.add_post'`        |
| `change_post`   | Can change post        | `'blog.change_post'`     |
| `delete_post`   | Can delete post        | `'blog.delete_post'`     |
| `view_post`     | Can view post          | `'blog.view_post'`       |

The format is always `'<app_label>.<action>_<modelname_lowercase>'`.

> Django's **admin** enforces these automatically. **Your own views don't** — you must check them yourself (Step 25).

### 23.2 The `Permission` model

| Field           | Example                          |
|-----------------|----------------------------------|
| `name`          | `'Can add post'`                 |
| `content_type`  | `ContentType` for `blog.Post`    |
| `codename`      | `'add_post'`                     |

### 23.3 Checking permissions

```python
user.has_perm('blog.add_post')                          # one permission
user.has_perms(['blog.add_post', 'blog.change_post'])   # ALL of these
user.has_module_perms('blog')                           # ANY permission in the blog app
user.get_all_permissions()                              # {'blog.add_post', 'blog.view_post', ...}
user.get_user_permissions()                             # only the ones given directly
user.get_group_permissions()                            # only the ones from groups
```

### 23.4 Giving and removing permissions

```python
from django.contrib.auth.models import Permission

perm = Permission.objects.get(codename='change_post', content_type__app_label='blog')

user.user_permissions.add(perm)          # give
user.user_permissions.remove(perm)       # take away
user.user_permissions.set([perm])        # replace all direct permissions
user.user_permissions.clear()            # remove all direct permissions
```

### 23.5 ⚠ Permission caching

The first `has_perm()` call loads and **caches** all permissions on that user object. Adding a permission afterwards isn't seen by the same object:

```python
user.has_perm('blog.change_post')        # False (and now cached)
user.user_permissions.add(perm)
user.has_perm('blog.change_post')        # STILL False — cached!

user = User.objects.get(pk=user.pk)      # re-fetch from the database
user.has_perm('blog.change_post')        # True
```

In views this rarely matters, because each request loads a fresh `request.user`.

---

## Step 24: Add Custom Permissions

Add your own permissions in the model's `Meta`. Update `blog/models.py`:

```python
from django.conf import settings
from django.db import models
from django.urls import reverse


class Post(models.Model):
    STATUS_CHOICES = [
        ('draft', 'Draft'),
        ('published', 'Published'),
    ]

    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, null=True, blank=True
    )
    content = models.TextField()
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default='draft')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-created_at']
        permissions = [
            ('publish_post', 'Can publish post'),          # (codename, human-readable name)
            ('feature_post', 'Can feature post on home page'),
        ]

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse('blog:post_detail', args=[self.pk])
```

Create and apply the migration:

```bash
python manage.py makemigrations blog
python manage.py migrate
```

Now `'blog.publish_post'` and `'blog.feature_post'` exist, appear in the admin under Users/Groups, and can be checked with `has_perm()`.

**Other `Meta` options for permissions:**

```python
class Meta:
    default_permissions = ('add', 'change', 'delete')   # don't create 'view' for this model
    # default_permissions = ()                         # create none of the defaults
```

**Create a permission in code** (not tied to a model's Meta):

```python
from django.contrib.auth.models import Permission
from django.contrib.contenttypes.models import ContentType
from blog.models import Post

content_type = ContentType.objects.get_for_model(Post)
Permission.objects.get_or_create(
    codename='export_posts',
    name='Can export posts',
    content_type=content_type,
)
```

---

## Step 25: Restrict Views by Permission

### 25.1 Function views — `@permission_required`

```python
from django.contrib.auth.decorators import login_required, permission_required


@login_required
@permission_required('blog.publish_post', raise_exception=True)
def post_publish(request, pk):
    ...
```

| Argument          | Purpose                                                                      |
|-------------------|------------------------------------------------------------------------------|
| `perm`            | A string, or a list/tuple — the user needs **all** of them                   |
| `login_url`       | Where to redirect on failure                                                  |
| `raise_exception` | `True` → return **403** instead of redirecting to login                      |

> **Gotcha:** without `raise_exception=True`, a logged-in user who lacks the permission gets redirected to the **login page** — confusing, and it can cause a redirect loop. Always use `raise_exception=True` together with `@login_required` on top.

### 25.2 Class-based views — `PermissionRequiredMixin`

```python
from django.contrib.auth.mixins import PermissionRequiredMixin


class PostCreateView(LoginRequiredMixin, PermissionRequiredMixin, CreateView):
    permission_required = 'blog.add_post'                       # or a tuple: ('blog.add_post', 'blog.view_post')
    model = Post
    fields = ['title', 'slug', 'content']
```

For dynamic rules, override `has_permission()`:

```python
    def has_permission(self):
        return super().has_permission() and self.request.user.is_active
```

### 25.3 Check inside the view

```python
from django.core.exceptions import PermissionDenied


def post_feature(request, pk):
    if not request.user.has_perm('blog.feature_post'):
        raise PermissionDenied              # → 403 Forbidden
    ...
```

### 25.4 Custom 403 page

When `DEBUG = False`, Django renders `403.html` from your templates for `PermissionDenied`. Create `templates/403.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Access denied{% endblock %}

{% block content %}
    <h2>Access denied</h2>
    <p>You don't have permission to view this page.</p>
    <p><a href="{% url 'blog:post_list' %}">Back to the blog</a></p>
{% endblock %}
```

(With `DEBUG = True` you see Django's default 403 message instead.)

---

## Step 26: Organize Permissions With Groups

Giving permissions user by user doesn't scale. A **group** is a named set of permissions; users in the group get all of them.

### 26.1 Plan the roles

| Group      | Permissions                                                        |
|------------|--------------------------------------------------------------------|
| `Authors`  | `add_post`, `view_post` (+ edit/delete **their own** via ownership) |
| `Editors`  | `add_post`, `change_post`, `delete_post`, `view_post`, `publish_post` |

### 26.2 Option A — create groups in the admin

1. **Admin → Groups → Add group**.
2. Name it `Authors`, choose `blog | post | Can add post` and `Can view post`. Save.
3. Add `Editors` with its permissions. Save.
4. **Admin → Users → (user)** → choose groups → Save.

### 26.3 Option B — create groups in code (repeatable)

Creating groups by hand must be redone on every new database. A management command does it with one command.

Create these files (the two `__init__.py` files are empty):

```
blog/
└── management/
    ├── __init__.py
    └── commands/
        ├── __init__.py
        └── setup_groups.py
```

`blog/management/commands/setup_groups.py`:

```python
from django.contrib.auth.models import Group, Permission
from django.core.management.base import BaseCommand

GROUPS = {
    'Authors': ['add_post', 'view_post'],
    'Editors': ['add_post', 'change_post', 'delete_post', 'view_post', 'publish_post'],
}


class Command(BaseCommand):
    help = 'Create the default groups and assign their permissions'

    def handle(self, *args, **options):
        for group_name, codenames in GROUPS.items():
            group, created = Group.objects.get_or_create(name=group_name)
            perms = Permission.objects.filter(
                content_type__app_label='blog',
                codename__in=codenames,
            )
            group.permissions.set(perms)

            missing = set(codenames) - set(perms.values_list('codename', flat=True))
            if missing:
                self.stdout.write(self.style.WARNING(f'{group_name}: missing {missing} — did you run migrate?'))

            action = 'Created' if created else 'Updated'
            self.stdout.write(self.style.SUCCESS(f'{action} {group_name} with {perms.count()} permissions'))
```

Run it (after `migrate`, so the permissions exist):

```bash
python manage.py setup_groups
```

> **Why not a data migration?** You can create groups in a migration, but on a fresh database the permissions don't exist yet during migrations (they're created by the `post_migrate` signal **after** all migrations run). A management command run after `migrate` avoids that problem.

### 26.4 Work with groups in code

```python
from django.contrib.auth.models import Group

editors = Group.objects.get(name='Editors')

user.groups.add(editors)                          # join
user.groups.remove(editors)                       # leave
user.groups.set([editors])                        # replace all groups
user.groups.clear()                               # leave all
user.groups.filter(name='Editors').exists()       # is the user an editor?

editors.user_set.all()                            # all users in the group
editors.permissions.all()                         # the group's permissions
```

**Check group membership in a view:**

```python
def is_editor(user):
    return user.groups.filter(name='Editors').exists()


@login_required
@user_passes_test(is_editor)
def editor_dashboard(request):
    ...
```

> Prefer checking **permissions** (`has_perm('blog.publish_post')`) over checking **group names**. Permissions describe *what* someone can do; groups are just a way of handing them out. Then you can rename or reorganize groups without changing code.

---

## Step 27: Use Permissions in Templates

The `perms` variable (from the `auth` context processor) lets templates check permissions:

```html
{% if perms.blog %}
    <p>You have at least one permission in the blog app.</p>
{% endif %}

{% if perms.blog.add_post %}
    <a href="{% url 'blog:post_create' %}">New post</a>
{% endif %}

{% if perms.blog.publish_post %}
    <button>Publish</button>
{% endif %}

{% if 'blog.delete_post' in perms %}
    <a href="#">Delete</a>
{% endif %}
```

| Syntax                         | Equivalent Python                          |
|--------------------------------|--------------------------------------------|
| `perms.blog`                   | `user.has_module_perms('blog')`            |
| `perms.blog.add_post`          | `user.has_perm('blog.add_post')`           |
| `'blog.add_post' in perms`     | `user.has_perm('blog.add_post')`           |

Other useful template checks:

```html
{% if user.is_authenticated %} ... {% endif %}
{% if user.is_staff %} ... {% endif %}
{% if user == post.author %} ... {% endif %}
{{ user.get_username }}
```

> **Hiding a button is not security.** Always enforce the same rule in the view (Steps 21–25). Templates only control what people see; anyone can still send the request directly.

---

## Step 28: Put It All Together — Protect the Blog

Now combine everything: anyone can read published posts, **Authors** can write posts and edit their own, **Editors** can edit/delete any post and publish.

### 28.1 Views

Replace `blog/views.py`:

```python
from django.contrib import messages
from django.contrib.auth.decorators import login_required, permission_required
from django.contrib.auth.mixins import (
    LoginRequiredMixin,
    PermissionRequiredMixin,
    UserPassesTestMixin,
)
from django.db.models import Q
from django.shortcuts import get_object_or_404, redirect, render
from django.urls import reverse_lazy
from django.views.decorators.http import require_POST
from django.views.generic import CreateView, DeleteView, UpdateView

from .models import Post


def post_list(request):
    posts = Post.objects.filter(status='published')
    if request.user.is_authenticated:
        # logged-in users also see their own drafts
        posts = Post.objects.filter(Q(status='published') | Q(author=request.user))
    return render(request, 'blog/post_list.html', {'posts': posts})


def post_detail(request, pk):
    post = get_object_or_404(Post, pk=pk)
    is_owner = request.user.is_authenticated and post.author == request.user
    if post.status == 'draft' and not (is_owner or request.user.has_perm('blog.change_post')):
        return redirect('blog:post_list')       # drafts are private
    return render(request, 'blog/post_detail.html', {'post': post, 'is_owner': is_owner})


class PostCreateView(LoginRequiredMixin, PermissionRequiredMixin, CreateView):
    permission_required = 'blog.add_post'
    model = Post
    fields = ['title', 'slug', 'content']
    template_name = 'blog/post_form.html'

    def form_valid(self, form):
        form.instance.author = self.request.user      # author = whoever is logged in
        messages.success(self.request, 'Post created as a draft.')
        return super().form_valid(form)               # redirects to post.get_absolute_url()


class OwnerOrPermissionMixin(UserPassesTestMixin):
    """Allow the post's author, or anyone with `owner_bypass_permission`."""
    owner_bypass_permission = None

    def test_func(self):
        post = self.get_object()
        user = self.request.user
        return post.author == user or user.has_perm(self.owner_bypass_permission)


class PostUpdateView(LoginRequiredMixin, OwnerOrPermissionMixin, UpdateView):
    owner_bypass_permission = 'blog.change_post'
    model = Post
    fields = ['title', 'slug', 'content']
    template_name = 'blog/post_form.html'


class PostDeleteView(LoginRequiredMixin, OwnerOrPermissionMixin, DeleteView):
    owner_bypass_permission = 'blog.delete_post'
    model = Post
    template_name = 'blog/post_confirm_delete.html'
    success_url = reverse_lazy('blog:post_list')


@login_required
@permission_required('blog.publish_post', raise_exception=True)
@require_POST
def post_publish(request, pk):
    post = get_object_or_404(Post, pk=pk)
    post.status = 'published'
    post.save(update_fields=['status', 'updated_at'])
    messages.success(request, f'"{post.title}" is now published.')
    return redirect(post)
```

### 28.2 URLs

Replace `blog/urls.py`:

```python
from django.urls import path

from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.post_list, name='post_list'),
    path('post/new/', views.PostCreateView.as_view(), name='post_create'),
    path('post/<int:pk>/', views.post_detail, name='post_detail'),
    path('post/<int:pk>/edit/', views.PostUpdateView.as_view(), name='post_update'),
    path('post/<int:pk>/delete/', views.PostDeleteView.as_view(), name='post_delete'),
    path('post/<int:pk>/publish/', views.post_publish, name='post_publish'),
]
```

### 28.3 Templates

`blog/templates/blog/post_form.html` (used for both create and edit):

```html
{% extends 'blog/base.html' %}

{% block title %}{% if object %}Edit post{% else %}New post{% endif %}{% endblock %}

{% block content %}
    <h2>{% if object %}Edit "{{ object.title }}"{% else %}New post{% endif %}</h2>
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Save</button>
    </form>
{% endblock %}
```

`blog/templates/blog/post_confirm_delete.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}Delete post{% endblock %}

{% block content %}
    <h2>Delete "{{ object.title }}"?</h2>
    <form method="post">
        {% csrf_token %}
        <button type="submit">Yes, delete</button>
        <a href="{{ object.get_absolute_url }}">Cancel</a>
    </form>
{% endblock %}
```

Update `blog/templates/blog/post_detail.html`:

```html
{% extends 'blog/base.html' %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
    <h2>{{ post.title }}{% if post.status == 'draft' %} <small>(draft)</small>{% endif %}</h2>
    <p><small>
        by {{ post.author.get_username|default:"unknown" }}
        on {{ post.created_at|date:"d M Y, H:i" }}
    </small></p>
    <p>{{ post.content|linebreaks }}</p>

    {% if is_owner or perms.blog.change_post %}
        <a href="{% url 'blog:post_update' post.pk %}">Edit</a>
    {% endif %}

    {% if is_owner or perms.blog.delete_post %}
        <a href="{% url 'blog:post_delete' post.pk %}">Delete</a>
    {% endif %}

    {% if perms.blog.publish_post and post.status == 'draft' %}
        <form method="post" action="{% url 'blog:post_publish' post.pk %}" style="display:inline">
            {% csrf_token %}
            <button type="submit">Publish</button>
        </form>
    {% endif %}
{% endblock %}
```

### 28.4 Try every role

1. `python manage.py setup_groups`
2. Sign up a new user → they're automatically an **Author** (Step 17). They can create posts and edit/delete their own, but not publish.
3. In the admin, put another user in **Editors**. They can edit, delete and publish anyone's posts.
4. Log out → you only see published posts, and visiting `/post/new/` redirects to the login page.
5. As an Author, open `/post/<someone-else's-id>/edit/` → **403 Forbidden**.

---

## Step 29: Authentication Backends

A **backend** is a class that answers two questions: *"are these credentials valid?"* (`authenticate`) and *"give me user #5"* (`get_user`). It can also answer permission questions.

### 29.1 Built-in backends

| Backend                                                       | Behavior                                                   |
|---------------------------------------------------------------|------------------------------------------------------------|
| `django.contrib.auth.backends.ModelBackend`                   | Default. Username + password from the database; groups/permissions; rejects inactive users |
| `django.contrib.auth.backends.AllowAllUsersModelBackend`      | Like `ModelBackend` but lets **inactive** users authenticate |
| `django.contrib.auth.backends.RemoteUserBackend`              | Trusts a web server that already authenticated the user (`REMOTE_USER` header) |
| `django.contrib.auth.backends.AllowAllUsersRemoteUserBackend` | Same, without the inactive check                           |
| `django.contrib.auth.backends.BaseBackend`                    | Empty base class for writing your own                      |

### 29.2 How Django uses several backends

```python
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'accounts.backends.EmailBackend',
]
```

- `authenticate()` tries each backend **in order** and stops at the first one that returns a user.
- If a backend raises `PermissionDenied`, Django stops immediately and authentication fails.
- The backend that succeeded is saved in the session (`_auth_user_backend`), and that same backend loads the user on later requests.
- `has_perm()` asks **every** backend and returns `True` if **any** says yes.
- With several backends, `login(request, user)` needs `backend='...'` if the user didn't come from `authenticate()` (e.g. right after sign-up — see Step 17).

### 29.3 Write a backend: log in with username **or** email

Create `accounts/backends.py`:

```python
from django.contrib.auth import get_user_model
from django.contrib.auth.backends import ModelBackend


class EmailBackend(ModelBackend):
    """Authenticate with an email address instead of a username."""

    def authenticate(self, request, username=None, password=None, **kwargs):
        UserModel = get_user_model()
        if username is None:
            username = kwargs.get(UserModel.USERNAME_FIELD)
        if username is None or password is None:
            return None

        try:
            user = UserModel.objects.get(email__iexact=username)
        except UserModel.DoesNotExist:
            # Run the hasher anyway so a missing user takes as long as a wrong password
            # (stops attackers from discovering which emails are registered by timing).
            UserModel().set_password(password)
            return None
        except UserModel.MultipleObjectsReturned:
            return None

        if user.check_password(password) and self.user_can_authenticate(user):
            return user
        return None
```

Because it inherits from `ModelBackend`, it also gets `get_user()` and all the permission methods. `user_can_authenticate()` rejects inactive users.

Register it in `config/settings.py`:

```python
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',   # username + password
    'accounts.backends.EmailBackend',              # email + password
]
```

Now the login form accepts either. You might relabel the field — create a custom form:

```python
# accounts/forms.py
from django.contrib.auth.forms import AuthenticationForm

class EmailOrUsernameAuthenticationForm(AuthenticationForm):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fields['username'].label = 'Username or email'
```

…and use it: `LoginView.as_view(authentication_form=EmailOrUsernameAuthenticationForm)` (or set `authentication_form` on your `RememberMeLoginView`).

### 29.4 The full backend interface

```python
from django.contrib.auth.backends import BaseBackend


class MyBackend(BaseBackend):
    def authenticate(self, request, **credentials):   # return a user or None
        ...

    def get_user(self, user_id):                       # return a user or None
        ...

    # Optional permission methods:
    def get_user_permissions(self, user_obj, obj=None): ...
    def get_group_permissions(self, user_obj, obj=None): ...
    def get_all_permissions(self, user_obj, obj=None): ...
    def has_perm(self, user_obj, perm, obj=None): ...
```

### 29.5 Object-level permissions

`has_perm()` accepts an `obj` argument: `user.has_perm('blog.change_post', post)`. The built-in `ModelBackend` **ignores** `obj` and returns `False` when one is passed. To get real per-object permissions (e.g. "Alice can edit post #7 only") either write a backend that handles `obj`, or use the **django-guardian** package. For simple "owner can edit" rules, the ownership check in Step 28 is enough.

> Changing `AUTHENTICATION_BACKENDS` doesn't log out existing sessions automatically. If you remove a backend, users logged in through it get logged out on their next request (Django can't find their backend).

---

## Step 30: React to Login, Logout and Failed Logins (Signals)

`django.contrib.auth.signals` provides:

| Signal              | Sent when                         | Arguments                                   |
|---------------------|-----------------------------------|---------------------------------------------|
| `user_logged_in`    | After `login()`                   | `sender` (user class), `request`, `user`    |
| `user_logged_out`   | During `logout()`                 | `sender`, `request`, `user` (`None` if wasn't logged in) |
| `user_login_failed` | `authenticate()` found no user    | `sender` (module name), `credentials` (password removed), `request` |

> Django itself connects `update_last_login` to `user_logged_in` — that's how `last_login` gets updated.

Create `accounts/signals.py`:

```python
import logging

from django.contrib import messages
from django.contrib.auth.signals import user_logged_in, user_logged_out, user_login_failed
from django.dispatch import receiver

logger = logging.getLogger(__name__)


@receiver(user_logged_in)
def on_login(sender, request, user, **kwargs):
    # fail_silently=True: client.login()/force_login() in tests use a request without message storage
    messages.success(request, f'Welcome back, {user.get_username()}!', fail_silently=True)
    logger.info('Login: %s from %s', user.get_username(), request.META.get('REMOTE_ADDR'))


@receiver(user_logged_out)
def on_logout(sender, request, user, **kwargs):
    if user is not None:
        logger.info('Logout: %s', user.get_username())


@receiver(user_login_failed)
def on_login_failed(sender, credentials, request, **kwargs):
    ip = request.META.get('REMOTE_ADDR') if request else None
    logger.warning('Failed login for %r from %s', credentials.get('username'), ip)
```

Connect the signals when the app starts. Edit `accounts/apps.py`:

```python
from django.apps import AppConfig


class AccountsConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'accounts'

    def ready(self):
        from . import signals  # noqa: F401  (importing registers the receivers)
```

> **Note:** `fail_silently=True` matters. `self.client.login()` and `force_login()` (Step 33) call `login()` with a bare request that never went through `MessageMiddleware`. Without it, every test that logs a user in crashes with `MessageFailure`.
>
> Messages added during logout (for example in `on_logout`) are **not** lost. `logout()` flushes the session, but new messages are only written to the cookie/session at the end of the request, after the flush, so they appear on the next page. Use `fail_silently=True` there too, because `self.client.logout()` sends the signal with a bare request. See [django-messages-guide.md](django-messages-guide.md) Steps 21–22.

---

## Step 31: Sessions and Security Under the Hood

### 31.1 What's stored in the session

After login, the session (row in `django_session`) contains:

| Key                  | Value                                                    |
|----------------------|----------------------------------------------------------|
| `_auth_user_id`      | The user's primary key (as a string)                     |
| `_auth_user_backend` | Dotted path of the backend that authenticated the user   |
| `_auth_user_hash`    | `user.get_session_auth_hash()` — HMAC of the password hash |

### 31.2 Logging out everywhere when the password changes

Because `_auth_user_hash` depends on the password hash, **changing a password invalidates all other sessions** for that user. If you change a password in **your own** view, keep the current session alive like this:

```python
from django.contrib.auth import update_session_auth_hash


@login_required
def my_change_password(request):
    if request.method == 'POST':
        form = PasswordChangeForm(user=request.user, data=request.POST)
        if form.is_valid():
            user = form.save()
            update_session_auth_hash(request, user)   # otherwise THIS session is logged out too
            return redirect('accounts:profile')
    else:
        form = PasswordChangeForm(user=request.user)
    return render(request, 'registration/password_change_form.html', {'form': form})
```

(The built-in `PasswordChangeView` already does this.)

**Force-logout a user from all devices** (e.g. after a security incident) — set a new password, or delete their sessions:

```python
from django.contrib.sessions.models import Session

for session in Session.objects.all():
    if session.get_decoded().get('_auth_user_id') == str(user.pk):
        session.delete()
```

### 31.3 Built-in protections you get for free

| Threat                       | Protection                                                             |
|------------------------------|------------------------------------------------------------------------|
| Stolen database              | Passwords are salted, slow hashes                                      |
| Session fixation             | `login()` rotates the session key                                      |
| Cross-site request forgery   | `{% csrf_token %}` + `CsrfViewMiddleware`; logout is POST-only         |
| Open redirects               | `LoginView`/`LogoutView` only follow `next` to safe hosts              |
| User enumeration via reset   | Reset form shows the same page whether the email exists or not         |
| Timing attacks               | Password comparison is constant-time; `ModelBackend` hashes even for unknown users |
| Leaking passwords in errors  | Auth views use `@sensitive_post_parameters` so passwords don't appear in error reports |
| Clickjacking                 | `XFrameOptionsMiddleware` (default)                                    |

### 31.4 What Django does NOT do (add yourself)

- **Brute-force protection / rate limiting** on login → use **django-axes** (locks accounts/IPs after N failures).
- **Two-factor authentication** → **django-otp** or **django-two-factor-auth**.
- **Email verification**, **social login** (Google, GitHub…) → **django-allauth**.
- **API token / JWT auth** → **Django REST Framework** with **djangorestframework-simplejwt**.

---

## Step 32: Async Auth API (Django 5.0+)

If you write `async def` views, use the async versions so you don't block the event loop:

| Sync                      | Async                          |
|---------------------------|--------------------------------|
| `request.user`            | `await request.auser()`        |
| `authenticate(...)`       | `await aauthenticate(...)`     |
| `login(request, user)`    | `await alogin(request, user)`  |
| `logout(request)`         | `await alogout(request)`       |
| `get_user(request)`       | `await aget_user(request)`     |

```python
from django.contrib.auth import aauthenticate, alogin
from django.http import JsonResponse


async def api_login(request):
    user = await aauthenticate(request, username=request.POST['username'], password=request.POST['password'])
    if user is None:
        return JsonResponse({'ok': False}, status=401)
    await alogin(request, user)
    return JsonResponse({'ok': True})


async def whoami(request):
    user = await request.auser()
    return JsonResponse({'username': user.get_username() if user.is_authenticated else None})
```

Since Django 5.1, `@login_required`, `@permission_required` and `@user_passes_test` also work on async views. Django 5.2 added async versions of user and permission methods too (e.g. `acreate_user()`, `ahas_perm()`, `aget_all_permissions()`).

---

## Step 33: Test Authentication

Add tests to `accounts/tests.py`:

```python
from django.contrib.auth import get_user_model
from django.contrib.auth.models import Permission
from django.core import mail
from django.test import TestCase
from django.urls import reverse

from blog.models import Post

User = get_user_model()


class AuthFlowTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.password = 'S3cret-pass!x'
        cls.user = User.objects.create_user('alice', 'alice@example.com', cls.password)
        cls.other = User.objects.create_user('bob', 'bob@example.com', cls.password)

    def test_password_is_hashed(self):
        self.assertNotEqual(self.user.password, self.password)
        self.assertTrue(self.user.check_password(self.password))

    def test_login_page_loads(self):
        response = self.client.get(reverse('login'))
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'registration/login.html')

    def test_login_with_correct_password(self):
        response = self.client.post(reverse('login'), {'username': 'alice', 'password': self.password})
        self.assertRedirects(response, reverse('blog:post_list'))
        self.assertEqual(int(self.client.session['_auth_user_id']), self.user.pk)

    def test_login_with_email(self):
        ok = self.client.login(username='alice@example.com', password=self.password)
        self.assertTrue(ok)   # works thanks to EmailBackend

    def test_login_with_wrong_password(self):
        response = self.client.post(reverse('login'), {'username': 'alice', 'password': 'nope'})
        self.assertEqual(response.status_code, 200)
        self.assertFalse(response.context['form'].is_valid())

    def test_logout_requires_post(self):
        self.client.force_login(self.user)
        self.assertEqual(self.client.get(reverse('logout')).status_code, 405)
        self.client.post(reverse('logout'))
        self.assertNotIn('_auth_user_id', self.client.session)

    def test_signup_creates_and_logs_in_user(self):
        response = self.client.post(reverse('accounts:signup'), {
            'username': 'carol',
            'email': 'carol@example.com',
            'first_name': 'Carol',
            'last_name': 'Jones',
            'password1': 'V3ry-g00d-pass!',
            'password2': 'V3ry-g00d-pass!',
        })
        self.assertRedirects(response, reverse('blog:post_list'))
        self.assertTrue(User.objects.filter(username='carol').exists())
        self.assertIn('_auth_user_id', self.client.session)

    def test_password_reset_sends_email(self):
        self.client.post(reverse('password_reset'), {'email': 'alice@example.com'})
        self.assertEqual(len(mail.outbox), 1)          # tests use an in-memory email backend
        self.assertIn('reset', mail.outbox[0].body.lower())

    def test_anonymous_redirected_from_create(self):
        url = reverse('blog:post_create')
        response = self.client.get(url)
        self.assertRedirects(response, f"{reverse('login')}?next={url}")

    def test_logged_in_without_permission_gets_403(self):
        self.client.force_login(self.user)
        response = self.client.get(reverse('blog:post_create'))
        self.assertEqual(response.status_code, 403)

    def test_with_permission_can_create(self):
        self.user.user_permissions.add(Permission.objects.get(codename='add_post'))
        self.client.force_login(self.user)
        response = self.client.get(reverse('blog:post_create'))
        self.assertEqual(response.status_code, 200)

    def test_cannot_edit_someone_elses_post(self):
        post = Post.objects.create(title='Bob post', slug='bob-post', content='x', author=self.other)
        self.client.force_login(self.user)
        response = self.client.get(reverse('blog:post_update', args=[post.pk]))
        self.assertEqual(response.status_code, 403)
```

**Test client helpers:**

| Helper                                       | Use                                                           |
|----------------------------------------------|---------------------------------------------------------------|
| `self.client.login(username=..., password=...)` | Logs in through the real backends; returns `True`/`False`  |
| `self.client.force_login(user, backend=None)`| Logs in **without** a password — faster, use in most tests    |
| `self.client.logout()`                       | Logs out                                                      |
| `mail.outbox`                                | List of emails "sent" during the test                         |

Run the tests:

```bash
python manage.py test
```

---

## Step 34: Production Checklist

Before going live, check every item:

- [ ] Custom user model set with `AUTH_USER_MODEL` from the first migration.
- [ ] `DEBUG = False` and a long, secret `SECRET_KEY` loaded from an environment variable.
- [ ] HTTPS everywhere:
  ```python
  SECURE_SSL_REDIRECT = True
  SESSION_COOKIE_SECURE = True
  CSRF_COOKIE_SECURE = True
  SECURE_HSTS_SECONDS = 31536000
  ```
- [ ] `SESSION_COOKIE_HTTPONLY = True` (the default — keep it).
- [ ] Strong `AUTH_PASSWORD_VALIDATORS` (consider `min_length` 10–12).
- [ ] Argon2 as the first hasher.
- [ ] Real SMTP `EMAIL_BACKEND` and a valid `DEFAULT_FROM_EMAIL` so password reset works.
- [ ] Login rate-limiting (django-axes).
- [ ] 2FA for staff accounts (django-otp).
- [ ] Admin URL changed from `admin/` and staff given minimum permissions via groups.
- [ ] Every protected view enforces access in Python, not only by hiding links.
- [ ] Custom `403.html` and `404.html` templates.
- [ ] Run `python manage.py check --deploy` and fix every warning.

---

## Reference: Built-in Auth Forms

All in `django.contrib.auth.forms`:

| Form                       | Used for                                    | Constructor                              |
|----------------------------|---------------------------------------------|------------------------------------------|
| `AuthenticationForm`       | Login                                       | `AuthenticationForm(request, data=...)`  |
| `UserCreationForm`         | Sign-up (username + 2 passwords)            | `UserCreationForm(data=...)`             |
| `BaseUserCreationForm`     | Same, without the case-insensitive username uniqueness check | `BaseUserCreationForm(data=...)` |
| `UserChangeForm`           | Admin's user edit form                      | `UserChangeForm(instance=user)`          |
| `PasswordChangeForm`       | Change password (needs old password)        | `PasswordChangeForm(user, data=...)`     |
| `SetPasswordForm`          | Set password without old one (reset confirm)| `SetPasswordForm(user, data=...)`        |
| `AdminPasswordChangeForm`  | Admin setting a user's password             | `AdminPasswordChangeForm(user, data=...)`|
| `PasswordResetForm`        | Ask for reset email                         | `PasswordResetForm(data=...)`            |

Useful `AuthenticationForm` methods: `get_user()`, `confirm_login_allowed(user)` (override to block certain users, e.g. raise `ValidationError` if `not user.email_verified`).

## Reference: Built-in Auth Views

All in `django.contrib.auth.views`:

| View                          | Default template                               | Default URL name           |
|-------------------------------|------------------------------------------------|----------------------------|
| `LoginView`                   | `registration/login.html`                      | `login`                    |
| `LogoutView` (POST only)      | `registration/logged_out.html`                 | `logout`                   |
| `PasswordChangeView`          | `registration/password_change_form.html`       | `password_change`          |
| `PasswordChangeDoneView`      | `registration/password_change_done.html`       | `password_change_done`     |
| `PasswordResetView`           | `registration/password_reset_form.html`        | `password_reset`           |
| `PasswordResetDoneView`       | `registration/password_reset_done.html`        | `password_reset_done`      |
| `PasswordResetConfirmView`    | `registration/password_reset_confirm.html`     | `password_reset_confirm`   |
| `PasswordResetCompleteView`   | `registration/password_reset_complete.html`    | `password_reset_complete`  |

Every view accepts `template_name` and `extra_context`; see Steps 12, 15 and 16 for each view's other options.

## Reference: Functions and Decorators

| Import                                                     | Purpose                                      |
|------------------------------------------------------------|----------------------------------------------|
| `django.contrib.auth.get_user_model`                       | Get the active user model class              |
| `django.contrib.auth.authenticate` / `aauthenticate`       | Verify credentials                           |
| `django.contrib.auth.login` / `alogin`                     | Log a user in                                |
| `django.contrib.auth.logout` / `alogout`                   | Log a user out                               |
| `django.contrib.auth.get_user` / `aget_user`               | Get the user for a request                   |
| `django.contrib.auth.update_session_auth_hash`             | Keep session alive after password change     |
| `django.contrib.auth.decorators.login_required`            | Require login                                |
| `django.contrib.auth.decorators.login_not_required`        | Opt out of `LoginRequiredMiddleware` (5.1+)  |
| `django.contrib.auth.decorators.permission_required`       | Require permission(s)                        |
| `django.contrib.auth.decorators.user_passes_test`          | Require a custom test                        |
| `django.contrib.auth.mixins.LoginRequiredMixin`            | Require login (CBV)                          |
| `django.contrib.auth.mixins.PermissionRequiredMixin`       | Require permission(s) (CBV)                  |
| `django.contrib.auth.mixins.UserPassesTestMixin`           | Require a custom test (CBV)                  |
| `django.contrib.auth.hashers.make_password` / `check_password` | Hash / verify passwords manually         |
| `django.contrib.auth.password_validation.validate_password`| Run password validators                      |
| `django.contrib.auth.tokens.default_token_generator`       | Make/check one-time tokens (`make_token(user)`, `check_token(user, token)`) |
| `django.utils.http.url_has_allowed_host_and_scheme`        | Validate a `next` URL                        |

---

## Troubleshooting

| Problem                                                           | Cause / fix                                                                 |
|-------------------------------------------------------------------|-----------------------------------------------------------------------------|
| `TemplateDoesNotExist: registration/login.html`                   | Create the template in `templates/registration/` and set `DIRS` (Step 10)   |
| Your reset/logout pages look like the **admin**                   | The admin's templates are winning — use a project-level `DIRS` folder (Step 10) |
| `405 Method Not Allowed` on logout                                | Logout must be a POST form, not a link (Step 13)                             |
| After login you get a 404 at `/accounts/profile/`                 | Set `LOGIN_REDIRECT_URL` (Step 9)                                            |
| `NoReverseMatch for 'accounts:login'`                             | Built-in URL names have no namespace — use `'login'`                        |
| User can't log in after `User.objects.create(...)`                | Password stored as plain text — use `create_user()` or `set_password()`     |
| `has_perm()` returns `False` right after adding a permission      | Permission cache — re-fetch the user (Step 23.5)                            |
| Logged-in user without permission is sent to the login page       | Add `raise_exception=True` to `@permission_required` (Step 25)             |
| `ValueError: You have multiple authentication backends configured…` | Pass `backend='...'` to `login()` (Step 29.2)                             |
| `AUTH_USER_MODEL refers to model 'accounts.User' that has not been installed` | Add `'accounts'` to `INSTALLED_APPS`                             |
| `InconsistentMigrationHistory` after adding a custom user          | Switched user models after migrating — reset the dev database (Step 3.4)   |
| `Manager isn't available; 'auth.User' has been swapped`           | You imported `django.contrib.auth.models.User` — use `get_user_model()`     |
| Password reset email never arrives                                | User has no email / is inactive / has unusable password, or `EMAIL_BACKEND` isn't configured |
| Reset link says invalid                                           | Already used, expired (`PASSWORD_RESET_TIMEOUT`), the user logged in since, or `SECRET_KEY` changed |
| CSRF verification failed on login                                 | Missing `{% csrf_token %}` in the form                                      |
| Logged out after changing password in your own view               | Call `update_session_auth_hash(request, user)` (Step 31.2)                  |

---

## Final Project Structure

```
myproject/
├── manage.py
├── db.sqlite3
├── config/
│   ├── settings.py              # AUTH_USER_MODEL, LOGIN_*, backends, email, validators
│   └── urls.py                  # includes auth.urls, accounts.urls, blog.urls
├── accounts/
│   ├── admin.py                 # CustomUserAdmin
│   ├── apps.py                  # ready() imports signals
│   ├── backends.py              # EmailBackend
│   ├── forms.py                 # SignUpForm, ProfileForm, EmailOrUsernameAuthenticationForm
│   ├── models.py                # custom User
│   ├── signals.py               # login/logout/failed receivers
│   ├── tests.py
│   ├── urls.py                  # signup, profile, profile_edit
│   ├── validators.py            # SymbolValidator
│   └── views.py                 # SignUpView, profile, profile_edit, RememberMeLoginView
├── blog/
│   ├── management/
│   │   └── commands/
│   │       └── setup_groups.py
│   ├── models.py                # Post with custom permissions
│   ├── urls.py
│   ├── views.py                 # protected create/update/delete/publish
│   └── templates/blog/
│       ├── base.html
│       ├── post_list.html
│       ├── post_detail.html
│       ├── post_form.html
│       └── post_confirm_delete.html
└── templates/
    ├── 403.html
    ├── accounts/
    │   ├── profile.html
    │   └── profile_edit.html
    └── registration/
        ├── login.html
        ├── logged_out.html
        ├── signup.html
        ├── password_change_form.html
        ├── password_change_done.html
        ├── password_reset_form.html
        ├── password_reset_done.html
        ├── password_reset_email.html
        ├── password_reset_subject.txt
        ├── password_reset_confirm.html
        └── password_reset_complete.html
```

---

## Quick Reference

| Task                                  | Code / Command                                                    |
|---------------------------------------|-------------------------------------------------------------------|
| Custom user model                     | `class User(AbstractUser)` + `AUTH_USER_MODEL = 'accounts.User'`  |
| Get user model                        | `User = get_user_model()`                                         |
| FK to user                            | `models.ForeignKey(settings.AUTH_USER_MODEL, ...)`                |
| Create user                           | `User.objects.create_user('name', 'e@x.com', 'pass')`             |
| Create superuser                      | `python manage.py createsuperuser`                                |
| Change password (CLI)                 | `python manage.py changepassword <username>`                      |
| Set password (code)                   | `user.set_password('x'); user.save()`                             |
| Check credentials                     | `authenticate(request, username=..., password=...)`               |
| Log in / out                          | `login(request, user)` / `logout(request)`                        |
| Built-in auth URLs                    | `path('accounts/', include('django.contrib.auth.urls'))`          |
| Current user                          | `request.user` / `{{ user }}`                                     |
| Logged in?                            | `request.user.is_authenticated`                                   |
| Require login (FBV / CBV)             | `@login_required` / `LoginRequiredMixin`                          |
| Require permission (FBV / CBV)        | `@permission_required('app.perm', raise_exception=True)` / `PermissionRequiredMixin` |
| Custom rule (FBV / CBV)               | `@user_passes_test(func)` / `UserPassesTestMixin` + `test_func()` |
| Check permission                      | `user.has_perm('blog.add_post')` / `{% if perms.blog.add_post %}` |
| Custom permission                     | `Meta.permissions = [('publish_post', 'Can publish post')]`       |
| Add user to group                     | `user.groups.add(group)`                                          |
| Give permission                       | `user.user_permissions.add(perm)`                                 |
| Keep session after password change    | `update_session_auth_hash(request, user)`                         |
| Session ends on browser close         | `request.session.set_expiry(0)`                                   |
| Login in tests                        | `self.client.force_login(user)`                                   |

---

## Next Steps

- **django-allauth** — email verification, social login (Google, GitHub), account management.
- **django-axes** — lock out brute-force login attempts.
- **django-otp** / **django-two-factor-auth** — two-factor authentication.
- **django-guardian** — per-object permissions.
- **Django REST Framework** + **simplejwt** — token/JWT authentication for APIs.
- Official docs:
  - Using the auth system: https://docs.djangoproject.com/en/stable/topics/auth/default/
  - Customizing auth: https://docs.djangoproject.com/en/stable/topics/auth/customizing/
  - Password management: https://docs.djangoproject.com/en/stable/topics/auth/passwords/
  - API reference: https://docs.djangoproject.com/en/stable/ref/contrib/auth/
