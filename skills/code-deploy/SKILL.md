---
name: code-deploy
description: >-
  Ships every docs/features requirement whose STATUS.md row is 待部署 as one
  new SemVer, or redeploys the current top version without a bump. Reads
  code-spec-common first. Runs the project's existing deploy entrypoint, then
  requires a passing health check before updating docs/CHANGELOG.md and those
  status rows. Independent of code-deploy-service (the code_deploy product
  itself). Use when the user asks for code-deploy, 部署, deploy, 发版,
  重新部署, or release.
---

# 部署

本技能是 `code-spec-common` 的专用层。只做两件事：发版，或重新部署。不设计方案，不写业务代码，不把别的状态改成 `待部署`。

本技能只约束**通用部署流程**（圈定需求、定版本、跑部署入口、健康检查、写 CHANGELOG / STATUS）。**code-deploy 服务**（应用登记、upload、exec 等）见独立技能 `code-deploy-service`，二者不互相包含，本文件不重复服务侧说明。

开始前先读 `../code-spec-common/SKILL.md`。术语、目录、状态、CHANGELOG 和共同约束以它为准，下文不重复。网络失败先读 `../code-network/SKILL.md`。

用户没有明确说「部署」「deploy」或「重新部署」时，不发布。

## 部署入口

仓库里已有、由文件名、目录名或项目文档标明用于部署的入口。找到就执行，版本按该入口既有方式传入。不规定语言、框架或命令形式。

## 健康检查门禁（写文档前必过）

部署入口**成功之后**、更新 `docs/CHANGELOG.md` / `docs/features/STATUS.md` / `LAST_SHIPPED` **之前**，必须做健康检查。未通过则整次发布失败：不改上述任何文档。

### 如何找到检查地址

1. 优先读项目 `AGENTS.md`（或等价 Agent 规则）里写明的健康检查 URL。
2. 其次读项目文档中的约定。
3. 仍没有：停止并询问，**不得**跳过检查，也不得臆造 URL。

### 判定标准

对健康检查 URL 发 `GET`（无特殊说明则无鉴权），须同时满足：

| 项 | 要求 |
| --- | --- |
| HTTP | 成功（通常 `200`） |
| 正文 | JSON，含 `ok: true` |
| `version` | 字符串，且 **等于** 本次发版/重新部署的目标 SemVer |

任一不满足（连不上、非 JSON、`ok` 不为 true、`version` 缺失或与目标不符）：停止；说明实际状态码与正文摘要；**不**写 CHANGELOG、**不**改 STATUS、**不**更新 `LAST_SHIPPED`、**不**追加 `redeploy`。

### 与「无部署入口」警告的关系

即使因「没有部署入口 / 分不清入口」只发出警告、未执行入口：仍须做健康检查且 `version` 等于目标 SemVer，才可更新文档。检查失败则同样不改文档（避免未上线却改状态）。

## 硬性约束

- 发布顺序固定：部署入口 → **健康检查通过** → 再更新 CHANGELOG / STATUS。
- 没有部署入口：明确警告「仓库中没有部署入口，本次未执行实际部署」，然后仍走健康检查；检查通过才更新文档。
- 有多份且分不清该执行哪一份：不擅自挑选。警告并列出找到的入口，然后仍走健康检查门禁。
- 部署入口执行失败，或健康检查失败：停下。不写 CHANGELOG，不改 `STATUS.md`，不更新 `LAST_SHIPPED`，说明失败原因。

## 先分清口令

| 用户原话 | 走哪条 |
| --- | --- |
| 部署、deploy、发版 | 发版 |
| 重新部署 | 重新部署 |

两个发版口令都没说清时，先问一条：这次是发版，还是用当前顶版重新部署。不要猜测。

## 发版

```
- [ ] 1. 圈定 STATUS.md 中状态为待部署的需求
- [ ] 2. 读这些需求的 SPEC / TECH / 已有 BRD，定一个 SemVer
- [ ] 3. 优先执行部署入口
- [ ] 4. 健康检查：ok 且 version 等于该 SemVer
- [ ] 5. 再按 code-spec-common 更新 CHANGELOG，并把圈定行改为已部署
```

