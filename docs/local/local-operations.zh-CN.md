# 本地安装、开发与部署

本文记录 **这台 Windows 工作区** 上 LobeHub 的实际用法。命令以仓库脚本和 2026-11-04 的本机核对为准，不代替上游官网教程。

上游说明仍在这些位置：

- 源码环境：[docs/development/basic/setup-development.zh-CN.mdx](../development/basic/setup-development.zh-CN.mdx)
- 开发约定：[AGENTS.md](../../AGENTS.md)
- Docker Compose 部署：[docs/self-hosting/platform/docker-compose.zh-CN.mdx](../self-hosting/platform/docker-compose.zh-CN.mdx)
- E2E：[e2e/docs/local-setup.md](../../e2e/docs/local-setup.md)

## 本机核对（2026-11-04）

| 项 | 结果 |
| --- | --- |
| 工作区 | `D:\lgq\ai-work\lobehub` |
| 系统 | Windows 10，PowerShell |
| 分支 | `canary`，与 `origin/canary` 对齐 |
| 上游 | `https://github.com/lobehub/lobehub.git` |
| 提交 | `d34f58ed`（`fix(cli): let the server pick the default model for lh gen image`） |
| Node.js | 已安装 `v22.23.3` |
| 仓库要求的 Node | `.nvmrc` 为 `lts/krypton`；`Dockerfile` 构建参数 `NODEJS_VERSION=24` |
| pnpm | `package.json` 锁定 `pnpm@12.4.1`。本机通过 Corepack 可执行，`pnpm -v` 为 `12.4.1` |
| Bun | 不在 `PATH`，`%USERPROFILE%\.bun\bin\bun.exe` 也不存在 |
| Docker | 客户端 `29.8.2` 已安装；引擎未启动（命名管道 `dockerDesktopLinuxEngine` 不存在） |
| 依赖 | 工作区没有 `node_modules`，尚未执行 `pnpm i` |
| 环境文件 | 根目录没有 `.env`。存在 `.env.example`、`.env.example.development`、`.env.desktop`。`docker-compose/dev/` 只有 `.env.example` |

因此当前检出 **还不能直接启动**。先补齐下面的缺口，再走「源码开发」或「Compose 部署」。

组织 fork：`https://github.com/lgq-open-source/lobehub`。本地远程名 `lgq`。日常改动推到 `lgq`，不要推到 `origin`（上游 `lobehub/lobehub`）。

## 先补齐的工具

1. **Node.js 24**。仓库 `.nvmrc` 与 Docker 构建都指向 Node 24 这一代。本机目前是 22.23.3，安装依赖前先切到 24，再在仓库根目录确认 `node -v`。
2. **pnpm 12.4.1**。已可用时不必重装。若 `pnpm` 不在 PATH：

   ```powershell
   corepack enable
   corepack prepare pnpm@12.4.1 --activate
   ```

3. **Bun**。`package.json` 的脚本用 Bun 执行；`scripts/devStartupSequence.mts` 里启动 Next 与 Vite 的命令也写死为 `bun`。只装 Node、不装 Bun 时，`bun run dev` 无法工作。安装后重新打开终端，确认 `bun -v`。
4. **Docker Desktop**。客户端已在，需要先启动 Desktop，直到下面的命令能列出引擎而不是管道错误：

   ```powershell
   docker info
   ```

5. **Git**。已克隆，无需再装。

密钥只放在被 git 忽略的 `.env` 里。`.gitignore` 已忽略各目录下的 `.env`。不要把 API Key、`KEY_VAULTS_SECRET`、`AUTH_SECRET` 写进提交。

## 路径 A：源码开发

在 `D:\lgq\ai-work\lobehub` 执行。PowerShell 没有 `cp`，用 `Copy-Item`。

### 1. 安装依赖

```powershell
pnpm i
```

这一步会安装根目录和 workspace（`packages/*`、`e2e`、`apps/share`、`apps/workbench` 等）。本机尚未跑过。

### 2. 准备环境文件

```powershell
Copy-Item docker-compose\dev\.env.example docker-compose\dev\.env
Copy-Item .env.example.development .env
```

两个文件的分工：

- `docker-compose/dev/.env`：开发用 PostgreSQL、Redis、RustFS。示例里数据库名 `lobechat`，宿主机端口 PostgreSQL `5432`、RustFS `9000`。
- `.env`：Next 开发服务器。示例指向 `APP_URL=http://localhost:3010`、`DATABASE_URL` 的本机 `5432/lobechat`、`REDIS_URL=redis://localhost:6379`、RustFS `http://localhost:9000`、SearXNG `http://localhost:8180`。

