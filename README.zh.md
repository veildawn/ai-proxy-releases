# AI Proxy Service

[English](README.md) | 简体中文

自托管的 AI 代理网关 + 多租户后台。把手里的 Claude、Codex、Kiro、Kimi、Cursor、
xAI 订阅账号与 API 账号池化，通过一个 OpenAI / Anthropic 兼容端点供整个团队使用，
配额、计费、用量统计与日志都内建。

单个 Go 二进制，前端已嵌入——一个进程，一个端口。

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

需要 Linux（amd64 或 arm64）与 PostgreSQL。

```sh
# 下载并解包最新二进制
gh release download --repo veildawn/ai-proxy-releases -p '*linux_amd64.tar.gz'
tar -xzf ai-proxy-service_*_linux_amd64.tar.gz
```

写一个 `config.yaml`。除 DSN 外都有可用默认值：

```yaml
server:
  port: 8080
  public_url: http://localhost:8080
database:
  dsn: postgres://postgres:postgres@127.0.0.1:5432/ai_proxy?sslmode=disable
```

密钥通过环境变量注入（或放在二进制同目录的 `.env`）：

```sh
export JWT_SECRET="$(openssl rand -base64 48)"        # 至少 32 字符
export ENCRYPTION_KEY="$(openssl rand -base64 32)"    # base64，解码后正好 32 字节
```

先跑迁移，再启动：

```sh
./ai-proxy-service migrate --config config.yaml
./ai-proxy-service serve   --config config.yaml
```

打开 `http://localhost:8080/setup` 创建管理员账号，然后在 **Accounts** 里添加上游账号。

## CLI

```sh
ai-proxy-service serve   --config config.yaml   # API + 后台
ai-proxy-service migrate --config config.yaml   # 执行 schema 迁移
ai-proxy-service oauth login --provider codex   # 关联上游账号
```

## 校验下载

每个 release 都带 `checksums.txt`：

```sh
gh release download --repo veildawn/ai-proxy-releases -p 'checksums.txt'
sha256sum -c checksums.txt --ignore-missing
```

## 许可

MIT
