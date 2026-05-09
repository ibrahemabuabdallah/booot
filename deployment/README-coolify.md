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

## 6. Troubleshooting: deployment fails with `git ls-remote` exit code 128

**Symptom:** `fatal: could not read Username for 'https://github.com': No such device or address`

**Cause:** Coolify is cloning your app over **HTTPS** but GitHub needs credentials for a **private** repository (or the linked Git source has no token / expired token).

**Fix (pick one):**

1. **Make the repository public** (GitHub → repo **Settings** → **Danger Zone** → change visibility). Easiest if the code is not secret. Redeploy in Coolify.

2. **Keep the repo private — connect GitHub in Coolify:** In Coolify go to **Sources** (or **Keys & Tokens** / **GitHub** integration, depending on version), add your GitHub account or a **Personal Access Token** (classic) with at least **`repo`** scope for private repos. Re-save the application and pick the repository again so clones use that source.

3. **Deploy key:** Add an SSH deploy key to the GitHub repo (**Settings → Deploy keys**) and configure the application in Coolify to use **SSH** clone URL if your Coolify version supports it for that source.

After fixing Git access, redeploy; the error is **not** caused by PostgreSQL or `FRONTEND_URL` env vars.

## 7. Troubleshooting: Nixpacks build fails on `rake assets:precompile` — Missing `secret_key_base`

**Symptom:** `ArgumentError: Missing 'secret_key_base' for 'production' environment` during `bundle exec rake assets:precompile`.

**Cause:** Coolify injects `RAILS_ENV=production` during the image build. Rails then requires `SECRET_KEY_BASE` while compiling assets. If `SECRET_KEY_BASE` exists only as a **runtime** variable (not available during build), the build fails.

**Fix (pick one):**

1. **Recommended (repo already includes this):** Push the root [`nixpacks.toml`](../nixpacks.toml), which sets a **build-only** `SECRET_KEY_BASE` placeholder and `NIXPACKS_NODE_VERSION=24` to match `package.json` engines. In Coolify, still add your **real** `SECRET_KEY_BASE` (from `bundle exec rake secret`) for **runtime** so sessions stay secure.

2. **In Coolify only:** Edit `SECRET_KEY_BASE` (and any other vars Rails needs at boot) so they are available at **build** time — e.g. enable **Available at Build Time** / **Build Secrets** (wording depends on Coolify version), then redeploy.

3. **Alternative:** Switch the application to build from [`docker/Dockerfile`](../docker/Dockerfile) instead of Nixpacks; that Dockerfile passes `SECRET_KEY_BASE=precompile_placeholder` only for the asset layer.

**Note:** Coolify may warn that `RAILS_ENV=production` during build affects asset precompilation; the placeholder in `nixpacks.toml` addresses the usual failure mode.

**After a successful redeploy:** Run `POSTGRES_STATEMENT_TIMEOUT=600s bundle exec rails db:chatwoot_prepare` once (see §3 above) and ensure Sidekiq uses the same environment variables as the web process.

## 8. Troubleshooting: Nixpacks build fails on `db:chatwoot_prepare` — PostgreSQL host not found

**Symptom:** During image build, `bundle exec rails db:chatwoot_prepare` fails with `ActiveRecord::ConnectionNotEstablished` / `PG::ConnectionBad` / `could not translate host name "<postgresql-service>"` (temporary failure in name resolution).

**Cause:** Nixpacks turns the `release:` line from the root [`Procfile`](../Procfile) into a **Docker build** step. The Coolify PostgreSQL hostname exists only on the **runtime** Docker network, not while the image is building.

**Fix (repo already includes this):** The root [`nixpacks.toml`](../nixpacks.toml) defines `[phases.release]` so the release phase no longer runs `db:chatwoot_prepare` at build time; it only writes `.git_sha` when `SOURCE_VERSION` is set. You **must** still run `POSTGRES_STATEMENT_TIMEOUT=600s bundle exec rails db:chatwoot_prepare` once after the first successful deploy (§3).

**If you deploy without Nixpacks** (e.g. plain `docker/Dockerfile`), this section does not apply; the upstream Dockerfile does not run `db:chatwoot_prepare` at image build time in the same way.
