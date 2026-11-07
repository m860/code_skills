---
name: code-network
description: >-
  Shared handling when outbound network access fails: one-shot
  http_proxy/https_proxy retries on 127.0.0.1:7890 then candidate 7897,
  without changing persistent git/npm/proxy config. Read from code-git,
  code-skills, or any skill that hits the network. Use on network errors,
  proxy retry, 代理, or 7890/7897.
---

# 网络失败处理

本技能是通用层。凡访问远程（git、`npx skills`、下载、API 等）因**网络**失败时，按下文处理。专用技能（如 `code-git`、`code-skills`）先读本文件，只写自己的流程，不重复代理细则。

不规定开发语言、框架、工程目录或具体业务命令。

## 1. 何时适用

- 适用：超时、连接拒绝、DNS/TLS 握手失败、无法到达主机等**网络类**错误。
- 不适用：鉴权失败、权限不足、冲突、远程拒绝、参数错误、业务 4xx（非网络）等。此类按原错误处理或向用户说明，不走代理重试。

## 2. 重试步骤

对**同一条失败命令**，按端口顺序各重试一次。**仅本次命令**设置环境变量，且 `http_proxy` 与 `https_proxy` 同值：

1. 优先：`http://127.0.0.1:7890`
2. 候选：`http://127.0.0.1:7897`（上一端口仍因网络失败时再用）

两个端口都仍因网络失败：停止代理重试，向用户说明已尝试的端口与错误摘要。

## 3. 禁止改持久配置

- 不修改 git 的 `http.proxy` / `https.proxy`，不改全局或仓库 `.gitconfig`。
- 不修改 npm / 包管理器的持久代理配置，不改用户级 shell 配置里的代理导出。
- 只允许命令前一次性环境变量（或等价的单次调用包装）。

## 完成时自检

1. 是否确认为网络类失败（非鉴权/冲突等）  
2. 是否按 7890 → 7897 各重试一次，且仅本次命令带代理  
3. 是否未改任何持久代理 / git / npm 配置  
