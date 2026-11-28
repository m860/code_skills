---
name: code-docker-pack
description: >-
  Constraints for Docker images and Compose-oriented release artifacts: SemVer
  image tags, no secrets in image or compose committed files, runtime env via
  host/env_file, and reproducible Dockerfile/compose. Use when the user asks for
  code-docker-pack, Dockerfile, docker compose, image build, 镜像打包, or Docker
  packaging for deploy.
---

# Docker / Compose 打包

约定业务仓如何写 **Docker 镜像与 Compose** 相关文件，使产物适合目标机 `pull` 后用 **Compose** 启动。

本技能**只**管镜像/Compose 产物与构建脚本约束。不写目标机切流/回滚步骤、不改需求状态、不写 CHANGELOG，除非用户同时要求发版。

- 目标机部署约束：`../code-docker-deploy/SKILL.md`
- GitHub Actions 构建位置与 workflow：`../code-github-ci/SKILL.md`
- PM2 / tgz 打包：`../code-node-pack/SKILL.md`

不规定具体应用框架或仓库目录名；跟随目标仓库已有约定。没有约定先问。

用户未要求写/改 Dockerfile、compose 或镜像构建脚本时，不擅自新增或大改。

## 产物与版本

| 项 | 要求 |
| --- | --- |
| 版本写入 | 版本体现在**镜像 tag**（及可选 OCI label）。**禁止**为发版去改仓库源码里的业务 version 字段（与 `code-node-pack` 同源约定）。 |
| 版本来源 | 优先：调用方传入（如 CI 从 `vX.Y.Z` tag 解析）→ `VERSION` → CLI → `docs/CHANGELOG.md` 最上 `## X.Y.Z`。皆无则失败说明，不臆造。 |
| 镜像 tag | 至少打上与 SemVer 对应的 tag（如 `{registry}/{name}:{SemVer}`）。可另打 `latest` 仅当仓库已有此习惯；回滚与保留版本以 **SemVer tag** 为准，不依赖 `latest`。 |
| Dockerfile | 仓库内可重复构建；多阶段构建跟随仓库；**不要**把真实 `.env`、密钥、私钥 `COPY` 进镜像。 |
| Compose | **默认**提供 `docker-compose.yml`（或仓库等价名，如 `compose.yaml`），供目标机 `pull` 后启动。服务镜像引用带版本的 tag（或由部署时替换的变量），不写死无法对应发版的随意名。 |
| 环境变量 | 运行时配置用宿主机 `env_file` / Compose `environment` 引用宿主或密钥注入；**禁止**把生产密钥写进已提交的 compose / Dockerfile。最多提交 `.env.example`。 |
| 可重复 | 构建命令/脚本可直接执行；注明 registry、镜像名、入参（版本）。 |

### 建议构建步骤（脚本或文档中写清）

```
1. 解析 SemVer（禁止改仓库源码 version）
2. docker build（或仓库既有 buildx 入口），tag 为 {image}:{SemVer}
3. docker push 到约定 registry（部署路径需要目标机 pull 时必须 push）
4. 打印镜像引用
```

本地仅验证构建、不进入发版流水线时，可不 push；正式发版路径必须能被目标机 pull 到。

## Compose 约定

| 项 | 要求 |
| --- | --- |
| 默认编排 | **Docker Compose**（不是默认裸 `docker run`；仓库明确单容器且无 compose 时再例外，并先问）。 |
| 镜像 | 服务 `image:` 指向将 pull 的引用；构建上下文若在 compose 里，发版仍以「已推送的版本 tag」为准，避免目标机现场 `build` 当作主路径。 |
| 数据与配置 | 持久数据用 volume；配置与密钥不进镜像层。 |
| 健康检查 | 可在 compose 写 `healthcheck`；发版门禁判定见 `code-docker-deploy` /（GitHub 时）`code-github-ci`。 |

## 层与 pull 友好（构建侧）

目标机用标准 `compose pull` 按层复用；构建侧宜让「少变的大层」与「常变的应用层」分开，避免日常只改业务也重拉整棵应用树。

| 项 | 要求 |
| --- | --- |
| 拆层 | runtime 阶段按「稳 → 变」分层 `COPY`（或等价）：大且稳的依赖目录与常变产物（如构建输出、带 SemVer 的清单）分开；含版本号的文件靠后。 |
| 可复现 digest | 同内容应尽量得到同层 digest。若安装/打包会写入易变 mtime，在进入最终镜像层之前对将拷贝的树做**统一固定 mtime**（含需覆盖的 symlink 策略跟随工具能力）；内容变则 digest 仍应变。 |
| 不做 | 不自研「差分包 + 目标机合并」替代标准镜像 tag；不把网络加速/私有 registry 拓扑写死进本技能。 |

具体 Dockerfile 阶段名与路径跟随仓库；没有约定先问。

## 口令边界

- 用户只说「写 Dockerfile / compose / code-docker-pack / 镜像打包」：只改/新增相关文件与必要说明，**不**改 STATUS、**不**写 CHANGELOG，除非同时要求发版。

## 完成时输出

只改 Docker 打包相关文件时：列出改动路径，并核对：镜像 tag 含 SemVer、无密钥进镜像/已提交 compose、有 Compose 默认编排、版本未写入源码 version 字段；若改 runtime 层，是否利于目标机按层复用（稳/变拆分，必要时固定 mtime）。
