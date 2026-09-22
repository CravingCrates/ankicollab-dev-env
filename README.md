# AnkiCollab local dev environment

One-command local setup for `AnkiCollab-Backend` + `AnkiCollab-Website`, to get you all ready.

## Layout

This repo is a *meta repo* — it holds the compose file and glue config,
and expects the two app repos checked out as siblings inside it:

```
ankicollab-dev-env/
├── docker-compose.yml
├── .env.docker
├── nginx/default.conf
├── backend/     <- clone of AnkiCollab-Backend (contains its own Dockerfile)
└── website/     <- clone of AnkiCollab-Website (contains its own Dockerfile)
```

The `backend/` and `website/` folder *names* matter — the backend's
`Cargo.toml` has a path dependency (`htmldiff = { path = "../website/htmldiff" }`)
that assumes exactly this sibling layout, both on your machine and
inside the Docker build.

Recommended: add backend/website as git submodules so a fresh clone
sets everything up in one shot:

```bash
git clone --recurse-submodules https://github.com/CravingCrates/ankicollab-dev-env ankicollab-dev-env
cd ankicollab-dev-env
```

(Or just `git clone` each repo manually into `backend/` and `website/`
if you don't want submodules.)

## First-time setup

1. Add these to your hosts file (`/etc/hosts` on Mac/Linux,
   `C:\Windows\System32\drivers\etc\hosts` on Windows) — `localhost`
   and `www.localhost` typically already resolve on their own, but
   `plugin.localhost` and `media.localhost` usually need to be explicit:

   ```
   127.0.0.1 plugin.localhost media.localhost
   ```

2. `.env.docker` in this repo already has working dummy credentials —
   nothing to edit to get a first run going.

3. Build and start everything:

   ```bash
   docker compose up --build
   ```

   First run compiles two Rust workspaces from scratch, so expect it to
   take a while. Subsequent runs reuse Docker's layer cache.

## What's running

| Service | What it is | Reachable at |
|---|---|---|
| `db` | Postgres 18, schema auto-loaded from `backend/schema-database.sql` | `localhost:5432` |
| `rustFS` | S3-compatible object storage, standing in for real S3 | API `localhost:9000`, console `localhost:9001` |
| `amazon/aws-cli` | Creates the application bucket once RustFS is ready | — |
| `backend` | the Rust API | `plugin.localhost` (via nginx) directly |
| `website` | the Rust website | `localhost` / `www.localhost` (via nginx) directly |
| `nginx` | reverse proxy, replicates `nginx_localhost_example` | `localhost:80` |

Point the Anki add-on at `http://plugin.localhost`:
The easiest way for that is to navigate to the addons folder in anki, open the AnkiCollab folder (`1957538407` or whatever test folder you're using), and edit the URL inside `var_defs.py` to: 
```
API_BASE_URL = "http://plugin.localhost"
```

## Known unknowns — please verify against source

I put this together from the repos' Cargo.toml/.env.example/nginx example
files, not by reading `main.rs`, so a few things are educated assumptions
rather than certainties:

- **`SENTRY_URL`/`DISCORD_WEBHOOK_URL` left blank/dummy.** If either
  app panics on an empty/fake value instead of treating it as "disabled",
  you'll need to either stub those integrations behind a feature flag or
  provide real-looking values.
- **CF-Connecting-IP trust.** nginx sets this header (spoofing what
  Cloudflare would send in production) and the app is expected to trust
  it. Fine for local dev since nginx is now always in front, but worth
  knowing that's what makes the rate limiting / IP logging behave
  correctly here.

## Never worked with Docker?

Here is a cheat sheet

| What I did                         | Command                                                               |
| ---------------------------------- | --------------------------------------------------------------------- |
| Start normally                     | `docker compose --env-file .env.docker up -d`                         |
| Stop everything                    | `docker compose --env-file .env.docker down`                          |
| See running containers             | `docker compose --env-file .env.docker ps`                            |
| See logs                           | `docker compose --env-file .env.docker logs -f`                       |
| Changed backend                    | `docker compose --env-file .env.docker up -d --build backend`         |
| Changed website                    | `docker compose --env-file .env.docker up -d --build website`         |
| Changed both                       | `docker compose --env-file .env.docker up -d --build backend website` |
| Restart Nginx                      | `docker compose --env-file .env.docker restart nginx`                 |
| Rebuild everything                 | `docker compose --env-file .env.docker up -d --build`                 |
| **Delete DB/data and start fresh** | `docker compose --env-file .env.docker down -v`                       |
