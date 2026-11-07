---
name: code-skills
description: >-
  Project-level agent skill lifecycle via the npm skills CLI (npx skills):
  add, list, find, check, update, remove. Prefer project scope, never hand-copy
  into agent skill dirs or edit skills-lock.json by hand. Use when installing,
  updating, removing, or listing skills, or when the user asks for 技能管理 /
  npx skills / skills.sh.
---

# 项目技能管理

本技能约束**业务项目**里 agent 技能的安装与维护。增删改查一律走 npm 包 [`skills`](https://www.npmjs.com/package/skills) 的 CLI（`npx skills`），不手搓目录。

本仓库内**编写**新技能（新建 `skills/code-*`、改 `SKILL.md`）不在本技能范围，遵循仓库 `AGENTS.md`。

不规定业务项目的开发语言、框架或工程目录布局。跟随项目已有约定；没有约定先问。

## 1. 唯一入口：`npx skills`

- 安装、列出、查找、检查更新、更新、移除：只用 `npx skills …`。
- 禁止：把技能文件手工复制/软链进各 agent 的 skills 目录；禁止绕过 CLI 改安装产物。
- 禁止：手改 `skills-lock.json`（或 CLI 生成的等价锁文件）；变更只通过 CLI。
- 不把 CLI 全局装进系统当默认前提；用 `npx skills` 即可。仓库另有约定时跟随。

## 2. 作用域

- **默认项目级**（不加 `-g`），技能随仓库共享给团队。
- 仅当用户明确要求全局（跨项目）时才加 `-g` / `--global`。
- 目标 agent：用户点名或项目已约定则按约定传 `-a`；拿不准先问，不擅自 `--all` 铺开。

## 3. 常用流程

先核对现状，再改动：

```bash
npx skills list
```

| 意图 | 命令形态 |
| --- | --- |
| 查找 | `npx skills find [query]` |
| 安装 | `npx skills add <source> [--skill <name>…] [-a <agent>…]` |
| 检查更新 | `npx skills check` |
| 更新 | `npx skills update [skills…]`（项目内可加 `-p`） |
| 移除 | `npx skills remove [skills…]` |

- 非交互场景（CI 或用户要求免确认）再加 `-y`；日常先让用户看清要装/要删的内容。
- `<source>` 用 CLI 支持的形式（如 `owner/repo`、git/HTTPS URL、本地路径）。项目锁文件里已有来源时，优先与之保持一致。
- 安装本技能包中的 `code-*` 技能时，默认源为 `m860/code_skills`；项目已锁定其他源或用户另有指定时，以项目/用户为准。

## 4. 网络失败

`npx skills` 因**网络**失败时：同一条命令重试一次，且**仅本次**设置

- `http_proxy=http://127.0.0.1:7890`
- `https_proxy=http://127.0.0.1:7890`

不改 npm / git 的持久代理配置。非网络错误不走代理重试。更细的 git 远程约定见 `code-git`。

## 5. 不应做

- 不把技能管理做成「下载 zip 解压进 `.cursor/skills`」之类手工流程。
- 不在未确认时对全部 agent 执行 `--all` 或 `remove --all`。
- 不把「编写本技能仓库里的新技能」误当成要用 `npx skills init` 替代仓库约定；消费端项目若要脚手架，才考虑 `npx skills init`，且仍须符合命名等已有约定。

## 完成时自检

1. 是否只用 `npx skills` 完成变更，无手改安装目录或锁文件  
2. 作用域是否为项目级（除非用户要求全局）  
3. 安装源是否与项目锁文件 / 用户指定一致（`code-*` 默认 `m860/code_skills`）  
4. 网络失败时是否仅一次性代理重试  
