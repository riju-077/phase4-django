# Phase 4 — Notes 01: Docker, Redis, Cloud & Engineering Mindset

> Reference document. Everything we discussed before writing a single line of Django code.
> Read top-to-bottom the first time. After that, jump to whatever section you need.

---

## 📚 Table of Contents

1. [Redis — the in-memory storeroom](#1-redis--the-in-memory-storeroom)
2. [Docker — the magic box](#2-docker--the-magic-box)
3. [Dockerfile vs Image vs Container vs Compose](#3-dockerfile-vs-image-vs-container-vs-compose)
4. [The full Dev → Deploy lifecycle](#4-the-full-dev--deploy-lifecycle)
5. [Stateless vs Stateful — the production split](#5-stateless-vs-stateful--the-production-split)
6. [Native install vs Docker — the "daily driver" rule](#6-native-install-vs-docker--the-daily-driver-rule)
7. [Server vs Client — the universal decoupling pattern](#7-server-vs-client--the-universal-decoupling-pattern)
8. [Rules of Thumb (consolidated)](#8-rules-of-thumb-consolidated)
9. [My confusions, cleared](#9-my-confusions-cleared)
10. [Engineer-mindset search cheatsheet](#10-engineer-mindset-search-cheatsheet)

---

# 1. Redis — the in-memory storeroom

## 📖 The story — "The Coffee Shop"

You run a coffee shop. A customer asks *"How many cups did you sell yesterday?"*

- **Option A** — Walk to the back, open the filing cabinet, count receipts (3 minutes).
- **Option B** — First time, count once. Write **"Yesterday: 473"** on a sticky note on the counter. Next 200 customers? Glance at the sticky note (1 second).

| Concept | Real-world thing |
|---|---|
| Filing cabinet | **PostgreSQL** — slow, on disk, stores everything forever, survives power cuts |
| Sticky note | **Redis** — fast, in RAM, temporary, vanishes on power loss (that's fine — re-count from the cabinet) |

## What it is

**Redis** = **RE**mote **DI**ctionary **S**erver.

A program that:
- Holds **key-value pairs in RAM**
- Lives in its own process
- Listens on a network port (default `6379`)
- Supports per-key **expiry timers** ("sticky notes that fall off after 5 min")

It's literally a Python dict that lives on a network port.

## Why it's needed (two jobs in Phase 4)

```
┌─────────────────────────────────────────────────────────────┐
│                         REDIS                               │
│                                                             │
│   JOB 1: CACHE              JOB 2: MESSAGE BROKER          │
│   ┌────────────────┐        ┌──────────────────────┐       │
│   │ Sticky note    │        │ Queue of jobs        │       │
│   │ "dashboard_    │        │ "fetch_weather:      │       │
│   │  stats" → 473  │        │  London, 51.5, -0.1" │       │
│   └────────────────┘        └──────────────────────┘       │
│         ▲                            ▲                      │
│         │ Django reads/writes        │ Celery worker pulls │
│         │ to skip slow DB queries    │ jobs from here      │
└─────────────────────────────────────────────────────────────┘
```

## What goes wrong WITHOUT Redis

- 100 dashboard visitors → 100 identical 500ms Postgres aggregations → server cries.
- With Redis: first hit fills the cache, next 99 read from RAM in 1ms. **100× faster.**

## What goes wrong with Redis used WRONG

| Mistake | Pain |
|---|---|
| Forget to `cache.delete()` when underlying data changes | Users see stale data for an hour |
| `cache.set()` with no timeout | Keys live forever, Redis runs out of RAM, server dies |

## 🧠 Memorize: The cache-or-compute pattern

```
  ┌─────────┐
  │ Request │
  └────┬────┘
       │
       ▼
  ┌─────────────────────┐
  │ cache.get('key')    │
  └────┬────────────────┘
       │
       ├──── HIT (data found) ──────► return data  ⚡ fast path
       │
       └──── MISS (returns None) ───► compute it (DB / API)
                                         │
                                         ▼
                                    cache.set('key', data, timeout=300)
                                         │
                                         ▼
                                      return data
```

> **Rule:** Every `cache.set()` MUST have a timeout. Treat "no expiry" like "no seatbelt."

---

# 2. Docker — the magic box

## 📖 The story — "The Italian Pizza Oven"

You ordered a pizza oven from Italy. It expects Italian voltage, Italian gas hookup, Italian stone. If you tried to install it natively in your kitchen — you'd rewire, replumb, break things.

**Docker is a magic box.** Drop the oven inside. Plug the *box* into your kitchen wall. Box handles all the translation. Your kitchen never gets touched.

That's a **container**. The pizza oven = Redis (or Postgres, or your Django app). The box = the container. Your kitchen = your laptop.

## What it is

Docker = a program that runs **isolated little environments** ("containers") on your machine. Each container has its own filesystem, libraries, processes — but shares your kernel. Lighter than a VM, heavier than a process.

## Why it matters

| Without Docker | With Docker |
|---|---|
| "Install Postgres 16, MySQL 8, Redis 7 on your laptop. Hope nothing conflicts." | One YAML file. `docker compose up`. Done. |
| "Works on my machine" bugs in prod | Same container runs locally and in cloud. Identical. |
| Cleaning up: registry hell, leftover services | `docker rm`. Gone. |

## Docker Desktop vs Docker Engine — critical distinction

```
   DOCKER DESKTOP                       DOCKER ENGINE
   (your Windows/Mac laptop)            (the Linux server in cloud)

   ┌──────────────────────────┐         ┌──────────────────────────┐
   │  GUI (whale icon)        │         │  No GUI, just CLI        │
   │  + Docker Engine inside  │         │  apt install docker      │
   │  + WSL2 magic            │         │                          │
   │  + Settings panel        │         │  Runs your containers    │
   │                          │         │  Same way as Desktop     │
   │  Paid for big companies  │         │  Free (open source)      │
   └──────────────────────────┘         └──────────────────────────┘
       DEV-ONLY tool                        PRODUCTION tool
```

> **Insight:** Docker Desktop is the friendly Windows/Mac wrapper. The actual *engine* underneath is what runs in production. On Linux servers, no "Desktop" needed — just the `docker` CLI. Same containers run identically.

---

# 3. Dockerfile vs Image vs Container vs Compose

The four words you'll keep confusing. Here's the map.

```
   DOCKERFILE  ───docker build──►  IMAGE  ───docker run──►  CONTAINER
   (recipe)                        (built dish,             (live, running
                                    static artifact)         instance)

   docker-compose.yml = orchestrates MULTIPLE containers together
```

## Detailed roles

| Word | What it is | Analogy |
|---|---|---|
| **Dockerfile** | Text recipe describing how to build an image | A cooking recipe |
| **Image** | Built, immutable artifact (think `.zip` of an entire OS + your app) | The cooked, packaged dish |
| **Container** | A running instance of an image | A served plate of the dish |
| **docker-compose.yml** | YAML config that runs multiple containers together with one command | The full restaurant menu — orders many dishes at once |

## You write a Dockerfile ONLY for your own code

```yaml
# docker-compose.yml — what you'll write later
services:
  backend:
    build: ./backend      # uses YOUR Dockerfile
  frontend:
    build: ./frontend     # uses YOUR Dockerfile
  db:
    image: postgres:16    # just pulls the official image, no Dockerfile needed
  redis:
    image: redis:7        # same — no Dockerfile
```

> **Rule:** Write a Dockerfile for code YOU wrote. For third-party services (Postgres, Redis, MongoDB), just reference the official image. Never reinvent.

---

# 4. The full Dev → Deploy lifecycle

The "what happens between writing code and seeing it on the internet" picture.

```
   YOUR LAPTOP                         GIT                            CI/CD                       REGISTRY                      SERVER
  ┌──────────────────┐              ┌────────┐                  ┌──────────────┐             ┌──────────┐               ┌─────────────┐
  │  Code + Docker-  │   git push   │ GitHub │   triggers       │ GitHub       │   pushes    │ ghcr.io  │   pulls       │ Cloud VPS / │
  │  file +          ├─────────────►│        ├─────────────────►│ Actions      ├────────────►│ /        ├──────────────►│ Container   │
  │  compose.yml     │              │        │                  │ (free CI)    │             │ Docker   │               │ orchestrator│
  │                  │              │        │                  │              │             │ Hub      │               │             │
  │  docker compose  │              │        │                  │ - builds img │             └──────────┘               │ docker run  │
  │  up (local test) │              │        │                  │ - tests      │                                        │             │
  └──────────────────┘              └────────┘                  │ - pushes img │                                        └─────────────┘
                                                                └──────────────┘
```

## Key insights

| Point | Reality |
|---|---|
| "I push the Dockerfile to the server" | No — you push the **image** to a registry. The Dockerfile is the recipe; the image is the built dish. |
| "I manually push images to Docker Hub" | True the first day. After that, **CI/CD does it for you** automatically when you `git push`. |
| Docker Hub vs ghcr.io vs ECR | All three are image registries. Docker Hub = popular for open source. GitHub Container Registry (`ghcr.io`) = popular for company/private images. AWS ECR = popular if you live on AWS. They all work the same. |
| "Deploy from git to cloud" | Conceptually right. In practice, GitHub Actions builds the image, pushes it to a registry, and the server pulls + runs it. |

> **You don't need to set up CI/CD for Phase 4.** But knowing this exists is the difference between "I learned Docker" and "I understand how teams ship code." File it.

---

# 5. Stateless vs Stateful — the production split

```
                "STATELESS"            vs              "STATEFUL"
                (your Django app)                      (Redis, Postgres)

   ┌───────────────────────────────┐       ┌───────────────────────────────┐
   │ • Has no memory of its own.   │       │ • Holds data. The data IS     │
   │ • You can run 10 copies.      │       │   the value.                  │
   │ • Crash one? Start another.   │       │ • Crashing one = data loss    │
   │   No drama.                   │       │   risk.                       │
   │                               │       │ • Needs backups, replication. │
   │ ► Containerize freely.        │       │                               │
   └───────────────────────────────┘       │ ► Most teams use managed      │
                                            │   services in prod.          │
                                            └───────────────────────────────┘
```

## How real teams run things in production

```
  ┌─────────────────────────────────────────────────────────┐
  │           DEVELOPER manages (your containers):          │
  │                                                         │
  │   ┌─────────────┐    ┌─────────────┐                   │
  │   │  Backend    │    │  Frontend   │                   │
  │   │  container  │    │  container  │                   │
  │   │  (Django)   │    │  (React)    │                   │
  │   └──────┬──────┘    └─────────────┘                   │
  │          │                                              │
  └──────────┼──────────────────────────────────────────────┘
             │
             │ talks to:
             ▼
  ┌─────────────────────────────────────────────────────────┐
  │       CLOUD PROVIDER manages (managed services):        │
  │                                                         │
  │   ┌─────────────┐    ┌─────────────┐                   │
  │   │ AWS RDS     │    │ ElastiCache │                   │
  │   │ (Postgres)  │    │ (Redis)     │                   │
  │   └─────────────┘    └─────────────┘                   │
  │                                                         │
  │   Backups, scaling, security patches → not your problem │
  │   You pay extra for the peace of mind                   │
  └─────────────────────────────────────────────────────────┘
```

Your Django container reads two env vars:

```
DATABASE_URL=postgres://user:pass@rds-xyz.amazonaws.com:5432/mydb
REDIS_URL=redis://elasticache-xyz.amazonaws.com:6379/0
```

And doesn't care that one is a container on your laptop in dev and a managed AWS service in prod. **That's the magic of the abstraction.**

> **Rule:** Stateless code (Django, React, Node) → container all day. Stateful services (Redis, Postgres, Mongo) → containers in dev, often managed services in prod.

---

# 6. Native install vs Docker — the "daily driver" rule

## Why courses still teach `apt install postgres`

The honest 4 reasons:

| Reason | Truth |
|---|---|
| 1. "It's how I learned" | Inertia. Instructor learned pre-2018. Updating tutorials is huge work. |
| 2. "Beginners don't have Docker" | Lowest-common-denominator teaching. Adding Docker = 1 more install = student drop-off. |
| 3. "You need to FEEL the DB" | Philosophical belief that abstraction hides "real" engineering. Mostly nostalgia. |
| 4. "It just works" | True. Native isn't WRONG — just suboptimal. |

None of these are *"because it's better."*

## Hidden costs of native install (your gut was right)

| Cost | Example |
|---|---|
| 📦 Disk bloat | Postgres ~500MB + MySQL ~600MB + Redis + Mongo + ES → laptop cries |
| 🔀 Version conflicts | Project A needs Postgres 14, project B needs 16. Native = pain. Docker = 1 yml line. |
| 🚪 Port conflicts | MySQL grabs 3306. Next project also wants 3306. Now what? |
| 🪲 Windows services pollution | Background services eat RAM forever. "Why is my laptop slow?" |
| 🧹 Uninstall nightmare | Try uninstalling MySQL cleanly from Windows. Registry leftovers, ghost services. |
| 🏭 Doesn't match production | Postgres 16 on Windows vs Postgres 14 on Linux in prod. "Works on my machine" bugs. |
| 🤝 Hard to share with team | "Install these 7 services with these versions" vs `git clone && docker compose up`. |

## When native IS the right choice — the decision tree

```
                  ┌──────────────────────────────────────┐
                  │  IS THIS YOUR DAILY DRIVER TOOL?     │
                  │  (Use it on every project, all day)  │
                  └──────────┬───────────────────────────┘
                             │
                ┌────────────┴────────────┐
                │ YES                     │ NO
                ▼                         ▼
       ┌─────────────────┐      ┌────────────────────┐
       │ INSTALL NATIVE  │      │ USE DOCKER         │
       │                 │      │                    │
       │ • Python        │      │ • Postgres         │
       │ • Node          │      │ • MySQL            │
       │ • Git           │      │ • Redis            │
       │ • Your editor   │      │ • MongoDB          │
       │ • Docker itself │      │ • Elasticsearch    │
       │                 │      │ • RabbitMQ         │
       │ (Touched every  │      │ • Anything you     │
       │  hour)          │      │  use per-project)  │
       └─────────────────┘      └────────────────────┘
```

> **Rule of thumb:** Daily driver = native. Per-project service = Docker.

## Modern laptop setup in 2026

```
  ✅ Install natively:    Python, Node, Git, your IDE, Docker Desktop, DBeaver

  🐳 Run via Docker:      Postgres, MySQL, Redis, MongoDB, Elasticsearch,
                          RabbitMQ, Kafka, anything else "a service"

  ☁️  Don't install:       AWS, S3, ElastiCache — those live in the cloud.
                          Your app just hits them via URL.
```

---

# 7. Server vs Client — the universal decoupling pattern

## 📖 The story — "The Restaurant"

A restaurant has two completely different jobs:

1. **The kitchen** — fridges, stoves, pantry. Cooks and stores food.
2. **The dining room** — tables, menus, customers. Where food is *consumed and viewed*.

**These are two separate things.** The kitchen doesn't need a dining room to function. The dining room doesn't cook. They just need to be connected.

## Database = same pattern

```
  THE KITCHEN                        THE DINING ROOM
  ───────────────────                ────────────────────
  Postgres server                    pgAdmin / DBeaver
  MySQL server                       MySQL Workbench
  Redis server                       RedisInsight
  
  "Stores and serves data"           "Lets you LOOK at the data"
  Runs as a service                  Runs as a desktop app
  Speaks SQL over network            Speaks SQL over network
  Can live anywhere:                 Always on your laptop
    - Native install
    - Docker container
    - AWS RDS
    - A friend's server
```

> **The big myth busted:** When you "installed Postgres," you actually got **two** programs bundled — the server (kitchen) AND a GUI tool called pgAdmin (dining room). You thought it was one. It wasn't.

## The mental model

```
                     YOUR LAPTOP
   ┌──────────────────────────────────────────────────────┐
   │                                                      │
   │   ┌──────────────────┐         ┌──────────────────┐ │
   │   │  GUI TOOL        │         │   Docker         │ │
   │   │  (e.g. DBeaver)  │         │  ┌────────────┐  │ │
   │   │                  │ ───SQL──┼─►│ Postgres   │  │ │
   │   │  Shows tables,   │  over   │  │ container  │  │ │
   │   │  rows, queries   │ network │  │ port 5432  │  │ │
   │   └──────────────────┘         │  └────────────┘  │ │
   │                                └──────────────────┘ │
   └──────────────────────────────────────────────────────┘

   The GUI doesn't CARE that Postgres is in a container.
   It connects to localhost:5432 the same way it would connect to:
     • a native install
     • AWS RDS in the cloud
     • a database on the moon (if it had a public IP)
```

## GUI tools — pick ONE for everything

| Tool | Speaks | Notes |
|---|---|---|
| **DBeaver** | Postgres, MySQL, SQLite, Mongo, Redis, Oracle, 20+ more | ⭐ FREE, all-in-one. The boring-but-correct choice. |
| pgAdmin | Postgres only | Free, bundled with Postgres installer. |
| MySQL Workbench | MySQL only | Free, bundled with MySQL. |
| TablePlus | Postgres, MySQL, Redis, Mongo | Paid (limited free tier). Beautiful UI. |
| DataGrip | Everything | Paid (JetBrains). Pro tool. |
| RedisInsight | Redis only | Free, official. Great for inspecting cache keys. |

> **Rule:** Pick **ONE** GUI tool that handles every DB you'll ever touch. Use it forever. **DBeaver** is the recommended default.

## The deeper insight — server + client EVERYWHERE

| Server (the engine) | Client (how you talk to it) |
|---|---|
| Postgres | DBeaver / pgAdmin / `psql` CLI |
| Redis | RedisInsight / `redis-cli` |
| Git server (GitHub) | Git CLI / GitHub Desktop / GitKraken |
| Docker daemon | `docker` CLI / Docker Desktop GUI |
| Your Django API | Browser / Postman / curl / React frontend |
| Email server (SMTP) | Gmail web / Outlook desktop / Thunderbird |

> **Senior-engineer brain:** Every system in software is `server + client(s) talking over a network`. Once you see this pattern everywhere, software stops being a thousand random tools and becomes one repeating shape.

---

# 8. Rules of Thumb (consolidated)

Pin these in your head.

| # | Rule |
|---|---|
| 1 | Every `cache.set()` needs a timeout. No exceptions. |
| 2 | Always `cache.delete()` when underlying data changes. Stale cache = silent bug. |
| 3 | One container = one process. Don't cram Postgres + Redis into one container. |
| 4 | Write a Dockerfile only for code YOU wrote. Use official images for everything else. |
| 5 | Daily driver tool → install native. Per-project service → run in Docker. |
| 6 | Pick ONE database GUI tool (DBeaver). Use it for everything. |
| 7 | Stateless code → container freely. Stateful services → managed in prod. |
| 8 | In CI/CD-driven teams, `git push` triggers image build + deploy. You don't `docker push` manually. |
| 9 | `localhost:6379` works the same whether Redis is in Docker, native, or a tunneled cloud connection. Apps don't care. |
| 10 | Course tutorials are often dated. Validate against current best practice (search "X best practice 2025"). |

---

# 9. My confusions, cleared

Each of these was a doubt I had. Here's the resolved answer.

### Q1. "Is Docker the most recommended in production?"

**A:** For LOCAL DEV — yes, absolutely standard in 2026. For PRODUCTION — nuanced:

- Most companies use **managed cloud services** (AWS ElastiCache, RDS) for stateful stuff.
- They use **Docker (in Kubernetes)** for stateless stuff (their own apps).
- Behind the scenes, managed services often run containers anyway — Docker is the underlying packaging at every level.

### Q2. "I have Docker — why not use it for Redis instead of installing natively?"

**A:** Exactly right. You should. Docker for services = production-matching habit. Installing Redis natively on Windows is a beginner trap (the abandoned MicrosoftArchive repo is the trap).

### Q3. "My understanding: wrap project in docker, push to cloud, done?"

**A:** Almost. Refined:

- You write a `Dockerfile` (recipe) → `docker build` → **image** (built artifact) → push **image** to a **registry** → server pulls and runs.
- You DON'T push the Dockerfile itself.
- In real teams, CI/CD (GitHub Actions) does the build + push for you automatically.

### Q4. "In prod, DB and Redis are handled by cloud, devs don't manage them?"

**A:** Spot on. This is the stateless/stateful split. Devs containerize their own code; cloud providers manage the stateful services.

### Q5. "Why do instructors teach native MySQL/Postgres install?"

**A:** Inertia, lowest-common-denominator teaching, philosophical bias, and "it works." Not wrong, just outdated. Modern best practice in 2026: Docker.

### Q6. "If I use Docker for the DB, how do I visualize the data? Don't I need MySQL/Postgres natively for that?"

**A:** NO. You're conflating two things:
- The **database server** (kitchen) — runs in Docker.
- The **database GUI client** (dining room) — separate program (DBeaver, pgAdmin, etc.) that connects to the server over a network.

Install **one GUI tool** (DBeaver), use it forever, connect to whatever DB lives wherever.

### Q7. "Do I still need the natively-installed Postgres/MySQL on my laptop?"

**A:** Not for new projects. They're not actively harmful, but they're dead weight. Eventually you can uninstall them and run everything via Docker + DBeaver. No rush.

### Q8. "Is Docker Desktop needed in production?"

**A:** No. Docker Desktop = friendly GUI wrapper for Mac/Windows devs. Linux servers in production just use the Docker Engine CLI. Same containers, no GUI.

---

# 10. Engineer-mindset search cheatsheet

When you don't know something, here's exactly what to type into Google.

| Situation | Search query |
|---|---|
| Need to install Redis on Windows in 2025 | `redis windows install 2025` |
| Comparing install options | `redis windows wsl vs memurai vs docker` |
| Need the official Redis Docker docs | `redis docker hub` → land on **hub.docker.com/_/redis** |
| Need to find a Docker image's flags | `docker run [imagename] options` or check the Docker Hub README |
| Confused about Dockerfile vs image vs container | `dockerfile image container difference` |
| Want a free DB GUI | `dbeaver community edition download` |
| Want a Redis GUI | `redisinsight download` |
| Generic "what's best practice now" | `[topic] best practice 2025` (always add the year) |

> **Engineer's golden rule:** The docs are the source of truth. Random blog posts are second-class. When the docs and a blog disagree, the docs win. Always start at the docs.

---

## 📌 Where to go next

When you're ready, the actual Phase 4 build picks up at:

1. Get Redis running in a Docker container (`docker run redis ...`)
2. Verify with `redis-cli ping` → expect `PONG`
3. Then `pip install redis django-redis`
4. Configure `CACHES` in Django `settings.py`
5. Test the cache-or-compute pattern in a view

But take your time. Ask anything else first. These notes are yours.

---

*Document created during Phase 4 setup conversation.*
*Add new sections as we cover Celery, signals, OTP, Debug Toolbar, etc.*
