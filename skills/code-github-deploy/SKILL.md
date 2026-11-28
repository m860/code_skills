---
name: code-github-deploy
description: >-
  GitHub-hosted release orchestration (Part A): scope 待部署 rows, choose SemVer,
  create annotated vX.Y.Z tag whose message carries release/redeploy kind and
  changelog lines (with feature directory names), push the tag, then stop.
  Does not watch CI or write CHANGELOG/STATUS — that is Part B
  (code-github-ci ship-docs / delete-failed-tag). Use when the user asks for
  code-github-deploy, 部署, deploy, 发版, 重新部署, or release on a
  GitHub-hosted repo.
---

# GitHub 部署（Part A）

本技能是 `code-spec-common` 的专用层，**仅适用于 GitHub 托管**仓库。只做发版或重新部署的 **Agent 侧编排**：圈定 → 定版 → 打并 push **annotated** `v{SemVer}`（annotation 含版本日志与需求目录名）→ **停止**。

机器上的发布、健康检查、成功后回写 STATUS/CHANGELOG、失败删 tag，见 `../code-github-ci/SKILL.md`（Part B）。本文件不重复 Part B 步骤，也不代替用户选择托管方式。

开始前先读 `../code-spec-common/SKILL.md`。术语、目录、状态、CHANGELOG 以它为准。网络失败先读 `../code-network/SKILL.md`。提交规范读 `../code-git/SKILL.md`（本技能的部署口令视为已明确要求提交与推送**发版相关**变更；**不含** CI 成功后的文档回写——那由 Part B 完成）。

用户没有明确说「部署」「deploy」「发版」或「重新部署」时，不发布。

## 适用范围

- **适用**：`origin`（或等价远程）为 GitHub；用户要发版或重新部署。
- **不适用**：非 GitHub 托管。停止并说明；不改 STATUS / CHANGELOG，不打 tag。

## 与 Part B 的衔接

```
Part A（本技能）                    Part B（code-github-ci）
圈定 / 定版
  → annotated tag（含日志）
  → push tag
  → 停止（不强制 watch）     →  构建/部署/健康检查
                               →  成功：ship-docs 写 STATUS/CHANGELOG 并 push
                               →  失败：回滚目标机 + delete-failed-tag
```

- Tag 名：**始终** `v{X.Y.Z}`（与 SemVer 对应，带 `v` 前缀）。不使用 `vX.Y.Z-时间戳`。
- Tag **必须**为 annotated；轻量 tag 不足以承载版本日志，CI 无法回写。
- 版本号传入 CI：以 tag 名为准；日志与需求清单以 annotation 为准。
- **STATUS / CHANGELOG / LAST_SHIPPED 由 Part B 在健康检查成功后回写。** Part A 在打 tag **之前与之后都不改**这些文件。
- `gh run watch` **可选**（仅人工想盯进度时）；**不是**完成本技能的必要条件，也不得据此本地改文档。

## 先分清口令

| 用户原话 | 走哪条 |
| --- | --- |
| 部署、deploy、发版 | 发版 |
| 重新部署 | 重新部署 |

两个口令都没说清时，先问一条。不要猜测。

## Annotated tag 消息体（版本日志）

打 tag 时用 `-a`（或等价）写入下列正文。这是 Part B `ship-docs` 与可选出站通知的真源。

### 正式发版（`kind: release`）

```text
kind: release
version: {SemVer}

- {type}: {短标题} | {需求目录名}
```

- 每个圈定需求一行；可多行。
- `{type}` = 该需求正文中的变更类型（`feat` / `fix` / `docs` / `chore` 等）。
- `{短标题}` = 写入 CHANGELOG 的对外说明（纯文本，不含目录名）。
- `{需求目录名}` = `docs/features/STATUS.md`「目录」列全文 = `docs/features/` 下文件夹名，**一字不差**。CI **仅**按目录名精确匹配改为 `已部署`；禁止指望短标题模糊匹配。
- 解析约定：以最后一个 ` | ` 分隔；右侧为目录名，左侧为 `- {type}: {短标题}`。

