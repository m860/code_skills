---
name: code-spec-common
description: >-
  Shared upper layer for requirement directories
  docs/features/{timestamp}-{title}/, status in docs/features/STATUS.md, SemVer,
  and docs/CHANGELOG.md. Read this before code-spec-design or code-deploy. Does not
  design a requirement or ship a release.
---

# 需求文档约定

本技能是通用层。只定义需求目录、状态、版本、CHANGELOG 和共同约束。不设计方案，不发布。

专用技能开始前先读本文件，只写自己的流程，不重复下文：

- `code-spec-design`：方案设计；用户明确要求后才实现
- `code-deploy`：发版或重新部署

在含 `docs/` 的产品仓库中使用。下文路径是文档角色，不代表某种开发语言、框架或命令。

## 术语

- **版本** = SemVer（如 `1.2.0`），与需求目录时间戳无关。
- **当前产品版本** = `docs/CHANGELOG.md` 最上一条 `## X.Y.Z`。没有任何 `## X.Y.Z` 区段 = 尚未发版。不以源码里的版本号反推。
- **需求** = `docs/features/{timestamp}-{title}/`。一个目录一个需求。目录名创建后不再改。
- **需求状态**只写在 `docs/features/STATUS.md`，不写进目录名。顺序：

  `方案设计中`（新建默认）→ `方案设计完成` → `已实现` → `待部署` → `已部署`

- **源码版本号**：仓库源码中的版本字段。设计、实现、发版、重新部署都不改。发布版本只出现在发布产物和 CHANGELOG。
- **发版**：圈定状态为 `待部署` 的需求，给出一个新 SemVer，发布上线后写 CHANGELOG，并把这些行改为 `已部署`。
- **重新部署**：用当前顶版再发布上线。不升版，不改需求状态，不改已有变更条目。

谁推进哪一步状态，由专用技能决定。

## 需求目录

目录名 = `{timestamp}-{title}`。

- `timestamp` = 创建时的本地时间 `YYYYMMDDHHmmss`（14 位）。
- `title` = 短标题。中文或英文，词与词只用 `-` 连接。不含空格、状态、`/` 和标点。
- 例：`docs/features/20260101140530-关键词匹配应用清单/`。
- 先对照已有目录和 `STATUS.md`。重名，或短标题拿不准时，先问用户。不另起时间戳，不改已有目录名。
- 引用旧需求时用它实际的文件夹名。

## 状态文件

`docs/features/STATUS.md` 是需求状态的唯一登记处。不存在则创建。

```markdown
# 需求状态

目录名创建后不变。状态只改本表对应行。

| 目录 | 状态 |
| --- | --- |
| 20260101140530-关键词匹配应用清单 | 方案设计中 |
```

- `目录` 列等于文件夹名，不带 `docs/features/`。
- 新需求插在表头下第一行（新的在上）。
- 一行一个需求。状态只允许术语里的五个。
- 不删行，不改其他需求的状态，不把状态写进目录名，不另建第二份清单。

## 文档位置

| 文件 | 作用 |
| --- | --- |
| `docs/features/{timestamp}-{title}/` | 一个需求的方案。状态不在目录名里 |
| `docs/features/STATUS.md` | 状态的唯一登记处 |
| `docs/CHANGELOG.md` | 仅发版、重新部署时写 |
| `docs/DB.md` | 数据模型全量。有结构或字段变化才改 |
| `docs/ARCHITECTURE.md` | 架构级事实。有变化才改对应陈述 |
| `docs/PLANNED-FEATURES.md` | 明确留到以后的能力。一条一个能力 |
| `docs/cases/` | 用例文档。只在实现时按模块增量更新 |

专用技能决定自己这一步改其中哪些。

## CHANGELOG

```markdown
# Changelog

LAST_SHIPPED: YYYYMMDDHHmmss

## X.Y.Z (YYYY-MM-DD)

- feat: …
- fix: …
```

- 新版本在上。`## X.Y.Z (YYYY-MM-DD)` 的日期是该版本首次发版日。
- 条目前缀用需求正文里的变更类型（`feat` / `fix` / `docs` / `chore`）。一条说清对外变化。纯文本，不写仓库路径或文件链接。
- `LAST_SHIPPED` 是最近一次记录的上线时刻（`YYYYMMDDHHmmss`）。
- 同版本多次上线：标题日期保持首次发版日。只在该区段下追加 `- redeploy: YYYYMMDDHHmmss`，不新开版本号。
- 设计与实现阶段不写、不改本文件（含 `LAST_SHIPPED`）。

## 共同约束

- 业务、数据、接口、流程以读到的代码或已有文档为依据。读不到、对不上、有多种解释：先问，禁止补故事。
- 功能实现优先评估仓库已有能力与成熟库，对照本需求后让用户选择；细则见 `code-architecture` 与 `code-spec-design`。
- 不改旧需求目录里的文档，不重命名任何需求目录。
- 不改源码中的版本号。
- 不自动提交。
- 不规定开发语言、框架、运行命令或工程目录。跟随仓库已有约定；没有约定先问，不另造一套。
