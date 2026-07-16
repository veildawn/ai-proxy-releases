# AI Proxy Service

English | [简体中文](README.zh.md)

Self-hosted AI gateway with a multi-tenant admin panel. Pool your Claude, Codex,
Kiro, Kimi, Cursor and xAI subscription and API accounts behind one
OpenAI/Anthropic-compatible endpoint, and hand the whole team API keys with
built-in quotas, billing, usage stats and logs.

A single Go binary with the web UI embedded — one process, one port.

This repository ships the release artifacts. The source is private.

## Features

**Gateway**

- One endpoint for every client: OpenAI `/v1/chat/completions` and Anthropic
  `/v1/messages`, translated across protocols in both directions.
- Streaming SSE, tools / `tool_choice`, image content, reasoning/thinking and
  streamed tool events survive the translation. Anything that cannot be
  preserved returns a 400 rather than being silently dropped.
- Providers: Codex, Claude, xAI, Kiro, Kimi, Cursor — via OAuth (PKCE / device
  code) or a manual access token. Any OpenAI-/Anthropic-compatible upstream
  (DeepSeek, Zhipu, OpenCode Zen, local vLLM, …) can be added from the admin UI.
- Model routing by explicit route or prefix, with upstream model rewriting.

**Account pool**

- Round-robin scheduling with session affinity.
- Bounded cross-account failover plus in-place retries on upstream errors.
- Per-account concurrency caps, so pooled subscriptions don't trip provider 429s.
- Per-account outbound proxy routing.
- Account health and pool utilization on the dashboard.

**Tenants & billing**

- Users, API keys, and subscription plans: fixed-window / daily / weekly quotas
  plus a monthly budget.
- Open or invite-code-only registration.
- Built-in model pricing, synced for built-in models on startup; custom pricing
  is left untouched.
- Usage and cost stats, with Analysis and Realtime dashboards.

**Operations**

- Versioned SQL migrations with a schema-version check at startup.
- Application, request-error and audit logs, each with a retention window.
- Optional Redis backend for quotas and OAuth refresh locks.
- One-click self-update from the admin panel.
- Bilingual UI (zh-CN / en), light and dark themes.

## Install

### One-line install (Linux amd64/arm64, systemd)

Downloads the binary, verifies its checksum, and installs a systemd service. You
do not supply any secrets — the server generates them on first start.

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/install.sh | sudo bash
```

Needs a reachable PostgreSQL. If yours is not at the local default, set
`DATABASE_URL`:

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/install.sh \
  | sudo DATABASE_URL='postgres://user:pass@host:5432/ai_proxy?sslmode=disable' bash
```

Re-running it upgrades in place; your `config.yaml` and `.env` are kept. It
refuses to downgrade unless you pass `ALLOW_DOWNGRADE=1`. Other knobs:
`VERSION`, `PORT` (8080), `INSTALL_DIR` (`/opt/ai-proxy-service`), `DATA_DIR`
(`/var/lib/ai-proxy-service`), `SERVICE_USER` (`aiproxy`).

```sh
systemctl status ai-proxy-service
journalctl -u ai-proxy-service -f
```

### Docker Compose

Brings its own PostgreSQL; the database password is the only thing to decide.

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/docker-compose.yml -o docker-compose.yml
printf 'POSTGRES_PASSWORD=%s\n' "$(openssl rand -hex 24)" > .env
docker compose up -d
```

`config.yaml` and the app secrets are created in the `/data` volume on first
start. For the other settings — `PORT`, `IMAGE_TAG` to pin a release instead of
tracking `latest`, and optional injected secrets — download `env.example` from
the same release and use it as your `.env`. Upgrade with
`docker compose pull && docker compose up -d`.

### Manual

```sh
gh release download --repo veildawn/ai-proxy-releases -p '*linux_amd64.tar.gz'
tar -xzf ai-proxy-service_*_linux_amd64.tar.gz    # binary + config.example.yaml + .env.example
cp config.example.yaml config.yaml                # everything has a usable default except the DSN
./ai-proxy-service migrate --config config.yaml
./ai-proxy-service serve   --config config.yaml
```

## First run

Open `http://localhost:8080/setup`. It asks for a **one-time setup token**,
printed to the log on first start and written next to `config.yaml` (mode 0600):

```sh
sudo cat /var/lib/ai-proxy-service/setup-token   # one-line install
sudo cat ./data/setup-token                      # docker compose mounts /data at ./data
docker compose logs app | grep -A2 'one-time token'
```

The token dies the moment the super-admin exists. Nobody can create that account
without it, so a public instance cannot be claimed by a passer-by. Then add your
upstream accounts under **Accounts**.

## Secrets

`JWT_SECRET` and `ENCRYPTION_KEY` are generated on first start into
`secrets.env` (0600) beside `config.yaml`, and reused from there — they are never
rotated for you. **Back up `ENCRYPTION_KEY` with your database: without it the
stored upstream credentials are permanently undecryptable.** Environment
variables win over the generated values, so you can inject your own at any time.
Several instances sharing one database *must* be given identical values, or
sessions and credentials won't work across them.

## Ports

| Port  | Purpose                                    |
|-------|--------------------------------------------|
| 8080  | API + admin panel                          |
| 1455  | Codex OAuth callback                       |
| 54545 | Anthropic OAuth callback                   |

The OAuth callback ports only need to be reachable from the browser you link
accounts with.

## CLI

```sh
ai-proxy-service serve   --config config.yaml               # API + admin panel
ai-proxy-service migrate --config config.yaml               # apply schema migrations
ai-proxy-service oauth login --provider codex|anthropic|xai # link an upstream account
ai-proxy-service version
```

## Verifying a download

Every release ships a `checksums.txt` (the installer checks it for you):

```sh
gh release download --repo veildawn/ai-proxy-releases -p 'checksums.txt' -p '*.tar.gz'
sha256sum -c checksums.txt --ignore-missing
```

## License

MIT
