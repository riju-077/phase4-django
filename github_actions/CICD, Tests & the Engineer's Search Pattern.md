# Phase 4 — Notes 04: GitHub Actions, CICD, Tests & the Engineer's Search Pattern

> Reference document. Everything we discussed about GitHub Actions, the CI/CD family, testing strategy, dependency decisions, and the **search-and-adapt** engineering mindset.

---

## 📚 Table of Contents

1. [Why this notes file exists](#1-why-this-notes-file-exists)
2. [GitHub Actions — the kitchen robot](#2-github-actions--the-kitchen-robot)
3. [Workflow anatomy — workflow / job / step / action / runner](#3-workflow-anatomy--workflow--job--step--action--runner)
4. [The search-and-adapt rule (most important section)](#4-the-search-and-adapt-rule-most-important-section)
5. [The Phase 4 CI workflow — line by line](#5-the-phase-4-ci-workflow--line-by-line)
6. [CI vs Tests — they work together, neither replaces the other](#6-ci-vs-tests--they-work-together-neither-replaces-the-other)
7. [The test pyramid](#7-the-test-pyramid)
8. [Where tests live in the repo](#8-where-tests-live-in-the-repo)
9. [pytest vs Django built-in test](#9-pytest-vs-django-built-in-test)
10. [The library decision framework — 5 questions](#10-the-library-decision-framework--5-questions)
11. [Dev vs Prod dependencies — the cost separation](#11-dev-vs-prod-dependencies--the-cost-separation)
12. [CI vs CD vs CD — precise definitions](#12-ci-vs-cd-vs-cd--precise-definitions)
13. [Phase 4 scope — Slice B](#13-phase-4-scope--slice-b)
14. [Reading GitHub Actions warnings](#14-reading-github-actions-warnings)
15. [The git credential warning explained](#15-the-git-credential-warning-explained)
16. [Rules of thumb](#16-rules-of-thumb)
17. [My confusions, cleared](#17-my-confusions-cleared)
18. [Search queries cheatsheet](#18-search-queries-cheatsheet)

---

# 1. Why this notes file exists

After setting up Redis (NOTES_01–03), we moved to **automation**: how to make a robot test your code on every push. This file is the complete reference for everything that touched.

After reading this, you should:
- Understand what GitHub Actions is, how it runs, and what every word in a workflow file means
- Know the engineering pattern for finding & adapting ANY config file template (not just YAML)
- Know the difference between CI, Continuous Delivery, and Continuous Deployment
- Decide intelligently about adding libraries to your project (the cost framework)
- Know where tests go and why they're never gitignored

---

# 2. GitHub Actions — the kitchen robot

## 📖 The story

Your hotel/bakery hired a tireless **kitchen robot**. Every time you change a recipe in the recipe book (push to GitHub), the robot:

1. **Wakes up automatically** (trigger fires)
2. **Walks into a brand-new kitchen** (a fresh Ubuntu VM, spun up in seconds)
3. **Follows the instructions you wrote** (the workflow YAML file)
4. **Reports back**: ✅ all good, or ❌ this step broke
5. **Then vanishes** — new robot, new kitchen, next time

That robot is **GitHub Actions**.

## What it actually is

A free CI/CD platform built into GitHub. Every public repo gets it. Every push or PR can trigger automated jobs. The jobs run on free virtual machines (called **runners**) hosted by GitHub.

## What it does for you

```
   Without GitHub Actions                With GitHub Actions
   ───────────────────────                ──────────────────────────
   
   • Push code                            • Push code
   • Hope it works                        • Robot tests on a clean VM
   • Maybe remember to run tests          • 60–90 seconds later, ✅ or ❌
   • Bugs reach prod days later           • Bugs caught the moment they're
   • "Did anyone test this?"                introduced
                                          • Audit trail of every change
                                          • Branch protection blocks bad
                                            merges
```

The robot turns *"discipline-based safety"* into *"automation-based safety."* That's the value.

---

# 3. Workflow anatomy — workflow / job / step / action / runner

The vocabulary that confuses every beginner. Let me lay it out cleanly.

## The hierarchy

```
   .github/workflows/ci.yml             ← THIS EXACT PATH or GitHub ignores it
              │
              ▼
   ┌─────────────────────────────────────────────────────────┐
   │  WORKFLOW                                                │
   │  ──────────                                              │
   │  name:    "CI"                ← shows up in GitHub UI    │
   │  on:                          ← TRIGGERS                 │
   │    push: branches [main]         "run when pushed"       │
   │                                                          │
   │  jobs:                        ← parallel work units      │
   │  ┌──────────────────────────────────────────────────┐    │
   │  │  JOB: "check"                                    │    │
   │  │  runs-on: ubuntu-latest    ← which machine        │   │
   │  │                                                  │    │
   │  │  steps:                    ← sequential actions  │    │
   │  │   1. Checkout code         (uses an action)      │    │
   │  │   2. Set up Python 3.12    (uses an action)      │    │
   │  │   3. Install dependencies  (runs a shell command)│    │
   │  │   4. Django check          (runs a shell command)│    │
   │  └──────────────────────────────────────────────────┘    │
   └──────────────────────────────────────────────────────────┘
```

## Each word, defined

| Word | Meaning |
|---|---|
| **Workflow** | The entire YAML file. One workflow = one automated process. |
| **Job** | A unit of work. Runs on its own fresh VM. Jobs run in **parallel** by default. |
| **Step** | One operation inside a job. Runs **sequentially**. |
| **Action** | A reusable, pre-built step (like `actions/checkout@v4`). Saves you from writing repetitive code. Use `uses:` to invoke. |
| **Runner** | The actual VM where a job runs. `ubuntu-latest` = a fresh Ubuntu Linux machine, free, provided by GitHub. |
| **Trigger** | What wakes the robot up. `on: push`, `on: pull_request`, `on: schedule` (cron), `on: workflow_dispatch` (manual button). |

## The magic folder location

GitHub Actions has a strict path rule:

```
   Your repo/
   └── .github/
       └── workflows/        ← EXACT path
           ├── ci.yml        ← can be any filename ending in .yml or .yaml
           ├── deploy.yml
           └── release.yml
```

Files anywhere else are ignored. The `.github/workflows/` location is the contract.

## Each workflow file = one workflow

You can have multiple workflows in the same repo:
- `ci.yml` — runs tests on every push
- `deploy.yml` — deploys when a release tag is created
- `nightly.yml` — runs every night at midnight to do reports

Each is independent.

---

# 4. The search-and-adapt rule (most important section)

This is the engineering mindset you asked to be taught. It applies to **any** config file you've never written before.

## 📖 The story — "The engineer's reflex"

You're a chef. Today's job: bake croissants. You've never baked them before.

**Bad engineer:** *"I'll invent the recipe from scratch."* → 12 hours, 3 failures, mediocre croissants.

**Good engineer:** *"There are millions of croissant recipes online. Find a trusted one. Read every step. Adapt for my oven. Test."* → 3 hours, one decent croissant, learned why each step exists.

> **Engineers don't memorise. They locate, adapt, test.**

## The rule applies to ALL of these

- `.github/workflows/*.yml` (GitHub Actions)
- `Dockerfile`
- `docker-compose.yml`
- `nginx.conf`, `Caddyfile`
- `pyproject.toml`, `package.json`
- `tsconfig.json`
- Helm charts, Terraform, Ansible
- `pytest.ini`, `pyproject.toml [tool.pytest]`
- Database migration files
- Anything ending in `.yml`, `.yaml`, `.toml`, `.conf`, `.json`, `.ini`

## The 5-step pattern

```
   STEP 1 — Identify WHAT TYPE of config you need
   ─────────────────────────────────────────────
   "I need a GitHub Actions workflow for a Django project."
   
   Be specific. The more precise the question, the better the search.


   STEP 2 — Go to the OFFICIAL source FIRST
   ────────────────────────────────────────
   Not Stack Overflow. Not random blogs. The official docs.

   For GitHub Actions:    https://docs.github.com/en/actions
   For Docker:            https://docs.docker.com
   For Django:            https://docs.djangoproject.com
   For pytest:            https://docs.pytest.org
   For nginx:             https://nginx.org/en/docs/
   
   Pattern: search "official docs for X 2025" (year keeps it current)


   STEP 3 — Look for STARTER TEMPLATES
   ───────────────────────────────────
   Most tools provide ready-made templates. Use them as a base.

   GitHub Actions starters: https://github.com/actions/starter-workflows
                            (literally a folder of templates per language)

   Docker official guides:  https://docs.docker.com/language/python/
   
   Inside GitHub itself:    Repo → Actions tab → "set up a workflow yourself"
                            shows templates in a sidebar.

   For Django CI specifically:
   https://github.com/actions/starter-workflows/blob/main/ci/django.yml


   STEP 4 — Read EVERY line, understand, adapt
   ──────────────────────────────────────────
   This is the engineering muscle. For each line, ask:
   
   • "What does this do?"
   • "Do I need it for my project?"
   • "Should this value be different for me?"
   
   Adapt for YOUR project:
     - Python version
     - Branch names
     - Dependencies (requirements.txt vs Pipfile vs pyproject.toml)
     - Test command
     - Environment-specific variables

   NEVER paste blindly. The "I'll just use this whole template" mindset
   is how copy-pasted insecure configs ship to production.


   STEP 5 — Push and watch the robot
   ────────────────────────────────
   Push the file → GitHub Actions runs → ✅ green or ❌ red
   
   If red: read the failure logs line by line. The error message tells
   you what's wrong. Don't guess. Don't randomly tweak.
   
   Iterate. Each red → fix → green is a small learning win.
```

## The thumb rule

> **For every config file you don't already know cold:**
>
> 1. Search official docs / starter templates first
> 2. Read EVERY line — what does it do, do I need it?
> 3. Adapt for YOUR project's specifics
> 4. Test (push, run, watch)
> 5. Iterate
>
> **Never invent. Never blindly paste. Always understand.**

That's the engineer's reflex. Build that muscle, and you can pick up any new tool's config in hours instead of weeks.

---

# 5. The Phase 4 CI workflow — line by line

The file we created: `.github/workflows/ci.yml`.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  check:
    name: Django system check
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run Django system check
        run: python manage.py check
```

## Walkthrough

```yaml
name: CI
```
Display name on the GitHub Actions UI. Pick anything.

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```
**Triggers.** Robot wakes when:
- Someone pushes commits to `main`
- Someone opens a PR targeting `main`

Add more branches in the list if you want CI on dev branches too: `branches: [main, dev, staging]`.

```yaml
jobs:
  check:
    name: Django system check
    runs-on: ubuntu-latest
```
One job, internally named `check`, display label "Django system check," runs on a fresh Ubuntu VM.

> Multiple jobs run in **parallel** by default. We have just one.

```yaml
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
```
**Step 1.** The VM starts empty — no code. `actions/checkout@v4` is a pre-built action that clones your repo into the VM.

`@v4` is a version pin (same rule as Docker image tags).

```yaml
      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
```
**Step 2.** Installs Python 3.12 on the VM. `cache: 'pip'` caches pip downloads between runs → faster installs after the first.

```yaml
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
```
**Step 3.** `run:` runs shell commands. `|` is YAML for "multi-line block." Two commands run sequentially.

```yaml
      - name: Run Django system check
        run: python manage.py check
```
**Step 4.** Django's built-in check command — validates settings, verifies INSTALLED_APPS load, catches common config errors. Exit code 0 = pass; non-zero = fail.

## What this gives you, even minimally

```
   Before:                                After:
   ─────────                              ──────────
   You push code.                         You push code.
   You hope it works.                     Robot tries it on a clean
                                          machine.
                                          Reports in ~60 seconds.
                                          
                                          Green ✅ = project at least loads.
                                          Red ❌ = something broke,
                                                   logs tell you what.
```

This is the **skeleton**. We grow it over Phase 4 (add migrations check, add pytest, add Postgres/Redis services, add Docker build, add image push).

---

# 6. CI vs Tests — they work together, neither replaces the other

## 📖 The story

The kitchen robot (CI) doesn't have taste buds. It can only check whether each step in the recipe executed without errors. It doesn't know if the bread is good.

The **QA inspectors** (your unit tests) are the ones with taste buds:
- Inspector 1: "Does the dough rise?"
- Inspector 2: "Is the temperature right?"
- Inspector 3: "Is the crust crispy?"
- Inspector 4: "Does it taste like bread or cardboard?"

Each inspector checks ONE specific thing.

**The relationship:**

> *"The robot's recipe says: at the end of every bake, call all 50 inspectors. Each inspector reports pass/fail. If any fail, the bake is marked failed."*

The robot runs the inspectors. **Tests are the inspectors. CI is the recipe that calls them.**

## In one sentence

> **Tests are what you write. CI is the automation that runs them on every push.**

## What each catches — alone

```
   ALONE: Just CI (no tests)               ALONE: Just tests (no CI)
   ──────────────────────────              ──────────────────────────
   ✅ Catches:                              ✅ Catches:
   • Syntax errors                          • Logic bugs in tested code
   • Import failures                        • Regressions IF you remember
   • Missing dependencies                     to run tests
   • Broken settings.py                     • Edge cases you wrote tests for
   
   ❌ Misses:                              ❌ Misses:
   • All logic bugs                         • Untested code paths
   • Wrong calculations                     • Bugs from teammates who didn't run
   • Database query mistakes                • "I forgot to run them" moments
   • API contract changes
   • Edge cases
   
   → Guard rail. Doesn't                   → Quality measure but
     measure correctness.                    requires discipline.
```

## Together

```
   CI + TESTS — the real safety net
   ─────────────────────────────────
   
   ✅ Catches everything above
   ✅ Bugs detected within minutes of being introduced
   ✅ No one can "forget" to run tests — robot always does
   ✅ Failed tests can block PR merges (with branch protection)
   ✅ Confidence to refactor — green = nothing broke
   
   This is what "production-grade engineering" looks like.
```

## The mental model

```
   WHAT YOU WRITE              WHEN/WHERE IT RUNS
   ──────────────              ───────────────────

   ┌─────────────────────┐    ┌─────────────────────────────┐
   │  TESTS (pytest /    │    │  CI (GitHub Actions)        │
   │   Django test)      │    │                             │
   │                     │    │  • Spins up clean VM        │
   │  • test_user_can_   │    │  • Installs Python + deps   │
   │    sign_up()        │    │  • Runs `pytest`            │
   │  • test_otp_expires │    │    (which runs ALL          │
   │    _after_5min()    │    │     your tests at once)     │
   │  • test_cache_      │    │  • Reports green/red        │
   │    invalidation()   │    │                             │
   │                     │    │                             │
   │  YOU author these.  │    │  CI runs them               │
   │  They live in your  │    │  automatically on every     │
   │  code repo.         │    │  push and PR.               │
   └─────────────────────┘    └─────────────────────────────┘
            │                              │
            └──────────┬───────────────────┘
                       ▼
            ┌─────────────────────────────┐
            │  TOGETHER:                  │
            │  Every git push triggers    │
            │  the robot, who runs every  │
            │  test you've written.       │
            └─────────────────────────────┘
```

---

# 7. The test pyramid

```
                          ╱│╲              ▲  fewer, slower,
                         ╱ │ ╲             │  more brittle
                        ╱E2E╲             │
                       ╱─────╲            │  test the whole flow
                      ╱       ╲           │  ("user signs up and
                     ╱Integra- ╲          │   sees dashboard")
                    ╱  tion     ╲         │
                   ╱─────────────╲        │  test interactions
                  ╱               ╲       │  ("when post_save fires,
                 ╱   UNIT TESTS    ╲      │   email is sent")
                ╱___________________╲     ▼  many, fast,
                                              focused, cheap
                                              ("function returns
                                              correct sum")
```

| Layer | What it tests | Speed | Count |
|---|---|---|---|
| **Unit** | A single function or method | Milliseconds | Hundreds–thousands |
| **Integration** | Two or more components working together (e.g., signal fires → email sent) | Seconds | Dozens |
| **E2E (end-to-end)** | The full user journey through the entire app | Minutes | Tens |

CI runs all of them. They serve different purposes.

> **For Phase 4:** mostly **unit tests** for individual functions/models, with a few **integration tests** when Celery comes in.

---

# 8. Where tests live in the repo

## 🚨 The number one rule

**Tests are FIRST-CLASS code. ALWAYS pushed to GitHub. NEVER gitignored.**

Gitignoring tests is a newbie disaster. Tests:
- Document expected behavior
- Are run by CI on every push (must be in repo)
- Are reviewed by teammates in PRs
- Are required for refactoring confidence
- Are part of the code contract

## Three common patterns

### Pattern A — Per-app `tests.py` (Django default, small projects)

```
   myproject/
   ├── manage.py
   ├── weather/                ← Django app
   │   ├── models.py
   │   ├── views.py
   │   ├── tests.py            ← ← ← all tests for this app
   │   └── ...
```

Django default. Run with `python manage.py test`.

### Pattern B — Per-app `tests/` folder (medium projects)

```
   myproject/
   ├── weather/
   │   ├── models.py
   │   ├── views.py
   │   └── tests/              ← folder, not single file
   │       ├── __init__.py
   │       ├── test_models.py
   │       ├── test_views.py
   │       └── test_signals.py
```

When `tests.py` grows beyond ~300 lines, split it.

### Pattern C — Centralized `tests/` at project root (pytest favorite)

```
   myproject/
   ├── manage.py
   ├── weather/                ← app code
   │   ├── models.py
   │   └── views.py
   └── tests/                  ← all tests in one place
       ├── conftest.py
       ├── test_models.py
       └── test_celery_tasks.py
```

Common in pytest-driven projects. Easier to find; harder when code moves.

## What we'll use in Phase 4

Likely **Pattern B** — per-app `tests/` folder. Django-idiomatic, pytest-friendly, scales well.

## Naming conventions

| Tool | File names | Function names |
|---|---|---|
| Django built-in | `tests.py` or `test_*.py` | `def test_<thing>(self):` (inside `TestCase` class) |
| pytest | `test_*.py` or `*_test.py` | `def test_<thing>():` (no class needed) |

Both tools auto-discover files starting with `test_`. Stick to that prefix.

---

# 9. pytest vs Django built-in test

Two ways to run tests in Django.

| Tool | Command | Notes |
|---|---|---|
| **Django built-in** | `python manage.py test` | Uses Python's `unittest`. Comes with Django. No extra install. |
| **pytest + pytest-django** | `pytest` | Most popular in modern Python. Better fixtures, cleaner syntax, plugin ecosystem (pytest-cov, pytest-mock, etc.) |

## Why pytest wins for most modern Django projects

```
   Django built-in (unittest-style)              pytest
   ────────────────────────────                   ──────
   
   class UserTestCase(TestCase):                  def test_user_can_sign_up(client):
       def setUp(self):                              response = client.post('/signup', ...)
           self.user = User.objects.create(...)      assert response.status_code == 200
       
       def test_user_can_sign_up(self):           # Simpler. Plain functions. No class boilerplate.
           response = self.client.post(...)       # Use fixtures (decorator) for setup.
           self.assertEqual(response.status, 200) # Better error output on failure.
                                                  # Plugins for coverage, mocking, async, etc.
```

## When Django built-in is fine

- You're learning, want zero extra deps
- Tiny project
- Strong unittest discipline already

## When pytest is worth it

- Anything beyond toy size
- You want clean fixtures and parametrized tests
- You want plugins (`pytest-cov` for coverage, `pytest-mock` for mocking)
- Your team uses pytest (most modern teams do)

## Our Phase 4 choice — pytest + pytest-django

Reasoning:
- Industry standard for new Python projects in 2025
- Small dev-dep cost (~6 MB)
- Confined to dev deps — doesn't bloat production
- Better DX, better plugins, better future-proofing

**When we get there**, we'll install via:
```powershell
uv add --dev pytest pytest-django
```

And configure `pyproject.toml` to tell pytest where Django settings are.

---

# 10. The library decision framework — 5 questions

This is the engineering muscle for "should I add this library?"

## The 5 questions

```
   ┌──────────────────────────────────────────────────────────────┐
   │  BEFORE adding any library, ask:                             │
   │                                                              │
   │  1. WHAT PROBLEM does this solve?                             │
   │     ▼ Is the problem real or am I optimising prematurely?   │
   │                                                              │
   │  2. WHAT DOES IT COST?                                       │
   │     ▼ Size (MB), learning curve, maintenance burden          │
   │     ▼ Transitive dependencies (libs of the lib)              │
   │                                                              │
   │  3. CAN IT BE CONFINED TO DEV?                               │
   │     ▼ Tests, linters, formatters → dev deps only             │
   │     ▼ They don't ship to prod, so runtime cost = ~0          │
   │                                                              │
   │  4. WHAT DOES THE COMMUNITY DO?                              │
   │     ▼ Search: "django pytest 2025"                           │
   │     ▼ Look at popular Django projects on GitHub             │
   │     ▼ Check maintenance: last commit, open issues           │
   │                                                              │
   │  5. WHAT'S THE ALTERNATIVE?                                  │
   │     ▼ Built-in / standard library version                    │
   │     ▼ A smaller library                                      │
   │     ▼ Writing it yourself (sometimes correct!)               │
   │                                                              │
   │  Decision: value > total cost? → adopt. Else → skip.         │
   └──────────────────────────────────────────────────────────────┘
```

## Applied to pytest vs Django built-in

| Question | Django built-in | pytest + pytest-django |
|---|---|---|
| **Problem solved?** | Basic test running | Better fixtures, parametrization, plugins |
| **Cost?** | $0 — comes with Django | ~6 MB total deps, dev-only |
| **Dev-only?** | N/A | ✅ YES — never ships to prod |
| **Community?** | Used; pytest dominant for new projects | Industry standard in 2025 |
| **Alternative?** | This IS the alt | unittest (built-in), nose (dead) |

**Decision:** pytest's value clearly exceeds cost. Adopt.

## When NOT to add a library

- The problem is trivial — write 10 lines yourself
- The library has 1 maintainer who's stopped responding
- It pulls in 50 transitive dependencies for a tiny feature
- It's the trending JS-style "one library for one tiny thing" pattern

> **Rule of thumb:** Every dependency has a cost. Question every addition. Confine to dev when possible. Audit periodically.

---

# 11. Dev vs Prod dependencies — the cost separation

## 🚨 KEY INSIGHT — Adding to `requirements.txt` does NOT bloat your repo

```
   ┌──────────────────────────────────────────────────────────────┐
   │  Your REPO contains:                                         │
   │  • Code files (your .py files)                              │
   │  • requirements.txt (just a list of NAMES)                  │
   │  • Configs                                                  │
   │                                                              │
   │  The actual library CODE lives in:                          │
   │  • .venv/  ← gitignored on your laptop                      │
   │  • Inside the Docker image when built                       │
   │  • NOT in the repo                                          │
   │                                                              │
   │  Adding a library name to requirements.txt = a few bytes.   │
   │  The library binary code never enters the repo.             │
   └──────────────────────────────────────────────────────────────┘
```

## The real question — does it bloat the PRODUCTION runtime?

```
   PRODUCTION DEPS                  DEV DEPS
   ────────────────                  ────────
   django                            pytest
   djangorestframework               pytest-django
   redis                             black (formatter)
   celery                            ruff (linter)
   httpx                             coverage
                                     ipython
   
   Ship to production image.         Only installed on dev
   Run inside Docker.                laptops + CI runners.
   Bloat = bad here.                 Never reach prod.
                                     Bloat = fine here.
```

## How to separate them in modern Python

### Option 1 — Two separate text files (simple)

```
   requirements.txt           ← production deps only
   requirements-dev.txt       ← dev tools, references prod
```

`requirements-dev.txt`:
```
-r requirements.txt
pytest
pytest-django
ruff
```

Install on laptop: `pip install -r requirements-dev.txt`
Install in production Docker image: `pip install -r requirements.txt`

### Option 2 — `pyproject.toml` with groups (modern, what `uv` uses)

```toml
[project]
dependencies = [
    "django>=5",
    "djangorestframework",
    "redis",
    "celery",
    "httpx",
]

[project.optional-dependencies]
dev = [
    "pytest",
    "pytest-django",
    "ruff",
]
```

Install on laptop:
```powershell
uv pip install -e ".[dev]"
```

Install in production Docker image:
```powershell
uv pip install -e .
```

The `[dev]` group is **optional** — production install skips it. Clean separation.

### Option 3 — uv groups (latest)

```toml
[dependency-groups]
dev = ["pytest", "pytest-django", "ruff"]
```

uv handles them automatically. Even cleaner.

## Why this matters

```
   WITHOUT separation                    WITH separation
   ──────────────────                    ────────────────
   
   Production image:                     Production image:
   • Django                               • Django
   • DRF                                  • DRF
   • Redis                                • Redis
   • Celery                               • Celery
   • httpx                                • httpx
   • pytest          ← dead weight       
   • pytest-django   ← dead weight       
   • ruff            ← dead weight       
   • coverage        ← dead weight       
   
   Image size: +200 MB unnecessary       Image size: production-lean
   Attack surface: bigger                 Attack surface: minimal
```

> **Engineering rule:** Any tool ONLY used during development (tests, linters, formatters, type checkers, debuggers) → dev deps. Anything the running application needs → prod deps. NEVER cross-contaminate.

---

# 12. CI vs CD vs CD — precise definitions

The acronyms are confusing because **"CD" means two different things.**

```
   CI  — Continuous INTEGRATION    Run tests on every code change.
                                    Verify the codebase stays healthy.

   CD  — Continuous DELIVERY        Automatically prepare DEPLOYABLE
                                    artifacts (Docker images, packages).
                                    Push to a registry, ready for someone
                                    to deploy.

   CD  — Continuous DEPLOYMENT      Same as Delivery + automatically deploy
                                    to live production servers when CI passes.
```

## How they stack

```
   ┌──────────────────────────────────────────────────────────────┐
   │  CI (run tests)                                              │
   │  ────────────────                                            │
   │  Every push → robot runs tests → ✅ or ❌                    │
   │                                                              │
   │  └─► If ✅ ...                                              │
   │                                                              │
   │      ┌──────────────────────────────────────────────────┐    │
   │      │  CONTINUOUS DELIVERY                             │    │
   │      │  ─────────────────────                           │    │
   │      │  • Build Docker image                            │    │
   │      │  • Tag with commit SHA                           │    │
   │      │  • Push to registry (ghcr.io / Docker Hub)      │    │
   │      │  • Now ANYONE can manually pull and deploy      │    │
   │      └──────────────────────────────────────────────────┘    │
   │                                                              │
   │           └─► If you ALSO want auto-deploy ...               │
   │                                                              │
   │               ┌──────────────────────────────────────┐       │
   │               │  CONTINUOUS DEPLOYMENT                │      │
   │               │  ─────────────────────                │      │
   │               │  • SSH into prod server               │      │
   │               │  • docker compose pull                │      │
   │               │  • docker compose up -d               │      │
   │               │  • Live in production. No humans.    │      │
   │               └──────────────────────────────────────┘      │
   └──────────────────────────────────────────────────────────────┘
```

Each layer builds on the previous.

---

# 13. Phase 4 scope — Slice B

What we agreed on:

```
   PHASE 4 = full CI + Continuous DELIVERY
            (no auto-deployment)

   ✅ CI (tests on push)
   ✅ Continuous Delivery (build + push image to ghcr.io)
   ❌ Continuous Deployment (auto-deploy to live server)
```

## Why not Slice C (full Continuous Deployment)?

Slice C would require:
- A real VPS (~$6/month)
- SSH key management
- HTTPS cert setup (Let's Encrypt)
- Server hardening
- Firewall config

That's a separate curriculum that distracts from learning Django/Celery/Redis. We can layer it on after Phase 4 ends — it's a one-weekend project once the rest is in place.

## The Phase 4 growth roadmap

```
   Day 1 (today):     CI runs `manage.py check`
                      ▲ you're here

   After models:      + Migrations check (no missing migrations)

   After first test:  + pytest run

   After Celery:      + spin up Postgres + Redis as service containers
                       in the CI workflow during tests

   Near end:          + Build Docker image of Django app

   Final:             + Push image to ghcr.io
                      ▲ end of Phase 4

   ─────────────────────────────────────────────────────

   Post-Phase 4 (optional, Slice C):
                      Auto-deploy from ghcr.io to a real VPS
                      on every successful CI run.
```

---

# 14. Reading GitHub Actions warnings

Warnings appear in the "Annotations" section of a workflow run.

## Example from your first run

```
   ⚠️ Django system check
   Node.js 20 actions are deprecated. The following actions are running
   on Node.js 20 and may not work as expected: actions/checkout@v4,
   actions/setup-python@v5.
```

## What it means

The pre-built actions (`actions/checkout@v4`, `actions/setup-python@v5`) internally use Node.js 20 as their runtime. GitHub is phasing out Node 20 support. The actions still WORK — they just may not work forever.

**This is not your fault.** It's about how those actions are built internally.

## How to handle it (the engineering reflex)

```
   Warning text  →  Google "github actions [exact warning text]"
                 →  Read the official deprecation notice
                 →  Check if action has a new version
                 →  Bump the @v4 → @v5 if available
                 →  Push → robot retries
```

For this specific warning — there's no new version yet that resolves the Node issue. So you do nothing and wait. ⏳

## The general rule

> **Warnings are not errors.** Errors break the build. Warnings tell you *"this works but may not forever."*
>
> When you see one, Google the exact text. Don't ignore, don't panic.

---

# 15. The git credential warning explained

You saw this on `git push`:

```
   git: 'credential-manager-core' is not a git command.
```

## What's happening

Git needs to remember your GitHub login (so you don't type your password every push). It uses a **credential helper** — a small companion program that stores credentials.

```
   You type:        git push
        │
        ▼
   Git says:        "I need to authenticate to github.com"
        │
        ▼
   Git looks in     "Which credential helper should I call?"
   its config:      → It finds: `credential-manager-core`
        │
        ▼
   Git tries:       "Run program: credential-manager-core"
        │
        ▼
   Windows:         "Sorry, no such program installed."
        │
        ▼
   Git prints:      "git: 'credential-manager-core' is not a git command."
        │
        ▼
   Git falls back:  Uses a different auth method.
        │
        ▼
   Push succeeds. ✅
```

## Why this happened

GitHub renamed their tool a few years ago:
- **Old name:** `git-credential-manager-core`
- **New name:** `git-credential-manager` (dropped `-core`)

Your Git config still references the old name. Git gracefully falls back, but prints the warning each time.

## What "harmless" means

> *"The symptom (warning text) appears, but the function (authentication) still works through a fallback path. Nothing was broken, no data was lost — the warning is just noise."*

## The one-line fix

```powershell
git config --global credential.helper manager
```

Verify after:
```powershell
git config --global credential.helper
```

Should now say `manager` (without `-core`).

---

# 16. Rules of thumb

| # | Rule |
|---|---|
| 1 | For ANY config file you don't know cold → search official docs → adapt → test. Never invent. |
| 2 | The `.github/workflows/` path is mandatory. Files anywhere else are ignored. |
| 3 | Pin action versions (`@v4`, `@v5`). Same rule as Docker tags. Never use bare names. |
| 4 | Tests are first-class code. ALWAYS in git. NEVER gitignored. |
| 5 | CI ≠ tests. Tests verify correctness; CI runs them automatically. You need both. |
| 6 | Adding to `requirements.txt` does NOT bloat your repo — only the production runtime IF the lib goes into the prod image. |
| 7 | Dev tools (pytest, linters, formatters) → dev deps. NEVER mixed with production. |
| 8 | "CD" is ambiguous: Continuous Delivery (prepare artifacts) vs Continuous Deployment (auto-deploy live). Know which one you mean. |
| 9 | Phase 4 = CI + Continuous Delivery (image to ghcr.io). Not Continuous Deployment. |
| 10 | Warnings ≠ errors. When you see a warning, Google the exact text. Don't ignore, don't panic. |
| 11 | Every dependency has a cost. Apply the 5-question framework before adding. |
| 12 | Engineers don't memorise config — they locate, adapt, test. That's the muscle. |

---

# 17. My confusions, cleared

### Q1. "I wrote my first workflow YAML — what did I just do?"

**A:** You taught the GitHub Actions robot a recipe. Every time you push to `main` (or open a PR), the robot wakes up, spins up a clean Ubuntu VM, follows your recipe (checkout code, install Python, install deps, run Django check), and reports pass/fail. That's CI.

### Q2. "Does using CI mean we don't need unit testing?"

**A:** No. CI and tests are different things that work together. CI is automation that RUNS whatever you tell it to. Tests are the things being run. Without tests, CI only catches startup/syntax issues. Together, they make a real safety net.

### Q3. "In production, do test files get pushed to GitHub or gitignored?"

**A:** Tests are ALWAYS in git. Never gitignored. They're documentation, regression guards, code review evidence, and CI input. Gitignoring tests is a major newbie mistake.

### Q4. "Where do tests live in the repo?"

**A:** Three common patterns: per-app `tests.py` file (small projects), per-app `tests/` folder (medium), or centralized `tests/` at project root (pytest favorite). Phase 4 will use per-app `tests/` folder.

### Q5. "Should I use Django built-in tests or pytest?"

**A:** pytest for most modern projects. Better fixtures, cleaner syntax, plugin ecosystem. Industry standard. Small cost (~6 MB, dev-only). Phase 4 uses pytest.

### Q6. "Won't adding pytest bloat my repo and make it less optimal?"

**A:** Adding to `requirements.txt` doesn't bloat the repo (the lib code never enters the repo). It can bloat the production runtime — BUT only if you don't separate dev from prod deps. Use `pyproject.toml` groups or `requirements-dev.txt` to keep them apart. pytest never reaches prod.

### Q7. "Are we doing only CI for Phase 4, no CD?"

**A:** We're doing CI + Continuous Delivery (build image, push to registry). We're NOT doing Continuous Deployment (auto-deploy to a live server). That's Slice C and requires a VPS — out of scope for Phase 4.

### Q8. "What does 'credential-manager-core is not a git command' mean and why is it harmless?"

**A:** Git was configured to call an old version of its credential helper. That binary doesn't exist anymore (renamed). Git printed a warning, then fell back to a working method, push succeeded. "Harmless" = visible noise, no functional damage. Fix: `git config --global credential.helper manager`.

### Q9. "How do I read a GitHub Actions warning like 'Node.js 20 deprecated'?"

**A:** Warnings tell you something works now but may not forever. Google the exact text. Check if affected actions have newer versions. Bump if so. If not, wait — the maintainers will release updates eventually.

### Q10. "What's the engineering pattern for writing any config file I've never written before?"

**A:** The 5-step search-and-adapt rule:
1. Identify what type of config you need
2. Go to official docs first
3. Find a starter template
4. Read every line, understand, adapt for YOUR project
5. Push and watch the robot

Never invent. Never blindly paste. Always understand. That's the engineering muscle.

---

# 18. Search queries cheatsheet

For when you forget how to find things.

| Situation | Search query |
|---|---|
| Want a GitHub Actions workflow for X | `github actions [X] ci yaml example` |
| Looking for official starter templates | `github actions starter workflows` → land on https://github.com/actions/starter-workflows |
| Action version is deprecated | Search the exact warning text |
| Want to know what an action does | `actions/[name]` on github.com (e.g., https://github.com/actions/checkout) |
| Want a Docker Hub image | `[name] official docker hub` |
| Want to compare two libraries | `[lib A] vs [lib B] 2025 python` |
| Want pytest + Django setup | `pytest django setup tutorial 2025` |
| Want to fix any CI error | Copy the exact error → Google → first result is usually GitHub Discussions or Stack Overflow |
| Want best practices for a tool | `[tool] best practices 2025` |
| Want to know if a library is alive | Check its GitHub repo: last commit, open issues, recent releases |

## The golden rule one more time

> **Docs are the source of truth. Random blog posts are second-class. When docs and a blog disagree, the docs win. Always start at the docs.**

---

## 📌 Where this fits in Phase 4

You now have:
- A working CI pipeline (`manage.py check` runs on every push) ✅
- A clear understanding of GitHub Actions, jobs, steps, runners ✅
- The search-and-adapt engineering muscle ✅
- The dev/prod dependency separation pattern ready to apply ✅
- A roadmap for growing CI as features ship ✅

Next up: actually building Django models + integrating Redis. Then we'll write the first pytest test and watch CI grow.

---

*Document created during Phase 4 GitHub Actions / CI setup conversation. The first green ✅ in your repo's Actions tab was the milestone this anchors.*
