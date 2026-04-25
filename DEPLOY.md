# Deploying the bot

This bot is a single Node.js service that uses **Telegram long-polling**, so it does not need a public URL or webhook setup. Any host that can run a Node.js / Docker process works.

## What you need on every host

- Set the env var **`TELEGRAM_BOT_TOKEN`** (from [@BotFather](https://t.me/BotFather)).
- Optional: `BOT_OWNER_ID` (your Telegram user ID — the owner can use admin commands), `OWNER_EMAIL`.
- A persistent volume mounted at `/app/data` (the bot stores its SQLite DB there). On hosts without volumes the bot still runs, but data resets on every redeploy.
- Set `DATABASE_URL=/app/data/bot.db` to use the mounted volume (already pre-set in the configs below).

The container exposes port `8080` for a health check at `GET /api/healthz`. The bot itself does not need an inbound port.

---

## 1. Render (`render.yaml`)

1. Push this repo to GitHub.
2. On Render → **New** → **Blueprint** → pick the repo.
3. Render reads `render.yaml`, creates the service + a 1 GB disk at `/var/data`, and asks for `TELEGRAM_BOT_TOKEN`.

## 2. Railway (`railway.json`)

1. Railway → **New Project** → **Deploy from GitHub repo**.
2. Railway picks up `railway.json` and uses the `Dockerfile`.
3. In **Variables**, set `TELEGRAM_BOT_TOKEN` (and optional `BOT_OWNER_ID`).
4. In **Settings → Volumes**, add a volume mounted at `/app/data`.

## 3. Fly.io (`fly.toml`)

```bash
fly launch --copy-config --no-deploy        # accept defaults; keep fly.toml
fly volumes create bot_data --size 1
fly secrets set TELEGRAM_BOT_TOKEN=...      # also: BOT_OWNER_ID, OWNER_EMAIL
fly deploy
```

## 4. Heroku (`heroku.yml` + `app.json`)

```bash
heroku create world-money-bot --stack=container
heroku config:set TELEGRAM_BOT_TOKEN=...
git push heroku main
```

> Heroku's filesystem is ephemeral. For persistent state, switch `DATABASE_URL` to a Postgres add-on path (you'd also need to swap the SQLite driver — keep this host as a quick try-out only).

## 5. Koyeb (`koyeb.yaml`)

```bash
koyeb secret create telegram-bot-token --value <token>
koyeb app init --file koyeb.yaml
```

Add a persistent volume mounted at `/app/data` from the Koyeb dashboard.

## 6. Any Docker host (VPS, Northflank, DigitalOcean App Platform, Cloud Run, AWS App Runner, etc.)

```bash
docker build -t world-money-bot .
docker run -d --name world-money-bot \
  -e TELEGRAM_BOT_TOKEN=... \
  -e DATABASE_URL=/app/data/bot.db \
  -v world_money_bot_data:/app/data \
  world-money-bot
```

## 7. Plain Node host (no Docker)

Requires Node 24 and pnpm 10.

```bash
pnpm install --frozen-lockfile
pnpm --filter @workspace/api-server run build
TELEGRAM_BOT_TOKEN=... \
DATABASE_URL=./data/bot.db \
PORT=8080 \
node --enable-source-maps artifacts/api-server/dist/index.mjs
```

The repo also includes a `Procfile` for buildpack-based hosts that prefer that flow.

---

## Local development

```bash
pnpm install
TELEGRAM_BOT_TOKEN=... PORT=8080 pnpm --filter @workspace/api-server run dev
```
