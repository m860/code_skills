---
name: code-github-ci
description: >-
  Prefer GitHub Actions + self-hosted runners when adding or editing CI on
  GitHub-hosted repos. Part B supports PM2 or Docker (user chooses hosting),
  plus post-deploy ship-docs and delete-failed-tag. Docker runtime rules live
  in code-docker-deploy; packaging in code-docker-pack; this skill adds Actions
  wiring and first-time build-location choice. Agent-side release is
  code-github-deploy. Use when adding or editing CI/workflows, PM2 or Docker
  deploy, self-hosted runner, or GitHub CI 规范.
---

# GitHub CI 约束

本技能约束 **GitHub 托管项目**的 CI 选型与写法：优先用 GitHub Actions Workflow + self-hosted runner；并定义 **Part B 部署流水线**，托管方式为 **PM2** 或 **Docker（Compose）** 二选一（可并存于同一仓库的不同服务，由用户选择本次走哪条）。

Agent 侧发版编排见 `../code-github-deploy/SKILL.md`（Part A）。

- PM2 产物：`../code-node-pack/SKILL.md`
- Docker 产物：`../code-docker-pack/SKILL.md`
- Docker 目标机部署：`../code-docker-deploy/SKILL.md`

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
- 选 **Docker** → 第 7 节 + `code-docker-pack` + `code-docker-deploy`  

Part A（annotated tag + push，不强制 watch）两种托管共用；**文档回写与失败删 tag** 也共用（第 10 节），不因托管方式分叉。

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

由 **tag 推送**触发。目标机行为与回滚/保留版本以 **`code-docker-deploy`** 为准；镜像/Compose 文件以 **`code-docker-pack`** 为准。本节只补 **GitHub Actions 接线**。

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

Registry、镜像名、凭证：跟随仓库；没有约定先问。密钥用 GitHub Secrets / 宿主已有登录，不写进 YAML 明文。

### 7.2 Actions 步骤顺序

```
【build · 按 7.1 已选位置 · 可 GitHub-hosted】
1. checkout（该 tag）— 若在 Actions 构建
2. 登录 registry
3. 按 code-docker-pack 构建并 push {image}:{SemVer}

【deploy job · self-hosted · needs: build（若 build 在同一 workflow）】
4. 按 code-docker-deploy：登录 → 钉 SemVer → compose pull && up -d
5. 健康检查（第 8 节 / code-docker-deploy）
6. 成功：保留最多 3 版；失败：回滚（第 9 节 / code-docker-deploy）
7. 其后：第 10 节 ship-docs / 失败时 delete-failed-tag（均在 GitHub-hosted）
```

若构建在开发机完成且镜像已 push：Actions 可仅跑 **deploy**（及第 10 节）；仍由 tag 触发。若无部署 workflow、只在开发机 up：须向用户说明未走 Part B，不假装 CI 已发布。

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
    runs-on: [self-hosted]   # 目标侧：见 code-docker-deploy
    steps:
      - run: <compose 指向 :{SemVer}；compose pull && up -d>
      - run: <健康检查或容器观察>
