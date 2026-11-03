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

因此当前检出 **还不能直接启动**。下面两条路径任选一条，不必两条都做。

组织 fork：`https://github.com/lgq-open-source/lobehub`。本地远程名 `lgq`。日常改动推到 `lgq`，不要推到 `origin`（上游 `lobehub/lobehub`）。

本机已经按文末「安装部署实况」跑过一遍。端口、项目名和当前停着哪一套，以那一节为准，不要照前面示例里的 `5432`、`6379`、`9000` 直接启动。

## 本地和 Docker 是什么关系

可以只用本机源码，也可以只用 Docker。两者都是在这台机器上得到一个能打开的 LobeHub，差别是 **网站进程放在哪**。它们不是「先本地、再 Docker」的流水线。

Docker 在两条路径里都会出现，担任的角色不一样：

| | 路径 A：源码开发 | 路径 B：Compose 部署 |
| --- | --- | --- |
| 你打开的地址 | `http://localhost:3010` | `http://localhost:3210` |
| LobeHub 进程 | 本机的 Node。命令是 `bun run dev`，改代码会热更新 | 容器 `lobehub`。默认镜像是 Docker Hub 上的 `lobehub/lobehub`，不读当前工作区 |
| PostgreSQL、Redis、RustFS、SearXNG | 也在 Docker 里，由 `bun run dev:docker` 单独拉起。这一步 **没有** 网站 | 和应用、网关在同一份 `docker-compose/deploy/docker-compose.yml` 里一起拉起 |
| Agent / Device Gateway | 这份开发编排不包含 `lobe-gateway` | 包含，`8787` / `8788` |
| 要不要装 Node、pnpm、Bun | 要。依赖装在本机 `node_modules` | 直接用官方镜像时不要。只有「用当前源码打镜像」才要 |
| 配置文件 | 仓库根目录 `.env`，外加 `docker-compose/dev/.env` | 只有 `docker-compose/deploy/.env` |
| 数据 | 开发库，卷和 `docker-compose/dev/data` | 另一套卷。两条路径的库 **不会** 自动共用 |

可以把它看成两套彼此独立的层，不要叠在一起跑：

```text
路径 A（改源码）
  浏览器 → 本机 Node :3010
              ↓
           Docker 只跑 PostgreSQL / Redis / RustFS / SearXNG
           文件：docker-compose/dev/docker-compose.yml

路径 B（整套容器）
  浏览器 → 容器 lobehub :3210
              ↓
           同一份 Compose 里还有 PostgreSQL / Redis / RustFS / SearXNG / Gateway
           文件：docker-compose/deploy/docker-compose.yml
```

路径 A 里的 Docker 只是给本机进程当数据库和对象存储。停掉 `bun run dev` 之后，容器还在，但浏览器里没有网站。反过来，只执行 `bun run dev:docker` 也打不开界面。

路径 B 里的 Docker 才是完整产品：网站、数据库、存储、搜索、网关都在容器里。不需要 `pnpm i`，也不需要 `bun run dev`。容器里跑的是已经发布的版本，工作区里刚改的文件不会进那个容器。

### 为什么不要同时开

两份 compose 都写了 `name: lobehub`，并共用这些容器名：`lobe-postgres`、`lobe-redis`、`lobe-rustfs`、`lobe-rustfs-init`、`lobe-searxng`。Docker 不允许同名容器各跑一份，宿主机上的 `5432`、`6379`、`9000`、`9001`、`8180` 也会被先启动的那一边占用。

从一条路径换到另一条时，先停掉正在跑的那一套，再启动另一套。开发库里的对话不会出现在部署库里。

### 另外两种容易和这两条搞混的跑法

**只把界面接到线上。** `bun run dev:spa` 之后终端里的 Debug Proxy，是本地 Vite 配 `app.lobehub.com` 的后端。它不用本机的 Next，也不用路径 B 的容器。只能看界面，不能验证本仓库的服务端。

**本机生产进程。** `bun run build` 然后 `bun run start` 仍然是本机 Node，端口 `3210`，没有热更新。数据库还得另有一份，通常继续用路径 A 已经起来的 PostgreSQL。这不是路径 B：没有 `lobehub` 容器，也没有 gateway 容器。

**用当前源码做镜像，再放进路径 B。** `bun run self-hosting:docker` 打出 `lobehub:local` 之后，把部署 compose 的镜像改成它再 `up`。这时跑的是容器，不再是 `bun run dev`。构建期间路径 A 可以停着；启动这份 compose 之前，路径 A 的同名容器必须先停。

