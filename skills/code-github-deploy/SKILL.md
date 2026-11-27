---
name: code-github-deploy
description: >-
  GitHub-hosted release orchestration (Part A): scope 待部署 rows, choose SemVer,
  push vX.Y.Z tag, watch the deploy Actions run with gh, then on success write
  CHANGELOG/STATUS and commit-push; on failure delete the tag and leave docs
  unchanged. Requires gh. Runtime deploy is code-github-ci (Part B). Use when
  the user asks for code-github-deploy, 部署, deploy, 发版, 重新部署, or release
  on a GitHub-hosted repo.
---

# GitHub 部署（Part A）

本技能是 `code-spec-common` 的专用层，**仅适用于 GitHub 托管**仓库。只做发版或重新部署的 **Agent 侧编排**：圈定 → 打 tag → 用 `gh` 等待 CI → **成功后**才改 CHANGELOG / STATUS 并提交推送。

机器上的发布（PM2 或 Docker/Compose，由用户/仓库选定）见 `../code-github-ci/SKILL.md`（Part B）。本文件不重复 Part B 步骤，也不代替用户选择托管方式。

开始前先读 `../code-spec-common/SKILL.md`。术语、目录、状态、CHANGELOG 以它为准。网络失败先读 `../code-network/SKILL.md`。提交规范读 `../code-git/SKILL.md`（本技能的部署口令视为已明确要求提交与推送发版相关变更）。

用户没有明确说「部署」「deploy」「发版」或「重新部署」时，不发布。

## 适用范围

- **适用**：`origin`（或等价远程）为 GitHub；用户要发版或重新部署。
- **不适用**：非 GitHub 托管。停止并说明；不改 STATUS / CHANGELOG，不打 tag。

## 依赖：GitHub CLI（`gh`）

