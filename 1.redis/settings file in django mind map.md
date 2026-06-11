# NOTES 00 — `settings.py`: when to touch it, when to leave it alone

> The mental model for the project's "control room." Learned this while wiring up Redis (Concept 1).

---

## 1. What `settings.py` actually is

Think of the project as the **Grand Hotel**.

- **Rooms** = individual pages/endpoints → `urls.py` → `views.py` → templates.
  Each serves ONE guest (one request).
- **`settings.py`** = the **building's control room / fuse box + reception's master directory.**
  It serves NO single guest. It declares **what the whole building is wired for**:
  which services exist, where the utilities connect, what the house rules are.

```
            ┌─────────────────────────────────────────┐
            │           settings.py                    │
            │   (control room — project-wide wiring)   │
            │  INSTALLED_APPS · CACHES · DATABASES ·   │
            │  MIDDLEWARE · EMAIL_* · CELERY_* · DEBUG │
            └───────────────┬─────────────────────────┘
                            │ applies to EVERY request
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   urls→views→tmpl    urls→views→tmpl     urls→views→tmpl
     (one page)         (one page)          (one page)
```

**The one question that resolves 90% of cases:**

> "Does the ENTIRE project need to know this, or just one view?"
> - Whole project → `settings.py`
> - One page      → `urls.py` / `views.py`

---

## 2. The big confusion: TWO separate "registers"

Installing a package and telling Django about it are **different steps.**
They answer different questions and live in different files.

| Register | File | Question it answers | Whose job |
|---|---|---|---|
| #1 Installed | `pyproject.toml` (`uv add`) / `requirements.txt` | "Is this package on my machine?" | pip / uv |
| #2 Activated | `settings.py` → `INSTALLED_APPS` | "Should Django treat this package as part of itself?" | Django |

A package can be in **#1 but not #2**. That's the NORMAL case.

```
   uv add  ─────────────►  pyproject.toml   (downloaded + installed)
                                 │
                                 ▼
                      Is it a *Django app*?
                       ├─ no  → STOP. Just `import` and use it.
                       └─ yes → also add to INSTALLED_APPS in settings.py
```

---

## 3. "Django app" vs "plain library" — the deciding test

- **Plain library** = normal Python package. You just `import` it and call functions.
  It has no idea Django exists. → `pyproject.toml` **ONLY**.
- **Django app** = ships Django *models, templates, admin pages, middleware, or
  management commands*. Django can't use those unless it's in `INSTALLED_APPS`.
  → **BOTH** lists.

| Package | `pyproject.toml`? | `INSTALLED_APPS`? | Why |
|---|---|---|---|
| numpy / pandas | ✅ | ❌ | plain library — `import numpy` |
| tensorflow / pytorch | ✅ | ❌ | plain library |
| selenium | ✅ | ❌ | plain library |
| httpx, pyotp, redis | ✅ | ❌ | plain libraries — just import + call |
| python-decouple | ✅ | ❌ | plain library — `from decouple import config` |
| **django-redis** | ✅ | ❌ | ⚠️ Django-aware but plugs in via `CACHES`, NOT INSTALLED_APPS |
| rest_framework (DRF) | ✅ | ✅ | Django app — ships admin/templates/views |
| debug_toolbar | ✅ | ✅ | Django app — ships middleware + UI |
| django_celery_beat | ✅ | ✅ | Django app — ships DB models |
| `weather` (your own app) | ❌* | ✅ | your own Django app |

\* Your own app isn't a pip package — it's just folders in your repo — so it's never
in `pyproject.toml`. But it IS a Django app → goes in `INSTALLED_APPS`.

**⚠️ Gotcha:** "starts with `django-`" does NOT mean it goes in `INSTALLED_APPS`.
`django-redis` is the counter-example. Always check the docs, don't pattern-match the name.

**How do you KNOW which bucket?** The install docs tell you (search-pattern habit):
- docs say *"add `'x'` to INSTALLED_APPS"* → Django app → both lists
- docs just say *"import x"* and show function calls → plain library → pyproject only

---

