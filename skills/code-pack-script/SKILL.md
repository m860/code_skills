---
name: code-pack-script
description: >-
  Constraints for writing pack/release scripts whose artifacts are uploaded to
  the code_deploy tool (HTTP upload into app root/incoming, then remote exec).
  Covers shared tarball rules plus Node and Web app layouts. Use when the user
  asks for code-pack-script, pack:release, 打包脚本, release.tgz, or packaging
  for code_deploy upload + exec.
---

# 打包脚本（面向 code_deploy）

约定业务仓如何写 **pack / release 脚本**，使产物可被 **code_deploy** 消费：

```
本地 pack → .tar / .tar.gz / .tgz
  → HTTP POST /v1/apps/:appId/upload → 写入应用 root/incoming/，返回 relativePath
  → 多次 HTTP 或 MCP exec(argv) 完成解压、装依赖、切版本、进程管理等
```

本技能**只**管打包脚本与产物约束。发版改 STATUS / CHANGELOG、跑部署入口 → 见 `../code-deploy/SKILL.md`。code_deploy 服务实现 → 改 **code_deploy 仓库**，不在本技能改业务服务代码。

不规定具体构建命令或仓库目录名；跟随目标仓库已有约定。没有约定先问。

用户未要求写/改打包脚本时，不擅自新增或大改 pack 脚本。

## 与 code_deploy 的分工

| 侧 | 职责 |
| --- | --- |
| 业务仓 pack 脚本 | 构建并打出扁平压缩包；版本只写入产物清单 |
| code_deploy `upload` | 把包落到应用 `root/incoming/<uploadId>--<file>` |
| code_deploy `exec` | 在应用 root 下跑白名单 argv（无业务命令封装、无运行时模板） |
| 调用方（人或 AI） | 用多次 `exec` 组合发布 / 回滚 / 排障步骤 |

MCP **无** upload Tool：上传必须走 HTTP。

## 部署机目录（调用方 exec 时常用约定）

脚本注释 / README 可提示运维侧布局（**非** code_deploy 强制模板）：

```
<root>/                    ← apps 登记的 root；exec 的 cwd
  .env                     ← 跨版本共用（运维维护；禁止打进包）
  incoming/                ← upload 落盘（TTL 可清理）
  current → releases/<id>  ← 可选：当前版本软链
  releases/<id>/           ← 解压后的本次包（+ 该版本依赖目录）
```

| 项 | 约定 |
| --- | --- |
| 环境配置 | 真实 `.env` 在 `<root>/`，跨版本共用，是运行时配置的**唯一来源**。包内最多带 `.env.example`。 |
| 依赖目录 | 如 `node_modules` 在 `releases/<id>/`，每版本独立；**默认不打进包**，由发布流程里的 `exec` 安装。 |
| 进程 cwd | 建议与 `.env` 同目录，即 `<root>/`，以便进程读到该 `.env`。 |

## 共性约束（所有应用）

| 项 | 要求 |
| --- | --- |
| 版本写入 | **仅**写入发布产物内的清单（如产物 `package.json` 的 `version`）。**禁止**改仓库源码里的版本字段。 |
| 版本来源 | 优先：`VERSION` → CLI `--version`/`-v` → `docs/CHANGELOG.md` 最上 `## X.Y.Z`。皆无则失败说明，不臆造。 |
| 产物目录 | 组装可部署的扁平目录（下文称 `release/`）：运行/托管所需文件、清单、锁（若需要）、可选 `.env.example`、可选进程配置。 |
| **不要**打进包 | 依赖安装缓存（如 `node_modules`）、真实 `.env` / 密钥、与运行无关的源码或测试产物（除非仓库明确要求）。 |
| 压缩包形态 | `.tar` / `.tar.gz` / `.tgz`。归档成员为**扁平根**（`tar -czf out.tgz -C release .`），**不要**再包一层 `release/` 目录名。 |
| 文件名 | 建议 `{SemVer}-{YYYYMMDDHHmmss}.tgz`（或等价），便于从文件名识别版本。 |
| 可重复 | 脚本可直接执行；注明最小依赖与入参。 |
| 本地清理 | 仓库忽略 `*.tgz` / `*.tar` / `*.tar.gz`。upload 且后续发布 `exec` **成功后**删除本次本地压缩包；失败可保留重试。 |

