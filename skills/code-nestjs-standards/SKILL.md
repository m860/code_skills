---
name: code-nestjs-standards
description: >-
  NestJS + TypeScript + Node.js ≥20 backend coding standards for agents.
  Reads code-architecture first for business-domain boundaries and reuse rules;
  this skill only adds NestJS stack conventions. Covers feature-module layout by
  business domain (not query/ingress-style technical facades), typing,
  controllers/providers boundaries, DTO/validation, errors, config, logging,
  testing, naming/DI, PNPM, and security basics. Use when writing or reviewing
  NestJS backend code, or when the user asks for NestJS standards, Nest 规范,
  domain modules, or Node/TS Nest coding conventions.
---

# NestJS 后端编码规范

本技能约束 **NestJS + TypeScript + Node.js ≥20** 后端实现。写或改 NestJS 代码前读本文件并遵守。仓库已有更严约定时跟随仓库；本文件补齐缺口，不另造第二套风格。

先读通用 `../code-architecture/SKILL.md`。业务域定义、复用、边界、分层、抽取时机等架构共性以它为准；本技能只写 NestJS 栈约定，不重复那些内容。

共性流程（需求目录、状态、发版）不在本技能内；有则先读对应 `code-spec-*` / `code-deploy`。

## 技术栈假设

- 框架：NestJS（官方模块与 DI 模型）
- 语言：TypeScript（严格模式）
- 运行时：Node.js **20+**（不兼容、不迁就更旧 Node）
- 包管理器：**PNPM 唯一**。安装、脚本、依赖增删一律用 `pnpm`；锁文件为 `pnpm-lock.yaml`。禁止使用 npm 或 yarn；禁止生成或提交 `package-lock.json`、`yarn.lock`。构建链其他部分跟随仓库；没有约定先问，但不得改用其他包管理器

## 1. 项目与模块布局（业务域 Feature Module）

Domain 判据见 `../code-architecture/SKILL.md` §1。Nest 侧落实为：

**一个业务域 = 一个（或一组紧密相关的）Nest feature module**，目录与模块名用业务语言。

### 允许的顶层目录

```
src/
  main.ts
  app.module.ts
  {domain}/                      # 业务域，如 project / event / analysis / notification / audit
    {domain}.module.ts
    {domain}.controller.ts       # 本域对外 HTTP（含管理面读接口）；可有多个 controller
    {domain}.service.ts
    dto/
    *.entity.ts                  # 仅本域表
    adapters/                    # 可选：本域专用外部适配（CLI、SDK、网关客户端）
      {adapter}/
        {adapter}.module.ts      # 通常只被本域 module import，不在 AppModule 平铺
  common/                        # 跨域横切：守卫、过滤器、管道、无业务语义工具
  config/                        # 配置与校验
```

### Nest 落位规则

| 东西 | 放哪 |
| --- | --- |
| 资源的 CRUD / 本域规则 / 本域实体 | `{domain}/` |
| 管理面「查某资源列表/详情」 | **该资源所属域**的 controller（禁止单独 `query` module） |
| Webhook / 入站适配 | **接收该业务事实的域**（如错误入站 → `event`），可 `ingress.controller.ts` 作文件名，但模块仍是业务域 |
| 多步编排（收事件 → 分析 → 通知） | 主责用例所在域（如 `analysis` 内 pipeline）；不要顶层空壳 `pipeline` 域 |
| OpenCode / Git / 第三方 LLM SDK | 使用方域下 `adapters/`（如 `analysis/opencode`），**不要**与业务域在 `AppModule` 平级并列成「技术域」 |
| 审计、鉴权工具 | `audit` 域或 `common/`（无业务规则的工具才进 common） |

### 禁止的顶层 module 名（典型坏味道）

- `query` / `read` / `bff` — 技术读门面  
- `ingress` / `egress`（单独成域时）— 传输角色；应并入业务域  
- `pipeline` / `orchestrator`（无自有模型、只 forward）— 编排壳  
- `opencode` / `git` / `llm`（与业务域平级）— 适配器误升格  

发现上述目录时：合并回业务域或降为 `adapters/`，而不是继续加文件。

### 其他约定

- 一个 feature module 只 `exports` 外域需要的 provider；默认不导出。
- 跨域复用放 `common/` 或独立 shared module；禁止把业务规则塞进 `common/`。
- `AppModule` 只组装：根配置、基础设施、**业务域** feature module；不写业务逻辑；不把适配器 module 与业务域平铺一长串（适配器由所属域 module import）。
- 禁止循环依赖：优先下沉共享数据访问、事件或拆模块；`forwardRef` 仅权宜。不要用新建 `query` 模块来「躲开」环。

## 2. TypeScript 严格性