```

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

## 10. 文档回写与失败删 tag（Part B · 共用）

健康检查成功后的 **STATUS / CHANGELOG** 回写、以及部署失败时的 **远程 tag 清理**，由 Actions 完成；**不**依赖 Part A `gh run watch`。目标机 self-hosted job **不做** git ref 写操作（不在部署机上 push 文档或删 tag）。

Annotation 格式与 Part A 约定见 `code-github-deploy`（`kind: release` / `kind: redeploy`；release 行含 `| {需求目录名}`）。

### 10.1 Secret

- 仓库 Secret（建议名 **`RELEASE_GIT_TOKEN`**，仓库另有约定则跟随）：PAT 或等价凭据，对该仓 **Contents: Read and write**（含删 tag 所需写 refs）。  
- `GITHUB_TOKEN` 默认权限往往不足以向默认分支 push 文档或删 tag；**不要**假定默认 token 够用。  
- 缺 Secret：`ship-docs` / `delete-failed-tag` 应明确失败或 warning，**不**回滚已健康的线上版本。

### 10.2 `ship-docs`（成功路径）

- 触发条件：部署 job（含健康检查）**成功**之后；推荐独立 job，`runs-on: ubuntu-latest`（GitHub-hosted），`needs: <deploy-job>`。  
- 步骤要点：  
  1. 用 `RELEASE_GIT_TOKEN` checkout **默认分支**（如 `main`；跟随仓库）最新提交  
  2. force-fetch 本次 annotated tag（避免 checkout 剥掉 annotation）  
  3. 读 `git tag -l --format='%(contents)' v{SemVer}`，按 annotation 改文档（仓库脚本，如 `scripts/ship-release-docs.*`；无则按下列规则实现，不臆造第二套 STATUS 语义）  
  4. commit + push 回默认分支；冲突时有限次 rebase 重试  
- **`kind: release`**：仅将 annotation 中列出且 STATUS 当前为 `待部署` 的目录改为 `已部署`；在 CHANGELOG 写入本版区段与 `- {type}: {短标题}`（**不**把目录名写入 CHANGELOG 正文）；更新 `LAST_SHIPPED`。缺目录名 / STATUS 无此行 / 状态不是 `待部署` → 该条 skip + warning，其它合法行照常。  
- **`kind: redeploy`**：只更新 `LAST_SHIPPED`（可追加 `- redeploy: {stamp}`）；**不**改需求状态行。  
- `kind` 无法解析：不写 STATUS/CHANGELOG；job 失败或显式 skip（跟随仓库，须可观测）。  
- **文档回写失败不触发目标机回滚**（线上已健康的 SemVer 保持）；summary 标明 deploy ok / docs failed。  
- 回写 push 的是文档提交，**不得**再触发一次 tag 部署（deploy 仅 `push: tags: v*`）。

### 10.3 `delete-failed-tag`（失败路径）

- 独立 job，GitHub-hosted；`needs: [build?, deploy]`（以仓库 job 名为准）；`if: always() && (build 或 deploy 失败)`。  
- 用 `RELEASE_GIT_TOKEN` 调 GitHub API 删除本次 `refs/tags/v{SemVer}`。  
- 缺 Secret / 404 / 非 2xx → warning，不改变目标机回滚结果。  
- **`ship-docs` 失败不删 tag**（避免文档问题误删已成功发版标记）。

### 10.4 示意（节选）

```yaml
  ship-docs:
    needs: deploy
    runs-on: ubuntu-latest
    if: success()
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main   # 或仓库默认分支
          token: ${{ secrets.RELEASE_GIT_TOKEN }}
          fetch-depth: 0
      - run: git fetch --force --no-tags origin "refs/tags/${{ github.ref_name }}:refs/tags/${{ github.ref_name }}"
      - run: |
          git tag -l --format='%(contents)' "${{ github.ref_name }}" \
            | <ship-release-docs 入口> "${VERSION}"
      - run: |
          git add docs/CHANGELOG.md docs/features/STATUS.md
          git diff --staged --quiet || git commit -m "docs: 记录 ${VERSION} 发版（CI）"
          git push origin HEAD:main

  delete-failed-tag:
    needs: [build, deploy]   # job 名跟随仓库；无 build 则只 needs deploy
    runs-on: ubuntu-latest
    if: always() && (needs.build.result == 'failure' || needs.deploy.result == 'failure')
    steps:
      - run: <DELETE refs/tags/${{ github.ref_name }} via API + RELEASE_GIT_TOKEN>
```

### 10.5 与 Part A 的分工

| 事项 | Part A | Part B |
| --- | --- | --- |
| 圈定 / 定 SemVer / annotated tag + push | 负责 | — |
| `gh run watch` | 可选，非必须 | — |
| 构建、部署、健康检查、回滚 | — | 负责 |
| 写 STATUS / CHANGELOG / `LAST_SHIPPED` | **不写** | 成功后 `ship-docs` |
| 删除失败的远程 tag | **不删** | `delete-failed-tag` |

job summary 建议写出：托管方式、version、是否回滚、当前运行指向、docs 是否已 push、失败 tag 是否已删。

## 11. 不应做

- 不为「顺便统一」迁移非 Actions 流水线，或擅自把 PM2 仓改成 Docker（或相反）。  
- 未确认时不把**部署** job 改成 GitHub-hosted。  
- Docker 路径不跳过「目标机 pull + Compose」；构建位置不在第一次未问清时擅自选定。  
- 不跳过健康检查 / 回滚 / 最多 3 版。  
- 不在目标机 self-hosted 上 push 文档或删 GitHub tag。  
- 不以短标题模糊匹配 STATUS；不在 CI 成功前由 Agent 改文档。  
- 托管方式不明时不擅自二选一。

## 完成时自检

1. 本次托管方式是否已由用户或仓库选定（并存时已问清）  
2. PM2 是否走第 6 节 + `code-node-pack`；Docker 是否走第 7 节 + `code-docker-pack` + `code-docker-deploy`  
3. Docker：第一次是否已问构建位置；之后是否跟随约定；目标机是否符合 `code-docker-deploy`  
4. 健康检查与回滚、最多 3 版是否满足（Docker 细则以 `code-docker-deploy` 为准）  
5. 是否具备 `ship-docs` + `delete-failed-tag`（或仓库等价），且文档回写失败不回滚已健康版本  
6. Secret（如 `RELEASE_GIT_TOKEN`）与 annotated tag 约定是否已文档化  

