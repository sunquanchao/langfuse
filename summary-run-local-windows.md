# langfuse 原生 Windows 本地开发部署要点总结

> 基于 2026-11-27 在 Windows 11 (26300) 上的完整实战记录。
> **核心结论**：langfuse 官方仅支持 macOS / Linux / devcontainer，原生 Windows 部署可行，但需要修补 **6 层**问题（工具链、权限、网络、行尾、上游依赖、运行时竞态）。全部解决后三服务可稳定运行。

---

## 0. 最终形态（验证通过）

| 服务 | 地址 | 健康检查 | 状态 |
|---|---|---|---|
| web (Next.js) | http://localhost:3000 | HTTP 200 | ✅ |
| worker (队列/摄取) | http://localhost:3030/api/health | `{"status":"ok"}` | ✅ |
| ai-gateway (Rust) | http://localhost:8080/health | HTTP 200 | ✅ |

基础设施（Docker）：`langfuse-postgres` / `langfuse-clickhouse` / `langfuse-redis` / `langfuse-minio` 全部 Healthy。

---

## 1. 前置条件与版本要求

| 组件 | 版本要求 | 说明 |
|---|---|---|
| Node.js | 主版本 **24**（engines.node=24，.nvmrc 锁 v24.22.1） | patch 差异可接受 |
| pnpm | **12.16.1 精确版本** | `devEngines.packageManager` 严格校验，版本不符直接拒绝 |
| Docker Desktop | 任意近期版本 | daemon 必须运行；安装路径可能在 `%LOCALAPPDATA%\Programs\DockerDesktop` |
| Rust | rustup + 仓库 pin **1.98.0**（rust-toolchain.toml 自动安装） | 需要 MSVC C++ 工具集（VS BuildTools 的 cl.exe/link.exe） |
| golang-migrate | 4.x | CH 迁移用；Windows release 资产是 `.zip` 不是 `.tar.gz` |
| ClickHouse 客户端 | 与服务端同版 | 官方无 Windows 原生版 → 用 Docker 包装器（见 §3.4） |
| LLVM/clang | 可选（sha2-asm 补丁后仅备用） | cc-rs 汇编 `.S` 文件需要 |
| Git | 支持 symlink | 开发者模式开启后配合 `core.symlinks=true` |

**国内网络镜像策略**（关键提速项）：

| 目标 | 直连 | 用镜像 |
|---|---|---|
| GitHub clone/release | ❌ 443 超时 | `gh-proxy.com` → 备选 `ghfast.top` / `mirror.ghproxy.com` |
| Docker Hub (docker.io) | ❌ auth.docker.io EOF | `docker.m.daocloud.io` 拉取后 retag |
| cgr.dev (minio) | ✅ 可直连 | — |
| crates.io | ⚠️ 极慢（几分钟零进度） | `rsproxy.cn` sparse 镜像（`~/.cargo/config.toml`） |
| npm | ⚠️ 公司源 tbcnpm.21tb.com 可用但慢（~60s/请求） | 无 @21tb 私有包时可考虑 npmmirror |

---

## 2. 系统级一次性准备（顺序敏感）

### 2.1 Windows 开发者模式（符号链接权限）

```powershell
# 需管理员（UAC），写入后对所有"新登录会话"生效
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock" /t REG_DWORD /f /v AllowDevelopmentMachineMode /d 1
```

⚠️ **关键陷阱**：已存在的登录会话（及其子进程树）拿不到新权限，symlink 创建仍会 EPERM，**注销重登后彻底生效**。临时绕过：用管理员 `mklink` 创建所需的个别链接。

### 2.2 git 配置（仓库本地）

```bash
git config core.longpaths true    # 仓库有超长路径，clone 时必须
git config core.symlinks true     # 开发者模式生效后启用
git config core.autocrlf false    # 必须！见 §2.3
```

### 2.3 行尾符（CRLF）——隐蔽杀手

全局 `autocrlf=true` 会把 `.sql` 迁移模板检出成 CRLF，而 langfuse 的 ClickHouse 迁移校验正则要求 `\n` 结尾，直接报错：

```
CLICKHOUSE_HISTORICAL_FINAL_NEWLINES must appear once at the end of a migration
```

修复：仓库本地 `autocrlf=false` 后，**删除文件再 checkout**（`git checkout-index` 有 stat 缓存坑，不删文件重检不生效）：

```bash
git config core.autocrlf false
find packages/shared/clickhouse/migrations -name "*.sql" -delete
git checkout -- packages/shared/clickhouse/migrations/
```

验证：`file packages/shared/clickhouse/migrations/canonical/*.sql | grep -c CRLF` 应为 0。

---

## 3. 工具链安装要点

### 3.1 pnpm（版本精确匹配）

