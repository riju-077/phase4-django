# Phase 4 — Notes 03: Request Flow — Dev (laptop) vs Production (server)

> Reference document. How a single request travels from a user → through ports, IPs, Docker → to your code → back. Compares the simple **laptop** setup with the full **production server** setup so you can keep them straight.

---

## 📚 Table of Contents

1. [Why this notes file exists](#1-why-this-notes-file-exists)
2. [The two scenarios at a glance](#2-the-two-scenarios-at-a-glance)
3. [Scenario A — On your laptop (dev)](#3-scenario-a--on-your-laptop-dev)
4. [Scenario B — On a production server (the Grand Hotel)](#4-scenario-b--on-a-production-server-the-grand-hotel)
5. [Port mapping — the heart of Docker networking](#5-port-mapping--the-heart-of-docker-networking)
6. [Decoding `0.0.0.0:6379->6379/tcp`](#6-decoding-0000637963790tcp)
7. [DNS — how `yourcoffeeshop.com` becomes an IP](#7-dns--how-yourcoffeeshopcom-becomes-an-ip)
8. [The Concierge (reverse proxy) — the production gatekeeper](#8-the-concierge-reverse-proxy--the-production-gatekeeper)
9. [Internal vs External networks](#9-internal-vs-external-networks)
10. [Dev vs Prod — what changes, what stays the same](#10-dev-vs-prod--what-changes-what-stays-the-same)
11. [Engineering rules of thumb](#11-engineering-rules-of-thumb)
12. [My confusions, cleared](#12-my-confusions-cleared)
13. [Cheatsheet](#13-cheatsheet)

---

# 1. Why this notes file exists

Two scenarios constantly trip up beginners:

- **"How does Django on my laptop reach Redis in a Docker container?"**
- **"How does a user on the internet reach my Django app in production?"**

They feel like the same question but they're not — and they're not entirely different either. This doc puts them side by side so the mental model is permanent.

After reading this you should be able to:
- Trace any HTTP request from origin to response
- Explain what `0.0.0.0:6379->6379/tcp` means without thinking
- Know exactly which ports are public and which are internal
- Move confidently between dev and prod environments

---

# 2. The two scenarios at a glance

```
   SCENARIO A — DEV (your laptop)               SCENARIO B — PROD (cloud server)
   ─────────────────────────────                ─────────────────────────────

   Just YOU + a few services                    Millions of users + many services
   on one physical machine.                     on a remote server.

   ┌─────────────────────────┐                  ┌─────────────────────────┐
   │  YOUR LAPTOP            │                  │  CLOUD SERVER           │
   │  ┌──────────────────┐   │                  │  ┌──────────────────┐   │
   │  │ Django (on host) │   │                  │  │ Concierge (nginx)│ ◄─┼─── INTERNET
   │  └────────┬─────────┘   │                  │  └────────┬─────────┘   │
   │           │             │                  │           │             │
   │           ▼             │                  │     ┌─────┼─────┐       │
   │  ┌──────────────────┐   │                  │     ▼     ▼     ▼       │
   │  │ Redis (Docker)   │   │                  │  ┌────┐┌────┐┌────┐     │
   │  └──────────────────┘   │                  │  │ F  ││ B  ││ W  │     │
   │                         │                  │  └─┬──┘└─┬──┘└─┬──┘     │
   │                         │                  │    │  ┌──┴──┐  │        │
   │                         │                  │    └──┤ D R ├──┘        │
   │                         │                  │       └─┬───┘           │
   │                         │                  │         └────► Stripe   │
   └─────────────────────────┘                  └─────────────────────────┘

   Talkers: Django ↔ Redis                      Talkers: User ↔ Concierge ↔
                                                            multiple services
                                                            ↔ external APIs
```

You're already living in Scenario A right now. Scenario B is the future of your project.

---

# 3. Scenario A — On your laptop (dev)

## 🎬 The setup right now

You have:
- Django (when it's running) — runs **natively** on your Windows machine via `python manage.py runserver`
- Redis — runs **inside a Docker container** with `-p 6379:6379` exposed

```
                    YOUR LAPTOP
   ┌──────────────────────────────────────────────────┐
   │                                                  │
   │   ┌────────────────────┐                         │
   │   │   Django           │                         │
   │   │   (Python process) │                         │
   │   │                    │                         │
   │   │   wants to reach   │                         │
   │   │   Redis at:        │                         │
   │   │   localhost:6379   │ ──┐                     │
   │   └────────────────────┘   │                     │
   │                            │                     │
   │                            ▼                     │
   │   ┌────────────────────────────────────────┐     │
   │   │   DOCKER ENGINE                        │     │
   │   │                                        │     │
   │   │   Sees: incoming on 0.0.0.0:6379       │     │
   │   │   Knows: forward to container "redis"  │     │
   │   │                                        │     │
   │   │   ┌──────────────────────┐             │     │
   │   │   │  redis container     │             │     │
   │   │   │  port 6379 inside    │ ◄───────────┘     │
   │   │   │  Redis is listening  │                   │
   │   │   └──────────────────────┘                   │
   │   └────────────────────────────────────────┘     │
   └──────────────────────────────────────────────────┘

   THERE IS NO INTERNET INVOLVED HERE.
   Everything happens inside your laptop.
```

## 🎬 The flow — Django talks to Redis

Imagine Django wants to ask Redis: *"What's the cached dashboard data?"*

```
   Step 1.  Django creates a TCP connection to localhost:6379
            (`localhost` = "this very machine", `6379` = the port)

   Step 2.  Windows networking sends the packet to itself.
            It hits whatever is listening on 0.0.0.0:6379.

   Step 3.  Docker Desktop is the listener.
            It looks up its port-mapping table:
            "0.0.0.0:6379 → container 'redis', port 6379"

   Step 4.  Docker forwards the packet INSIDE the container.
            The Redis server process there gets the connection.

   Step 5.  Redis reads the request, looks up the key,
            responds with the cached data.

   Step 6.  Response travels back the same path in reverse.
            Django receives the data.
```

The whole round trip takes **less than 1 millisecond** because nothing leaves the laptop.

## 🧠 Key insight — `localhost` is the magic word

```
   "localhost"  ≈  "127.0.0.1"  ≈  "this machine I'm sitting on"
```

When Django says `redis://localhost:6379`, Windows networking interprets that as *"talk to whoever is listening on port 6379 on this very computer."* Docker has registered itself as the listener. Docker forwards to the container. Done.

> **Rule:** On your laptop, services running on the host talk to containerized services via `localhost:<exposed_port>`.

---

# 4. Scenario B — On a production server (the Grand Hotel)

Now scale it up to a real cloud server with many services and millions of potential users.

## 🏨 The hotel cast

| Room | What lives there |
|---|---|
| 🎩 **Concierge** | The reverse proxy (nginx / Traefik / Caddy). The only one with public phone numbers. |
| 🟢 **Room F** | Frontend container (React/HTML/JS) |
| 🟢 **Room B** | Backend container (Django) |
| 🟡 **Room W** | Background workers (Celery) |
| 🔴 **Room D** | Database (Postgres) — INTERNAL ONLY |
| 🔴 **Room R** | Redis cache — INTERNAL ONLY |
| 🟡 **Room P** | Payments service |
| 🟡 **Room M** | Mailer (sends emails) |

## 🚨 THE ONE RULE OF THE HOTEL

> **Only the Concierge has public phone numbers (ports 80 and 443). Everything else is internal-only.**

```
                          THE INTERNET
                          (the whole world)
                                  │
                                  ▼
                       ┌────────────────────┐
                       │     CONCIERGE      │   ◄── Ports 80 + 443 PUBLIC
                       │  nginx / Traefik   │
                       └──────────┬─────────┘
                                  │
                ──── internal Docker network from here ────
                                  │
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
       ┌──────────┐         ┌──────────┐         ┌──────────┐
       │ Room F   │         │  Room B  │         │ Room W   │
       │Frontend  │         │ Backend  │         │ Workers  │
       └──────────┘         └────┬─────┘         └────┬─────┘
                                 │                    │
                  ┌──────────────┼────────────────────┘
                  │              │
              ┌───┴────┐    ┌────┴────┐    ┌──────────┐
              │ Room D │    │ Room R  │    │ Room P   │
              │Postgres│    │ Redis   │    │ Payments │
              └────────┘    └─────────┘    └────┬─────┘
                                                │
                                                │ outbound only
                                                ▼
                                          ┌──────────┐
                                          │  STRIPE  │
                                          │ (external│
                                          │  hotel)  │
                                          └──────────┘
```

Rooms D, R, P, M, W have **no public ports** — they're invisible to the internet. Outside callers can't even see they exist.

## 🎬 The Akash story — full step-by-step

A user named **Akash** opens his phone, goes to `yourcoffeeshop.com`, signs up, and buys a $10 coffee subscription.

### 🪜 Step 1 — DNS lookup

```
   Akash's browser   ── "where is yourcoffeeshop.com?" ──► DNS
   Akash's browser   ◄──         "at IP 203.0.113.42"   ── DNS
```

DNS is the global address book of the internet. Now his browser knows the server's IP.

### 🪜 Step 2 — Akash dials the hotel

```
   Akash   ─── connect to 203.0.113.42:443 (HTTPS) ───►  Concierge
```

Port 443 is the standard public port for HTTPS. The Concierge picks up.

### 🪜 Step 3 — Concierge routes the request

The Concierge looks at the URL: `https://yourcoffeeshop.com/`. The Concierge has a rulebook (config file):

```
   /            → Room F (Frontend)
   /api/...     → Room B (Backend)
   /static/...  → Room F (static assets)
   /admin/...   → Room B (Django admin)
```

This request goes to Room F.

```
   Concierge  ── internal call ──►  Room F
   Concierge  ◄── HTML homepage ──  Room F
   Concierge  ──── HTML to ───►     Akash's browser
```

Akash sees the page. Note: Akash never knew Room F existed — only the Concierge.

### 🪜 Step 4 — Akash signs up

He fills the form. Browser POSTs to `https://yourcoffeeshop.com/api/signup`. Concierge sees `/api/...` → routes to Room B.

```
   Room B  → Room D:    "Create user akash@..."
   Room D  → Room B:    "Done. User ID 1042."

   Room B  → Room R:    "Store session token abc123 for user 1042"
   Room R  → Room B:    "Stored, 24h expiry"

   Room B  → Room M:    "Send welcome email to akash@..."
   Room M  → Gmail:     (outbound to external SMTP)
   Room M  → Room B:    "Email queued"

   Room B  → Concierge → Akash:  "Signup successful, here's your session"
```

**Five internal calls happened — none visible to the outside world.**

### 🪜 Step 5 — Akash buys the subscription

```
   Concierge → Room B:           "POST /api/purchase, user 1042, sub X"

   Room B → Room R:              "Is session valid?" → "Yes"
   Room B → Room D:              "Sub X price?" → "$10"

   Room B → Room P:              "Charge user 1042 $10"
   Room P → Stripe (outbound):   "Charge customer X, $10"
   Stripe → Room P:              "Approved, txn #xyz"

   Room P → Room B:              "Paid"
   Room B → Room D:              "Record subscription for 1042"
   Room B → Room W:              "Schedule first delivery"
   Room B → Room M:              "Send receipt"
   Room B → Concierge → Akash:   "Confirmed! 🎉"
```

About 200ms total. Akash sees the success screen.

## 🧠 Key insight — services talk by NAME, not by IP

Inside the hotel, Room B doesn't say *"call IP 172.17.0.4:6379"*. It says *"call **redis** on port 6379"*. Docker has its own internal DNS that resolves the name `redis` to the right container.

This is why your `docker-compose.yml` matters:

```yaml
services:
  redis:        # ← the NAME
    image: redis:7-alpine
  backend:
    build: ./backend
    # backend code says: redis://redis:6379
    #                            ▲
    #              this is the service NAME, not a hostname
```

In dev, Django talks to Redis at `localhost:6379`.
In prod (or with compose), Django talks to Redis at `redis:6379`.
**Same Redis. Different addressing because the network context changed.**

---

# 5. Port mapping — the heart of Docker networking

The single most important Docker networking concept. It's all `-p`.

## The flag

```
   -p     <host-port>    :    <container-port>
   │           │                     │
   │           │                     └── Port INSIDE the container
   │           │                         (where the service is listening)
   │           │
   │           └────────────────────── Port on your laptop/server
   │                                   (what apps OUTSIDE the container dial)
   │
   └────────────────────────────────── "Publish" port flag
```

## What it does

Tells Docker: *"When something comes to the host machine on `<host-port>`, forward it inside the container to `<container-port>`."*

## Examples

| Flag | Meaning |
|---|---|
| `-p 6379:6379` | Host 6379 → container 6379 (most common, same number both sides) |
| `-p 8080:6379` | Host 8080 → container 6379 (Redis still uses 6379 inside; we dial 8080 outside) |
| `-p 127.0.0.1:6379:6379` | Only localhost can reach it (safer dev mode) |
| `-p 6379` | Host port chosen at random by Docker (use `docker ps` to find it) |
| (no `-p` at all) | No external access. Only other containers on the same Docker network can reach it. ← **production pattern for Redis/Postgres!** |

## The visual

```
   YOUR LAPTOP                          DOCKER CONTAINER
   ───────────                          ────────────────

   ┌─────────────────┐                  ┌──────────────────┐
   │  port 6379      │  ◄── -p 6379:6379 ──►  │  port 6379       │
   │  (host side)    │                  │  (container side)│
   │                 │                  │                  │
   │  apps OUTSIDE   │                  │  Redis listens   │
   │  the container  │                  │  here            │
   │  dial THIS port │                  │                  │
   └─────────────────┘                  └──────────────────┘
```

## The production reversal

In production, you DON'T expose internal services. Your `docker-compose.yml` would look like:

```yaml
services:
  concierge:
    image: nginx
    ports:
      - "80:80"        # ✅ public
      - "443:443"      # ✅ public

  backend:
    build: ./backend
    # NO ports section — internal only

  redis:
    image: redis:7-alpine
    # NO ports section — internal only
    # Other containers reach it via redis:6379

  postgres:
    image: postgres:16-alpine
    # NO ports section — internal only
```

> **Rule:** In dev, expose ports so you can poke services with tools (DBeaver, redis-cli) from your host. In production, expose ONLY the reverse proxy. Everything else internal.

---

# 6. Decoding `0.0.0.0:6379->6379/tcp`

The string `docker ps` shows in the PORTS column.

## Breakdown

```
   0.0.0.0   :   6379   ->   6379   /tcp
      │            │      │      │       │
      │            │      │      │       └── Protocol: TCP
      │            │      │      │           (vs UDP, ICMP)
      │            │      │      │
      │            │      │      └────────── Port INSIDE the container
      │            │      │
      │            │      └────────────────── Direction of mapping
      │            │
      │            └────────────────────────── Port on the HOST
      │
      └─────────────────────────────────────── Bind address
                                               (which network interfaces accept connections)
```

## Plain English translation

> *"I (Docker) am listening on the host machine's port 6379, on ALL network interfaces. Any incoming TCP connection here will be forwarded to port 6379 inside the container."*

## What the bind address controls

```
   0.0.0.0            ── "Accept from ANY interface"
                          (localhost, your LAN IP, etc.)

   127.0.0.1          ── "ONLY localhost"
                          (only programs on THIS machine)

   192.168.1.5        ── "ONLY this specific network interface"
                          (niche)
```

In dev, `0.0.0.0` is fine because nobody's targeting your laptop. In production, you'd use `127.0.0.1:...` if you wanted to lock services to localhost-only on the server itself.

## Why you sometimes also see `[::]:6379`

That's the **IPv6** version of `0.0.0.0`. Same thing, different IP version. Modern Docker supports both.

---

# 7. DNS — how `yourcoffeeshop.com` becomes an IP

DNS is the **phonebook of the internet**. Without it, you'd have to remember IPs.

```
   Akash's browser   ── "where is yourcoffeeshop.com?" ──► DNS server
   Akash's browser   ◄──         "at 203.0.113.42"     ── DNS server

   Then:
   Akash's browser   ─── connect to 203.0.113.42:443 ───► your server
```

## How does DNS know?

You bought the domain from a registrar (Namecheap, GoDaddy, Cloudflare) and pointed it at your server's IP. The DNS system propagates this globally within minutes.

## DNS exists at multiple levels

```
   GLOBAL DNS         Public — translates yourcoffeeshop.com → 203.0.113.42

   DOCKER DNS         Inside Docker — translates "redis" → container IP
                      (only works for containers on the same Docker network)
```

Same concept, different scope. Both let you use **names** instead of **numbers**.

---

# 8. The Concierge (reverse proxy) — the production gatekeeper

The single most important new piece in production.

## What it is

A specialized program (usually **nginx**, **Traefik**, or **Caddy**) that sits at the front door and:

1. **Terminates HTTPS** — handles SSL certificates so backend doesn't have to
2. **Routes by URL** — `/api/...` → backend, `/static/...` → frontend, etc.
3. **Adds rate limiting** — protect against abuse
4. **Logs every request** — central log
5. **Hides everything behind it** — internal services are invisible

## Why every production system uses one

```
   WITHOUT a reverse proxy           WITH a reverse proxy
   ─────────────────────────         ──────────────────────────

   Each service has its own          ONE public port (443).
   public port.                      Concierge routes by URL.
                                     
   Frontend on :3000                  yourcoffeeshop.com/        → Frontend
   API on :8000                       yourcoffeeshop.com/api/    → Backend
   Admin on :9000                     yourcoffeeshop.com/admin/  → Backend admin
                                     
   Users see ugly URLs               Clean URLs.
                                     
   Each service must handle HTTPS    ONE place handles HTTPS.
                                     
   Security mess —                   ONE attack surface.
   every port is exposed             Everything else hidden.
```

## In dev — do you need a Concierge?

No. On your laptop you talk directly to services (Django on `:8000`, Redis on `:6379`). The Concierge only earns its keep when there are real users and real attackers. Same code runs the same way; only the wrapper changes.

---

# 9. Internal vs External networks

The hidden plumbing.

## Two networks coexist

```
   ┌────────────────────────────────────────────────────────┐
   │                                                        │
   │   EXTERNAL NETWORK (the internet)                      │
   │   ────────────────────────────────                     │
   │   Only the Concierge touches this.                     │
   │   Public IPs and DNS names.                            │
   │                                                        │
   │   ┌────────────────────────────────────────────────┐   │
   │   │                                                │   │
   │   │   INTERNAL DOCKER NETWORK                      │   │
   │   │   ─────────────────────────                    │   │
   │   │   Containers reach each other by NAME          │   │
   │   │   (redis, backend, postgres, ...)              │   │
   │   │                                                │   │
   │   │   Private IPs (172.x.x.x range)                │   │
   │   │   Invisible from outside                       │   │
   │   │                                                │   │
   │   └────────────────────────────────────────────────┘   │
   └────────────────────────────────────────────────────────┘
```

## Outbound vs inbound

| Direction | Status |
|---|---|
| **Inbound** (internet → server) | ⚠️ Tightly controlled. Only Concierge ports accept this. |
| **Outbound** (container → internet) | ✅ Free. Any container can call Stripe, Gmail, weather API, etc. |

This asymmetry is fundamental — **locked doors don't stop you from leaving.**

## Why this matters for security

If Postgres has no public port:
- Attackers scanning the internet won't even see it
- Brute-force password attacks are impossible from outside
- The only way to reach it is to first compromise another container that already has access

That's called **defense in depth**. Lots of layers, each one harder.

---

# 10. Dev vs Prod — what changes, what stays the same

The most important table in this document.

| Aspect | Dev (laptop) | Prod (server) |
|---|---|---|
| **What's running** | Django on host + Redis in container | Concierge + Frontend + Backend + Redis + Postgres + Workers + Payments + Mailer, ALL in containers |
| **Reverse proxy** | ❌ Not needed | ✅ Required (nginx/Traefik) |
| **DNS** | ❌ `localhost` everywhere | ✅ Public DNS + Docker DNS |
| **HTTPS** | ❌ Plain HTTP is fine | ✅ Required (certs via Let's Encrypt) |
| **Public ports** | Whatever's convenient (`:6379`, `:8000`) | ONLY `:80` and `:443` on the Concierge |
| **Service-to-service addressing** | `localhost:6379` (host talks to Docker) | `redis:6379` (container name resolution) |
| **Redis port exposed?** | ✅ Yes, so DBeaver/redis-cli can poke it | ❌ NO — internal only |
| **Postgres port exposed?** | ✅ Yes for tooling | ❌ NO — internal only |
| **Speed** | Sub-ms (same machine) | A few ms (same Docker network) |
| **Cost** | $0 | Whatever the VPS/cloud charges |

## What stays the same — and why this matters

```
   Django code that talks to Redis:
   ─────────────────────────────────
   redis_client.set('key', 'value')
   
   Same code in dev. Same code in prod.
   Only the connection URL differs:
       Dev:  redis://localhost:6379
       Prod: redis://redis:6379
   
   That URL lives in an environment variable.
   The code doesn't know or care.
```

This is the magic of containers and config-driven networking. **You write code once, it runs anywhere.**

---

# 11. Engineering rules of thumb

| # | Rule |
|---|---|
| 1 | Only the reverse proxy faces the internet in production. Everything else is internal. |
| 2 | Stateful services (DB, Redis) NEVER have public ports in prod. Period. |
| 3 | Containers communicate by **service NAME** on the internal Docker network. |
| 4 | Connection URLs should always come from environment variables, not be hardcoded. |
| 5 | In dev, expose ports for tooling. In prod, expose only `80` and `443`. |
| 6 | Outbound internet from containers is fine — Stripe, Gmail, weather APIs all work. |
| 7 | `localhost` only ever means "this very machine." It doesn't reach containers from other containers. |
| 8 | Port mapping rule: `-p host:container` — the LEFT side is what apps outside dial. |
| 9 | DNS is just a phonebook. Inside Docker there's a private one. Globally there's the public one. |
| 10 | If you can't reach your service, check (in order): is the container running? Is the port mapped? Is the bind address right? |

---

# 12. My confusions, cleared

### Q1. "What does `0.0.0.0:6379->6379/tcp` mean?"

**A:** Docker is listening on the host machine's port 6379, on every network interface (`0.0.0.0`). Anything that connects to that port gets forwarded inside the container to port 6379, using TCP. That's it.

### Q2. "Why does Django use `localhost:6379` instead of the container's IP?"

**A:** Because Docker mapped the container's port 6379 to the host's port 6379. From Django's perspective (which runs on the host), Redis appears to be just another local service on `localhost`. Docker handles the forwarding invisibly.

### Q3. "What if the same scenario happens but it's a real server, not my laptop?"

**A:** Things change dramatically:
- A reverse proxy (Concierge) is added in front
- Only ports 80 and 443 are public
- Internal services use the Docker network and talk by name (`redis:6379`)
- DNS routes `yourdomain.com` to the server's IP
- Multiple services coordinate via the same patterns you learned on laptop

The COMPUTE is identical. The NETWORKING surface area is what scales.

### Q4. "Who 'forwards' requests in production?"

**A:** Two layers:
1. The **OS firewall** of the server lets port 443 traffic through
2. The **reverse proxy** (nginx/Traefik) routes by URL to the right container
3. The **Docker network** routes container-to-container by service name

Three layers, all invisible to the user — they only see the URL they typed.

### Q5. "Why do containers in prod NOT expose ports for Redis/Postgres?"

**A:** Security. The internet has bots constantly scanning every IP for exposed databases. If your Postgres has a public port, it's getting hammered within minutes. The fix: don't have a public port at all. Internal services live on the Docker network only.

### Q6. "What's the difference between Docker DNS and global DNS?"

**A:** Both are phonebooks, different scopes:
- **Global DNS** — translates `yourcoffeeshop.com` → `203.0.113.42` (public, worldwide)
- **Docker DNS** — translates `redis` → `172.18.0.4` (private, inside one Docker network)

You use both. They never overlap.

### Q7. "Can a container call out to Stripe?"

**A:** Yes, no problem. Outbound calls are unrestricted by default — the lockdown is only on inbound (the internet calling you). That's why Room P can call Stripe.

### Q8. "Why does Django code not change between dev and prod?"

**A:** Because the only thing different is the connection string in an environment variable. Code reads `os.getenv('REDIS_URL')`. In dev, that's `redis://localhost:6379`. In prod, it's `redis://redis:6379`. The code itself is identical.

---

# 13. Cheatsheet

## Networking concepts

```
   localhost / 127.0.0.1   "This very machine"
   0.0.0.0                  "All network interfaces — anyone can call"
   Port                     A numbered communication channel (0–65535)
   Port 80                  Default for HTTP
   Port 443                 Default for HTTPS
   Port 6379                Default for Redis
   Port 5432                Default for Postgres
   Port 8000 / 8080         Common dev ports for Django/Flask/etc.
   TCP                      Reliable connection-based protocol (most apps)
   UDP                      Fire-and-forget protocol (video streams, DNS queries)
```

## Docker port flags

```powershell
docker run -p 6379:6379 redis           # standard
docker run -p 127.0.0.1:6379:6379 redis # localhost-only (safer)
docker run -p 8080:6379 redis            # host:8080 → container:6379
docker run redis                         # NO port mapping (internal only)
```

## Useful inspection commands

```powershell
docker ps                          # what's running
netstat -ano | findstr :6379       # who is listening on 6379 (Windows)
docker port redis                  # show what's mapped for one container
docker network ls                  # list all Docker networks
docker network inspect bridge      # see what's on the default network
docker exec -it redis redis-cli    # talk to Redis inside the container
```

## The two URLs you'll keep seeing

```
   DEV:   redis://localhost:6379       (host talks to container via port mapping)
   PROD:  redis://redis:6379           (container talks to container via Docker DNS)
```

---

## 📌 Where this fits in Phase 4

You're in **Scenario A** right now — Django (when running) on your host, Redis in a Docker container, talking via `localhost:6379`. Next we'll wire `django-redis` into `settings.py` so Django actually USES the Redis you started.

Later in Phase 4, we'll write a `docker-compose.yml` that defines services by name and demonstrates **Scenario B** patterns even on your laptop — that's the bridge to production thinking.

---

*Document created during Phase 4 dev/prod flow conversation. The hotel story (Concierge, rooms, internal vs external) is the metaphor anchor — return to it whenever networking gets confusing.*
