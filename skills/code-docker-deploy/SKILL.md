---
name: code-docker-deploy
description: >-
  Stack-agnostic Docker Compose runtime deploy constraints: build/push off the
  target host, target only pull + Compose up, SemVer image cutover/rollback,
  host env_file and volumes, keep last 3 versions, no git ref ops on the target.
  Image/Compose packaging is code-docker-pack; GitHub Actions wiring is
  code-github-ci. Use when deploying with Docker/Compose, writing deploy or
  rollback scripts, or when the user asks for code-docker-deploy, Docker 部署,
  compose pull, or 镜像回滚.
---

# Docker / Compose 部署

本技能约定 **Docker Compose 运行时部署**的通用约束：目标机如何切流、回滚与保留版本。不绑定某家 CI 主机或具体业务框架。

- **镜像与 Compose 文件怎么写**：`../code-docker-pack/SKILL.md`
- **GitHub Actions 触发、runner、ship-docs**：`../code-github-ci/SKILL.md`
- **Agent 打 tag 发版编排**：`../code-github-deploy/SKILL.md`

不规定部署根路径、registry 名或脚本文件名；跟随仓库 / `AGENTS.md`；没有约定先问。

用户未要求部署或改 Docker 部署脚本/约定时，不擅自改目标机流程。

## 1. 核心原则

1. **构建与运行分离**：镜像在构建机（CI 托管 runner 或开发机等）`build` + `push`；**目标机只 `pull` + Compose 启动**。  
2. **禁止**把「目标机现场 `docker build`」当作正式发版主路径（紧急抢修且用户明确要求除外，并须说明未走正式路径）。  
3. **版本真源**是镜像的 **SemVer tag**（如 `{registry}/{name}:{SemVer}`）。回滚与保留版本不依赖 `latest`。  
4. **密钥与运行配置**在宿主：`.env` / `env_file` / 密钥注入与 volume；不进镜像层、不进已提交的生产密钥。  
5. **目标机不做 git ref 写操作**（不在部署机上 push 文档、删远程 tag）；文档与失败 tag 清理由 CI 托管侧完成（见 `code-github-ci`）。

## 2. 部署目录（概念布局）

仓库已有则跟随。默认概念：

```text
<deploy-root>/
  docker-compose.yml   # 或部署时从发版产物拷入/渲染；image 钉到 {SemVer}
  .env                 # 宿主密钥与运行配置（不进 git / 不进镜像）
  data/                # 可选：持久数据 volume 源
  logs/                # 可选
  .deploy/             # 可选：当前版本、历史、渲染后的 image 钉扎等
    current            # 当前成功 SemVer
    history            # 可选
```

- 首次 Docker 发版前：宿主须已有可用运行配置（如 `.env`）；旧进程（如曾用 PM2）应停掉以免占端口。  
- Compose 中服务 `image:` 必须能解析到本次要上线的 `{image}:{SemVer}`（变量替换或渲染文件均可）。

## 3. 切流顺序

```
1. 登录 registry（若需要；凭证来自宿主或 CI Secret，不写进已提交 YAML 明文）
2. 更新/渲染 compose（或等价），使服务指向 {image}:{SemVer}
3. 记录上一成功版本（供回滚），再 docker compose pull
4. docker compose up -d（或仓库等价）
5. 健康检查（第 5 节）
6. 成功：写入当前版本标记；按第 6 节保留最多 3 个 SemVer
7. 失败：按第 7 节回滚，然后判定本次部署失败
```

脚本入口跟随仓库（如 `docker-deploy.sh` / `docker-rollback.sh` / `docker-prune-images.sh`）；没有则先问再新增，不平行造第二套。

## 4. 与打包的关系

- 构建须满足 `code-docker-pack`（SemVer tag、无密钥进镜像、默认可 Compose 启动）。  
- 部署侧**信任**已 push 的 `{image}:{SemVer}`；不在目标机重新解释 Dockerfile 作为主路径。  
- 为加快目标机 `pull`：打包侧宜拆分稳定层与常变层、必要时固定可复现 mtime（见 `code-docker-pack`）；部署侧仍只做标准 `compose pull`，**不**自研差分包合并协议。

## 5. 健康检查

- **有健康检查 URL**（项目文档 / `AGENTS.md`）：`GET` 后 JSON 含 `ok: true`，且 `version` 等于本次 SemVer；不满足则失败并走回滚。  
- **无 URL**：不得臆造。改为观察窗（时长跟随仓库；没有约定先问，问不清约 60s）：容器 `running` / `healthy`（若有 healthcheck）、无持续重启或崩溃日志；否则失败。

## 6. 保留版本

- 成功后目标机本地：**最多保留 3 个**已成功发布的 SemVer 镜像（及必要的 compose 钉扎记录）。  
- 清理更旧镜像/悬空层的方式跟随仓库；没有约定先问，避免误删共享基础镜像。  
- 回滚目标 = **上一成功 SemVer** 的 image + compose，不是 `latest`。

## 7. 失败与回滚

| 失败点 | 处理 |
| --- | --- |
| 尚未 `up` / 未切到新版本 | 失败退出；运行中旧服务保持 |
| 已 `up` 或健康检查失败 | Compose 指回上一成功 SemVer 并 `up -d`（或仓库等价回滚脚本），再标记失败 |
| 回滚自身失败 | 失败；summary/日志标明状态不明，需人工检查 |

回滚成功后：清理本次失败产物（未采用的新镜像引用等），避免占满 3 槽。

## 8. 出站通知（可选）

- 若仓库有上线通知：在健康检查**成功之后**发送；失败应**软忽略**，不触发回滚。  
- 读发版说明时若依赖 git annotated tag：须能拿到完整 annotation（注意 checkout 可能剥 annotation；force-fetch tag 再读）。具体通道跟随仓库。

## 9. 不应做

- 不以目标机 `docker build` 作为正式发版主路径。  
- 不在目标机上 push 文档或删除远程 git tag。  
- 不依赖 `latest` 做回滚或「当前版本」判定。  
- 不把生产密钥写入镜像或已提交的 compose。  
- 不跳过健康检查与「最多 3 版」。  
- 不在本技能重复 GitHub Part A / `ship-docs` 细节。

## 完成时自检

1. 目标机是否仅为 pull + Compose；构建是否在约定构建机完成并已 push  
2. 切流是否钉 SemVer；失败能否回到上一成功 SemVer  
3. 宿主配置/卷是否与镜像分离；无密钥进镜像  
4. 成功后是否最多保留 3 版  
5. 目标机是否未做 git ref 写操作  