```bash
# 必须在仓库外执行（仓库内 npm 会被 devEngines 拦截报 EBADDEVENGINES）
npm install -g pnpm@12.16.1
```

### 3.2 Rust

```bash
rustup-init.exe -y --default-toolchain stable-msvc   # static.rust-lang.org 国内可直连
# 仓库 pin 的 1.98.0 会在首次 cargo 时自动安装；可提前：
rustup toolchain install 1.98.0
```

crates.io 加速（写入 `~/.cargo/config.toml`，删除即还原）：

```toml
[source.crates-io]
replace-with = 'rsproxy-sparse'
[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"
```

### 3.3 golang-migrate + 路径转换 shim（最隐蔽的坑）

CH 迁移脚本（up.sh/drop.sh）以 `-source "file://${MIGRATIONS_DIRECTORY}"` 调用 migrate，路径含 `..` 段 → **MSYS 拒绝转换含 `..` 的路径** → migrate.exe 收到 `file:///e/...` 报"找不到路径"。

解法：`~/bin/migrate` bash 包装器（`command -v migrate` 会优先命中它而非 migrate.exe），把 `/e/foo` → `E:/foo`、`file:///e/foo` → `file://E:/foo`：

```bash
#!/bin/bash
real="$HOME/bin/migrate.exe"
declare -a out=()
for a in "$@"; do
  if [[ "$a" =~ ^file:///([a-z])(/.*)$ ]]; then
    out+=("file://$(tr 'a-z' 'A-Z' <<<"${BASH_REMATCH[1]}"):${BASH_REMATCH[2]}")
  elif [[ "$a" =~ ^/([a-z])(/.*)?$ ]]; then
    out+=("$(tr 'a-z' 'A-Z' <<<"${BASH_REMATCH[1]}"):${BASH_REMATCH[2]}")
  else
    out+=("$a")
  fi
done
exec "$real" "${out[@]}"
```

安装本体：GitHub release 资产名为 **`migrate.windows-amd64.zip`**（tar.gz 会 404），解压 `migrate.exe` 到 `~/bin`。

### 3.4 ClickHouse 客户端 shim

官方无 Windows 原生客户端。`~/bin/clickhouse`：

```bash
#!/bin/bash
# dev-tables.sh 通过 stdin 喂 SQL；容器内 localhost 就是 CH 服务本身
exec docker exec -i langfuse-clickhouse clickhouse "$@"
```

### 3.5 LLVM/clang（按需）

sha2-asm 未打补丁时必须（cc-rs 汇编 `.S` 文件用）：

```powershell
choco install llvm -y    # 管理员 UAC；自动加系统 PATH
```

---

## 4. 依赖安装（pnpm install）

- lockfile 为 pnpm-lock.yaml，`pnpm install` 即可
- **postinstall 链条**：根 postinstall → `scripts/postinstall.sh` → `agents:sync`（生成 `.claude/.cursor/.codex/.vscode` 的 shim 配置 + symlink）
- 失败模式 1：`EPERM symlink AGENTS.md` → 见 §2.1/2.2，用管理员 `mklink AGENTS.md .agents\AGENTS.md` 一次性创建；`.claude/skills/*` 的 37 个目录链接用 **NTFS junction**（`symlinkSync(target, dest, 'junction')`，免提权且能通过脚本的校验）
- 失败模式 2：`os error 1450 系统资源不足`（硬链接导入时）→ 瞬时资源问题，重试
- 一次性环境变量（如需绕过脚本校验）：依赖本体装完但 postinstall 失败时，手动跑 `bash scripts/postinstall.sh` 排查

---

## 5. 基础设施（Docker）

```bash
docker compose -f docker-compose.dev.yml up -d --wait   # --wait 等全部健康
```

- 4 服务：postgres:17 / clickhouse 26.8 LTS / redis 7.2 / minio（floci 仅 worker-tests profile）
- Docker Hub 被墙时逐镜像走代理源 + retag：

```bash
docker pull docker.m.daocloud.io/postgres:17 && docker tag docker.m.daocloud.io/postgres:17 postgres:17
# clickhouse/redis 同理；cgr.dev/chainguard/minio 可直连
```

- `.env` 从 `.env.dev.example` 复制（默认值即可跑通）；`.env.test` 从 `.env.test.example` 复制（跑测试用，非必需）

---

## 6. 数据库初始化（顺序敏感！）

```
pg reset → ch reset → seed examples
```

⚠️ **ch:reset 里的 ch:seed 依赖 Postgres schema 已存在**（seeder 会查 `public.projects`），必须先完成 pg 迁移。