聊天需要至少一组模型密钥。把对应行写进 `.env`（文件里已有注释位），例如 `OPENAI_API_KEY` 或 `ANTHROPIC_API_KEY`。示例密钥只用于本地开发，不能用于生产。

### 3. 启动依赖容器

```powershell
bun run dev:docker
```

实际执行：

```text
docker compose -f docker-compose/dev/docker-compose.yml up -d --wait postgresql redis rustfs searxng
```

| 服务 | 容器名 | 本机地址 |
| --- | --- | --- |
| PostgreSQL（ParadeDB PG17） | `lobe-postgres` | `localhost:5432`，库名 `lobechat` |
| Redis 7 | `lobe-redis` | `localhost:6379` |
| RustFS | `lobe-rustfs` | API `http://localhost:9000`，控制台 `http://localhost:9001` |
| SearXNG | `lobe-searxng` | `http://localhost:8180` |

这条命令 **不启动 LobeHub 本身**。查看状态：

```powershell
docker compose -f docker-compose/dev/docker-compose.yml ps
```

停掉容器（保留数据卷）：

```powershell
bun run dev:docker:down
```

清空数据并重新迁移：

```powershell
bun run dev:docker:reset
```

该脚本会 `down -v`、删除 `docker-compose/dev/data`、再次 `dev:docker`，然后执行 `pnpm db:migrate`。

### 4. 迁移数据库

Docker 健康检查通过后再执行：

```powershell
pnpm db:migrate
```

成功时输出 `database migration pass`。需要看表结构时：

```powershell
pnpm db:studio
```

### 5. 启动开发服务器

完整前后端（Next + Vite SPA）：

```powershell
bun run dev
```

`scripts/devStartupSequence.mts` 的端口规则：

- Next 默认从 **3010** 起找空闲端口。设置了 `PORT`，或命令行带了 `-p`，就固定用该端口，冲突时直接失败。
- Web SPA 的 Vite 默认从 **9876** 起找空闲端口，可用 `SPA_PORT` 固定。
- 编排器把选中的 Vite 端口写进 `VITE_DEV_PORT`，Next 靠这个变量找 SPA，不读端口文件。

浏览器打开 Next 打印的地址，默认为 `http://localhost:3010`。

只跑前端、把 `/api`、`/trpc`、`/webapi`、`/oidc` 代理到 `localhost:3010`：

```powershell
bun run dev:spa
```

终端会打印 Debug Proxy。形如：

```text
Debug Proxy: https://app.lobehub.com/_dangerous_local_dev_proxy?debug-host=http%3A%2F%2Flocalhost%3A9876
```

该地址把本地 Vite 接到 **生产后端** `app.lobehub.com`，只适合联调界面，不能当作本分支的验收环境。验收必须跑正在交付的这份代码及其后端。

其他入口：

| 目的 | 命令 | 默认端口 |
| --- | --- | --- |
| 仅 Next | `bun run dev:next` | 3010 |
| 移动端 SPA | `bun run dev:spa:mobile` | 3012（`MOBILE_SPA_PORT`） |
| 认证 SPA | `bun run dev:spa:auth` | 3013（`AUTH_SPA_PORT`） |
| 桌面端 Electron | `bun run dev:desktop` | 见 `apps/desktop` |
| 独立 Hono 服务 | `pnpm --filter @lobechat/server dev` | 见该包启动日志 |

Windows 上查端口占用：

```powershell
Get-NetTCPConnection -LocalPort 3010,5432,6379,9000,9001,8180,9876 -ErrorAction SilentlyContinue |
  Select-Object LocalPort, State, OwningProcess
```

上游文档里的 `lsof` 在本机 PowerShell 中不可用。

### 6. 改完代码后的检查

只检查本次改动的文件：

```powershell
bun run check
```

指定文件时：

```powershell
bun run check src/path/to/file.ts
```

不要用 `bun run test`，那会跑完整测试。类型检查是全仓库的 `bun run type-check`（`tsrs`）。纯文档改动不需要跑这套检查。

## 路径 B：用 Docker Compose 跑服务

这条路径使用已发布镜像 `lobehub/lobehub`，不编译当前工作区。配置在 `docker-compose/deploy/`。

官方一键脚本是 Bash：

```sh
bash <(curl -fsSL https://lobe.li/setup.sh) -l zh_CN
```

本机默认壳是 PowerShell，该脚本需要 **WSL 2 或 Git Bash**。已经克隆了本仓库时，可以不跑一键脚本，直接用仓库里的 compose 文件。

### 1. 准备部署目录

在 Git Bash 或 WSL 中：

```sh
cp docker-compose/deploy/.env.example docker-compose/deploy/.env
```

