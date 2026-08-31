# Node.js + Postgres with Docker Compose

A two-container stack: a Node.js application and a Postgres database, wired together
with Docker Compose. Service-name networking, healthcheck-gated startup, a named
volume for persistence, and every environment-specific value passed in as
configuration rather than written into the code.

Built from scratch as a working reference for containerised application deployment.

---

## The problem

Running an application and its database in separate containers introduces three
problems that a naive `docker-compose.yml` does not solve:

**The application starts before the database is ready.** Containers start in seconds;
Postgres takes longer to accept connections. The app connects, fails, and exits. This
usually works on a fast machine and fails on a slow one — which makes it an
intermittent bug rather than an obvious one.

**Data disappears.** Without a named volume, everything in the database is deleted the
moment the container is removed.

**Configuration ends up in the code.** If a database host, username or password is
written into the source, the same code cannot run in a different environment without
being edited — which means maintaining two copies of it.

This repository is a small stack that solves all three.

---

## What it does

```
                    ┌──────────────────┐
   localhost:3000 ──▶  app             │
                    │  Node.js         │
                    │  Express + pg    │
                    └────────┬─────────┘
                             │  reaches the database as "db"
                             │  no IP addresses anywhere
                    ┌────────▼─────────┐
                    │  db              │
                    │  Postgres 16     │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  pgdata volume   │  survives container removal
                    └──────────────────┘
```

The application exposes one endpoint. It queries the database and returns the result,
which is enough to prove the connection works end to end:

```bash
curl http://localhost:3000
{"status":"ok","db_time":"2026-08-31T00:13:40.437Z"}
```

That timestamp comes from Postgres, not from Node.

---

## Layout

```
docker-compose.yml     # both services, network, volume
.env                   # local config — not committed
.env.example           # the keys, without the values
app/
  Dockerfile           # builds the application image
  index.js             # Express server, reads config from environment
  package.json
```

---

## Key decisions

### 1. Services find each other by name, not by IP

The application connects to `DB_HOST: db` — the name of the database service.

Compose puts both containers on a shared network where each service is resolvable by
its service name. No IP addresses appear anywhere in the configuration, so nothing
breaks when containers are recreated and get different addresses.

### 2. The application waits for the database to be *ready*, not just *started*

```yaml
depends_on:
  db:
    condition: service_healthy
```

Plain `depends_on` waits for the database container to start. It does not wait for
Postgres inside it to accept connections — and the gap between those two moments is
where the application crashes.

Pairing `depends_on` with a healthcheck closes that gap:

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
  interval: 5s
  retries: 5
```

Compose now polls until Postgres genuinely answers before starting the application.

### 3. Database storage lives in a named volume

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
```

Postgres writes to a volume rather than the container's own filesystem. Stop and
recreate the containers and the data is still there — visible in the logs on restart:

```
PostgreSQL Database directory appears to contain a database; Skipping initialization
```

Removing the volume explicitly (`docker compose down -v`) starts clean.

### 4. Every environment-specific value is configuration, not code

```js
const pool = new Pool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
});
```

Nothing about where the database lives is written into the application. The same image
runs against the Postgres container here, or against a managed database such as RDS,
by changing values rather than code.

This matters more than it first appears. The moment a hostname or credential is
hardcoded, running in a second environment requires editing the source — and that is
how one codebase quietly becomes two that have to be maintained separately.

### 5. Dependencies are installed before the application code is copied

```dockerfile
COPY package*.json ./
RUN npm install --omit=dev
COPY . .
```

Docker caches each build step and reuses it when its inputs have not changed. Copying
only `package.json` before installing means the install layer depends on the dependency
list alone. Changing application code no longer invalidates it, so rebuilds skip the
slowest step entirely.

Reversing these lines — copying everything first — is a common and easily missed
mistake. It works, and it makes every single rebuild slow.

---

## Running it

**Requires:** Docker and Docker Compose.

```bash
git clone https://github.com/0dow0ri7s3/node-postgres-docker-compose.git
cd node-postgres-docker-compose

cp .env.example .env      # then set your own values
docker compose up --build
```

```bash
curl http://localhost:3000
```

**Stop:**
```bash
docker compose down       # keeps the data
docker compose down -v    # removes the volume too
```

---

## Troubleshooting log — FATAL errors from a healthy application

On the first run, the database logs filled with an error repeating every five seconds:

```
FATAL:  database "appuser" does not exist
```

`appuser` is the database *user*. The database is `appdb`. Something was connecting
with the username in the database field.

**Ruling out the application.** Two details pointed away from it. The errors began at
00:09:00, five seconds *before* the application reported `listening on 3000` — so they
predated the app being able to send anything. And they repeated on a precise five-second
cycle, which is not how an application behaves when nothing is calling it.

**Finding the five-second cycle.** The only thing in the stack running on a five-second
interval was the healthcheck:

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
  interval: 5s
```

**Root cause.** `pg_isready -U appuser` specifies a user but no database. Postgres
defaults the database name to the username when it is not given, so it was looking for
a database called `appuser`, which does not exist.

**Fix.** Pass the database explicitly:

```yaml
test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
```

### What this one teaches

The application was working correctly the entire time. `curl` returned a valid timestamp
from Postgres while the logs were still filling with `FATAL`.

The healthcheck also reported *healthy* throughout, because `pg_isready` treats a server
that responds — even with an error — as a server that is up. So the check was passing for
the wrong reason.

Two lessons worth keeping. A log full of alarming errors is not automatically the problem
you are looking for. And the timing of an error is evidence: a message on a fixed
interval is coming from something scheduled, not from something reacting.

---

## What I would improve

**Add database migrations.** There is no schema — the app runs a single `SELECT NOW()`.
A real stack needs migrations that run in a defined, repeatable order.

**Run the application as a non-root user.** The container currently runs as root.
Production images should add a dedicated user and drop to it.

**Add a health endpoint for the application itself.** The database has a healthcheck; the
app does not. A `/health` route would let Compose or an orchestrator detect a hung
process rather than only a stopped one.

**Pin the base image by digest.** `node:20-alpine` will change over time. Pinning the
digest makes builds reproducible.

**Handle secrets properly.** Passwords currently pass through environment variables,
which is acceptable locally and not sufficient in production. Docker secrets or a
managed secret store would be the correct pattern.

---

**Built by [Odoworitse Afari](https://github.com/0dow0ri7s3)** — Cloud & DevOps Engineer.
