# AI Proxy Service

English | [简体中文](README.zh.md)

Got Claude, Codex, Kiro, Kimi, Cursor, or xAI subscriptions (or API keys) and want the whole team on them? Drop the accounts into a pool, share one endpoint, and see who used what, what it cost, and which account is down — from one admin panel.

One binary, one port.

## What you get

**Clients**

- Point your usual AI clients at one address and you're in.
- Streaming, tool calls, images, and thinking/reasoning come through. If something can't be translated, you get an error — nothing is silently dropped.
- Codex, Claude, xAI, Kiro, Kimi, Cursor — sign in via the browser, or paste a token. Other compatible upstreams (DeepSeek, Zhipu, local models, …) can be added in the admin UI.
- Route models by name or prefix, and rewrite to whatever the upstream actually expects.

**Account pool**

- Accounts take turns; the same conversation sticks to the same account when it can.
- If one account fails, another picks up — the whole pool doesn't die with one bad key.
- Cap concurrency per account so you don't burn upstream rate limits.
- Each account can use its own outbound proxy.
- Dashboard shows which accounts are alive and how hard the pool is working.

**Team & billing**

- Users, API keys, plans: daily / weekly / fixed-window quotas, plus a monthly budget.
- Open registration, or invite codes only.
- Built-in prices for common models, refreshed on startup; your custom prices stay put.
- Usage and spend panels.

**Day-to-day**

- Upgrades migrate the database for you; a version mismatch refuses to start so you don't walk into a broken schema.
- App logs, request errors, and audit logs — each with its own retention.
- Optional Redis for quotas and OAuth refresh locks.
- One-click self-update from the admin panel.
- Chinese and English UI, light and dark themes.

## Install

### One-line install (Linux amd64 / arm64)

Downloads, checksums, and installs a systemd service. You don't supply secrets — the server generates them on first start.

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/install.sh | sudo bash
```

Needs a reachable PostgreSQL. If yours isn't at the local default, pass the URL:

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/install.sh \
  | sudo DATABASE_URL='postgres://user:pass@host:5432/ai_proxy?sslmode=disable' bash
```

Run it again to upgrade in place; `config.yaml` and `.env` are kept. It won't downgrade unless you pass `ALLOW_DOWNGRADE=1`. Other knobs: `VERSION`, `PORT` (8080), `INSTALL_DIR` (`/opt/ai-proxy-service`), `DATA_DIR` (`/var/lib/ai-proxy-service`), `SERVICE_USER` (`aiproxy`).

```sh
systemctl status ai-proxy-service
journalctl -u ai-proxy-service -f
```

### Docker Compose

Brings its own PostgreSQL; you only pick the database password.

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/docker-compose.yml -o docker-compose.yml
printf 'POSTGRES_PASSWORD=%s\n' "$(openssl rand -hex 24)" > .env
docker compose up -d
```

Config and secrets land in the `/data` volume on first start. To change the port, pin a version (`IMAGE_TAG`), or inject your own secrets, download `env.example` from the same release and use it as `.env`. Upgrade with:

```sh
docker compose pull && docker compose up -d
```

### Manual

```sh
gh release download --repo veildawn/ai-proxy-releases -p '*linux_amd64.tar.gz'
tar -xzf ai-proxy-service_*_linux_amd64.tar.gz    # binary + example config
cp config.example.yaml config.yaml                # usable defaults except the database URL
./ai-proxy-service migrate --config config.yaml
./ai-proxy-service serve   --config config.yaml
```

## First run

Open `http://localhost:8080/setup`. It asks for a **one-time setup token** — printed in the log on first start, and saved next to the config file:

```sh
sudo cat /var/lib/ai-proxy-service/setup-token   # one-line install
sudo cat ./data/setup-token                      # docker compose mounts /data at ./data
docker compose logs app | grep -A2 'one-time token'
```

Once the admin exists, the token is gone. Nobody can create that account without it, so a public instance can't be claimed by a passer-by. Then add your upstream accounts under **Accounts**.

## Secrets

`JWT_SECRET` and `ENCRYPTION_KEY` are generated on first start into `secrets.env` beside the config (mode 0600), and reused from there — they are never rotated for you. **Back up `ENCRYPTION_KEY` with your database: without it, stored upstream credentials are permanently unreadable.** Environment variables win, so you can inject your own anytime. Several instances on one database **must** share the same values, or sessions and credentials won't line up.

## Ports

| Port  | What for                    |
|-------|-----------------------------|
| 8080  | API + admin panel           |
| 1455  | Codex OAuth callback        |
| 54545 | Anthropic OAuth callback    |

The OAuth ports only need to reach the browser you use to link accounts.

## CLI

```sh
ai-proxy-service serve   --config config.yaml               # API + admin panel
ai-proxy-service migrate --config config.yaml               # apply DB migrations
ai-proxy-service oauth login --provider codex|anthropic|xai # link an upstream account
ai-proxy-service version
```

## Verifying a download

Every release ships a `checksums.txt` (the installer checks it for you):

```sh
gh release download --repo veildawn/ai-proxy-releases -p 'checksums.txt' -p '*.tar.gz'
sha256sum -c checksums.txt --ignore-missing
```
