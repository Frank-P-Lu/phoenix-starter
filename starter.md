# Stack Setup Guide (Phoenix + LiveView + Oban + Postgres + Fly.io + GitHub Actions)

This document is a lightweight, version-agnostic reference for bootstrapping a *new* project with the same stack. It intentionally avoids pinning dependency versions so a future agent can select the latest stable releases at the time of setup.

## Goals

- Use Phoenix with LiveView for the web app
- Use Postgres for persistence
- Use Oban for background jobs and scheduled work
- Deploy to Fly.io
- Use GitHub Actions for CI (and deploy on `main` if desired)

## Stack Summary

- **Web Framework:** Phoenix (with LiveView)
- **Database:** Postgres
- **Job Queue:** Oban (Postgres-backed)
- **Hosting:** Fly.io
- **CI/CD:** GitHub Actions

## Bootstrap Steps (High Level)

1. Create a new Phoenix project with LiveView enabled.
2. Add Postgres config for dev/test/prod.
3. Add Oban with Postgres engine and queues.
4. Add Fly.io deploy config.
5. Add GitHub Actions workflow for CI (tests, lint, formatting).

## Phoenix Project Creation

When generating a new Phoenix app:

- Enable LiveView
- Use Postgres as the DB
- Keep assets pipeline standard (Tailwind + esbuild)

Example (adjust name and options as needed):

```bash
mix phx.new my_app --live
```

After creation:

```bash
cd my_app
mix ecto.create
mix ecto.migrate
```

## Database (Postgres)

Use Postgres in all environments:

- **dev:** local Postgres
- **test:** isolated test database
- **prod:** managed Postgres (Fly.io Postgres or external)

Typical config elements:

- `username`, `password`, `hostname`, `database`
- `pool_size` and SSL options for prod
- `DATABASE_URL` env var for production

## Oban (Background Jobs)

Oban uses Postgres and should be configured early since it requires migrations.

Checklist:

- Add Oban dependency
- Configure Oban in `config/config.exs`
- Run Oban migrations
- Add Oban to supervision tree
- Set queues and plugins

Example config shape (names are illustrative):

```elixir
config :my_app, Oban,
  repo: MyApp.Repo,
  plugins: [
    {Oban.Plugins.Pruner, max_age: 60 * 60 * 24 * 7}
  ],
  queues: [
    default: 10,
    low: 5
  ]
```

## Fly.io Deployment

Use Fly.io for hosting. Basic flow:

1. Install Fly CLI
2. Initialize app
3. Create Postgres cluster or use external DB
4. Set secrets
5. Deploy

Commands (adjust app name):

```bash
fly launch --no-deploy
fly postgres create
fly postgres attach --app my_app
fly secrets set SECRET_KEY_BASE=... DATABASE_URL=...
fly deploy
```

Notes:

- Keep `fly.toml` in the repo
- Ensure a release command runs migrations (e.g., `mix ecto.migrate`)
- Configure health checks and HTTP port

## GitHub Actions (CI/CD)

Minimum CI steps:

- Install Elixir/Erlang
- Install deps
- Set up Postgres service
- Compile with warnings as errors
- Check formatting
- Run Credo
- Run Sobelow
- Run tests

Optional CD step:

- On `main` merge, deploy to Fly.io (requires `FLY_API_TOKEN`)

Suggested workflow outline:

1. `mix deps.get`
2. `mix compile --warnings-as-errors`
3. `mix format --check-formatted`
4. `mix credo --strict`
5. `mix sobelow --config`
6. `mix test`

## Pre-Commit (Local Quality Gate)

Mirror the local pre-commit flow used in this repo. Create a `precommit` alias in `mix.exs` and run it before merges.

Current flow (in order):

1. `mix compile --warning-as-errors`
2. `mix deps.unlock --unused`
3. `mix format`
4. `mix dialyzer`
5. `mix credo --strict`
6. `mix sobelow --config`
7. `mix test`

Notes:

- Set `preferred_envs: [precommit: :test]` in `mix.exs` so the alias runs in the test environment.
- Keep `.dialyzer_ignore.exs` and a `priv/plts/dialyzer.plt` location for Dialyzer.
- Keep `.sobelow-conf` and run Sobelow with `--config` (this repo uses `exit: "high"`).
- Ensure these deps exist in `mix.exs`: `:credo`, `:sobelow`, `:dialyxir` (versions should be latest stable).

## Security & Static Analysis

Use the same baseline as this repo:

- **Credo** for linting and style (`mix credo --strict`)
- **Sobelow** for security checks (`mix sobelow --config`)
- **Dialyzer** for static analysis (`mix dialyzer`)

Optionally add Dialyzer to CI if runtime allows, but keep it in local pre-commit.

## Fly.io `fly.toml` (Minimal Starter)

Use a small, explicit config and adjust values as needed:

```toml
app = "my_app"
primary_region = "iad"
kill_signal = "SIGTERM"

[build]

[deploy]
  release_command = "/app/bin/migrate"

[env]
  PHX_HOST = "myapp.example.com"
  PORT = "8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 0
  processes = ["app"]
```

Add sizing and concurrency settings later if needed.

## Environment Variables (Common)

- `DATABASE_URL` (prod)
- `SECRET_KEY_BASE`
- `PHX_HOST`
- `PORT`
- `OBAN_LICENSE_KEY` (if using Oban Pro)

## Practical Defaults

- Keep dependency versions unpinned in this document; use latest stable releases when setting up a new project.
- Prefer Postgres-backed queues for reliability.
- Keep the CI workflow minimal and fast; add extra checks only as needed.

## AGENTS.md (General Guidelines)

Phoenix projects already include the Phoenix-specific agent guidance. Add a short, project-agnostic section for how agents should work in this repo:

- Run `mix precommit` after changes and fix any failures.


## Next Decisions for a New Project

- Auth strategy (e.g., magic link vs. password)
- Email provider and templates
- Queue structure (number of queues, priority)
- Production Postgres provider (Fly.io vs. managed external)
- Deployment frequency (manual vs. automatic on `main`)
