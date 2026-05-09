# Chatwoot on Coolify (PostgreSQL + domain)

This folder supports the self-hosted wiring described in your deployment plan. Chatwoot reads configuration from environment variables (`config/database.yml`, etc.); do not commit real secrets to git.

## 1. PostgreSQL (Coolify)

1. Open your **PostgreSQL** resource in Coolify and copy the **internal** hostname (sometimes labeled Internal DNS, FQDN, or shown inside a connection URI).
2. In your **Chatwoot web** (and **Sidekiq**) application → **Environment Variables**, add the variables from [`coolify-env.template`](coolify-env.template), replacing placeholders.
3. Prefer Coolify’s **link database to application** feature when available so `POSTGRES_*` values stay correct.

## 2. Domain

Set `FRONTEND_URL` and `FORCE_SSL` exactly as in the template so they match the URL users open in the browser. Keep Coolify **Domains** aligned with `FRONTEND_URL`. Redeploy after changes.

## 3. First-time database setup

After the web container can reach PostgreSQL, run once (Coolify **Post-deployment** or one-off command):

```bash
POSTGRES_STATEMENT_TIMEOUT=600s bundle exec rails db:chatwoot_prepare
```

On your workstation (after `bundle install` and a working `.env` pointing at PostgreSQL), the same command applies:

```bash
POSTGRES_STATEMENT_TIMEOUT=600s bundle exec rails db:chatwoot_prepare
```

## 4. Local development against remote Postgres

Copy lines from [`local-remote-postgres.fragment.env`](local-remote-postgres.fragment.env) into your root `.env` (gitignored) and uncomment/fill values. Ensure port `5432` is reachable from your PC or use an SSH tunnel.

## 5. Leaked credentials

If a database password or other secret was exposed in chat or a ticket, rotate it in Coolify and update every copy (Coolify env, local `.env`, backup scripts). See [SECURITY.md](../SECURITY.md).