`searxng-settings.yml` 已在仓库中，compose 会直接挂载它。`.env` 必须改掉所有 `YOUR_*` 占位，至少包括：

- `KEY_VAULTS_SECRET`
- `AUTH_SECRET`
- `GATEWAY_SERVICE_TOKEN`
- `JWKS_KEY` 与 `JWKS_PUBLIC_KEY`（公钥给 gateway，私钥留在应用容器）
- `POSTGRES_PASSWORD`
- `RUSTFS_SECRET_KEY`

生成方式与字段说明以 [Docker Compose 部署文档](../self-hosting/platform/docker-compose.zh-CN.mdx) 的「自定义部署」为准。本地体验保持示例中的：

- `APP_URL=http://localhost:3210`
- `LOBE_PORT=3210`
- `AGENT_GATEWAY_URL=http://localhost:8787`

端口模式或域名模式要改 `APP_URL` 和网关的浏览器可见地址，否则上传、图片生成和 Gateway 会指向错误主机。

### 2. 启动

先确认 Docker Desktop 已运行，且 `3210`、`9000`、`9001`、`8787`、`8788`、`5432`、`6379` 没有被路径 A 的开发容器占用。开发栈和部署栈都映射了 `5432` 与 `6379`，不能同时占用这些宿主机端口。

```powershell
docker compose -f docker-compose/deploy/docker-compose.yml --env-file docker-compose/deploy/.env up -d
```

在 `docker-compose/deploy` 目录里也可以：

```powershell
Set-Location docker-compose\deploy
docker compose up -d
```

compose 会拉起应用、PostgreSQL、Redis、RustFS、SearXNG，以及镜像 `ghcr.io/lobehub/lobehub-gateway:0.5.0`。

| 入口 | 地址 |
| --- | --- |
| LobeHub | `http://localhost:3210` |
| RustFS API / 控制台 | `http://localhost:9000` / `http://localhost:9001` |
| Agent Gateway | `http://localhost:8787` |
| Device Gateway | `http://localhost:8788` |

应用容器启动时会自己跑数据库迁移。确认就绪：

```powershell
docker logs -f lobehub
```

日志中出现 migration 通过，以及 Next 在 `3210` 上 Ready，即可打开 `http://localhost:3210`。

更新已有部署时先看上游文档的「升级已有部署」。新的 compose 需要 Gateway 相关变量，缺了网关容器会重启，智能体会退回浏览器里执行。

可选 Elasticsearch 默认关闭，由 `COMPOSE_PROFILES` 打开。不要在没做回填的情况下把 `FTS_SEARCH_PROVIDER` 改成 `elasticsearch`。

### 3. 用当前源码打本地镜像

需要验证本仓库代码、而不是 Docker Hub 上的 `lobehub/lobehub` 时：

```powershell
bun run self-hosting:docker
```

这会 `docker build -t lobehub:local .`，构建阶段使用 Node 24，内存参数 `--max-old-space-size=8192`。国内镜像构建用 `bun run self-hosting:docker-cn`。构建完成后，把 deploy compose 里的 `image: lobehub/lobehub` 改成 `lobehub:local` 再启动。这个改动只留在本地，不要提交。

不走 Docker、在本机直接跑生产进程：

```powershell
bun run build
bun run start
```

`start` 是 `next start -p 3210`。构建前同样需要可用的 `DATABASE_URL` 和密钥。完整 `build` 会编 SPA、auth、workbench、share 和 Next，耗时长、占用内存大。

## 日常维护

| 动作 | 命令 |
| --- | --- |
| 拉上游（在跟踪 `origin` 的 `canary` 上） | `git pull --rebase` |
| 把文档或改动推到组织 fork | `git push lgq HEAD` |
| 停开发依赖 | `bun run dev:docker:down` |
| 停部署栈 | 在 `docker-compose/deploy` 执行 `docker compose down` |
| 清依赖后重装 | `bun run clean:node_modules`，然后 `pnpm i`（脚本内部调用 bash） |

从上游更新 fork 时，在本地 `canary` 上先 `git pull --rebase`（远程是 `origin`），再把需要保留的提交推到 `lgq`。不要对 `lobehub/lobehub` 做 force push。

## 和本机现状的对应关系

按 2026-11-04 的核对，要在这台机器上第一次把界面跑起来，顺序是：

1. 安装 Node 24 与 Bun，启动 Docker Desktop。
2. `pnpm i`。
3. 复制两份开发用 `.env`，写入模型密钥。
4. `bun run dev:docker`，然后 `pnpm db:migrate`。
5. `bun run dev`，打开 `http://localhost:3010`。

要的是一套不编译源码的私有化服务时，走路径 B，并避开与开发栈冲突的 `5432` / `6379`。
