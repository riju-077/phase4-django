# Phase 4 — Notes 06: HTTPS is the king — but other protocols still matter

> Reference document. HTTPS won the internet, but not every conversation uses it. This doc covers WHY HTTPS dominates, WHEN SMTP and other protocols actually make sense, and WHEN you'd reach for them.

---

## 📚 Table of Contents

1. [HTTPS is the king (and why)](#1-https-is-the-king-and-why)
2. [Why we never `curl smtp://...`](#2-why-we-never-curl-smtp)
3. [When SMTP still matters (the real cases)](#3-when-smtp-still-matters-the-real-cases)
4. [Other non-HTTP protocols you'll meet](#4-other-non-http-protocols-youll-meet)
5. [How to decide which protocol to use](#5-how-to-decide-which-protocol-to-use)
6. [Rules of thumb](#6-rules-of-thumb)
7. [Cheatsheet](#7-cheatsheet)

---

# 1. HTTPS is the king (and why)

**Most internet conversations in 2025 use HTTPS.** Period.

## The 5 reasons HTTPS won

```
   1. PORT 443 IS OPEN EVERYWHERE
      Every firewall, every router, every coffee-shop WiFi
      lets port 443 through. Because everyone needs the web.

   2. UNIVERSAL TOOLING
      curl, Postman, fetch(), requests, axios.
      Every language has an HTTP client. Zero adoption friction.

   3. TEXT-FRIENDLY & DEBUGGABLE
      You can read HTTP headers and JSON bodies with your eyes.
      Binary protocols need special decoders.

   4. TLS HANDLES ENCRYPTION FOR YOU
      The "S" in HTTPS = TLS. Browsers and certs do the crypto.
      You don't invent encryption.

   5. REST + JSON IS LINGUA FRANCA
      Stripe, SendGrid, OpenAI, GitHub — all expose HTTPS REST.
      Learn the pattern once, use everywhere.
```

## What this means in practice

Most modern dev work — calling APIs, fetching data, sending messages, deploying code — is HTTPS. When a new service launches today, **assume the API is HTTPS** unless told otherwise.

---

# 2. Why we never `curl smtp://...`

You noticed something true and sharp: people `curl https://...` all the time, but you've never seen anyone `curl smtp://...`. Here's why.

## The painful reality of speaking SMTP directly

```
   Sending an email via raw SMTP:
   ──────────────────────────────

   $ telnet smtp.gmail.com 587
   220 smtp.gmail.com ESMTP ready
   > EHLO mydomain.com
   250-smtp.gmail.com at your service
   > STARTTLS
   220 Ready to start TLS
   [TLS handshake — usually done with openssl s_client]
   > AUTH LOGIN
   334 VXNlcm5hbWU6
   > [base64-encoded username]
   334 UGFzc3dvcmQ6
   > [base64-encoded password]
   235 Accepted
   > MAIL FROM:<you@gmail.com>
   250 OK
   > RCPT TO:<friend@gmail.com>
   250 OK
   > DATA
   354 Go ahead
   Subject: Test

   Hello!
   .
   250 Queued
   > QUIT
```

15+ lines just to send "Hello." TLS, base64 auth, message formatting — all manual.

## The HTTPS equivalent (via SendGrid)

```powershell
curl -X POST https://api.sendgrid.com/v3/mail/send `
  -H "Authorization: Bearer YOUR_KEY" `
  -H "Content-Type: application/json" `
  -d '{
    "personalizations": [{"to": [{"email": "friend@gmail.com"}]}],
    "from": {"email": "you@yourdomain.com"},
    "subject": "Test",
    "content": [{"type": "text/plain", "value": "Hello!"}]
  }'
```

**10× simpler.** That's why SMTP usage migrated INTO libraries (Django's `send_mail`, Python's `smtplib`) and into HTTPS-fronted services (SendGrid's API). We almost never speak raw SMTP from the CLI.

## The mental model

```
   In modern dev, SMTP is still alive — just BURIED.
   
   Your app  ──►  HTTPS API  ──►  SendGrid's servers  ──►  SMTP  ──►  Gmail
              (your code)       (the SaaS provider)      (under the hood)
              ──────────────────────────────────────    ────────────
                  YOU touch this                          You DON'T touch this
```

You enjoy SMTP's reach without paying SMTP's cost. SaaS providers handle the raw protocol pain.

---

# 3. When SMTP still matters (the real cases)

SMTP isn't dead. There are scenarios where you'd touch it directly. Here are the real ones.

## Case 1 — Inside Django's `send_mail()`

Django's email module uses Python's `smtplib` under the hood. You configure SMTP details in `settings.py`:

```python
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = 'youremail@gmail.com'
EMAIL_HOST_PASSWORD = 'app-password'
```

You're not writing SMTP commands. You're telling Django *which* SMTP server to talk to. Django speaks the protocol for you.

> **You touch SMTP config** even if you don't touch SMTP commands. This is the most common modern interaction.

## Case 2 — Testing email flow locally with Mailpit / MailHog

You run a fake SMTP server in Docker on your laptop. It captures emails Django sends and shows them in a web UI. No real delivery — perfect for dev.

```python
# settings.py for dev
EMAIL_HOST = 'localhost'
EMAIL_PORT = 1025  # Mailpit's default
EMAIL_USE_TLS = False
```

You still don't write SMTP commands — Mailpit is the listener, Django is the client. But you're explicitly working with the SMTP protocol.

## Case 3 — Debugging email deliverability

Production emails not arriving? You might:
- Use `swaks` (a CLI tool built for SMTP testing) to manually send a test
- Use `openssl s_client -connect smtp.provider.com:587` to inspect TLS handshakes
- Check DNS records (SPF, DKIM, DMARC) that SMTP uses

These are admin tasks, not daily code. But when something breaks, this is where you go.

## Case 4 — Self-hosted email server (rare, advanced)

Some companies run their own Postfix/Exim mail server instead of using SendGrid. This is **hard** and most teams don't do it:

- Cloud providers block port 25 outbound (anti-spam)
- Deliverability requires perfect DNS (SPF, DKIM, DMARC, reverse DNS)
- IP reputation takes months to build
- Bounce handling, abuse reports, blacklists — all manual

You'd only do this if you have a really specific reason (extreme volume, compliance, cost at scale).

## Case 5 — Building an email library or SaaS

If you're building "the next SendGrid" — yes, you'd write raw SMTP code. But that's a niche product.

---

# 4. Other non-HTTP protocols you'll meet

HTTPS isn't alone. Here are the protocols you'll encounter throughout your career.

## SSH — Remote terminal access

```
   When you SSH into a server:
   
   ssh user@server.example.com
   
   That's NOT HTTP. It's the SSH protocol on port 22.
   You CANNOT use a browser for this.
```

Tools: OpenSSH, PuTTY, VS Code Remote, GitHub also speaks SSH for git operations.

**When you'd use it:** managing remote servers, secure git, file transfers (SCP/SFTP).

## Database wire protocols (Postgres, MySQL, Mongo, etc.)

```
   When DBeaver talks to your Postgres server:
   
   DBeaver  ──►  Postgres wire protocol  ──►  Postgres server
                  (binary, not HTTP)            (port 5432)
```

Each DB has its own wire protocol. SQL is the language; the wire protocol is the envelope.

**When you'd use it:** any time you connect to a DB. Your ORM (Django ORM, SQLAlchemy) speaks it for you.

## gRPC — High-performance internal service-to-service

```
   Microservices inside Google / Netflix / Stripe often use gRPC
   between each other:
   
   Service A  ──►  gRPC (HTTP/2 + Protobuf)  ──►  Service B
                    (binary, fast, typed)
```

gRPC is technically built ON HTTP/2 but uses a binary format (Protobuf). It's much faster than JSON over HTTPS for high-volume internal traffic.

**When you'd use it:** building microservices that need 10× more performance than REST. Niche but real.

## WebSocket — Real-time bidirectional

```
   For live chat, live dashboards, multiplayer games:
   
   Browser  ───WebSocket───  Server
            (persistent connection, both sides can speak anytime)
```

WebSocket upgrades from HTTP, but it's a different protocol once connected. The server can push messages without the client asking — which HTTP can't.

**When you'd use it:** chat apps, live notifications, collaborative editors, online games.

## DNS — Address book

```
   "Where is google.com?"
   
   Your computer  ──►  DNS server  ──►  "It's at 142.250.X.X"
                       (UDP port 53)
```

DNS uses UDP (not TCP, not HTTPS). It's the phonebook of the internet.

**When you'd use it:** literally every time you visit a website. Your computer does this silently.

## SMTP, IMAP, POP3 — Email family

```
   SMTP  → SENDING email (we covered this)
   IMAP  → READING email (modern, syncs across devices)
   POP3  → READING email (legacy, downloads and deletes)
```

When you set up Thunderbird or Outlook with a custom email account, you configure all three.

## FTP / SFTP — File transfer

Old protocol for moving files between machines. Mostly replaced by HTTPS uploads (S3, Cloudinary). Still used in some legacy enterprise setups.

## MQTT / AMQP — IoT and message queues

Specialized protocols for sensors, message queues (RabbitMQ uses AMQP). Niche.

---

# 5. How to decide which protocol to use

When two services need to talk, how do you pick?

```
   ┌──────────────────────────────────────────────────────────────┐
   │                                                              │
   │   1. DEFAULT TO HTTPS                                         │
   │      For any service-to-service or API call, start here.     │
   │      It's the easiest, most debuggable, most supported.      │
   │                                                              │
   │   2. REACH FOR A SPECIFIC PROTOCOL IF:                       │
   │                                                              │
   │      • Real-time bidirectional needed → WebSocket            │
   │      • Talking to a database → DB's wire protocol            │
   │      • Sending email → SMTP (via library/SaaS)               │
   │      • Remote server access → SSH                            │
   │      • Internal high-perf microservices → gRPC               │
   │      • Name resolution → DNS                                 │
   │      • Pushing to a message queue → AMQP / Redis pubsub      │
   │      • File transfer in legacy systems → SFTP                │
   │                                                              │
   │   3. WHEN IN DOUBT — HTTPS                                    │
   │      Even niche use cases often have HTTPS frontends         │
   │      (databases via REST, email via SendGrid API, etc.)      │
   │                                                              │
   └──────────────────────────────────────────────────────────────┘
```

## A simple test

> *"Is this a request-response interaction over the internet, where I want easy debugging?"* → **HTTPS**.
>
> *"Is this something deeply tied to a specific system (DB, email, terminal, queue)?"* → **That system's protocol**.

---

# 6. Rules of thumb

| # | Rule |
|---|---|
| 1 | Default to HTTPS for new service-to-service communication. |
| 2 | SMTP is alive but buried. You configure it; you don't usually speak it. |
| 3 | When picking a SaaS provider, prefer one with an HTTPS API even if their backend uses SMTP/FTP/whatever. |
| 4 | Don't run your own SMTP/email server unless you have a specific compelling reason. Pay a provider. |
| 5 | Database tools (DBeaver, ORM) speak the DB's wire protocol — not HTTP. Don't confuse SQL (language) with the protocol (transport). |
| 6 | SSH is irreplaceable for server admin. Don't try to "HTTP" your way around it. |
| 7 | WebSocket exists when you need real-time push. HTTPS request/response can't do this efficiently. |
| 8 | DNS quietly underlies everything. UDP port 53. Worth knowing but rarely worth touching directly. |
| 9 | If `curl` can do it, it's HTTPS. If `curl` can't, you'll use a different tool. |

---

# 7. Cheatsheet

## Common protocols + their default ports

| Protocol | Default port | What it's for | When you touch it |
|---|---|---|---|
| HTTPS | 443 | Web, APIs | Constantly |
| HTTP | 80 | Old web (now mostly redirects to HTTPS) | Rarely |
| SSH | 22 | Remote terminal, git over SSH | When SSHing to a server |
| SMTP | 25 / 587 / 465 | Sending email | When configuring email backend |
| IMAP | 143 / 993 | Reading email | When setting up an email client |
| POP3 | 110 / 995 | Reading email (legacy) | Rarely now |
| Postgres | 5432 | Database wire protocol | When connecting to Postgres |
| MySQL | 3306 | Database wire protocol | When connecting to MySQL |
| Redis | 6379 | Cache/queue protocol | When connecting to Redis |
| MongoDB | 27017 | Database wire protocol | When connecting to Mongo |
| DNS | 53 | Name resolution | Indirectly, always |
| FTP / SFTP | 21 / 22 | File transfer | Rarely (legacy systems) |
| WebSocket | 80 / 443 (same as HTTP) | Real-time bidirectional | When building chat / live apps |

## Search queries when you don't know

| Situation | Search |
|---|---|
| "Should I use HTTPS or X for Y?" | `[X] vs https for [use case]` |
| "How do I connect to [service]?" | `[service] connection string format` |
| "What protocol does [tool] use?" | `[tool] protocol port` |
| "How to test SMTP from CLI" | `swaks smtp test command` |
| "Best email SaaS for transactional" | `sendgrid vs postmark vs mailgun 2025` |

---

## 📌 Where this fits in your knowledge

NOTES_03 introduced protocols via the dev-vs-prod request flow.
NOTES_05 cleared the `localhost` vs `0.0.0.0` confusion.
This file (NOTES_06) is the **definitive answer to "is HTTPS everything?"** — no, but it's the default. Other protocols matter in specific scenarios.

---

*Document created during Phase 4. Pin this when you wonder "should I use HTTP for this?"*
