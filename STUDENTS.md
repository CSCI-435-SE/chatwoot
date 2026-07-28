# Chat Help Desk — Student Setup Guide

This is a **course fork of [chatwoot/chatwoot](https://github.com/chatwoot/chatwoot)** — an
open-source customer support and live-chat platform (**Ruby on Rails** API + **Vue 3** frontend,
PostgreSQL with pgvector, Redis, and Sidekiq for background jobs).

📚 **Official documentation:** <https://developers.chatwoot.com/self-hosted>

Verified end to end on macOS (Apple Silicon). The commands are POSIX — on Windows use Git Bash or
WSL.

---

## ⚠️ Read this first

**Use the Docker setup below, not a native install.** Chatwoot needs **Ruby 3.4.4**, and getting
that plus its native gems onto a laptop is far more painful than the container route.

**This fork carries a `docker-compose.override.yaml` that you need.** Upstream's dev compose has
three bugs that stop a fresh setup dead; the override file fixes all three and remaps ports so
Chatwoot can run alongside the other course projects. Compose loads it automatically — just don't
delete it. Each fix is commented in the file if you're curious.

---

## 1. Prerequisites

| Tool | Version | Notes |
| --- | --- | --- |
| **Docker Desktop** | any recent | Everything runs in containers. |
| **Git** | any recent | |

Nothing else — no Ruby, no Node, no Postgres on your machine.

**Give Docker at least 6 GB of RAM** (Docker Desktop → Settings → Resources). The stack idles at
roughly 900 MB, but 4 GB is genuinely tight — see the gotchas.

**macOS:** if Docker Desktop won't start and reports `VZErrorDomain Code=1`, **disable Rosetta** in
its settings.

Clone under a path with **no spaces or apostrophes**.

---

## 2. First-time setup

```bash
git clone <your-team-fork-url>
cd chat-help-desk

cp .env.example .env
openssl rand -hex 64        # paste as SECRET_KEY_BASE in .env
```

Then set these in `.env`:

```ini
SECRET_KEY_BASE=<the value you just generated>
FRONTEND_URL=http://localhost:3004
ENABLE_ACCOUNT_SIGNUP=true

POSTGRES_HOST=postgres
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=

REDIS_URL=redis://redis:6379
REDIS_PASSWORD=

SMTP_ADDRESS=mailhog
SMTP_PORT=1025
```

Build the images. **The order matters** — `rails` and `vite` are built FROM the `base` image, so
`base` has to exist first. Running plain `docker compose build` first will fail with
`chatwoot:development: pull access denied`, because Compose tries to *pull* an image it should be
building.

```bash
docker compose build base          # ~7 min; the bundle install step is the slow part
docker compose build rails vite    # ~2 min
```

Create and seed the database:

```bash
docker compose up -d postgres redis mailhog
docker compose run --rm --no-deps rails bundle exec rails db:chatwoot_prepare
```

Start everything:

```bash
docker compose up -d
```

Give it a minute on the first boot. The app is at **<http://localhost:3004>**.

---

## 3. Log in

The seed creates one SuperAdmin:

| Field | Value |
| --- | --- |
| URL | <http://localhost:3004> |
| Email | `john@acme.inc` |
| Password | `Password1!` |

You land in **Acme Inc** with a sample conversation, contact, and inbox already present. A second
account, **Acme Org**, also exists for testing multi-account behaviour.

Outgoing email goes to **MailHog** at <http://localhost:8028> — there is no real mailbox, so
password resets and invitations all land there.

---

## 4. Ports

Remapped so Chatwoot doesn't collide with the other course projects:

| Service | URL / port |
| --- | --- |
| **App (Rails)** | **<http://localhost:3004>** |
| Vite dev server | 3036 |
| PostgreSQL | 5435 |
| Redis | 6381 |
| MailHog | <http://localhost:8028> (SMTP 1028) |

For reference, the other projects own 3000 (cal.diy), 3001 (Actual), 3002 (Excalidraw),
3003 (Outline), 3030 (Gitea), 4000 (Discourse).

---

## 5. Run the tests

**JavaScript (Vitest)** — about 3.5 minutes:

```bash
docker compose run --rm --no-deps vite pnpm test
```

Expected baseline: **385 files, 3800 tests, 0 failures.**

**Ruby (RSpec)** — run it **per directory**, not all at once:

```bash
docker compose run --rm --no-deps -e RAILS_ENV=test rails bundle exec rspec spec/models
```

Expected baseline for `spec/models`: **972 examples, 2 failures, 23 pending** in about 70 s.

The 2 failures are in `spec/models/conversation_spec.rb` (lines 1132 and 1160) and are
**timing-sensitive, not real breakage** — they assert an elapsed duration is within 1 second of an
hour, and on a container-constrained machine you get `expected 3602.0 to be within 1 of 1 hour`.
They pass on faster hardware.

> ⚠️ **Do not run the whole `rspec` suite.** It **hangs indefinitely** at
> `spec/controllers/widgets_controller_spec.rb`, on the very first example (`/widget → GET
> /widget`). That spec renders a Vite-dependent view, and in the test environment the asset
> pipeline spins at 100 % CPU forever instead of failing. Reproducible in isolation; we let it run
> for over an hour with no progress. Until it's fixed, run directories individually:
>
> ```bash
> for d in models services jobs mailers workers; do
>   docker compose run --rm --no-deps -e RAILS_ENV=test rails bundle exec rspec spec/$d
> done
> ```
>
> Sorting this out is, incidentally, a genuinely useful first contribution.

A single file while you work:

```bash
docker compose run --rm --no-deps -e RAILS_ENV=test rails bundle exec rspec spec/models/user_spec.rb
```

---

## 6. Gotchas we actually hit

1. **`docker compose build` fails with `chatwoot:development: pull access denied`.** Build `base`
   first — see step 2. `rails` and `vite` are derived from it.
2. **Postgres restarts in a loop** with *"Database is uninitialized and superuser password is not
   specified."* Upstream's compose passes an empty `POSTGRES_PASSWORD`, which the Postgres image
   rejects. The override file fixes this with `POSTGRES_HOST_AUTH_METHOD=trust`. If you deleted the
   override, this is why.
3. **The app returns HTTP 500 with `JavaScript heap out of memory`.** This is the big one. The Vite
   dev server binds `::1` (IPv6 loopback) *inside its own container*, so Rails can't reach it and
   falls back to building assets itself — a build that needs ~3 GB of heap and gets OOM-killed.
   The override fixes it by putting Vite in the Rails container's network namespace so both agree
   on `localhost:3036`. **Don't "fix" this by raising `NODE_OPTIONS`** — you don't need the build
   at all.
4. **Give Docker ≥ 6 GB.** At 4 GB the app runs fine, but anything that triggers an asset build
   (including the fallback above) will be killed by the kernel with no useful error.
5. **First boot is slow.** Rails waits for Postgres, runs `bundle check`, and Vite installs
   packages. A minute of nothing is normal; `docker compose logs -f rails` shows progress.
6. **Editing `.env` requires a restart** — `docker compose restart rails sidekiq`.

---

## 7. Daily workflow

```bash
docker compose up -d           # start
docker compose logs -f rails   # follow the app log
docker compose down            # stop (keeps the database)
```

Reset the database to a clean seeded state:

```bash
docker compose run --rm --no-deps rails bundle exec rails db:reset
docker compose run --rm --no-deps rails bundle exec rails db:seed
```

Rails console:

```bash
docker compose run --rm --no-deps rails bundle exec rails console
```

---

## 8. Where things live

| Path | What's in it |
| --- | --- |
| `app/models/`, `app/controllers/` | Rails domain model and REST API |
| `app/javascript/dashboard/` | Vue 3 agent dashboard — most UI work happens here |
| `app/javascript/widget/` | the embeddable chat widget visitors see |
| `app/jobs/`, `app/workers/` | Sidekiq background jobs |
| `enterprise/` | licensed features — **avoid**, it's under a different licence |
| `spec/` | RSpec tests |

Good starter work: dashboard UI, labels and canned responses, contact management, reporting. Be
careful in the real-time messaging path (ActionCable + Sidekiq), where bugs are easy to introduce
and hard to see.
