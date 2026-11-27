---
name: code-github-ci
description: >-
  Prefer GitHub Actions + self-hosted runners when adding or editing CI on
  GitHub-hosted repos. Part B supports PM2 or Docker (user chooses hosting).
  Docker: ask build location on first project setup (GitHub-hosted vs developer
  machine), then follow that; target pulls + Compose; artifacts via
  code-docker-pack. Shared gates: health check, keep 3 versions, rollback.
  Agent-side release is code-github-deploy. Use when adding or editing
  CI/workflows, PM2 or Docker deploy, self-hosted runner, or GitHub CI 规范.
---

# GitHub CI 约束

本技能约束 **GitHub 托管项目**的 CI 选型与写法：优先用 GitHub Actions Workflow + self-hosted runner；并定义 **Part B 部署流水线**，托管方式为 **PM2** 或 **Docker（Compose）** 二选一（可并存于同一仓库的不同服务，由用户选择本次走哪条）。

Agent 侧发版编排见 `../code-github-deploy/SKILL.md`（Part A）。

- PM2 产物：`../code-node-pack/SKILL.md`
- Docker 产物：`../code-docker-pack/SKILL.md`

- **新增 CI**：选型与写法按本技能。
- **修改 CI**：若目标已是 Actions + self-hosted（或本次改成这种方式），须按本技能检查并保持合法；Docker **构建** job 按下文允许使用 GitHub-hosted。
- 不禁止其他 CI，也不要求把已有非此方式的流水线迁过来。

不规定业务构建命令细节。跟随仓库已有约定；没有约定先问。

## 1. 何时适用

- 适用：仓库在 GitHub 上，且用户要**新增**或**修改** CI / workflow / Actions，或要落地/调整 **PM2 或 Docker 部署** workflow。
- 修改时：仅当改动的流水线采用（或改为）本技能约定方式时，才做合法性检查；其他 CI 方案不按本技能审计，除非用户明确要求。
- 不适用：非 GitHub 托管；只改业务代码、不碰 CI。

## 2. 选型优先（新增）

1. **首选**：GitHub Actions（`.github/workflows/`）+ **self-hosted** runner（**部署 / 目标机侧** job）  
2. **例外 — Docker 镜像构建**：构建机可以是 **GitHub-hosted**（如 `ubuntu-latest`）或**开发机**；**不要**把「目标机 self-hosted 上 docker build」当作默认。具体选哪边见第 7.1 节（**项目第一次**问用户）。  
3. 仓库或用户已明确指定其他方案：跟随，并说明未用首选的原因  
4. 拿不准：先问一条，再动手  

## 3. 托管方式选择（Part B）

同一仓库可以**并存** PM2 与 Docker 两套部署配置；**本次**新增或修改部署流水线时必须选定一种：

| 优先级 | 动作 |
| --- | --- |
| 用户明确说 PM2 或 Docker | 跟用户 |
| 用户未说、仓库仅有一种约定/现成 workflow | 跟随仓库 |
| 两种都有或都没有、无法判断 | **先问用户选哪一种**，不擅自默认 |

- 选 **PM2** → 第 6 节 + `code-node-pack`  
- 选 **Docker** → 第 7 节 + `code-docker-pack`  

Part A（打 tag / `gh watch` / 成功后写文档）两种托管共用，不因托管方式分叉。

## 4. Workflow 通用写法

- 文件在 `.github/workflows/`，YAML；命名跟随仓库习惯。
- 部署类 job 的 `runs-on`：**优先** self-hosted；labels 跟随仓库约定，没有则先问。
- Docker **build** job：优先 GitHub-hosted（见第 2 节例外）。
- `permissions` / secrets / 触发：最小必要；没有约定先问。
- 具体安装与测试命令用仓库已有入口，无则问。

## 5. 修改时的合法性检查

改动目标采用本技能约定方式时，对照检查：