本技能**硬依赖**已安装且已登录的 [`gh`](https://cli.github.com/)，且能读取该仓库的 Actions：

- 开始监视 CI 前确认 `gh` 可用（如 `gh auth status`）。不可用或未登录：停止并说明，不假装部署完成。
- push tag 之后必须用 `gh run list` / `gh run watch` 等待本次部署；失败时用 `gh run view`（或 `--log-failed`）取摘要与 run URL。
- 不引入私有 webhook 作为 Part A 的主通知通道；给人看的通知由仓库自便，不替代 `gh run watch`。

## 与 Part B 的衔接

```
Part A（本技能）          Part B（code-github-ci · PM2 或 Docker）
圈定/定版 → push tag  →  PM2：self-hosted 打包→切流→PM2
                         Docker：按约定构建 push（首次问 GitHub-hosted 或开发机）→ self-hosted pull+Compose
         ← gh run watch ← 成功 / 失败（失败时目标侧回滚）
成功 → 写文档并 commit/push
失败 → 删 tag，不改文档
```

- Tag 名：**始终** `v{X.Y.Z}`（与 SemVer 对应，带 `v` 前缀）。不使用 `vX.Y.Z-时间戳`。
- 版本号传入 CI：以 tag 为准；workflow 侧解析见 `code-github-ci`。
- **STATUS / CHANGELOG / LAST_SHIPPED 仅在 CI 成功之后修改。** 打 tag 之前不改这些文件。

## 先分清口令

| 用户原话 | 走哪条 |
| --- | --- |
| 部署、deploy、发版 | 发版 |
| 重新部署 | 重新部署 |

两个口令都没说清时，先问一条。不要猜测。

## 发版

```
- [ ] 1. 确认 GitHub 托管；gh 可用
- [ ] 2. 圈定 STATUS.md 中待部署需求；定一个 SemVer
- [ ] 3. 打 tag vX.Y.Z 并 push（此时不改 STATUS/CHANGELOG）
- [ ] 4. gh 定位并 watch 由该 tag 触发的 deploy run
- [ ] 5a. 成功：写 CHANGELOG、LAST_SHIPPED、STATUS→已部署；提交并推送
- [ ] 5b. 失败或超时：报告；删除本地与远程该 tag；不改文档
```

### 1. 圈定

读 `docs/features/STATUS.md`。

- 用户没点名：圈定全部 `待部署` 行。
- 用户点了目录：只圈定点名的那些，且每一行都必须已经是 `待部署`。有任一行不是，停下来说明，不发布。
- 一行都没有：停止。告知没有待发版需求。若只是要把当前顶版再上线一次，请改口令为「重新部署」。
- `已实现` 及其他状态不纳入，也不要改成 `待部署`。

### 2. 定一个版本号

读圈定需求的 SPEC、TECH，以及已有的 BRD。变更类型以正文里写的为准。相对当前产品版本只升一档：

- 有不兼容的对外行为变化 → major（`1.2.0` → `2.0.0`）
- 否则有新能力且保持兼容 → minor（`1.2.0` → `1.3.0`）
- 否则 → patch（`1.2.0` → `1.2.1`）

同一批里几种变更都有，取更高的那一档。尚未发版时，不自拟第一个版本号，先问用户。

行为算不算不兼容、读不到依据时，先问一条。问清之前不打 tag，也不写 CHANGELOG。

### 3. 打 tag 并 push

- Tag：`v{SemVer}`，打在**当前待发布提交**（一般为当前 `HEAD`；以仓库约定为准）。
- 若本地或远程已存在同名 tag：先确认是否为上次失败残留。是则按「失败清理」删掉后再打；若对应一次**已成功**发版且用户要的是新版本，停下来说明，不要覆盖。
- `git push` 该 tag 到 GitHub。push 失败：停止，不改文档。

### 4. 用 `gh` 等待 CI

1. 用该 tag 对应 commit（或 `--branch`/`tag` 名）列出 runs，锁定**本次**部署 workflow（文件名或 workflow 名跟随仓库；没有约定先问，不臆造）。
2. `gh run watch <run_id> --exit-status`（或等价等待至 conclusion）。
3. **超时**（默认 30 分钟，仓库另有约定则跟随）：视为未完成。报告 run URL；**删除**该 tag（本地+远程）；不改 STATUS/CHANGELOG。不假装成功。
4. **失败**：`gh run view` 取失败摘要与 URL；**删除**该 tag（本地+远程）；不改 STATUS/CHANGELOG。Runner 侧回滚由 Part B 负责，Part A 不上机。
5. 删远程 tag：`git push origin --delete refs/tags/vX.Y.Z`（或等价）。删除失败须说明，不能假装已清理。

### 5. 成功后写文档并推送

仅当 run conclusion 为成功：

1. 按 `code-spec-common` 在 `docs/CHANGELOG.md` 顶部写入新版本；日期为当天；每个圈定需求一条，前缀用该需求正文中的变更类型。
2. 更新 `LAST_SHIPPED` 为当前本地时间。
3. 圈定行改为 `已部署`；其他行不动。
4. 提交上述文档（本会话发版相关文件）并推送到 GitHub。提交信息遵循 `code-git`。

## 重新部署

```
- [ ] 1. 确认 GitHub 托管；gh 可用；CHANGELOG 已有顶版
- [ ] 2. 不升版；如需触发 CI：删除远程/本地 v{顶版} 后在目标提交上重建并 push 同名 tag
- [ ] 3. gh watch 本次 run
- [ ] 4a. 成功：更新 LAST_SHIPPED，可追加 redeploy 行；提交推送；不改需求状态
- [ ] 4b. 失败或超时：报告；删除该 tag；不改文档
```

1. 没有顶版：停止，告知需先发版。
2. 版本号用当前顶版。为再次触发 Part B：**删除**已有 `v{顶版}` 后，在目标提交（默认当前 `HEAD`）上重建同名 tag 并 push。这是 CI 扳机，不是升版。
3. `gh run watch` 与失败删 tag 的规则同发版。
4. 成功后：按 `code-spec-common` 更新 `LAST_SHIPPED`，并可在该顶版区段下追加 `- redeploy: YYYYMMDDHHmmss`。**不**改需求状态。然后提交并推送文档。

## 完成时输出

发版成功：

1. 新版本号与升档依据  
2. tag 名与 commit  
3. Actions run URL  
4. `LAST_SHIPPED`  
5. 改为 `已部署` 的目录  
6. CHANGELOG 条目摘要  

重新部署成功：

1. 顶版号（未升版）  
2. 同名 tag 已重打并推送  
3. run URL  
4. `LAST_SHIPPED`  
5. 是否追加了 `redeploy` 行  

失败或超时：run URL（若有）、失败摘要、是否已删除 tag；明确未改 STATUS/CHANGELOG。

## 不应做

- 非 GitHub 仓库不走本技能流程。  
- CI 成功前不改 STATUS / CHANGELOG / `LAST_SHIPPED`。  
- 缺 `gh` 时不跳过监视、不把「已 push tag」当成部署完成。  
- 失败时不自动改需求状态；不上机替代 Part B 回滚。  
- 不把 Part B 的打包/切流步骤写进本文件。  