### 怎么选

- 要改这个仓库的界面或服务端：路径 A。
- 只想在这台机器上用一套私有化 LobeHub，不编译源码：路径 B。Node 和 Bun 可以先不装。
- 要确认「我改过的代码打进镜像之后」能否起来：停掉路径 A，构建 `lobehub:local`，再走路径 B。

## 先补齐的工具

路径 B 只要第 4 项 Docker Desktop。Node、pnpm、Bun 是路径 A 需要的；用当前源码打镜像时也需要它们。

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

网站跑在本机，Docker 只提供数据库和存储。在 `D:\lgq\ai-work\lobehub` 执行。PowerShell 没有 `cp`，用 `Copy-Item`。若路径 B 的容器已在运行，先到 `docker-compose/deploy` 执行 `docker compose down`，避免同名容器和端口冲突。

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

网站、数据库、存储和网关都在容器里。默认使用已发布镜像 `lobehub/lobehub`，不编译当前工作区，也不使用路径 A 的 `.env` 和开发库。配置在 `docker-compose/deploy/`。若路径 A 已启动，先执行 `bun run dev:docker:down`。

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
| 停路径 A 依赖 | 见文末实况里的 `lobehub-dev` `down`。不要用 `bun run dev:docker:down`，那个项目名是 `lobehub`，对不上这次的卷 |
| 停路径 B | 见文末实况里的 `lobehub-deploy` `down`。不要加 `-v` |
| 清依赖后重装 | `bun run clean:node_modules`，然后 `pnpm i`（脚本内部调用 bash） |

从上游更新 fork 时，在本地 `canary` 上先 `git pull --rebase`（远程是 `origin`），再把需要保留的提交推到 `lgq`。不要对 `lobehub/lobehub` 做 force push。

## 安装部署实况

记录时间：2026-11-04。工作区 `D:\lgq\ai-work\lobehub`，分支 `docs/local-operations`，基线提交 `d34f58ed`。下面是这台机器上实际发生的事，不是上游文档的理想步骤。

### 开始时缺什么

- Node 已有 `v22.23.3`（x64）。`.nvmrc` 写的是 `lts/krypton`，Dockerfile 用 Node 24。`nvm install 24` 去下 `24.9.0-arm64`，镜像站没有这个包，所以没有换成 Node 24。后来的登录页是在 22.23.3 上打开的。
- pnpm 通过 Corepack 使用仓库锁定的 `12.4.1`。
- Bun 一开始不在 PATH。安装脚本装上了 `1.4.4`（`%USERPROFILE%\.bun\bin\bun.exe`）。这个安装程序把用户 PATH 写成了带 `REG_SZ` 前缀的坏值，随后已去掉该前缀，并把 Bun 目录留在用户 PATH 里。
- Docker 客户端 `29.8.2` 已在，引擎没启动。拉起 Docker Desktop 之后 `docker info` 才通。
- 没有 `node_modules`，没有根目录 `.env`，也没有 `docker-compose/dev/.env`。

### 安装依赖时踩到的点

第一次 `pnpm i` 在根目录 `prepare` 失败：脚本是 `git rev-parse ... || true`，cmd 没有 `true`，`/dev/null` 也不是 Windows 路径。包已经下完，安装被判失败。

处理：在临时目录放一个 `true.cmd`，内容是 `@exit /b 0`，把它加到本次命令的 PATH 最前面，再执行 `pnpm i`。第二次 `Done in 6.8s`，退出码 0。

环境文件只做了复制，没有提交：

```powershell
Copy-Item .env.example.development .env
Copy-Item docker-compose\dev\.env.example docker-compose\dev\.env
```

根目录 `.env` 里的 `DATABASE_URL` 改成了 `localhost:5433`。Redis 和 RustFS 后来改成 `localhost:6380`、`http://localhost:9010`。密钥仍是开发示例里的值，不能用于生产。

### 不能占用的已有端口

项目 `deploy` 里原来就有这些容器，全程没有停、也没有连它们的库：

| 容器 | 宿主机端口 | 状态 |
| --- | --- | --- |
| `certification_postgres` | `5432` | 一直在跑 |
| `certification_redis` | `6379` | 当时是停着的，端口仍算它的 |
| `minio-server` | `9000`，另外还有一个端口 | 停着，端口仍算它的 |

