# Node.js + Postgres with Docker Compose

A two-container stack: a Node.js application and a Postgres database, wired together
with Docker Compose. Service-name networking, a healthcheck that actually tests the
database, a named volume for persistence, and every environment-specific value passed
in as configuration rather than written into the code.

Built from scratch as a working reference for containerised application deployment.

---

## The problem

Running an application and its database in separate containers introduces three
problems a naive `docker-compose.yml` does not solve:

**The application starts before the database is ready.** Containers start in seconds;
Postgres takes longer to accept connections. The app connects, fails, and exits. This
usually works on a fast machine and fails on a slow one — which makes it an
intermittent bug rather than an obvious one.

**Data disappears.** Without a named volume, everything in the database is deleted the
moment the container is removed.

**Configuration ends up in the code.** If a database host, username or password is
written into the source, the same code cannot run in a different environment without
being edited — which means maintaining two copies of it.

This repository is a small stack that solves all three, and a worked example of a
fourth problem that is harder to see: a healthcheck that reports success for the wrong
reason.

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

One endpoint, which queries the database and returns the result — enough to prove the
connection works end to end:

```bash
curl http://localhost:3000
{"status":"ok","db_time":"2026-09-12T20:26:09.318Z"}
```

That timestamp comes from Postgres, not from Node.

---

## Layout

```
docker-compose.yml     # both services, network, volume
.env                   # local config — not committed
.env.example           # the keys, ready to copy
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

### 2. The healthcheck runs a real query

```yaml
healthcheck:
  test: ["CMD-SHELL", "psql -U ${DB_USER} -d ${DB_NAME} -c 'select 1' || exit 1"]
  interval: 5s
  retries: 5
```

Paired with:

```yaml
depends_on:
  db:
    condition: service_healthy
```

Plain `depends_on` waits for the database container to start. It does not wait for
Postgres inside it to accept connections, and the gap between those two moments is
where the application crashes.

The check above authenticates as the application's user, connects to the application's
database, and executes a query. Every one of those steps can fail independently, and
each produces a red.

**This replaced `pg_isready`, which could not.** See the troubleshooting log below.

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
runs against the Postgres container here, or against a managed database such as RDS, by
changing values rather than code.

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

cp .env.example .env      # then set a password
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

## Troubleshooting log — a healthcheck that was green and wrong

### The symptom

On the first run, the database logs filled with an error repeating every five seconds:

```
FATAL:  database "appuser" does not exist
```

`appuser` is the database *user*. The database is `appdb`. Something was connecting
with the username in the database field.

Meanwhile the application was working. `curl` returned a valid timestamp read straight
from Postgres — not "appeared to work", actually working, against the database
something claimed did not exist.

### Ruling out the application

Two details pointed away from it.

**The errors started before the app did.** First error at 00:09:00; the app reported
`listening on 3000` at 00:09:05. They predated it being able to send anything.

**They repeated on an exact five-second cycle.** Applications do not behave that way
when nothing is calling them. Scheduled things do.

### Finding the five-second cycle

The only thing in the stack running on a five-second interval was the healthcheck:

```yaml
test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
interval: 5s
```

`pg_isready -U appuser` specifies a user but no database. Postgres defaults the database
name to the username when it is not given, so it was asking for a database called
`appuser`, which does not exist.

### The part that mattered more than the fix

Adding `-d ${DB_NAME}` stopped the errors. It did not fix the check.

**The healthcheck had reported `healthy` throughout.** Every failed check came back
green, because `pg_isready` treats any response from the server — including an error —
as the server being up.

It was not passing because things were fine. It was passing for the wrong reason, and
the two are indistinguishable from outside.

`pg_isready` does not authenticate and does not run a query. The only red it can
produce is roughly "the server is not accepting connections". Credentials, permissions,
whether the schema exists — all of it sits outside what the check can see.

*(This distinction was sharpened by [Nguyen Thanh Vinh](https://www.linkedin.com/in/vinhnguyen203/),
who pointed out that `-d` fixes the noise rather than the check.)*

### The fix, and how it was tested

The check was replaced with one that connects and runs a query:

```yaml
test: ["CMD-SHELL", "psql -U ${DB_USER} -d ${DB_NAME} -c 'select 1' || exit 1"]
```

A fix for a check that cannot fail is worth nothing unless you can make it fail. So the
new one was tested by injecting a fault.

**First attempt — changing `DB_NAME` to a nonexistent value — did not work as a test.**
`DB_NAME` sets `POSTGRES_DB` on the database service *and* is read by the healthcheck.
Changing it moved both sides together: Postgres created a database with the new name and
the check asked for the new name. Consistent, and correctly green.

So only one side was broken. The check was hardcoded to a database that does not exist:

```yaml
test: ["CMD-SHELL", "psql -U ${DB_USER} -d nosuchdb -c 'select 1' || exit 1"]
```

**Result:**

```
db-1  | FATAL:  database "nosuchdb" does not exist
db-1  | FATAL:  database "nosuchdb" does not exist
Container node-postgres-docker-compose-db-1  Error
dependency db failed to start
```

The application never started.

Compare that to the original failure: identical FATAL lines in the log, container marked
`healthy`, application started anyway.

**Same symptom. Opposite outcome.** The old check could not distinguish a working
database from a missing one. The new one holds the application at the door until a real
query succeeds.

### What this one teaches

A log full of alarming errors is not automatically the problem you are looking for — the
application was fine the entire time.

The timing of an error is evidence. A message on a fixed interval comes from something
scheduled, not something reacting. That single observation cut the search in half.

And a check that cannot go red is not a check. If it gates anything — and with
`condition: service_healthy` it gates the whole application — then a green for the wrong
reason is not log noise. It is a release on a signal that was never measuring the thing
it was gating.

---

## A smaller thing worth noting

Compose and Postgres disagree about what to do with missing configuration.

Running without a `.env` file, Compose warned and continued:

```
level=warning msg="The \"DB_PASSWORD\" variable is not set. Defaulting to a blank string."
```

Postgres refused:

```
Error: Database is uninitialized and superuser password is not specified.
```

Same missing input. One fails open, one fails closed. Postgres is right — a database
created with a blank superuser password is worse than a database that did not start.

`.env.example` in this repository is ready to copy directly, with only the password left
blank. An example file whose lines are all commented out reproduces the Compose failure
above for anyone cloning it.

---

## What I would improve

**Add database migrations.** There is no schema — the app runs a single `SELECT NOW()`.
A real stack needs migrations that run in a defined, repeatable order.

**Run the application as a non-root user.** The container currently runs as root.
Production images should add a dedicated user and drop to it.

**Add a health endpoint for the application itself.** The database has a healthcheck;
the app does not. A `/health` route would let Compose or an orchestrator detect a hung
process rather than only a stopped one.

**Pin the base image by digest.** `node:20-alpine` will change over time. Pinning the
digest makes builds reproducible.

**Handle secrets properly.** Passwords currently pass through environment variables,
which is acceptable locally and not sufficient in production. Docker secrets or a
managed secret store would be the correct pattern.

---

**Built by [Odoworitse Afari](https://github.com/0dow0ri7s3)** — Cloud & DevOps Engineer.
