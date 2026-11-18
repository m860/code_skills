---
name: code-deploy
description: >-
  Ships every docs/features requirement whose STATUS.md row is 待部署 as one
  new SemVer, or redeploys the current top version without a bump. Also guides
  writing pack/release scripts whose artifacts satisfy the code_deploy tool
  (upload + node-pm2 install/switch/reload). Reads code-spec-common first.
  Use when the user asks for code-deploy, 部署, deploy, 发版, 重新部署, release,
  pack:release, 打包脚本, or release.tgz for code_deploy.
---

# 部署

本技能是 `code-spec-common` 的专用层。覆盖：

1. **发版 / 重新部署**（改 STATUS / CHANGELOG，并跑仓库部署入口）
2. **面向 code_deploy 工具的打包脚本约定**（产物如何被 upload → deploy 消费）

不设计业务方案；不把别的状态擅自改成 `待部署`。

开始前先读 `../code-spec-common/SKILL.md`。术语、目录、状态、CHANGELOG 和共同约束以它为准，下文不重复。网络失败先读 `../code-network/SKILL.md`。

用户没有明确说「部署」「deploy」「重新部署」，也没有要求写/改打包脚本时，不主动发版、不擅自改 pack 脚本。

---

## 面向 code_deploy 工具的打包脚本

当应用由 **code_deploy**（`template: node-pm2`）发布时，仓库应有可重复执行的打包入口（常见名：`pack:release` / `scripts/pack-release.*`）。Agent 编写或修改该脚本时遵守下列契约（与具体业务框架无关；构建命令跟随仓库已有约定）。

### 发布侧怎么消费包

```
本地 pack → .tar / .tar.gz / .tgz
  → HTTP POST /v1/apps/:appId/upload（multipart 字段 file）→ uploadId
  → HTTP 或 MCP deploy({ uploadId, releaseId? })
  → 服务：解压到 root/releases/<id>/
       → 在该目录执行 pnpm install --prod --frozen-lockfile
       → 切 current → pm2 reload <pm2Name>
```

MCP **没有** upload Tool：上传必须走 HTTP；再 MCP/HTTP deploy。

### 打包脚本必须做到

| 项 | 要求 |
| --- | --- |
| 版本写入 | **仅**写入发布产物内的清单（如产物 `package.json` 的 `version`）。**禁止**改仓库源码里的版本字段。 |
| 版本来源 | 优先：环境变量 `VERSION` → CLI `--version`/`-v` → `docs/CHANGELOG.md` 最上 `## X.Y.Z`。缺失则失败并说明，不臆造。 |
| 产物目录 | 组装一个可运行的扁平目录（下文称 `release/`）：可启动入口、生产依赖声明、锁文件、示例配置等。 |
| **锁文件** | 必须带上与安装器匹配的 lock（node-pm2 当前为 **`pnpm-lock.yaml`**），否则服务端 `--frozen-lockfile` 会失败。 |
| **不要**打进包 | `node_modules`（及同等依赖缓存）。安装在部署机完成。 |
| 压缩包形态 | 生成 `.tar` / `.tar.gz` / `.tgz`。**归档成员应是扁平根**（`dist/`、`package.json`… 在包根），**不要**再包一层 `release/` 目录，以便解压到 `releases/<id>/` 后布局正确。 |
| 文件名 | 建议 `{SemVer}-{YYYYMMDDHHmmss}.tgz`（或等价），便于未传 `releaseId` 时从文件名推导。也可用显式 `releaseId`。 |
| 进程名 | 若生成 PM2 配置，其中的 name 须与 apps 登记里的 `pm2Name` 一致。 |
| 可重复 | 脚本可直接执行；注明最小依赖与入参（`VERSION` / `--version`）。 |

### 建议步骤（实现模板）

```
1. 解析 SemVer（VERSION / --version / CHANGELOG；禁止改仓库源码 version）
2. 按仓库既有方式完成生产构建
3. 清空并组装 release/（扁平：启动文件、清单、lock、.env.example、可选 ecosystem）
4. 断言关键入口与 lock 存在
5. tar：在 release/ 目录内打包当前目录内容（等价于 tar -czf out.tgz -C release .）
6. 打印产物路径；提醒：upload → deploy(uploadId)；依赖由 code_deploy 安装
```

### 与「发版」口令的关系

