---
name: cockpitify-own-postgres
description: Connect an app whose database is its own Postgres (a self-managed server, AWS RDS / Aurora, Neon, Google Cloud SQL, Railway…) to Cockpitify by running the ops-runner container next to it. Use when the user has a Cockpitify pairing code or a cockpitify.app/setup link, asks to "connect our Postgres to Cockpitify", "run the ops-runner", "set up Cockpitify on our server", or when the runner doesn't pair or connect.
---

# Cockpitify on your own Postgres (ops-runner)

Cockpitify reads an app's users, revenue and AI cost through a small, isolated `ops` layer inside the
app's **own** database. On a Postgres outside Supabase, one container does that job: the
**ops-runner** (`ghcr.io/cockpitify/ops-runner`). It installs the `ops` layer, runs its scheduled jobs,
and serves Cockpitify's app and web console at an HTTPS address. Cockpitify never connects to the
database; the runner talks to Cockpitify. Use only what is written here; don't invent settings.

## What you need from the person

- **The pairing code** (8 letters and digits, shown as `K7QM-4TXP`). It is in the Cockpitify app
  (Connect → Your own Postgres) or after `#` in a link like `https://cockpitify.app/setup#K7QM4TXP`.
  It works once and expires (7 days); the app can make a new one. Pass it without the dash.
- **Which database** the app uses, and a way to reach it from the machine you work on.
- **Which machine runs the container**, and whether it has a domain name you can point at it.

Ask for what's missing. Never print database passwords or keys back in your messages or commit them.

## 1. Check the database

Run against the app's database (with `psql` or the app's own tooling):

```sql
select current_setting('server_version_num')::int >= 150000 as postgres_15_or_newer;
select has_database_privilege(current_user, current_database(), 'CREATE') as can_create_schemas;
select count(*) > 0 as pgcrypto_available from pg_available_extensions where name = 'pgcrypto';
```

All three must be true. If not: Postgres older than 15 can't run it; a user without `CREATE` needs
the database owner (on RDS the master user, on Neon the project owner); `pgcrypto` missing means the
provider doesn't offer it (every major one does). The runner also creates one role, `cockpit_reader`
(read-only questions in Ask), so the user needs `CREATEROLE` or to be the owner.

## 2. Run the container

Generate the host key once and keep it next to the other secrets of that server (a `.env` file
outside git, the platform's secret store). Without it the service keys added later can't be read.

```sh
openssl rand -hex 32   # → OPS_HOST_KEY
```

Docker Compose (preferred: the settings live in one file):

```yaml
services:
  cockpitify-ops:
    image: ghcr.io/cockpitify/ops-runner:0.81.0
    restart: unless-stopped
    ports: ["8080:8080"]
    environment:
      DATABASE_URL: ${DATABASE_URL}        # postgres://USER:PASSWORD@HOST:5432/DB
      OPS_HOST_KEY: ${OPS_HOST_KEY}        # from openssl rand -hex 32; keep it
      OPS_PUBLIC_URL: https://ops.example.com
      OPS_PAIRING_CODE: K7QM4TXP           # first start only
```

Or one command:

```sh
docker run -d --name cockpitify-ops -p 8080:8080 --restart unless-stopped \
  -e DATABASE_URL='postgres://USER:PASSWORD@HOST:5432/DB' \
  -e OPS_HOST_KEY="$OPS_HOST_KEY" \
  -e OPS_PUBLIC_URL=https://ops.example.com \
  -e OPS_PAIRING_CODE=K7QM4TXP \
  ghcr.io/cockpitify/ops-runner:0.81.0
```

| Setting | |
|---|---|
| `DATABASE_URL` | The app's database. Add `?sslmode=require` when the provider needs TLS (RDS, Neon, Cloud SQL). |
| `OPS_HOST_KEY` | 32+ characters. Encrypts the service keys stored in the database. Never change or lose it. |
| `OPS_PUBLIC_URL` | The runner's public HTTPS address, no path needed (step 3). |
| `OPS_PAIRING_CODE` | The code. Only read on the first start; leave it or remove it afterwards. |
| `PORT` | 8080 by default. |

- Run **one** container. A second one serves requests too but leaves the scheduled jobs to the first.
- If the database runs in Docker on the same machine, put the runner on the same Docker network and
  use the database's service name as host (not `localhost`).

## 3. Give it an HTTPS address

The Cockpitify app, its web console and Cockpitify's backend call `OPS_PUBLIC_URL`, so it must be
reachable from the internet over HTTPS, at a public name (never an IP in a private range).
Pick one: a domain name with Caddy in front, or a Cloudflare Tunnel when there is no open port or the
server is behind a firewall. Exact files: [the HTTPS reference](references/https.md).

## 4. Check that it paired

```sh
docker logs cockpitify-ops --tail 50
curl -s https://ops.example.com/healthz
```

- The log shows `"step":"paired"` once, and `/healthz` answers `{"ok":true,"paired":true,…}`.
- Within a few minutes the Cockpitify app moves through Runner paired → First heartbeat and the owner
  gets a push. Tell the person to look there.
- Not paired? Read the log line with `"level":"error"`:
  - `pairing code has expired or was already used` → ask for a new code in the app, set it, restart.
  - `OPS_PUBLIC_URL must be …` → not https, not public, or has a query: fix the address.
  - Connection refused / password / SSL errors → `DATABASE_URL` (host, network, `sslmode`).
  - `not set on this connection` (ops.host_key) → `OPS_HOST_KEY` missing or shorter than 32.
  - Paired but no heartbeat after 15 minutes → the container can't reach the internet, or it was
    stopped; `queued` in `/healthz` should go back to 0.

## 5. After it connects

- **The users table.** Cockpitify needs to know which table holds the app's users. Find it (look at
  the schema: a `users` / `profiles` / `accounts` table with an id, created-at, e-mail and name) and
  tell the person which table and columns to pick in the web console: **Settings → People**.
- **Usage events from the app's server** (works with any sign-in). Make an ingest key and store it in
  the database, encrypted under the host key:
  ```sql
  set ops.host_key = '<OPS_HOST_KEY>';
  select ops.secret_set('ops_ingest_key', '<a new random key, e.g. openssl rand -hex 24>');
  ```
  Then the server posts `{ "user": "<the user's id>", "batch": { … } }` to
  `https://ops.example.com/v1/ingest/server` with the header `x-ops-ingest-key: <key>`. The batch is
  the Ingest Protocol v1 body; the `cockpitify-usage-tracking` skill's HTTP reference has its shape.
  Keep the key in the server's secrets.
- **Webhooks** from RevenueCat, Sentry and Expo go to `https://ops.example.com/webhooks/revenuecat`,
  `/webhooks/sentry` and `/webhooks/expo` (set them up from the Cockpitify app's Connections).
- **AI cost** per user: the `cockpitify-ai-cost` skill.

## Update and remove

- Update: change the image tag to the new version and start it again. It upgrades the `ops` layer in
  one transaction on start; data stays.
- Remove: the owner removes the app in Cockpitify (App settings → Remove), then stop the container.
  The `ops` layer stays in the database until someone runs
  `drop schema ops cascade; drop schema cockpit cascade; drop owned by cockpit_reader; drop role cockpit_reader;`.
  Never run that without the person asking for it.

## Report

End with: where the runner runs, its address, how HTTPS is done, where `OPS_HOST_KEY` is kept (never
its value), whether `/healthz` says `paired: true`, and what the person still has to do (users table
in Settings → People, webhooks, usage events). Help: https://cockpitify.app/help/connect-own-postgres/