1. 文件在 `.github/workflows/`，有效 YAML  
2. 部署 job 仍为 self-hosted（无说明时未改成 hosted）；Docker build job 可为 GitHub-hosted  
3. labels 符合仓库约定  
4. permissions / secrets / 触发未无故放大  
5. 步骤仍走仓库已有入口；PM2 / Docker 部署还须符合第 6 / 7 节  

只检查本次改动触及的 workflow / job。

## 6. Part B · PM2

由 **tag 推送**触发（`v{X.Y.Z}`）。在 **self-hosted** 上执行发布。默认 runner 能直达部署目录；需要 SSH 时跟随仓库，没有则先问。

### 6.1 版本与触发

- 触发：`push` tags（`v*`）或仓库等价约定。
- SemVer：从 tag 解析（去 `v`）。打包时传入仓库 pack 入口（对齐 `code-node-pack`），**不**改源码 version 字段。

### 6.2 步骤顺序

```
1. checkout（该 tag）
2. 打包（pack / pack:release → 扁平 tgz；见 code-node-pack）
3. 解压到 releases/<version>/，pnpm install --prod --frozen-lockfile（安装器跟随仓库）
4. 切流：current → 新 release
5. PM2 startOrReload（ecosystem）；不在 ecosystem/workflow 注入业务 env
6. 健康检查（第 8 节；无 URL 时观察 PM2）
7. 成功：最多保留 3 个 release 目录；失败：回滚（第 9 节）
```

布局仓库已有则跟随；默认：

```text
<deploy-root>/
  current -> releases/<version>/
  releases/
  shared/          # 可选：.env、日志等
```

### 6.3 示意

```yaml
name: deploy-pm2
on:
  push:
    tags: ['v*']
jobs:
  deploy:
    runs-on: [self-hosted]
    steps:
      - uses: actions/checkout@v4
      - run: <pack 入口，版本来自 tag>
      - run: <解压 + install --prod + 切 current + pm2 reload>
      - run: <健康检查或 PM2 观察>
```

## 7. Part B · Docker（Compose）

由 **tag 推送**触发。目标机 **pull** 镜像后用 **Compose** 启动。构建与部署拆开。

### 7.1 构建位置（第一次询问）

可选两边（都 push 到约定 registry，供目标机 pull）：

| 选项 | 含义 |
| --- | --- |
| **GitHub-hosted** | Actions 用平台 runner（如 `ubuntu-latest`）`build` + `push`；**不是** self-hosted |
| **开发机** | 在开发者机器上 `build` + `push`；Actions 侧可只跑 deploy |

**何时问、何时跟：**

1. **项目第一次**落地 Docker 部署（仓库尚无构建位置约定：无文档/AGENTS、无已有 build workflow、用户也未点名）→ **必须询问**用户选哪一种；问清之前不擅自写死。  
2. 已有约定（文档、现有 workflow、或用户曾选定并留下的配置）→ **跟随**，不再每次发版重问。  
3. 用户本次明确改口 → 跟本次，并更新仓库约定/workflow，避免两套打架。  
4. 已选 GitHub-hosted 但实际不可用（权限、额度、网络等）→ 说明原因，再问是否改走开发机；不静默改道。

建议把选定结果写进项目 `AGENTS.md`（或等价 Agent 规则）/ 部署文档一句，便于下次跟随。

目标机（self-hosted 部署 job）**只 pull + Compose**，不把「目标机现场 docker build」当作默认主路径。

Registry、镜像名、凭证：跟随仓库；没有约定先问。密钥用 GitHub Secrets / 宿主已有登录，不写进 YAML 明文。

### 7.2 步骤顺序

```
【build · 按 7.1 已选位置】
1. checkout（该 tag）— 若在 Actions 构建
2. 登录 registry
3. 按 code-docker-pack 构建镜像，tag={image}:{SemVer}，push

【deploy job · self-hosted · needs: build（若 build 也在同一 workflow）】
4. 登录 registry（若需要）
5. 在部署目录更新/渲染 compose，使服务 image 指向 {image}:{SemVer}
6. docker compose pull && docker compose up -d（或仓库等价）
7. 健康检查（第 8 节；无 URL 时观察容器状态/日志）
8. 成功：最多保留 3 个 SemVer 镜像引用/标签策略（见 7.4）；失败：回滚（第 9 节）
```