- 用户说「部署 / 发版」：按下文发版流程；部署入口常即上述打包脚本，版本用本技能算出的 SemVer 传入（如 `VERSION=x.y.z`）。
- 用户只说「写打包脚本 / pack:release」：只改/新增打包脚本与必要说明，**不**改 STATUS、**不**写 CHANGELOG，除非同时要求发版。

### 应用登记（运维侧，脚本可在 README 提示）

```json
{
  "my-app": {
    "root": "/path/to/app",
    "key": "<secret>",
    "template": "node-pm2",
    "pm2Name": "my-app"
  }
}
```

`root` 可由 code_deploy 在加载时自动创建；改 apps 文件可热加载，无需为加应用而重启（以服务当前能力为准）。

---

## 部署入口（发版 / 重新部署时执行）

仓库里已有、由文件名、目录名或项目文档标明用于部署的入口。找到就执行，版本按该入口既有方式传入。

若入口即面向 code_deploy 的打包脚本：产出须满足上一节契约。

## 硬性约束

- 发布顺序固定：先执行部署入口，再更新 CHANGELOG。
- 没有部署入口：不停下。明确警告「仓库中没有部署入口，本次未执行实际部署」，然后继续更新文档。
- 有多份且分不清该执行哪一份：不擅自挑选。警告并列出找到的入口，然后继续更新文档。
- 执行失败：停下。不写 CHANGELOG，不改 `STATUS.md`，说明失败原因。

## 先分清口令

| 用户原话 | 走哪条 |
| --- | --- |
| 部署、deploy、发版 | 发版 |
| 重新部署 | 重新部署 |
| 打包脚本、pack:release、release.tgz（未要求发版） | 只写/改打包脚本（见上文） |

「部署」与「重新部署」都没说清时，先问一条：这次是发版，还是用当前顶版重新部署。不要猜测。

## 发版

```
- [ ] 1. 圈定 STATUS.md 中状态为待部署的需求
- [ ] 2. 读这些需求的 SPEC / TECH / 已有 BRD，定一个 SemVer
- [ ] 3. 优先执行部署入口
- [ ] 4. 再按 code-spec-common 更新 CHANGELOG，并把圈定行改为已部署
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

### 4. 再更新 changelog

入口成功，或已经给出「没有入口 / 分不清入口」的警告之后，才做下面三件事：

1. 按 `code-spec-common` 在 `docs/CHANGELOG.md` 顶部写入新版本。日期是这次的年月日。每个圈定需求一条，前缀用该需求正文中的变更类型。
2. 按 `code-spec-common` 把 `LAST_SHIPPED` 改为当前本地时间。没有执行成功的部署入口时仍写入，并在回复中重复警告：文档已更新，但没有执行实际部署。
3. 本次圈定行改为 `已部署`。其他行不动。

## 重新部署

```
- [ ] 1. 确认 CHANGELOG 已有顶版
- [ ] 2. 用该顶版执行部署入口
- [ ] 3. 再按 code-spec-common 更新 LAST_SHIPPED，并可追加 redeploy 记录
```

1. 读 `docs/CHANGELOG.md`。没有顶版：停止，告知尚未发版，需要先发版。
2. 按 `code-spec-common` 的重新部署执行：不升版，不改需求状态，不改已有变更条目。版本号用当前顶版。
3. 部署入口的查找和执行与发版相同。没有入口或分不清入口时警告并继续。执行失败则停止，不改 `LAST_SHIPPED`，不追加 `redeploy` 行。
4. 进入文档更新时：按 `code-spec-common` 更新 `LAST_SHIPPED`，并可在该顶版区段下追加 `redeploy` 行。没有执行成功的部署入口时，同样警告文档已更新、但没有执行实际部署。

## 完成时输出

发版走到文档更新：

1. 新版本号，以及升档依据（major / minor / patch）
2. 是否执行了部署入口；没有执行时，警告原文要出现在回复里
3. `LAST_SHIPPED`
4. 改为 `已部署` 的目录名
5. 写入 CHANGELOG 的条目

重新部署走到文档更新：

1. 顶版号（未升版）
2. 是否执行了部署入口；没有执行时，警告原文要出现在回复里
3. `LAST_SHIPPED`
4. 是否追加了 `redeploy` 行

只改打包脚本时：列出改动的脚本路径与是否满足 code_deploy 扁平包 / lock / 无 node_modules 等要点。

入口失败或尚未发布就停止时：只说明停因，不改 CHANGELOG 和 `STATUS.md`。
