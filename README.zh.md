# AI Proxy Service

[English](README.md) | 简体中文

手里有 Claude、Codex、Kiro、Kimi、Cursor、xAI 的订阅或 API 号，想给整个团队用？
丢进这个池子，大家共用一个入口；谁用了多少、花了多少、哪个号挂了，后台一眼能看清。

一个程序、一个端口。

## 能干什么

**接客户端**

- 平时用的那些 AI 客户端，改个地址就能接进来。
- 边生成边吐字、工具调用、带图、思考过程都能正常过；实在过不去的会直接报错，不会偷偷丢掉。
- 支持 Codex、Claude、xAI、Kiro、Kimi、Cursor——浏览器登录授权，或自己贴 token。别的兼容上游（DeepSeek、智谱、本地模型等）也能在后台加。
- 模型可以指定走哪路，也能改成上游真正要用的名字。

**账号池**

- 多个号自动轮着用；同一段对话尽量黏在同一个号上。
- 某个号挂了会换别的号接着试，不会一炸全停。
- 每个号能限制同时跑几路，免得把上游额度打爆。
- 每个号可以单独走代理。
- 后台直接看哪些号还活着、池子榨得怎么样。

**给团队用**

- 用户、API Key、套餐：按天 / 周 / 固定时段限量，也能设月预算。
- 可以开放注册，也可以只靠邀请码。
- 常见模型价钱内置，启动时会同步；你自己定的价不会被盖掉。
- 用量和花费有统计面板。

**日常维护**

- 升级会自动改库表；版本对不上会拒启动，避免踩坑。
- 应用日志、出错记录、操作审计，各自能设留多久。
- 需要的话可以挂 Redis，用来管配额和登录刷新。
- 后台可以一键自己升级。
- 中英文界面，深浅色都行。

## 安装

### 一键安装（Linux amd64 / arm64）

下载、校验、装成系统服务。密钥不用你填——第一次启动时自己生成。

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/install.sh | sudo bash
```

需要一台能连上的 PostgreSQL。不在本机默认位置的话，先带上数据库地址：

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/install.sh \
  | sudo DATABASE_URL='postgres://user:pass@host:5432/ai_proxy?sslmode=disable' bash
```

再跑一遍就是原地升级，`config.yaml` 和 `.env` 不会被覆盖。默认不让降级，真要降就加 `ALLOW_DOWNGRADE=1`。还能改：`VERSION`、`PORT`（默认 8080）、`INSTALL_DIR`（`/opt/ai-proxy-service`）、`DATA_DIR`（`/var/lib/ai-proxy-service`）、`SERVICE_USER`（`aiproxy`）。

```sh
systemctl status ai-proxy-service
journalctl -u ai-proxy-service -f
```

### Docker Compose

自带 PostgreSQL，你只需要定一个数据库密码。

```sh
curl -fsSL https://github.com/veildawn/ai-proxy-releases/releases/latest/download/docker-compose.yml -o docker-compose.yml
printf 'POSTGRES_PASSWORD=%s\n' "$(openssl rand -hex 24)" > .env
docker compose up -d
```

配置和密钥第一次启动时会写到 `/data` 卷里。想改端口、钉死某个版本（`IMAGE_TAG`）、或自己注入密钥，从同一个 release 下载 `env.example` 当 `.env` 用。升级：

```sh
docker compose pull && docker compose up -d
```

### 手动安装

```sh
gh release download --repo veildawn/ai-proxy-releases -p '*linux_amd64.tar.gz'
tar -xzf ai-proxy-service_*_linux_amd64.tar.gz    # 二进制 + 示例配置
cp config.example.yaml config.yaml                # 除了数据库地址，其他都有能用的默认值
./ai-proxy-service migrate --config config.yaml
./ai-proxy-service serve   --config config.yaml
```

## 第一次打开

访问 `http://localhost:8080/setup`。页面会要一个**一次性初始化令牌**——第一次启动时打在日志里，也在配置文件旁边存了一份：

```sh
sudo cat /var/lib/ai-proxy-service/setup-token   # 一键安装
sudo cat ./data/setup-token                      # docker compose 把 /data 挂在 ./data
docker compose logs app | grep -A2 'one-time token'
```

管理员建好，令牌立刻作废。没有它谁也建不了管理员，公网实例不会被路人抢注。之后去 **Accounts** 把上游账号加进去。

## 密钥

`JWT_SECRET` 和 `ENCRYPTION_KEY` 第一次启动时生成，写在配置文件旁边的 `secrets.env`（只有自己能读），之后一直复用，不会自动换。**`ENCRYPTION_KEY` 要跟数据库一起备份：丢了，已经存进去的上游凭证就永远解不开。** 环境变量优先级更高，想自己填随时可以；多台机器共用同一个库时，**必须**用同一套值，否则登录态和凭证对不上。

## 端口

| 端口 | 干什么 |
|------|--------|
| 8080 | 服务和后台 |
| 1455 | Codex 登录回调 |
| 54545 | Anthropic 登录回调 |

后面两个端口，只需要对「你用来关联账号的那个浏览器」能访问就行。

## 命令行

```sh
ai-proxy-service serve   --config config.yaml               # 启动服务和后台
ai-proxy-service migrate --config config.yaml               # 升级数据库结构
ai-proxy-service oauth login --provider codex|anthropic|xai # 关联上游账号
ai-proxy-service version
```

## 校验下载

每个版本都带 `checksums.txt`（一键安装会自动帮你核）：

```sh
gh release download --repo veildawn/ai-proxy-releases -p 'checksums.txt' -p '*.tar.gz'
sha256sum -c checksums.txt --ignore-missing
```