## 4. The 3 triggers to TOUCH `settings.py`

You edit `settings.py` in exactly these three situations — nothing else:

**Trigger 1 — registering a Django app (or middleware).**
Django doesn't auto-discover plugins. You tell it they exist:
- a Django app → add to `INSTALLED_APPS`
- a request-level hook → add to `MIDDLEWARE`
```python
INSTALLED_APPS = [
    ...,
    "rest_framework",   # installed package that's a Django app
    "weather",          # your own app
]
```

**Trigger 2 — a package needs a CONFIG block.**
Some libraries need a dict telling them HOW to behave. These are the ALL-CAPS blocks.
You copy the block from the docs and adapt the values.
```python
CACHES = {                       # Redis
    "default": {"BACKEND": "django_redis.cache.RedisCache", ...}
}
# later in Phase 4:
CELERY_BROKER_URL = ...          # Celery
EMAIL_HOST = "smtp.gmail.com"    # Email
REST_FRAMEWORK = {...}           # DRF
```

**Trigger 3 — a value is secret or machine-specific.**
DB password, API keys, Redis URL, DEBUG. Pull these OUT into `.env` via `config()`.
```python
from decouple import config
"LOCATION": config("REDIS_URL", default="redis://127.0.0.1:6379/0"),
```
`.env` holds the value; `.gitignore` keeps it out of git.
`default=` = safety net so the project still boots if `.env` is missing.

---

## 5. When to LEAVE IT ALONE

Most of `settings.py` is **Django's generated factory wiring** — treat as off-limits
unless a doc explicitly says otherwise:
`TEMPLATES`, `AUTH_PASSWORD_VALIDATORS`, `WSGI_APPLICATION`, `USE_TZ`, `SECRET_KEY` logic…

Beginners break projects by "tidying" these. The only generated values you edit early:
- `DEBUG`           — on for dev, off for prod
- `ALLOWED_HOSTS`   — when you deploy
- `DATABASES`       — when moving off SQLite (did this in Phase 3)
- `TIME_ZONE` / `LANGUAGE_CODE`

---

## 6. The decision flow (memorize this)

```
Adding any package?
│
├─ 1. uv add <package>          ← ALWAYS. pyproject.toml + installs it.
│
└─ 2. Read its docs:
      ├─ "add to INSTALLED_APPS"?   → add the line       (Trigger 1)
      ├─ "here's a CONFIG = {...}"? → add that block      (Trigger 2)
      ├─ value is secret/per-machine? → config() + .env   (Trigger 3)
      └─ just "import it"?          → DONE. Don't touch settings.py.
```

---

## 7. One-sentence version

> **Touch `settings.py` when wiring a project-wide capability — registering an app,
> configuring a library, or externalizing a secret. Page-specific stuff goes in
> views/urls; Django's generated defaults, leave alone.**

---

## 8. Phase 4 preview — you'll be back here a lot

Almost every upcoming concept STARTS with one `settings.py` change:

| Concept | settings.py change | Which trigger |
|---|---|---|
| Redis cache ✅ done | `CACHES` block + `config('REDIS_URL')` | 2 + 3 |
| weather app | `INSTALLED_APPS += 'weather'` | 1 |
| Celery | `CELERY_*` block | 2 + 3 |
| Celery Beat | `INSTALLED_APPS += 'django_celery_beat'` | 1 |
| Email/OTP | `EMAIL_*` block | 2 + 3 |
| Debug Toolbar | `INSTALLED_APPS` + `MIDDLEWARE` + `INTERNAL_IPS` | 1 |

---

## Recap of what we actually changed for Redis (Concept 1)

- `pyproject.toml` — added `django-redis`, `redis`, `python-decouple` (via `uv add`)
- `settings.py` — added `CACHES` block + `from decouple import config`, read `REDIS_URL` from `.env`
- `.env` — `REDIS_URL=redis://127.0.0.1:6379/0`  (Trigger 3: externalized config)
- `.gitignore` — added `.env` so secrets never get committed
- Verified: `redis-cli ping` → PONG, `cache.set/get` round-trip → `'world'` ✅