所以 LobeHub 没有使用 `5432`、`6379`、`9000`、`9001`。

### 路径 A：先跑通

Compose 项目名用 `-p lobehub-dev`，这样卷是 `lobehub-dev_redis_data` 和 `lobehub-dev_rustfs-data`，不会和别的项目混。PostgreSQL 数据在 `docker-compose/dev/data`（绑定挂载，不跟项目名走）。

`docker-compose/dev/host-ports.yml` 被 git 忽略，内容是：

```yaml
services:
  postgresql:
    ports: !override
      - '5433:5432'
  redis:
    ports: !override
      - '6380:6379'
  network-service:
    ports: !override
      - '9010:9000'
      - '9011:9001'
```

第一次拉起时还只覆盖了 `5433`，Redis 用了 `6379`，RustFS 用了 `9000`。路径 B 也这样绑过，和上面的已有容器冲突。最后一次启动路径 A 之前改成了上表，并且不再把 `3210` 映射出来。

实际启动：

```powershell
docker compose -p lobehub-dev -f docker-compose/dev/docker-compose.yml -f docker-compose/dev/host-ports.yml --env-file docker-compose/dev/.env up -d --wait postgresql redis rustfs searxng
pnpm db:migrate
bun run dev
```

`pnpm db:migrate` 输出 `database migration pass`，大约 33 秒。随后在这个库里建了标记表 `path_isolation_marker`，插入 `path-a-dev`。`bun run dev` 选了 Next `3010`、Vite `9876`。浏览器打开 `http://localhost:3010/signin`，登录页渲染完成，请求返回 200。

### 路径 B：换一套库跑通，然后停掉

停路径 A 时用 `down`，没有 `-v`。`docker-compose/dev/data/PG_VERSION` 还在，`lobehub-dev_*` 卷也还在。

路径 B 使用 `-p lobehub-deploy`。PostgreSQL 数据在 `docker-compose/deploy/data`，和开发库不是同一个目录。宿主机端口当时只把数据库改成了 `5434`，应用容器内部仍是 `postgresql:5432`。`docker-compose/deploy/.env` 按示例生成过密钥，文件被 git 忽略，这里不记录密钥内容。

`docker compose up -d --wait` 成功。`docker logs lobehub` 里有 `database migration pass` 和 Next Ready。`http://localhost:3210/signin` 登录页返回 200。在这个库里查 `path_isolation_marker`，表不存在。

随后发现路径 B 的 Redis 发布在 `6379`、RustFS 发布在 `9000`，和已有的 `certification_redis`、`minio-server` 冲突。按要求把路径 B 整套停掉，只留路径 A：

```powershell
docker compose -p lobehub-deploy -f docker-compose/deploy/docker-compose.yml -f docker-compose/deploy/host-ports.yml --env-file docker-compose/deploy/.env down
```

没有加 `-v`。`lobehub-deploy_redis_data`、`lobehub-deploy_rustfs-data` 和 `docker-compose/deploy/data` 都还在。

### 现在留着什么

只跑路径 A。`docker ps` 里 LobeHub 相关容器是：

| 容器 | 宿主机端口 |
| --- | --- |
| `lobe-postgres` | `5433` |
| `lobe-redis` | `6380` |
| `lobe-network`（RustFS 的端口从这里出去） | `9010`、`9011` |
| `lobe-searxng` | `8180` |
| `lobe-rustfs` | 不单独映射，走 `lobe-network` |

开发服务器听 `3010`。`certification_postgres` 仍在 `5432`。路径 B 没有容器在跑。开发库里的 `path-a-dev` 还在。

再开路径 A（路径 B 已经停着时）：

```powershell
docker compose -p lobehub-dev -f docker-compose/dev/docker-compose.yml -f docker-compose/dev/host-ports.yml --env-file docker-compose/dev/.env up -d --wait postgresql redis rustfs searxng
bun run dev
```

再开路径 B 之前，必须先停 `bun run dev` 和 `lobehub-dev`，并且给 Redis、RustFS 另选宿主机端口（例如 `6381`、`9020`、`9021`）。现在的 `docker-compose/deploy/host-ports.yml` 只把数据库改到了 `5434`，直接 `up` 会再次占用 `6379` 和 `9000`。两条路径的容器名相同，不能同时存在。

`bun run dev:docker` 不带 `-p`，项目名会落成 compose 文件里的 `lobehub`，数据库端口也是 `5432`。这台机器不要用这条快捷命令。
