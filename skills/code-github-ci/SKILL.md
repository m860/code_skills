---
name: code-github-ci
description: >-
  Prefer GitHub Actions + self-hosted runners when adding or editing CI on
  GitHub-hosted repos. Also defines Part B Node/PM2 deploy on self-hosted:
  pack, install, switch current, start, health check (or PM2 log probe), keep
  3 releases, rollback on failure. Agent-side release (tag, gh watch, docs) is
  code-github-deploy. Use when adding or editing CI/workflows, Node deploy pipelines,
  self-hosted runner, or GitHub CI / Actions / workflow 规范.
---

# GitHub CI 约束

本技能约束 **GitHub 托管项目**的 CI 选型与写法：优先用 GitHub Actions Workflow + self-hosted runner；并定义 **Node 应用在 self-hosted 上的部署流水线（Part B）**。

Agent 侧发版编排（圈定、打 tag、`gh run watch`、成功后改文档）见 `../code-github-deploy/SKILL.md`（Part A）。打包产物形态见 `../code-node-pack/SKILL.md`。

- **新增 CI**：选型与写法按本技能。
- **修改 CI**：若目标已是 Actions + self-hosted（或本次改成这种方式），须按本技能检查并保持合法。
- 不禁止其他 CI，也不要求把已有非此方式的流水线迁过来。

不规定业务构建命令细节。跟随仓库已有约定；没有约定先问。

## 1. 何时适用

- 适用：仓库在 GitHub 上，且用户要**新增**或**修改** CI / workflow / Actions，或要落地/调整 **Node 部署** workflow。
- 修改时：仅当改动的流水线采用（或改为）Actions + self-hosted 时，才做合法性检查；其他 CI 方案不按本技能审计，除非用户明确要求。
- 不适用：非 GitHub 托管；只改业务代码、不碰 CI。

## 2. 选型优先（新增）

1. **首选**：GitHub Actions（`.github/workflows/`）+ **self-hosted** runner  
2. 仓库或用户已明确指定其他方案：跟随，并说明未用首选的原因  
3. 拿不准：先问一条，再动手  

不主动推荐 GitHub-hosted runner（如 `ubuntu-latest`）作为新增默认；仅当用户明确要求、或仓库文档写明只能用 hosted 时再用。

## 3. Workflow 通用写法

- 文件在 `.github/workflows/`，YAML；命名跟随仓库习惯。
- `runs-on`：**优先** self-hosted；labels 跟随仓库约定，没有则先问。
- `permissions` / secrets / 触发：最小必要；没有约定先问。
- 具体安装与测试命令用仓库已有入口，无则问。

## 4. 修改时的合法性检查

改动目标是 Actions + self-hosted（或本次改为这种方式）时，对照检查：

1. 文件在 `.github/workflows/`，有效 YAML  
2. 相关 job 仍为 self-hosted（无说明时未改成 hosted）  
3. labels 符合仓库约定  
4. permissions / secrets / 触发未无故放大  
5. 步骤仍走仓库已有入口；若为 Node 部署 job，还须符合下文 Part B  

只检查本次改动触及的 workflow / job。

## 5. Node 部署流水线（Part B）

由 **tag 推送**触发（与 `code-github-deploy` 约定的 `v{X.Y.Z}` 一致）。在 **self-hosted** runner 上执行实际发布。默认假设 runner 能直达部署目录（runner 即目标机或可本地写发布根）；需要 SSH 跳板时跟随仓库约定，没有则先问。

### 5.1 版本与触发

- 触发：`push` tags（匹配 `v*`）或仓库已有等价约定。
- SemVer：从 tag 解析（去掉前导 `v`）。打包时把该版本传给仓库 pack 入口（对齐 `code-node-pack` 的版本来源，**不要**改仓库源码里的 version 字段）。

### 5.2 步骤顺序（必须按序）

