---
name: code-node-pack
description: >-
  Constraints for Node.js pack/release scripts and PM2-oriented artifacts:
  flat tarball layout, version only in the artifact, pnpm lock, engines.node,
  ecosystem.config without injected env, and no real .env or node_modules in
  the package. Use when the user asks for code-node-pack, pack:release,
  打包脚本, Node 打包, PM2 ecosystem, release.tgz, or Node/PM2 packaging.
---

# Node / PM2 打包

约定业务仓如何写 **Node 服务端** 的 pack / release 脚本与产物，使产物适合用 **PM2** 启动（含 `ecosystem` 接线）。

本技能**只**管 **PM2 路径**的打包脚本与产物约束。不写目标机目录/发布步骤，不改需求状态、不写 CHANGELOG，除非用户同时要求发版。

Docker / Compose 镜像产物见 `../code-docker-pack/SKILL.md`；目标机 Docker 部署见 `../code-docker-deploy/SKILL.md`。CI 中选 PM2 还是 Docker 见 `../code-github-ci/SKILL.md`。

不规定具体构建命令或仓库目录名；跟随目标仓库已有约定。没有约定先问。

用户未要求写/改打包脚本时，不擅自新增或大改 pack 脚本。

## 产物约束

| 项 | 要求 |
| --- | --- |
| 版本写入 | **仅**写入发布产物内的清单（如产物 `package.json` 的 `version`）。**禁止**改仓库源码里的版本字段。 |
| 版本来源 | 优先：`VERSION` → CLI `--version`/`-v` → 调用方传入（如 CI 从 `vX.Y.Z` tag 解析）→ `docs/CHANGELOG.md` 最上 `## X.Y.Z`。皆无则失败说明，不臆造。 |
| 产物目录 | 组装可部署的扁平目录（下文称 `release/`）：运行所需文件、`package.json`、锁文件、可选 `.env.example`、**PM2** `ecosystem.config.cjs`（或仓库等价名）。 |
| **不要**打进包 | `node_modules`、真实 `.env` / 密钥、与运行无关的源码或测试产物（除非仓库明确要求）。 |
| 压缩包形态 | `.tar` / `.tar.gz` / `.tgz`。归档成员为**扁平根**（`tar -czf out.tgz -C release .`），**不要**再包一层 `release/` 目录名。 |
| 文件名 | 建议 `{SemVer}-{YYYYMMDDHHmmss}.tgz`（或等价），便于从文件名识别版本。 |
| 可重复 | 脚本可直接执行；注明最小依赖与入参。 |
| 本地清理 | 仓库忽略 `*.tgz` / `*.tar` / `*.tar.gz`。打包流程成功产出后，按仓库约定清理本次本地压缩包；失败可保留重试。 |

### 建议步骤

```
1. 解析 SemVer（禁止改仓库源码 version）
2. 按仓库既有方式完成生产构建
3. 清空并组装 release/（扁平；无 node_modules；无真实 .env）
4. 断言启动入口与 ecosystem 存在（见下文）
5. tar：tar -czf out.tgz -C release .
6. 打印产物路径
```

## Node + PM2 要求

| 项 | 要求 |
| --- | --- |
| 锁文件 | 必须带上与安装器匹配的 lock（当前常见 **`pnpm-lock.yaml`**），以便目标环境可 `pnpm install --prod --frozen-lockfile`。 |
| `engines.node` | 产物 `package.json` 建议含 **`engines.node`**；缺则由运行方自行保证 Node 版本。 |
| 启动入口 | 包内含可启动文件（如 `dist/main.js` 或仓库约定入口）。 |
| PM2 配置 | 包内带 `ecosystem.config.cjs`（或仓库既有等价文件）：只写进程接线（`name` / `script` / `cwd` / `instances` / `exec_mode` 等）。`script` 使用包内相对路径，解压后仍成立。 |
| 环境变量 | **禁止**在 `ecosystem` 里写入 / 注入运行时环境变量（如 `PORT`、`NODE_ENV`、密钥、业务配置）。也不要从 `.env` 读出再写进 ecosystem。真实 `.env` **不**打进包（最多 `.env.example`）。 |
| 依赖 | **不**把 `node_modules` 打进包；由运行方在解压后安装生产依赖。 |
| 同包静态资源 | 若同仓前端构建产物一并打进 Node 包：静态根与服务端入口的相对布局须一致，并在脚本注释写清。纯静态、无 Node/PM2 进程的托管不在本技能范围。 |

## 口令边界

- 用户只说「写打包脚本 / pack:release / code-node-pack / Node 打包 / PM2 打包」：只改/新增打包脚本与必要说明，**不**改 STATUS、**不**写 CHANGELOG，除非同时要求发版。

## 完成时输出

只改打包脚本时：列出改动路径，并核对：扁平包、无真实 `.env`、无 `node_modules`、启动入口、`ecosystem` 未注入运行时环境变量、锁文件与 `engines.node`（若约定需要）齐全。
