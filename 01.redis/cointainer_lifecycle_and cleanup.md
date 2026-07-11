# Phase 4 — Notes 02: Container Lifecycle, Disk Hygiene & the Engineer's Workflow

> Reference document. Everything we discussed about WHY/HOW to manage Docker images, containers, and volumes — and the "code is permanent, images are ephemeral" engineering philosophy.

---

## 📚 Table of Contents

1. [The engineer's source-of-truth philosophy](#1-the-engineers-source-of-truth-philosophy)
2. [Container lifecycle — birth to death](#2-container-lifecycle--birth-to-death)
3. [Exit codes decoded](#3-exit-codes-decoded)
4. [Image vs Container — what's mutable vs immutable](#4-image-vs-container--whats-mutable-vs-immutable)
5. [What eats disk space (and what doesn't)](#5-what-eats-disk-space-and-what-doesnt)
6. [Where Docker data lives on Windows](#6-where-docker-data-lives-on-windows)
7. [Image tag pinning — the silent disaster prevention](#7-image-tag-pinning--the-silent-disaster-prevention)
8. [One service instance per project](#8-one-service-instance-per-project)
9. [Named vs Anonymous volumes](#9-named-vs-anonymous-volumes)
10. [The cleanup command family](#10-the-cleanup-command-family)
11. [Rules of Thumb (consolidated)](#11-rules-of-thumb-consolidated)
12. [My confusions, cleared](#12-my-confusions-cleared)
13. [Cheatsheet](#13-cheatsheet)

---

# 1. The engineer's source-of-truth philosophy

The mental model that separates beginners from professionals.

```
                  THE ENGINEER'S SOURCE-OF-TRUTH HIERARCHY
                  ────────────────────────────────────────

   PERMANENT, PRECIOUS                 EPHEMERAL, DISPOSABLE
   ──────────────────────              ──────────────────────────

   ┌──────────────────────┐            ┌────────────────────────┐
   │  Code in GitHub      │            │  Images on your laptop │
   │  (Dockerfile,        │   ──►      │  (pulled when needed,  │
   │   compose.yml,       │            │   deleted when not)    │
   │   source code)       │            │                        │
   │                      │            │  Containers            │
   │  This is the         │            │  (recreated from       │
   │  RECIPE.             │            │   images anytime)      │
   │                      │            │                        │
   │  Without this,       │            │  Volumes (if data not  │
   │  you have nothing.   │            │  important / already   │
   │                      │            │  deployed elsewhere)   │
   └──────────────────────┘            └────────────────────────┘
       Backup religiously                 Wipe without fear
```

## The workflow in one sentence

> *"Pull images when a project needs them, use them during development, push code to GitHub, delete everything from the laptop. Re-pull / rebuild when needed again."*

## Why this works

| Aspect | Without this discipline | With this discipline |
|---|---|---|
| Disk space | Slowly fills with abandoned images and containers | Stays clean. Reclaim space anytime. |
| Reproducibility | "Works on my machine" — depends on local image state | Anyone with the GitHub repo can `git clone && docker compose up` and get the same setup |
| Confidence | "Am I depending on something only on my machine?" | All dependencies declared in code. No hidden state. |
| Team collaboration | "Hey, can you send me your local container?" | "Just clone the repo." |

## When to break the rule

- **Daily-driver tools** (Python, Node, your IDE) → install native, they're not project-scoped
- **Base images you pull constantly** (`python:3.12-slim`) → fine to keep, you'll re-pull anyway
- **Active in-progress work** → don't delete the image of a project you're still working on

> **Rule of thumb:** Code in GitHub = permanent. Images and containers = ephemeral. Volumes = case-by-case (if the data matters, it should be backed up; if not, treat as ephemeral).

---

# 2. Container lifecycle — birth to death

Containers, like processes, have well-defined states.

```
                   CONTAINER LIFECYCLE
                   ───────────────────

       docker run                   docker stop / process exits
          │                                  │
          ▼                                  ▼
    ┌─────────┐    docker start       ┌──────────┐
    │ CREATED │ ──────────────► RUNNING ◄── ──── │  EXITED  │
    └─────────┘                  ▲              └──────────┘
                                  │                    │
                                  │ docker start       │ docker rm
                                  └────────────────────│────────► GONE
                                                       ▼          (deleted)
```

## State definitions

| State | Meaning | Filesystem | Disk usage |
|---|---|---|---|
| **Created** | Container defined but never started. Rare. | Initialized | Minimal |
| **Running** | Active, executing. | Live, writable layer on top of image | Container size |
| **Paused** | Frozen mid-execution. Rarely used. | Preserved | Container size |
| **Exited** | Was running. Stopped. NOT deleted — can be restarted with `docker start`. | Preserved | Container size |
| **Dead** | Broken state, unusable. Rare. | Preserved but corrupt | Container size |
| **(removed)** | After `docker rm`. Container is GONE from disk. Image still exists. | Wiped | 0 |

## Important: Exited ≠ Deleted

A container that "exited" is **stopped**, not removed. It still sits on disk taking up space. To actually delete it: `docker rm <name>`.

`docker ps` shows only running containers. `docker ps -a` shows all, including exited.

---

# 3. Exit codes decoded

When a container "exits," it leaves an exit code. These map directly to Unix process signals.

| Exit code | What it means | When you'll see it |
|---|---|---|
| **`0`** | Normal, clean exit. Success. | Process finished its work and stopped on its own. |
| **`1`** | Generic application error. | Code crashed with an error. Check logs. |
| **`125`** | Docker daemon error. | The container couldn't even start (bad command, missing image). |
| **`126`** | Command found but not executable. | Permission issue inside the container. |
| **`127`** | Command not found. | Tried to run something that doesn't exist in the container. |
| **`130`** | Killed by Ctrl+C (`SIGINT`, signal 2). | You manually stopped it from terminal. |
| **`137`** | Killed by `SIGKILL` (signal 9). | Forced kill. Docker Desktop quit, WSL shut down, OOM killer, or `docker kill` was used. |
| **`139`** | Segmentation fault. | Native code crashed. Rare in normal apps. |
| **`143`** | Killed by `SIGTERM` (signal 15). | Graceful "please stop" signal — `docker stop` sends this by default, waiting 10s before escalating to SIGKILL. |

## The math behind 137 and 143

Unix convention: exit code = `128 + signal number`.

- Signal 9 (`SIGKILL`) → `128 + 9 = 137`
- Signal 15 (`SIGTERM`) → `128 + 15 = 143`

## When you see Exit 137 on old containers

> Almost always means *"Docker Desktop was running this, then Windows/WSL shut down, killing the container forcefully."* Not a real error — just normal Windows shutdown.

---

# 4. Image vs Container — what's mutable vs immutable

This is the most important Docker distinction. Most beginners confuse these.

```
                  IMAGE                          vs           CONTAINER
                  ─────────────────────                       ─────────────────────
   What it is     Immutable, frozen template                  A running instance
                  (literally the same bytes for                of an image, with
                  everyone who pulls it)                       its OWN writable layer

   Built from     A Dockerfile (recipe)                       An image (template)

   Storage        Layered, read-only                          Image layers + a
                                                              writable layer on top

   Lifecycle      Pulled / built once, used many times        Created → Running →
                                                              Exited → Removed

   Can it be      ❌ NO. Images are read-only.                 ✅ YES. Containers
   "contaminated" Same `redis:alpine` for you,                 have a writable layer
   by past use?   for me, for AWS, for Google.                 where past data sits.
                  Pulling it is like downloading
                  a sealed DVD.

   Reuse safety   ✅ Safe to reuse across projects             ❌ Don't reuse
                  (the image doesn't "know"                     containers — old data
                  which project ran it)                         taints new projects

   Where the      No app data. Just code +                    Whatever the running
   "state" lives  binaries + OS files in layers.              app wrote (cache,
                                                              database files, logs).
```

## Concrete example

You pull `redis:7-alpine` today, run `docker run --name redis-A`. Redis starts, accepts commands, fills up with cache keys.

Now imagine pulling the *same* `redis:7-alpine` image again on a different laptop. **It's literally the same bytes.** Docker Hub serves it from a CDN, identical for everyone. The image doesn't carry "history."

Your `redis-A` container, however, has the cache data you put into it. Even if you stop it, that data sits in the writable layer until you `docker rm redis-A`.

> **The myth:** "I used this image before, it might be contaminated."
> **The truth:** Images can't be contaminated. They're read-only. **Containers** can carry stale state — and that's why you wipe and recreate them per project.

## So why re-pull anyway?

Three good reasons, even though the image itself is pristine:

1. **Tag freshness** — `redis:alpine` 6 months ago ≠ `redis:alpine` today. Tags get updated upstream silently. Re-pulling gets security patches.
2. **Version pinning hygiene** — pulling fresh forces you to specify what you want (`redis:7-alpine`). Makes dependencies explicit.
3. **Mental model clarity** — saying "this project depends on Redis, here's the image" beats relying on whatever happens to be on your laptop.

---

# 5. What eats disk space (and what doesn't)

Where the bytes actually go.

```
   THING            SIZE        WHAT IT IS
   ──────────────   ─────────   ──────────────────────────────────────────
   IMAGES           HEAVY       The "templates" (full OS + your app).
                                Sit on disk forever until you `docker rmi`.
                                A single image: 100 MB to 4 GB.

   CONTAINERS       LIGHT       Tiny writable layer ON TOP of an image.
                                Usually 1-50 MB unless your app wrote a lot.

   VOLUMES          VARIABLE    Data your apps explicitly saved
                                (DB data, persistent state).
                                Survive container deletion (intentionally).

   BUILD CACHE      HEAVY       Layers Docker reuses to speed up rebuilds.
                                Grows silently. Can be many GB.

   NETWORKS         NEGLIGIBLE  Virtual networks Docker creates.
                                Few KB each.
```

## Real example — what 11 GB of Docker looks like

From your laptop's state before cleanup:

| Image | Size |
|---|---|
| `image-scraper` | 3.65 GB |
| `projectfastapi-api` | 2.1 GB |
| `agno_project-agent-app` | 1.61 GB |
| `grafana/grafana` | 971 MB |
| `postgres:latest` | 643 MB |
| `ankane/pgvector` | 628 MB |
| `postgres:15` | 608 MB |
| `prom/prometheus` | 507 MB |
| `searxng/searxng` | 303 MB |
| `redis:alpine` | 100 MB |
| **TOTAL** | **~11.1 GB** |

After `docker system prune -a --volumes`: **14.24 GB reclaimed** (the extra ~3 GB came from build cache and anonymous volumes).

---

# 6. Where Docker data lives on Windows

Important to know if you're worried about your C: drive (SSD).

```
   Docker Desktop + WSL2 stores everything in a virtual disk file:

   C:\Users\<you>\AppData\Local\Docker\wsl\disk\docker_data.vhdx

   ▲ This is a virtual hard disk file.
     Grows as you pull images and create containers.
     Lives on your C: drive (the SSD) by default.
```

## Important quirk — VHDX doesn't auto-shrink

When you delete images/containers, Docker reclaims space *inside* the VHDX file. But the VHDX file itself doesn't shrink on disk automatically.

To reclaim real disk space:

1. **Docker Desktop GUI**: Settings → Resources → "Disk image size" → Compact
2. **Or restart Docker Desktop** — sometimes triggers compaction
3. **Or manually** with `Optimize-VHD` PowerShell command (advanced)

> **In practice:** Don't worry about this for normal cleanup. The "free space inside VHDX" is reusable for future images, so you don't lose anything. Only compact if you're truly out of disk space.

---

# 7. Image tag pinning — the silent disaster prevention

Image tags are version labels. The choice matters more than beginners realize.

| Tag pattern | Example | Behavior |
|---|---|---|
| `:latest` | `redis:latest` | Whatever maintainers tagged as "latest" today. **Can change under you.** Disaster waiting to happen. |
| Rolling major | `redis:7` | Latest 7.x. Patches auto-update, but no major version jumps. Good for dev. |
| Variant | `redis:alpine`, `redis:7-alpine` | Smaller image, Alpine Linux base. |
| Pinned major | `redis:7-alpine` | Latest 7.x on Alpine. Stable for dev. |
| Fully pinned | `redis:7.4.1-alpine` | Exact patch version. Deterministic. Use for production. |

## Why `:latest` is dangerous

```
   Day 1, you write:                            docker pull redis:latest
                                                # Got Redis 7.2

   Day 30, maintainers ship Redis 8:            (silently)

   Day 45, your CI runs:                        docker pull redis:latest
                                                # Got Redis 8 — BREAKING CHANGES
                                                # CI fails mysteriously
                                                # Your app code expected 7.x
```

This has caused countless production outages.

> **Rule of thumb:** In dev, pin by major version (`redis:7-alpine`). In production, pin by full version (`redis:7.4.1-alpine`). Never use `:latest` for anything you care about.

---

# 8. One service instance per project

A core engineering rule for local development.

## The principle

> **Each project gets its own Postgres, its own Redis, its own everything. They're defined in the project's `docker-compose.yml` and live and die with the project.**

## Why

```
                  SHARED SERVICES                      PER-PROJECT SERVICES
                  (the bad pattern)                    (the right pattern)
                  ─────────────────────                ─────────────────────────

   Data isolation  ❌ Phase 4 cache mixed              ✅ Each project's data
                      with Project B's data               lives in its own container

   Versions         ❌ Stuck with one version           ✅ Pick the version each
                      across all projects                 project needs

   Config           ❌ Compromise config                ✅ Each project tunes
                      for everyone                        its own service

   Debugging        ❌ "Why does my Redis have          ✅ FLUSHALL nukes ONLY
                      keys from 4 different             this project's data
                      projects?"

   Cleanup          ❌ Can't delete the shared          ✅ docker compose down -v
                      service or you break              wipes ONE project entirely
                      other projects
```

## The exception

There IS a niche pattern where a single shared Redis hosts multiple projects using Redis's built-in 16 logical databases (db 0, db 1, ...). It works because Redis supports `SELECT 1` to switch DBs. But it's an optimization, not a default. Skip it unless you have a specific reason.

> **Rule:** When in doubt, give each project its own container.

---

# 9. Named vs Anonymous volumes

Volumes are how Docker persists data outside containers. There are two kinds.

```
                  ANONYMOUS VOLUMES                    NAMED VOLUMES
                  ─────────────────────                ─────────────────────────

   How created     Auto-generated by Docker            You explicitly name them
                   when a container declares a         in docker-compose.yml
                   VOLUME but you didn't name it       or `-v myname:/data`

   Name            Random hash like                    Human-readable: `mydata`,
                   `2212d4233c66f3c7...`               `postgres-data`, etc.

   `--volumes`     ✅ Deleted                          ❌ NOT deleted (preserved)
   prune flag         (because they're "junk")            (assumed important)

   To delete       docker system prune --volumes       docker volume prune -a
                                                       or
                                                       docker volume rm <name>

   Use case        Default behavior of some            Persistent data you
                   official images (Postgres,          care about across
                   MySQL) for ephemeral data           container restarts
```

## What surprised me

After `docker system prune -a --volumes`, you still had 5 volumes (~290 MB) left. These were **named** volumes from old projects. The `--volumes` flag only kills **anonymous** ones.

To finish the cleanup, you'd use either:

```powershell
docker volume ls          # see them
docker volume prune -a    # nuke all unused named volumes
```

or surgically with `docker volume rm <name>`.

> **Rule:** If you don't recognize a named volume, it's safe to delete (you saved it for a reason; if you can't recall the reason, you don't need it).

---

# 10. The cleanup command family

The full toolkit, from surgical to nuclear.

```
   GENTLE                                                    NUCLEAR
   ────────────────────────────────────────────────────────────────►

   docker stop <name>           Stops a running container

   docker rm <name>             Removes a stopped container

   docker rmi <image>           Removes an image
                                (fails if container uses it)

   docker container prune       Removes ALL stopped containers
                                (running ones safe)

   docker image prune           Removes "dangling" images
                                (no tag, unused)

   docker image prune -a        Removes ALL images not used by
                                any container

   docker volume prune          Removes anonymous volumes only

   docker volume prune -a       Removes named volumes too

   docker network prune         Removes unused networks

   docker builder prune         Clears build cache

   docker system prune          Containers + dangling images +
                                networks + build cache

   docker system prune -a       + ALL unused images

   docker system prune -a       + Anonymous volumes
   --volumes                    (named volumes still survive)
                                ← MOST NUCLEAR for typical use
```

## When to use which

| Goal | Command |
|---|---|
| Stop a container temporarily | `docker stop <name>` |
| Permanently remove a specific container | `docker rm <name>` |
| Clean up stopped containers after experimenting | `docker container prune` |
| Reclaim disk space — keep nothing unused | `docker system prune -a --volumes` |
| Check disk usage before/after | `docker system df` |
| See what's still on disk after cleanup | `docker images`, `docker ps -a`, `docker volume ls` |

## Order matters

You can't delete an image while a container (running OR stopped) uses it:

```
   Step 1: Stop running containers   →  docker stop <name>
   Step 2: Remove containers         →  docker rm <name>
   Step 3: Remove images             →  docker rmi <name>
   Step 4: Remove volumes            →  docker volume rm <name>
```

The `prune` family handles this ordering for you automatically.

---

# 11. Rules of Thumb (consolidated)

Burn these in.

| # | Rule |
|---|---|
| 1 | Code in GitHub = permanent. Images/containers/volumes = ephemeral. |
| 2 | Each project gets its own services (its own Postgres, its own Redis). |
| 3 | Pin image tags by major version (`redis:7-alpine`). Never use `:latest`. |
| 4 | Images are immutable and read-only — they can't be "contaminated." |
| 5 | Containers carry writable state — wipe and recreate per project. |
| 6 | "Exited" ≠ deleted. A stopped container still uses disk. Run `docker rm` to truly remove. |
| 7 | The disk weight is in IMAGES (and build cache), not containers. |
| 8 | `--volumes` flag on prune only deletes anonymous volumes. Named volumes need `docker volume prune -a`. |
| 9 | Always run `docker system df` before and after cleanup to see the win. |
| 10 | Re-pulling an image is cheap (often instant from cache). Don't fear deletion. |
| 11 | Exit code 137 = killed by SIGKILL (force). Not a bug, usually a forced shutdown. |
| 12 | VHDX on Windows doesn't auto-shrink. Use Docker Desktop "Compact" if you need real disk space back. |

---

# 12. My confusions, cleared

### Q1. "If I reuse an old Redis container, isn't that wrong since it was for a different project?"

**A:** Yes, reusing a *container* is wrong (it can carry stale data). But the *image* it's based on is pristine and reusable across projects.

In practice: don't reuse the container, create fresh per project. That's the modern pattern. Each project's `docker-compose.yml` defines its own services that live and die with the project.

### Q2. "What does 'Exited' mean in `docker ps -a`?"

**A:** The container was running and has stopped, but isn't deleted yet. It still sits on disk preserving its state. You can restart it with `docker start <name>` or remove it with `docker rm <name>`.

### Q3. "What's a 'graveyard' of containers?"

**A:** My informal term for stopped containers from old projects that pile up on your machine. They take up disk space silently. Clean them with `docker container prune`.

### Q4. "Are the stopped containers eating my SSD space?"

**A:** YES, indirectly. The containers themselves are small, but the IMAGES they depend on are heavy (often hundreds of MB to multiple GB each). On Windows, all of this lives in a VHDX file on your C: drive (SSD).

Your laptop had ~11 GB of images. After `docker system prune -a --volumes`, you reclaimed **14.24 GB**.

### Q5. "What happens when I delete an image from Docker Desktop GUI?"

**A:**
- ✅ Removes from your local cache
- ✅ Frees the disk space it used
- ✅ You can always re-pull from Docker Hub later
- ❌ Refuses to delete if any container (running or stopped) is using it — delete the container first

### Q6. "What if `redis:alpine` was used by some past project — is it contaminated?"

**A:** No. Images are immutable and read-only — they can't carry state from past use. The bytes are the same `redis:alpine` for everyone who pulls it.

BUT — re-pulling is still good practice for:
1. Getting latest security patches (tags get updated upstream)
2. Forcing explicit version choice (`redis:7-alpine`)
3. Mental clarity about dependencies

### Q7. "What about the 5 leftover volumes after my big prune?"

**A:** Those are **named volumes** (e.g. `mydata`) from old projects. The `--volumes` flag on `docker system prune` only deletes **anonymous volumes** (random hash names). To remove named volumes:

```powershell
docker volume ls          # see them
docker volume prune -a    # delete all unused named volumes
```

### Q8. "Engineers pull images when needed, use them, delete on completion — is my approach right?"

**A:** Yes. This is *exactly* the mature workflow:

1. Code (Dockerfile, compose.yml, source) lives in GitHub — permanent
2. Pull images when starting a project
3. Develop, use containers, iterate
4. Push code to GitHub
5. Delete images + containers locally → reclaim disk
6. If you ever need to revisit: `git clone && docker compose up` rebuilds everything

---

# 13. Cheatsheet

The commands you'll reach for most.

## Inspection

```powershell
docker ps                        # running containers
docker ps -a                     # all containers (incl. stopped)
docker images                    # local images
docker volume ls                 # local volumes
docker system df                 # disk usage breakdown
docker info                      # daemon status + machine info
```

## Running stuff

```powershell
docker pull <image>:<tag>        # download image
docker run -d --name <n> \
       -p <host>:<container> \
       --restart unless-stopped \
       <image>:<tag>             # start a container in background

docker start <name>              # restart an exited container
docker stop <name>               # gracefully stop
docker restart <name>            # stop + start
docker exec -it <name> <cmd>     # run command inside running container
docker logs <name>               # see container output
docker logs -f <name>            # follow logs in real time
```

## Cleanup

```powershell
docker rm <name>                 # remove a stopped container
docker rmi <image>               # remove an image
docker container prune           # all stopped containers
docker image prune -a            # all unused images
docker volume prune -a           # all unused volumes (incl. named)
docker system prune -a --volumes # everything unused (most nuclear)
```

## The Phase 4 setup (for reference)

```powershell
docker pull redis:7-alpine
docker run -d --name redis -p 6379:6379 --restart unless-stopped redis:7-alpine
docker exec -it redis redis-cli ping   # expect PONG
```

---

## 📌 Where this fits

After this notes file, you have:
- A clean Docker setup
- A pinned, named Redis container running on port 6379
- A reproducible cleanup workflow you can apply to every future project
- The engineer's mental model: "code permanent, infrastructure ephemeral"

Next up: wiring Redis into Django (`django-redis`, `CACHES` config, cache-or-compute pattern). That's `NOTES_03_*.md` when we get there.

---

*Document created during Phase 4 Docker cleanup conversation. The "14.24 GB reclaimed" was real.*