```bash
# 1) Postgres：Prisma 有 AI 代理防护，检测到 Claude Code 会拒绝 reset，
#    必须携带用户明确同意的原文作为环境变量
PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION="同意，执行 reset" \
  pnpm exec dotenv -e ../../.env -- pnpm exec prisma migrate reset --force

# 2) ClickHouse（SKIP_CONFIRM 避免交互卡住；依赖 §3.3 的 migrate shim）
SKIP_CONFIRM=1 pnpm --filter=shared run ch:reset

# 3) Prisma client 生成（seeder 的 zod enum 依赖 Prisma client 里的枚举，
#    未 generate 会报 "Cannot convert undefined or null to object"）
pnpm run db:generate

# 4) 示例数据
pnpm --filter=shared run db:seed:examples
```

---

## 7. 构建产物（先建产物，再启动 dev）

**直接 `pnpm dev` 会竞态翻车**：turbo 同时拉起各包 dev 任务，web 在 shared 的 tsc 编译完成前就启动 → `Can't resolve '@langfuse/shared'` 崩溃退出。正确顺序：

```bash
# 1) JS 工作区包（web/worker 的运行时依赖）
pnpm exec turbo run build --filter=@langfuse/shared --filter=@langfuse/icons --filter=@langfuse/ee

# 2) native addon（Rust release，首次 2-15 分钟，之后增量秒级）
pnpm --filter=@langfuse/native run rust:build
```

### 7.1 sha2-asm 与 Windows（上游硬伤，必须补丁）

`sha2-asm 0.6.4` 源码硬编码 `compile_error!("crate does not support Windows targets")`，而仓库给 native 的 sha2 无条件开了 `asm` feature。已打本地补丁（`packages/native/Cargo.toml` 末尾，带 LOCAL WINDOWS DEV PATCH 注释）：

```toml
[target.'cfg(not(windows))'.dependencies]
sha2 = { version = "0.10", features = ["asm"] }

[target.'cfg(windows)'.dependencies]
sha2 = { version = "0.10" }
```

Windows 下回落到 sha2 纯 Rust 实现（开发环境性能差异可忽略）。**还原**：`git checkout -- packages/native/Cargo.toml`。

### 7.2 编译成功标志

```
Finished `release` profile [optimized] target(s) in ...
packages/native/langfuse-native.win32-x64-msvc.node  (≈6.8MB)
```

---

## 8. 启动与验证

```bash
pnpm dev   # = turbo watch dev，常驻进程
```

启动期三个已知故障模式：

| 症状 | 原因 | 处置 |
|---|---|---|
| `EPERM: rename .napi-rs-filesystem-transaction.candidate.*.swp -> .swp` | Defender/索引器锁新建文件的瞬时竞态 | 删 `packages/native/.napi-rs-filesystem-transaction*`，重启 dev |
| gateway 卡 `no slot in concurrency group "langfuse-rust"` 且持有 pid 已死 | turbo 槽位死锁（进程死亡未释放） | 删 `%LOCALAPPDATA%\pnpm-state\run-slots\langfuse-rust` 整个目录 |
| web 起来了但 curl localhost 超时 | curl 的 localhost 解析到 `::1` 挂起 | 测试用 `curl --noproxy "*" http://127.0.0.1:3000`；浏览器无此问题 |

监控建议（本次实战的教训）：用**端口实据**（`netstat -ano | grep ":3000.*LISTENING"`）判定就绪，不要用日志文本正则（会误报）；看门狗同时盯"日志冻结 2 分钟 + 无编译进程"的静默死亡模式。

验证命令：

```bash
curl --noproxy "*" -o /dev/null -w "%{http_code}" http://127.0.0.1:3000        # 200
curl --noproxy "*" http://127.0.0.1:3030/api/health                            # {"status":"ok"}
curl --noproxy "*" -o /dev/null -w "%{http_code}" http://127.0.0.1:8080/health # 200
```

---

## 9. 本地改动清单（与上游差异）

| 位置 | 改动 | 性质 | 还原方式 |
|---|---|---|---|
| `packages/native/Cargo.toml` | sha2 asm feature 按平台 gate | **git 脏文件，提交前必须处理** | `git checkout -- Cargo.toml` |
| `.env` / `.env.test` | 从 example 复制 | 未跟踪，正常 | — |
| `.claude/skills/*` | 37 个 junction（Windows 替代 symlink） | gitignore 范畴 | 删除即可 |
| 仓库 git config | `core.symlinks=true` / `core.autocrlf=false` / `core.longpaths=true` | 本地配置 | 可保留 |
| `~/.cargo/config.toml` | rsproxy 镜像 | 机器级 | 删文件 |
| `~/bin/{migrate,clickhouse}` | Windows shim | 机器级 | 删文件 |
| 注册表 | 开发者模式 | 机器级 | 设置界面关闭 |
| `packages/native/langfuse-native.win32-x64-msvc.node` | 构建产物 | gitignore 范畴 | — |