- 开启并保持严格相关选项（`strict`、`noImplicitAny`、`strictNullChecks` 等）。不为图省事关闭。
- 禁止 `any`；确需逃逸用 `unknown` 并收窄。禁止无意义的 `as` 断言；断言前须有类型守卫或校验。
- 公共 API（controller 入参/出参、跨模块导出类型）必须显式类型；不靠上下文推断糊弄边界。
- 优先 `interface` / `type` 描述契约；实体与 DTO 分离，不把 ORM 实体直接当响应体。
- `enum` 仅在有稳定闭合集合时使用；否则用联合字面量类型。
- 异步函数返回 `Promise<T>`；不要把 Promise 当值漏掉 `await`。

## 3. Node.js 20+ 约定

- 以 Node 20+ API 为准：优先 `node:` 协议导入内置模块（如 `node:fs/promises`、`node:path`）。
- 禁止依赖已弃用或文档标明将移除的 API；禁止为兼容 Node 16/18 写垫片，除非仓库明确要求。
- 文件系统、加密、HTTP 客户端等优先用 Promise / async API，避免无必要的 callback 风格。
- 模块格式（ESM/CJS）**跟随 Nest/仓库既有配置**，不在本技能中强行切换；新增文件与现有 `import`/`require` 风格一致。
- 不引入仅在浏览器或非 Node 环境可用的 API。

## 4. Controller / Provider / Module 边界

| 层 | 职责 | 禁止 |
| --- | --- | --- |
| Controller | HTTP 映射、取参、调 service、返回约定形状 | 业务规则、直接访问 DB、复杂分支 |
| Service / Provider | 业务逻辑、编排、事务边界 | 解析 HTTP 细节、依赖 `Request`/`Response`（除非明确适配层） |
| Module | 组装 providers、imports/exports | 业务代码 |

- Controller 方法保持薄：校验后的 DTO 进 service，service 返回领域结果，再映射为响应。
- 可注入依赖一律构造器注入；标清 `@Injectable()`。
- 只在 module 的 `exports` 中放出外域需要的 provider；默认不导出。
- 守卫 / 拦截器 / 管道：横切能力用它们；业务 if/else 仍留在 service。
- 不要在 provider 里 `new` 本可注入的依赖，以免绕过 DI 与测试替身。

## 5. DTO / 校验 / Pipe

- 入参一律 DTO 类（或仓库等价方案）+ `class-validator` / `class-transformer`（或仓库已采用的校验库）。
- 全局或路由级启用 `ValidationPipe`：`whitelist: true`、`forbidNonWhitelisted: true`、`transform: true`（除非仓库另有更严配置）。
- 创建 / 更新 DTO 分开；更新 DTO 字段多为可选，但须仍有约束。
- Query / Param / Body 分不同 DTO，不混用一个「万能」对象。
- 校验失败返回统一校验错误形状（见错误处理）；不要吞掉 pipe 错误后改抛含糊 500。
- 出参可用专用 Response DTO / serializer；禁止直接泄露实体内部字段（密码哈希、内部标志等）。

## 6. 错误处理与 HTTP 响应

- 业务可预期失败用 Nest `HttpException` 子类或领域异常 + 全局过滤器映射；禁止裸 `throw new Error` 当 HTTP 结果。
- 不在 controller 里随手 `try/catch` 再 `res.status().json()`，除非过滤器无法覆盖的边界。
- 响应形状在仓库内统一。若无既有约定，采用：

```ts
// 成功：直接返回资源或约定包装（跟随仓库；二者选一，不混用）
// 失败：
{
  statusCode: number;
  message: string | string[];
  error?: string;
  // 可加 requestId / code；字段名跟随仓库
}
```

- 状态码语义正确：`400` 校验/非法入参，`401`/`403` 认证授权，`404` 资源不存在，`409` 冲突，`422` 语义不可处理（若团队使用），`5xx` 仅未处理异常。
- 不要把内部堆栈、SQL、文件路径返回给客户端。

## 7. 配置与环境变量

- 通过 Nest `ConfigModule` / 等价配置模块读取环境变量；禁止在业务代码散落 `process.env.X`。
- 启动时校验必需配置（schema 或自定义 validate）；缺关键配置应**启动失败**，不要带错配置跑起来。
- 配置键命名稳定、有文档或类型；区分 `development` / `test` / `production`。
- 密钥、令牌、连接串只来自环境或密钥管理，不写进源码、示例提交物或默认值里的真密钥。
- `.env*` 是否入库跟随仓库；哪怕示例文件也只用占位符。

## 8. 日志

- 使用 Nest `Logger` 或仓库统一日志封装；禁止长期保留 `console.log` 作为正式日志。
- 日志带上下文：模块名、关键业务 id（非敏感）、请求关联 id（若有）。
- 级别：`error` 失败与异常，`warn` 可恢复异常，`log`/`info` 关键路径，`debug`/`verbose` 细节。
- **禁止**记录密码、令牌、cookie、Authorization、完整支付信息、身份证等密钥与 PII；需要排障时记脱敏后的标识。
- 生产环境不打冗长 debug；不依赖日志代替指标与审计。

