---
name: code-github-ci
description: >-
  Prefer GitHub Actions workflows with self-hosted runners when newly adding CI
  to GitHub-hosted repos. When editing CI that already uses this pattern, also
  check it against these constraints. Does not ban other CI systems or force
  migration of non-Actions pipelines. Use when adding or editing CI, GitHub
  Actions, workflow, self-hosted runner, or when the user asks for GitHub CI /
  Actions / workflow 规范.
---

# GitHub CI 约束

本技能约束 **GitHub 托管项目**在涉及 CI 时的选型与写法：优先用 GitHub Actions Workflow + self-hosted runner。

- **新增**：选型与写法按本技能。
- **修改**：若目标已是 Actions + self-hosted（或本次改成这种方式），须按本技能检查并保持合法。
- 不禁止其他 CI，也不要求把已有非此方式的流水线迁过来。

不规定开发语言、框架、工程目录或具体构建/测试命令。跟随仓库已有约定；没有约定先问。

## 1. 何时适用

- 适用：仓库在 GitHub 上，且用户要**新增**或**修改** CI / workflow / Actions 相关配置。
- 修改时：仅当改动的流水线采用（或改为）Actions + self-hosted 时，才做本技能的合法性检查；其他 CI 方案不按本技能审计，除非用户明确要求按本技能改。
- 不适用：非 GitHub 托管；只改业务代码、不碰 CI。

## 2. 选型优先（新增）

新增 CI 时按此优先序：

1. **首选**：GitHub Actions（`.github/workflows/`）+ **self-hosted** runner
2. 仓库或用户已明确指定其他方案：跟随该方案，并在回复里点明未用首选的原因
3. 拿不准：先问一条，再动手

不主动推荐 GitHub-hosted runner（如 `ubuntu-latest`）作为新增默认；仅当用户明确要求、或仓库文档写明只能用 hosted 时再用。

## 3. Workflow 写法

新增，以及修改已采用本方式的 workflow，均遵守：

- 文件放在 `.github/workflows/`，YAML；命名跟随仓库已有习惯，没有习惯则用能表达用途的短名。
- Job 的 `runs-on`：**优先** self-hosted。labels 跟随仓库已有 runner / 文档约定；没有约定先问，不臆造 label 集合。
- 触发条件（`on`）、权限（`permissions`）、密钥与变量：跟随仓库已有约定；没有约定先问最小必要项，不擅自放大 `contents: write` 等权限。
- 具体步骤（安装、构建、测试、部署命令）不由本技能规定；用仓库已有脚本或入口，没有则问。

示意（label 与步骤仅为形态，以仓库约定为准）：

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:
jobs:
  check:
    runs-on: [self-hosted]
    steps:
      - uses: actions/checkout@v4
      # 后续步骤用仓库已有入口；无则先问
```

## 4. 修改时的合法性检查

改动的目标是 Actions + self-hosted（或本次改为这种方式）时，改完前对照检查；不合规先修正或问清，再交付：

1. workflow 是否在 `.github/workflows/`，且为有效 YAML
2. 相关 job 的 `runs-on` 是否仍为 self-hosted（未在无说明的情况下改成 GitHub-hosted）
3. labels 是否仍符合仓库约定（未臆造或偷偷换掉既有 label）
4. `permissions` / secrets / 触发范围是否仍为最小必要，未无故放大
5. 步骤是否仍走仓库已有入口，未塞进本技能臆造的命令

只检查**本次改动触及**的 workflow / job；不为「顺便统一」去扫全仓其他流水线。

## 5. 不应做

- 不为「顺便统一」去改或迁移已有非 Actions / 非 self-hosted 流水线。
- 不在未确认时把新增或已有 self-hosted job 改成 GitHub-hosted。
- 不把本技能当成禁止其他 CI 的禁令。
- 修改本方式流水线时，不跳过第 4 节合法性检查。

## 完成时自检

1. 本次是新增还是修改；修改时目标是否为本方式（是则已做合法性检查）  
2. 新增是否优先采用 Actions + self-hosted；若否，是否已说明原因或已获用户确认  
3. `runs-on` labels / 触发 / 权限是否跟随仓库约定或已先问清  
4. 构建与测试步骤是否使用仓库已有入口，而非本技能臆造命令  
5. 是否未擅自迁移或审计其他 CI 方案  