其余：`.cursor/environment.json`、`.vscode/settings.json` 会被 agents:sync 重写产生 diff，属正常现象。

---

## 10. 日常使用

```bash
# 日常启动（全部缓存命中：native 0.5s / web 热启动 / gateway 秒级）
cd E:\ws_python\langfuse && pnpm dev

# 仅前端
pnpm run dev:web
# 仅 worker
pnpm run dev:worker

# 停止：Ctrl+C；基础设施
docker compose -f docker-compose.dev.yml down      # 保数据卷
docker compose -f docker-compose.dev.yml down -v   # 清数据卷（= 重置环境）
```

**注销重登后的收益**：symlink 权限对所有进程生效，`pnpm i` 的 postinstall 不再需要任何手工干预；新 clone 的仓库也能正确检出符号链接。

---

## 11. 故障排查速查表

| 报错特征 | 根因 | 一句话解法 |
|---|---|---|
| `Filename too long`（clone/checkout） | Windows MAX_PATH | `git config core.longpaths true` + 重检 |
| `EPERM: symlink`（pnpm i / git checkout） | 无 symlink 权限 | 开发者模式 + 重登；或管理员 mklink 单个补齐 |
| `CLICKHOUSE_HISTORICAL_FINAL_NEWLINES must appear...` | .sql 被 CRLF 污染 | autocrlf=false + 删文件重 checkout |
| `failed to open source, "file:///e/..."` | MSYS 不转换含 `..` 的路径 | ~/bin/migrate shim |
| `clickhouse binary could not be found` | 无 Windows 原生客户端 | ~/bin/clickhouse docker 包装器 |
| `crate does not support Windows targets` | sha2-asm 上游硬禁 | Cargo.toml 平台 gate 补丁 |
| `LNK1181 cannot open .o` + `cl: 无法识别 src/*.S` | cc-rs 找不到汇编器 | 装 LLVM/clang（或打 §7.1 补丁后不再需要） |
| Prisma 拒绝 reset 并输出 AI 警告 | AI 代理防护 | `PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION` |
| `Cannot convert undefined or null to object`（zod enum） | Prisma client 未生成 | `pnpm run db:generate` |
| `The table public.projects does not exist`（ch:seed） | 初始化顺序错误 | 先 pg 迁移再 ch:reset |
| napi `EPERM rename *.swp` | AV 锁新文件 | 删 `.napi-rs-filesystem-transaction*` 重试 |
| gateway 永久等待 slot 且持有 pid 不存在 | turbo 槽位死锁 | 删 `pnpm-state/run-slots/langfuse-rust` |
| `Command "dev" not found`（pnpm） | cwd 不在仓库根 | `cd` 到仓库再跑 |

---

## 12. Docker 容器日常操作

### 12.1 容器清单

| 容器名 | 镜像 | 端口（宿主机） | 作用 |
|---|---|---|---|
| `langfuse-postgres` | postgres:17 | 5432 | 主数据库（账号/项目/元数据） |
| `langfuse-clickhouse` | clickhouse 26.8 | 8123 / 9000 | 追踪/观测数据仓库 |
| `langfuse-redis` | redis 7.2 | 6379 | 队列/缓存 |
| `langfuse-minio` | chainguard/minio | 9090 / 9091 | S3 对象存储（媒体/事件） |

### 12.2 日常命令（在仓库根目录执行）

```bash
# 看当前状态（是否在跑、是否健康）
docker ps

# 全部启动（含健康检查等待，最常用）
docker compose -f docker-compose.dev.yml up -d --wait

# 全部停止（容器删除，数据卷保留，下次 up 数据还在）
docker compose -f docker-compose.dev.yml down

# 只停不删（保留容器，下次 start 秒起）
docker compose -f docker-compose.dev.yml stop
docker compose -f docker-compose.dev.yml start

# 单个容器操作（例：postgres）
docker restart langfuse-postgres
docker logs langfuse-postgres --tail 50                # 看日志
docker exec -it langfuse-postgres psql -U postgres     # 进数据库

# ⚠️ 危险：删容器 + 删数据卷（数据库清空，需重新迁移+seed）
docker compose -f docker-compose.dev.yml down -v
```

### 12.3 数据安全等级

| 命令 | 容器 | 数据 | 用途 |
|---|---|---|---|
| `stop` / `start` | 保留 | ✅ 保留 | 日常关机/重启 |
| `down` / `up -d` | 重建 | ✅ 保留（卷在） | 编排变更后 |
| `down -v` | 重建 | ❌ **清空** | 想完全重置环境时 |

### 12.4 `--wait` 的含义

compose 会等每个容器的健康检查通过才返回。命令正常结束 = 4 个容器全部就绪，此时再跑 `pnpm dev` 才不会报连接错误。