若构建在开发机完成且镜像已 push：Actions 可仅跑 **deploy** job（仍由 tag 或用户约定触发）；Part A 仍 `gh run watch` 部署 run。若无部署 workflow、只在开发机 up：须向用户说明未走 Part B，不假装 CI 已发布。

### 7.3 示意

```yaml
name: deploy-docker
on:
  push:
    tags: ['v*']
jobs:
  build:
    runs-on: ubuntu-latest   # 若 7.1 选了 GitHub-hosted；选开发机则无此 job
    steps:
      - uses: actions/checkout@v4
      - run: <login registry>
      - run: <build & push {image}:{SemVer}>
  deploy:
    needs: build             # 仅当 build 在同一 workflow；开发机构建时可去掉 needs
    runs-on: [self-hosted]   # 目标侧：pull + compose
    steps:
      - run: <compose 指向 :{SemVer}；compose pull && up -d>
      - run: <健康检查或容器观察>
```

### 7.4 保留版本（Docker）

- 默认与 PM2 对齐：**最多保留 3 个**已成功发布的 SemVer（镜像 tag 与/或部署目录中的 compose 钉扎记录）。成功后清理更旧的镜像 tag/本地悬空镜像，方式跟随仓库；没有约定先问，避免误删共享基础镜像。  
- 回滚目标为**上一成功 SemVer** 的 image + compose，不依赖 `latest`。

## 8. 健康检查（共用）

- **有健康检查 URL**（`AGENTS.md` 或项目文档）：`GET` 后须 JSON 含 `ok: true`，且 `version` 等于本次 tag 的 SemVer。不满足则失败。  
- **无 URL**：不得臆造。改为观察窗（时长跟随仓库；没有约定则先问，问不清约 60s）：  
  - PM2：进程 online、无持续重启/崩溃  
  - Docker：容器 running / healthy（若有 healthcheck）、无持续重启/崩溃日志  
  否则失败。

## 9. 失败与回滚（共用）

| 失败点 | 处理 |
| --- | --- |
| 构建/打包/push（尚未在目标机切流） | job 失败；目标机 `current`/运行中服务不变 |
| 已切流或启动后 / 健康检查失败 | **回滚**到上一成功版本（PM2：`current`+reload；Docker：compose 指回上一 SemVer 并 `up -d`），然后 job 失败 |
| 回滚自身失败 | job 失败；summary 标明状态不明 |

回滚成功后：建议清理本次失败产物（半成品目录或未采用的新 tag 引用），避免占满 3 槽。

## 10. 与 Part A 的分工

- Part B **不**改 `docs/CHANGELOG.md` / `STATUS.md`。  
- Part A 用 `gh run watch` 看**整次** workflow（含 Docker 的 build+deploy）；失败时由 Part A 删 tag。  
- job summary 建议写出：托管方式、version、是否回滚、当前运行指向。

## 11. 不应做

- 不为「顺便统一」迁移非 Actions 流水线，或擅自把 PM2 仓改成 Docker（或相反）。  
- 未确认时不把**部署** job 改成 GitHub-hosted。  
- Docker 路径不跳过「目标机 pull + Compose」；构建位置不在第一次未问清时擅自选定。  
- 不跳过健康检查 / 回滚 / 最多 3 版。  
- 不在 Part B 写 STATUS / CHANGELOG。  
- 托管方式不明时不擅自二选一。

## 完成时自检

1. 本次托管方式是否已由用户或仓库选定（并存时已问清）  
2. PM2 是否走第 6 节 + `code-node-pack`；Docker 是否走第 7 节 + `code-docker-pack`  
3. Docker：第一次是否已问构建位置；之后是否跟随约定；部署是否 self-hosted pull + Compose  
4. 健康检查与回滚、最多 3 版是否满足  
5. 是否未在 CI 中改需求文档  