### 建议步骤

```
1. 解析 SemVer（禁止改仓库源码 version）
2. 按仓库既有方式完成生产构建
3. 清空并组装 release/（扁平；无 node_modules；无真实 .env）
4. 按应用类型断言关键入口存在（见下文 Node / Web）
5. tar：tar -czf out.tgz -C release .
6. 打印产物路径；提醒：HTTP upload → 多次 exec；.env 在 root 共用
```

## Node 应用（服务端）

在共性之上额外要求：

| 项 | 要求 |
| --- | --- |
| 锁文件 | 必须带上与安装器匹配的 lock（当前常见 **`pnpm-lock.yaml`**），以便目标机 `pnpm install --prod --frozen-lockfile`（经 `exec`）成功。 |
| `engines.node` | 产物 `package.json` 建议含 **`engines.node`**，便于目标机选对运行时；缺则由调用方自行保证 Node 版本。 |
| 启动入口 | 包内含可启动文件（如 `dist/main.js` 或仓库约定入口）。 |
| 进程配置（若用 PM2） | 可带 `ecosystem.config.cjs`：只写进程接线（`name` / `script` / `cwd` / `instances` / `exec_mode` 等）。`cwd` 指向应用 **root**（与 `.env` 同目录）；`script` 指向版本目录内入口（相对包内路径，解压到 `releases/<id>/` 后仍成立）。 |
| 环境变量 | **优先且只认**应用 root 的 `.env`。**禁止**在 `ecosystem`（或等价进程配置）里写入 / 注入运行时环境变量（如 `PORT`、`NODE_ENV`、密钥、业务配置）。也不要从 `.env` 读出再写进 ecosystem。宿主或进程管理器若已有同名变量，应用启动时应让 `.env` 覆盖它们（勿依赖 ecosystem 来“钉死”）。 |
| 安装时机 | **不**假设 code_deploy 自动 `pnpm install`；由发布流程 `exec` 在解压目录执行。 |

## Web 应用（静态 / 前端构建产物）

在共性之上额外要求：

| 项 | 要求 |
| --- | --- |
| 构建产物 | 包内含可托管的静态根（如 `index.html` 及资源；或仓库约定的 `public/` / `dist/`）。 |
| 与 Node 同包 | 若同仓前后端打成**一个**包：静态资源与服务端入口的相对布局须与运行时托管方式一致（例如服务端读 `public/`），并在脚本注释写清。 |
| 无锁文件场景 | 纯静态托管、目标机不安装 Node 依赖时：可不带 `pnpm-lock.yaml`；**不要**假装需要 `pnpm install`。 |
| 路由 | SPA 若依赖服务器 fallback，在 README/注释说明，不靠 code_deploy 模板处理。 |

## 应用登记（运维提示）

code_deploy 每应用仅需：

```json
{
  "my-app": {
    "root": "/path/to/app",
    "key": "<secret>"
  }
}
```

无 `template` / `deploy` / `rollback` / `pm2Name` / `logFile`。

## 与「发版」口令的关系

- 用户说「部署 / 发版 / 重新部署」：走 `code-deploy`；若部署入口即 pack 脚本，产物须满足本技能。
- 用户只说「写打包脚本 / pack:release / code-pack-script」：只改/新增打包脚本与必要说明，**不**改 STATUS、**不**写 CHANGELOG，除非同时要求发版。

## 完成时输出

只改打包脚本时：列出改动路径，并核对：扁平包、无真实 `.env`、无不当依赖目录、Node/Web 关键入口、ecosystem（若有）未注入运行时环境变量、以及「upload → exec」而非旧 `deploy(uploadId)` 话术。