## 9. 测试约定（高层）

- **核心流程 / 核心业务逻辑必须写单元测试**（典型落在 service、领域纯函数、关键 pipe/guard 等）。单测测行为与边界，不测框架本身；文件名与命令跟随仓库（如 `*.spec.ts`）。
- **涉及外部系统时可跳过单元测试**：依赖真实第三方 API、支付、短信、不可替身的外部 IO 等，且无法在不连外网/外部服务的前提下稳定测通时，可跳过单测。须在代码或邻近测试说明里标明跳过原因与依赖的外部系统；未标明不得擅自跳过。
- 能 mock / 替身的依赖（DB、HTTP 客户端、跨模块 provider）仍应 mock 后写单测，不算「可跳过」。
- **e2e / 集成**（按仓库需要）：HTTP 路径、校验、鉴权、模块装配；用测试模块与测试配置，不连真实生产资源。
- 测试数据可重复、可隔离；不依赖执行顺序。仓库已有测试结构时跟随；没有约定先问再写。

## 10. 命名、文件名与 DI 风格

- 文件：kebab-case + 角色后缀，如 `order.service.ts`、`create-order.dto.ts`、`orders.controller.ts`、`orders.module.ts`。
- 类：PascalCase，并带角色后缀：`OrdersController`、`OrdersService`、`OrdersModule`、`CreateOrderDto`。
- 方法 / 变量：camelCase；常量：UPPER_SNAKE 或仓库既有风格。
- 路由路径：复数资源名、小写、kebab-case（如 `/order-items`）；HTTP 动词与语义一致。
- DI：构造器注入 + `private readonly`；接口 token 用 `string` / symbol / 类，并在 module 中显式 `provide`/`useClass`（或仓库等价写法）。
- 同一概念命名在 controller / service / dto / 测试间保持一致。

## 11. 安全基线

- 所有外部输入经校验与白名单；不信任客户端字段。
- 鉴权与鉴权后的授权分开：认证守卫 + 权限/角色检查；默认拒绝。
- 密钥与连接串不进日志、不进错误响应、不进前端可见配置。
- 生产关闭过宽 CORS、明文密钥默认值、以及暴露内部错误细节的调试开关。
- 文件上传、重定向 URL、模板渲染等按仓库要求做类型/大小/允许列表限制。
- 依赖变更注意已知漏洞；不引入无必要的动态 `eval` / 任意 `exec`。

## 12. Agent 应做 / 不应做

### 应做

- 改代码前先读邻近模块的现有写法，保持一致。
- 安装与脚本一律用 **pnpm**（如 `pnpm install`、`pnpm add`、`pnpm run …`）；只维护 `pnpm-lock.yaml`。
- 新功能落在**对应业务域** feature module；边界清晰：controller 薄、service 厚、DTO 校验入参。
- 新外部依赖客户端放使用方域的 `adapters/`，不新建顶层技术域。
- 补齐类型、校验、错误映射；**核心流程/逻辑补单元测试**；配置走 Config 模块。
- 因外部系统跳过单测时写明原因与依赖；不确定的域边界、响应包装、鉴权方案：**先问**，不臆造全局约定。
- 只改与任务相关的文件；保持变更可审阅。

### 不应做

- 不用 npm / yarn 安装或跑脚本；不生成、不提交 `package-lock.json` 或 `yarn.lock`。
- 不新建 `query` / 独立 `ingress` 域 / 空壳 `pipeline` 域 / 与业务平级的 SDK 域。
- 不在业务 service 里直接操作 `request`/`response`，或绕过 Nest 管道/过滤器手写一套平行框架。
- 不把 ORM 实体当 API 契约；不返回敏感字段。
- 不关 TypeScript 严格项、不滥用 `any`、不吞异常后返回假成功。
- 不硬编码密钥；不 `console.log` 密钥或完整请求头。
- 不引入与仓库冲突的风格（随意换 ORM、换校验库、换模块格式），除非用户明确要求。
- 不对可 mock 的核心逻辑省略单测，或不经说明就以「外部系统」为由跳过。
- 不自动提交、不擅自发版；提交与发布遵守仓库与用户指示。
- 不把教程式长注释或大段脚手架粘进无关文件。

## 完成时自检

写完 NestJS 改动后快速核对：

1. 顶层 module 是否皆为业务域；有无技术门面 / 适配器平级冒充域（对照 §1）
2. 模块边界与命名是否符合本规范与仓库现状
3. 入参是否有 DTO + 校验；出参是否无敏感字段
4. 错误是否走统一异常/过滤器；状态码是否合理
5. 配置是否经 Config；日志是否无密钥
6. 核心流程/逻辑是否有单元测试；若跳过，是否因外部系统并已标明