### 重新部署（`kind: redeploy`）

```text
kind: redeploy
version: {SemVer}
```

- 不列功能行；不改需求状态。

## 发版

```
- [ ] 1. 确认 GitHub 托管
- [ ] 2. 圈定 STATUS.md 中待部署需求；定一个 SemVer
- [ ] 3. 打 annotated tag vX.Y.Z（消息体含 kind/release 行与目录名）并 push
- [ ] 4. 报告 tag / 预计触发的 workflow；停止（不改 STATUS/CHANGELOG；不强制 watch）
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

行为算不算不兼容、读不到依据时，先问一条。问清之前不打 tag。

### 3. 打 annotated tag 并 push

- Tag：`v{SemVer}`，打在**当前待发布提交**（一般为当前 `HEAD`；以仓库约定为准）。
- 消息体：按上文「正式发版」模板；**每一行都必须带 `| {需求目录名}`**。缺目录名则停下来补全，不推残缺 annotation。
- 若本地或远程已存在同名 tag：先确认是否为上次失败残留。是则删除后再打；若对应一次**已成功**发版且用户要的是新版本，停下来说明，不要覆盖。
- `git push` 该 tag 到 GitHub。push 失败：停止，不改文档。

### 4. 完成后停止

- 向用户报告：SemVer、升档依据、tag 名与 commit、圈定的目录列表、annotation 摘要、（若能解析）Actions 入口或「已触发 tag 部署」。
- **不要** `gh run watch` 作为必做步骤；**不要**本地改 STATUS / CHANGELOG / `LAST_SHIPPED`；**不要**在成功或失败后删 tag（失败清理由 Part B `delete-failed-tag`）。
- 用户明确要求「盯到结束」时，可用 `gh` 只读查看进度；仍以 Part B 回写结果为准，Part A 不写文档。

## 重新部署

```
- [ ] 1. 确认 GitHub 托管；CHANGELOG 已有顶版
- [ ] 2. 不升版；删除远程/本地 v{顶版} 后在目标提交上重建并 push 同名 annotated tag（kind: redeploy）
- [ ] 3. 报告并停止（不改需求状态；不强制 watch）
```

1. 没有顶版：停止，告知需先发版。
2. 版本号用当前顶版。为再次触发 Part B：**删除**已有 `v{顶版}` 后，在目标提交（默认当前 `HEAD`）上重建同名 **annotated** tag（消息体为 `kind: redeploy`）并 push。这是 CI 扳机，不是升版。
3. 停止；文档由 Part B 在成功后只更新 `LAST_SHIPPED`（可追加 `redeploy` 行）。

## 完成时输出

发版（Part A 完成 = tag 已 push）：

1. 新版本号与升档依据  
2. tag 名与 commit  
3. annotation 中的功能行摘要（含目录名）  
4. 明确：STATUS / CHANGELOG 将由 CI 在部署成功后回写；本侧未改文档  

重新部署（Part A 完成 = 同名 tag 已重打并推送）：

1. 顶版号（未升版）  
2. 同名 annotated tag 已推送（`kind: redeploy`）  
3. 明确：需求状态不会因本次改写；`LAST_SHIPPED` 由 CI 更新  

## 不应做

- 非 GitHub 仓库不走本技能流程。  
- 不打轻量 tag；不推缺少目录名的 release annotation。  
- 不在 Part A 改 STATUS / CHANGELOG / `LAST_SHIPPED`（成功前、成功后都不改）。  
- 不以「已 push tag」假装线上已健康；也不以「未 watch」阻塞发版编排完成。  
- 不强制 `gh run watch`；失败删 tag、文档回写不由 Part A 执行。  
- 不上机替代 Part B 回滚。  
- 不把 Part B 的打包/切流/ship-docs 步骤写进本文件。  
