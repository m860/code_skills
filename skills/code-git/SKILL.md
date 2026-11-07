---
name: code-git
description: >-
  Git commit constraints for agents: never auto-commit, stage only files
  changed in the current session, Conventional Commits (Commitizen) messages.
  For remote network failures, read code-network first. Use when committing,
  pushing, fetching, or when the user asks for git / 提交 / Conventional
  Commits guidance.
---

# Git 约束

本技能约束 agent 的 git 提交与远程访问行为。涉及提交、推送、拉取或访问远程时遵守下文。

远程因**网络**失败时，先读 `../code-network/SKILL.md`，代理重试以它为准，下文不重复。

不规定开发语言、框架、工程目录或仓库布局。跟随仓库已有约定；没有约定先问。

## 1. 禁止自动提交

- 未经用户明确要求，不执行提交。
- 「改完了」「做完了」「可以推了」不等于要求提交；用户需明确说出提交、commit，或等价口令。
- 用户未要求时，可说明工作区有哪些改动，但不要自行 `git add` / `git commit` / `git push`。

## 2. 只提交当前会话改动的文件

- 提交范围仅限**本会话**内创建或修改的文件。
- 不把工作区里其他未改动、他人改动、或无关脏文件一并纳入。
- 暂存前先核对：`git status` / `git diff`，按路径精确暂存本会话文件，避免 `git add .` / `git add -A` 扫进无关内容。
- 有本会话之外的改动且拿不准是否该带上：先问用户，不擅自纳入或丢弃。

## 3. 提交信息：Conventional Commits（Commitizen）

提交信息遵循 [Conventional Commits](https://www.conventionalcommits.org/)，与 Commitizen 常用约定一致：

```text
<type>(<scope>): <subject>

[optional body]

[optional footer(s)]
```

- `type` 常用：`feat` / `fix` / `docs` / `style` / `refactor` / `perf` / `test` / `chore` / `ci` / `build`。仓库另有约定时跟随仓库。
- `scope` 可选；有明确模块或技能名时再写，不硬凑。
- `subject`：简洁说明意图；不加句号；与仓库既有语言（中/英）保持一致。
- 正文与脚注按需；破坏性变更用 `BREAKING CHANGE:` 或在 `type` 后加 `!`。
- 一条提交只表达一件事；不把无关改动揉进同一条信息。

## 4. 远程与网络

推送、拉取、fetch 或其他访问远程时：

- **网络**失败 → 按 `code-network` 处理。
- 非网络原因（鉴权失败、冲突、拒绝推送等）不走代理重试；按失败原因处理或向用户说明。

## 完成时自检

1. 用户是否明确要求了提交；否则未执行 commit / push  
2. 暂存文件是否仅含本会话改动  
3. 提交信息是否符合 Conventional Commits  
4. 远程网络失败时是否已按 `code-network` 处理  
