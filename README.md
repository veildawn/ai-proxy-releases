# AI Proxy Service

English | [简体中文](README.zh.md)

Self-hosted AI gateway with a multi-tenant admin panel. Pool your Claude, Codex,
Kiro, Kimi, Cursor and xAI subscription and API accounts behind one
OpenAI/Anthropic-compatible endpoint, and hand the whole team API keys with
built-in quotas, billing, usage stats and logs.

A single Go binary with the web UI embedded — one process, one port.

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

Requires Linux (amd64 or arm64) and PostgreSQL.

```sh
# download and unpack the latest binary
gh release download --repo veildawn/ai-proxy-releases -p '*linux_amd64.tar.gz'
tar -xzf ai-proxy-service_*_linux_amd64.tar.gz
```

Write a `config.yaml`. Everything has a usable default except the DSN:

```yaml
server:
  port: 8080
  public_url: http://localhost:8080
database:
  dsn: postgres://postgres:postgres@127.0.0.1:5432/ai_proxy?sslmode=disable
```

Supply the secrets through the environment (or a `.env` beside the binary):

```sh
export JWT_SECRET="$(openssl rand -base64 48)"        # at least 32 chars
export ENCRYPTION_KEY="$(openssl rand -base64 32)"    # exactly 32 bytes, base64
```

Apply the migrations, then serve:

```sh
./ai-proxy-service migrate --config config.yaml
./ai-proxy-service serve   --config config.yaml
```

Open `http://localhost:8080/setup` to create the admin account, then add your
upstream accounts under **Accounts**.

## CLI

```sh
ai-proxy-service serve   --config config.yaml   # API + admin panel
ai-proxy-service migrate --config config.yaml   # apply schema migrations
ai-proxy-service oauth login --provider codex   # link an upstream account
```

## Verifying a download

Every release ships a `checksums.txt`:

```sh
gh release download --repo veildawn/ai-proxy-releases -p 'checksums.txt'
sha256sum -c checksums.txt --ignore-missing
```

## License

MIT