```
1. checkout（该 tag 对应提交）
2. 打包（仓库 pack / pack:release 入口 → 扁平 tgz；约束见 code-node-pack）
3. 安装依赖：解压到 releases/<version>/（或仓库约定 id），在该目录 pnpm install --prod --frozen-lockfile（安装器跟随仓库）
4. 切流：将 current（或等价指针）指向新 release
5. 启动：pm2 startOrReload 包内 ecosystem（或仓库等价命令）；不在 ecosystem/workflow 注入业务 env
6. 健康检查（见 5.3）
7. 成功：裁剪历史版本，默认最多保留 3 个 release 目录；失败：回滚后让 job 失败（见 5.4）
```

发布根目录布局仓库已有则跟随；没有约定时采用：

```text
<deploy-root>/
  current -> releases/<version>/
  releases/
  shared/          # 可选：宿主机 .env、日志等，不进包
```

`shared` 中的 `.env` 按仓库方式链到或供进程使用；**禁止**把生产密钥写进 YAML 或打进包。

### 5.3 健康检查

- **有健康检查 URL**（`AGENTS.md` 或项目文档）：`GET` 后须 JSON 含 `ok: true`，且 `version` 字符串等于本次 tag 解析出的 SemVer。不满足则失败。
- **无 URL**：不得臆造 URL。改为在一段观察窗内查看 PM2 进程/日志（时长跟随仓库；没有约定则先问，问不清可用约 60s）：进程保持 online、无持续重启/崩溃栈，才算通过；否则失败。

### 5.4 失败与回滚

| 失败点 | 处理 |
| --- | --- |
| checkout / 打包 / 新目录内安装（尚未切流） | job 失败；不改 `current`；可删除半成品目录 |
| 已切流或启动后 / 健康检查失败 | **回滚**：`current` 指回上一版 release，再 PM2 reload；然后 job 失败 |
| 回滚自身失败 | job 失败；在日志/summary 中标明状态不明，需人工检查 |

回滚成功后：建议删除本次失败的新 release 目录，避免占满保留名额。

### 5.5 保留版本数

- 默认 **最多保留 3 个** `releases/*` 目录（含当前）。成功发布并切流后，删除更旧的。
- 仓库文档另有数量则跟随。

### 5.6 与 Part A 的分工

- Part B **只**负责 runner 上发布与回滚；**不**改 `docs/CHANGELOG.md` / `STATUS.md`。
- Part A 用 `gh run watch` 看本 job 结论；失败时由 Part A 删 tag。
- job summary 建议写出：version、是否回滚、当前 `current` 指向，便于 `gh run view`。

### 5.7 示意（非可照抄命令）

```yaml
name: deploy
on:
  push:
    tags: ['v*']
jobs:
  deploy:
    runs-on: [self-hosted]
    steps:
      - uses: actions/checkout@v4
      - run: <仓库 pack 入口，版本来自 tag>
      - run: <解压到 releases/… + install --prod>
      - run: <切 current + pm2 reload>
      - run: <健康检查或 PM2 观察>
      # 失败：回滚；成功：只留 3 个版本
```

## 6. 不应做

- 不为「顺便统一」迁移非 Actions / 非 self-hosted 流水线。  
- 未确认时不把 self-hosted job 改成 GitHub-hosted。  
- 不把本技能当成禁止其他 CI 的禁令。  
- Node 部署 job 不跳过健康检查/回滚/保留 3 版约定。  
- 不在 Part B 里写 STATUS / CHANGELOG。  

## 完成时自检

1. 新增是否优先 Actions + self-hosted；修改是否做了合法性检查  
2. Node 部署是否按 5.2 顺序，且失败可回滚、成功最多 3 版  
3. 健康检查：有 URL 用 URL，无 URL 用 PM2 观察，未臆造 URL  
4. 版本是否来自 tag，打包是否对齐 `code-node-pack`  
5. 是否未在 CI 中改需求文档  
