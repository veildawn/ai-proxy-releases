# AI Proxy Service

[English](README.md) | 简体中文

自托管的 AI 代理网关 + 多租户后台。把手里的 Claude、Codex、Kiro、Kimi、Cursor、
xAI 订阅账号与 API 账号池化，通过一个 OpenAI / Anthropic 兼容端点供整个团队使用，
配额、计费、用量统计与日志都内建。

单个 Go 二进制，前端已嵌入——一个进程，一个端口。

本仓库只放发布产物，源码仓库是私有的。

## 功能

**网关**

- 一个端点喂给所有客户端：OpenAI `/v1/chat/completions` 与 Anthropic
  `/v1/messages` 双向跨协议转换。
- 流式 SSE、tools / `tool_choice`、图片内容、reasoning/thinking、流式 tool 事件
  都在转换中保留；无法保留的字段直接返回 400，不做静默丢弃。
- 供应商：Codex、Claude、xAI、Kiro、Kimi、Cursor，支持 OAuth（PKCE / 设备码）
  或手动填 access token。任何 OpenAI / Anthropic 兼容上游（DeepSeek、智谱、
  OpenCode Zen、本地 vLLM 等）都能在后台直接添加。
- 模型路由支持显式路由与前缀匹配，并可改写上游模型名。

**账号池**

- 轮询调度 + 会话亲和。
- 上游出错时有限次跨账号故障转移与原账号重试。
- 每账号并发上限，池化订阅账号时不至于触发上游 429。
- 每账号独立的出站代理。
- 仪表盘直接看账号健康度与池利用率。

**多租户与计费**

- 用户、API Key、订阅套餐：固定窗口 / 日 / 周配额 + 月预算。
- 开放注册或邀请码注册。
- 内置模型定价，启动时同步内置模型价格；自定义定价不受影响。
- 用量与成本统计，含 Analysis 与 Realtime 面板。

**运维**

- 版本化 SQL 迁移，启动时校验 schema 版本。
- 应用日志、请求错误日志、审计日志，各有保留期。
- 配额与 OAuth 刷新锁可选 Redis 后端。
- 后台一键自更新。
- 双语界面（zh-CN / en），深浅色主题。

## 安装

### 一键安装（Linux amd64/arm64，systemd）

下载二进制、校验 checksum、装成 systemd 服务。密钥不用你填——首次启动时服务端
自己生成。

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/install.sh | sudo bash
```

需要一个可连的 PostgreSQL。数据库不在本机默认位置的话，先设好 `DATABASE_URL`：

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/install.sh \
  | sudo DATABASE_URL='postgres://user:pass@host:5432/ai_proxy?sslmode=disable' bash
```

重复执行即为原地升级，`config.yaml` 与 `.env` 不会被覆盖；除非传
`ALLOW_DOWNGRADE=1`，否则拒绝降级。其他可选变量：`VERSION`、`PORT`（8080）、
`INSTALL_DIR`（`/opt/ai-proxy-service`）、`DATA_DIR`（`/var/lib/ai-proxy-service`）、
`SERVICE_USER`（`aiproxy`）。

```sh
systemctl status ai-proxy-service
journalctl -u ai-proxy-service -f
```

### Docker Compose

自带 PostgreSQL，只有数据库密码需要你定。

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/docker-compose.yml -o docker-compose.yml
printf 'POSTGRES_PASSWORD=%s\n' "$(openssl rand -hex 24)" > .env
docker compose up -d
```

`config.yaml` 与应用密钥在首次启动时落到 `/data` 卷里。其余设置——`PORT`、
用 `IMAGE_TAG` 钉住某个版本而不是跟着 `latest` 跑、以及可选的自注入密钥——从同一个
release 下载 `env.example` 当作 `.env` 用即可。升级：
`docker compose pull && docker compose up -d`。

### 手动安装

```sh
gh release download --repo veildawn/ai-proxy-releases -p '*linux_amd64.tar.gz'
tar -xzf ai-proxy-service_*_linux_amd64.tar.gz    # 二进制 + config.example.yaml + .env.example
cp config.example.yaml config.yaml                # 除了 DSN，其他都有能用的默认值
./ai-proxy-service migrate --config config.yaml
./ai-proxy-service serve   --config config.yaml
```

## 首次启动

访问 `http://localhost:8080/setup`。页面会要一个**一次性初始化令牌**：首次启动时
打在日志里，也在 `config.yaml` 旁边存了一份（0600）：

```sh
sudo cat /var/lib/ai-proxy-service/setup-token   # 一键安装
sudo cat ./data/setup-token                      # docker compose 把 /data 挂在 ./data
docker compose logs app | grep -A2 'one-time token'
```

超级管理员一建好，令牌立即失效并删除。没有它谁也建不了管理员——公网实例不会被路人
抢注。之后在 **Accounts** 里添加上游账号。

## 密钥

`JWT_SECRET` 与 `ENCRYPTION_KEY` 在首次启动时生成，写入 `config.yaml` 旁边的
`secrets.env`（0600），此后一律复用，不会自动轮换。**`ENCRYPTION_KEY` 要随数据库
一起备份：丢了，已存的上游凭证就永久解不开。** 环境变量优先级更高，随时可以注入
自己的值；多实例共用同一个数据库时**必须**注入相同的值，否则 session 与凭证互不通用。

## 端口

| 端口  | 用途                     |
|-------|--------------------------|
| 8080  | API + 后台面板           |
| 1455  | Codex OAuth 回调         |
| 54545 | Anthropic OAuth 回调     |

OAuth 回调端口只需要对「你用来关联账号的那个浏览器」可达。

## CLI

```sh
ai-proxy-service serve   --config config.yaml               # API + 后台面板
ai-proxy-service migrate --config config.yaml               # 执行 schema 迁移
ai-proxy-service oauth login --provider codex|anthropic|xai # 关联上游账号
ai-proxy-service version
```

## 校验下载

每个 release 都带 `checksums.txt`（一键安装脚本会自动校验）：

```sh
gh release download --repo veildawn/ai-proxy-releases -p 'checksums.txt' -p '*.tar.gz'
sha256sum -c checksums.txt --ignore-missing
```

## License

MIT