### 1. 圈定

读 `docs/features/STATUS.md`。

- 用户没点名：圈定全部 `待部署` 行。
- 用户点了目录：只圈定点名的那些，且每一行都必须已经是 `待部署`。有任一行不是，停下来说明，不发布。
- 一行都没有：停止。告知没有待发版需求。若只是要把当前顶版再上线一次，请改口令为「重新部署」。
- `已实现` 及其他状态不纳入，也不要改成 `待部署`。回复里可以列出这些目录，标明未纳入。

### 2. 定一个版本号

读圈定需求的 SPEC、TECH，以及已有的 BRD。变更类型以正文里写的为准。相对当前产品版本只升一档：

- 有不兼容的对外行为变化 → major（`1.2.0` → `2.0.0`）
- 否则有新能力且保持兼容 → minor（`1.2.0` → `1.3.0`）
- 否则 → patch（`1.2.0` → `1.2.1`）

同一批里几种变更都有，取更高的那一档。尚未发版时，不自拟第一个版本号，先问用户。

行为算不算不兼容、读不到依据时，先问一条。问清之前不执行入口，也不写 CHANGELOG。此时还不改 `STATUS.md`。

### 3. 优先执行部署入口

- 没有部署入口：警告「仓库中没有部署入口，本次未执行实际部署」，然后进入第 4 步。
- 有多份且分不清：警告并列出这些入口，不执行其中任何一份，然后进入第 4 步。
- 执行失败：停止，不进入第 4 步。

### 4. 健康检查

按上文「健康检查门禁」请求项目约定 URL；目标 SemVer 为第 2 步所定版本。失败则停止。

### 5. 再更新 changelog

健康检查通过后，才做下面三件事：

1. 按 `code-spec-common` 在 `docs/CHANGELOG.md` 顶部写入新版本。日期是这次的年月日。每个圈定需求一条，前缀用该需求正文中的变更类型。
2. 按 `code-spec-common` 把 `LAST_SHIPPED` 改为当前本地时间。
3. 本次圈定行改为 `已部署`。其他行不动。

## 重新部署

```
- [ ] 1. 确认 CHANGELOG 已有顶版
- [ ] 2. 用该顶版执行部署入口
- [ ] 3. 健康检查：ok 且 version 等于该顶版
- [ ] 4. 再按 code-spec-common 更新 LAST_SHIPPED，并可追加 redeploy 记录
```

1. 读 `docs/CHANGELOG.md`。没有顶版：停止，告知尚未发版，需要先发版。
2. 按 `code-spec-common` 的重新部署执行：不升版，不改需求状态，不改已有变更条目。版本号用当前顶版。
3. 部署入口的查找和执行与发版相同。没有入口或分不清入口时警告并进入健康检查。执行失败则停止，不改文档。
4. 健康检查：目标 SemVer 为当前顶版；失败则停止，不改 `LAST_SHIPPED`，不追加 `redeploy`。
5. 健康检查通过后：按 `code-spec-common` 更新 `LAST_SHIPPED`，并可在该顶版区段下追加 `redeploy` 行。**不**改需求状态。

## 完成时输出

发版走到文档更新：

1. 新版本号，以及升档依据（major / minor / patch）
2. 是否执行了部署入口；没有执行时，警告原文要出现在回复里
3. 健康检查 URL 与返回的 `version`
4. `LAST_SHIPPED`
5. 改为 `已部署` 的目录名
6. 写入 CHANGELOG 的条目

重新部署走到文档更新：

1. 顶版号（未升版）
2. 是否执行了部署入口；没有执行时，警告原文要出现在回复里
3. 健康检查 URL 与返回的 `version`
4. `LAST_SHIPPED`
5. 是否追加了 `redeploy` 行

入口失败、健康检查失败或尚未发布就停止时：只说明停因，不改 CHANGELOG 和 `STATUS.md`。
