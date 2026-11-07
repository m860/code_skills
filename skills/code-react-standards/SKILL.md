---
name: code-react-standards
description: >-
  React + TypeScript frontend coding standards for agents. Reads
  code-architecture first for shared architecture/reuse rules; this skill only
  adds React stack conventions. Pins PNPM, MUI, dayjs, axios, and TanStack
  Query (@tanstack/react-query). Covers components, hooks, props typing,
  effects, styling, a11y, performance, testing, and security basics. Use when
  writing or reviewing React code, or when the user asks for React standards,
  React 规范, or frontend React coding conventions.
---

# React 前端编码规范

本技能约束 **React + TypeScript** 前端实现。写或改 React 代码前读本文件并遵守。下文钉死的库（pnpm / MUI / dayjs / axios / TanStack Query）必须使用，不另起竞争栈；其余未钉死项跟随仓库，没有约定先问。

先读通用 `../code-architecture/SKILL.md`。复用、边界、分层、抽取时机等架构共性以它为准；本技能只写 React 栈约定，不重复那些内容。

共性流程（需求目录、状态、发版）不在本技能内；有则先读对应 `code-spec-*` / `code-deploy`。

## 技术栈假设

- 框架：React（函数组件 + Hooks）
- 语言：TypeScript（严格模式）
- UI 运行时：现代浏览器；构建与打包跟随仓库（Vite / Next / 其他）
- 包管理器：**pnpm 唯一**。安装、脚本、依赖增删一律用 `pnpm`；锁文件为 `pnpm-lock.yaml`。禁止 npm / yarn；禁止生成或提交 `package-lock.json`、`yarn.lock`
- UI 组件库：**MUI（Material UI）**。优先用 MUI 组件与主题；不新引入 Ant Design、Chakra、裸造组件库等竞争 UI 栈
- 日期时间：**dayjs**。解析、格式化、运算用 dayjs；不新引入 `moment`、`date-fns` 等（存量除外，新代码仍用 dayjs）
- HTTP 客户端：**axios**。接口请求用 axios（可集中封装实例）；不另起 fetch 包装层或 ky / ofetch 等竞争客户端作为默认方案
- 服务端状态 / 查询与变更：**TanStack Query**（`@tanstack/react-query`），使用 `useQuery` / `useMutation` 等。列表与远端读写走 Query；不新引入 SWR、RTK Query、手写全局请求缓存作为默认
- 路由、客户端全局 UI 状态、测试运行器等未钉死项：**跟随仓库已有约定**；没有约定先问，不擅自引入第二套

## 1. 组件结构与文件命名

- 优先函数组件；不要新写 class 组件（除非维护存量 class）。
- 文件命名跟随仓库；无约定时：组件用 PascalCase 文件名（如 `UserCard.tsx`），hooks / 工具用 camelCase（如 `useUserMenu.ts`、`formatDate.ts`）。
- 一文件一主组件（或一个紧密相关的小组件簇）；默认导出 / 具名导出跟随仓库，新增代码保持一致。
- 按**业务功能域**组织目录（与后端业务语言对齐更佳），不按「所有 components / 所有 hooks」无限堆平；也不按纯技术角色（如单独 `query/` 页面桶）切目录。共享 UI 放仓库既有的 shared / ui 位置。
- 组件文件保持可读：渲染为主；复杂派生、请求、副作用抽到 hooks 或邻近模块。
- 展示与数据编排可拆：容器 / 页面负责取数与状态，展示组件尽量纯 props 驱动（若仓库已有此分层则跟随）。

## 2. Hooks 与状态边界

- 遵守 Hooks 规则：只在组件或自定义 Hook 顶层调用；不在条件 / 循环 / 嵌套函数里调用。
- 状态放最近够用的地方：仅本组件用 → 本地 `useState` / `useReducer`；多兄弟共享 → 上提或仓库既有的上下文 / 客户端状态方案。
- **服务端/远端数据**用 TanStack Query（`useQuery` / `useMutation` 等）管理缓存、加载与错误；不要把接口列表塞进随意的全局 store 里手写同步。
- 派生数据优先在渲染中计算或抽纯函数；不要为「镜像 props」再开一份 state。
- 自定义 Hook 命名以 `use` 开头，封装可复用的状态与副作用；Hook 内不要偷偷做与调用方无关的全局副作用。
- Context 用于少量、稳定、树深共享的值（主题、会话）；高频变化的大对象不要无脑塞进单一 Context；MUI 主题走既有 `ThemeProvider` 模式。
- 服务端组件 / 客户端组件边界（若用 RSC）：跟随框架约定；仅在需要浏览器 API 或交互时标客户端，不把整树默认打成 client。

## 3. Props 与 TypeScript

- 开启并保持严格相关选项。禁止 `any`；逃逸用 `unknown` 并收窄。
- Props 用 `type` 或 `interface` 显式描述；导出给外域用的组件 props 必须显式。
- 布尔 props 命名清晰（`isOpen` / `disabled`）；回调用 `onXxx`；避免意义不明的 `data` / `info`。
- 区分必选与可选；可选 props 提供合理默认或在类型上表达互斥（联合类型），不要一堆互相矛盾的可选字段。
- 不要把庞大领域实体整包塞进 props；只传视图需要的字段。
- Children、render props、多态组件（如 `as`）的类型写清楚；不靠宽松断言糊弄。

## 4. Effects 与数据获取（高层）

- `useEffect` 用于同步外部系统（订阅、命令式 DOM、与非 React 库对接），不是「随便放副作用」的垃圾桶。
- 能从 props / state 算出的不要放进 effect 再 `setState`。
- HTTP：用 **axios**（共享实例、拦截器、错误映射跟随仓库封装）；查询用 **`useQuery`**，变更用 **`useMutation`**（`@tanstack/react-query`）。不要在组件里用 `fetch` + `useEffect` 手写一套并行请求/缓存层。
- `queryKey` 稳定、可序列化；mutation 成功后按需 `invalidateQueries` / 更新缓存，行为与仓库现有 Query 用法一致。
- Effect 若仍需要：必须处理清理（取消订阅、忽略过期响应等）。依赖数组写真实依赖；不靠漏依赖或盲目 `eslint-disable`。
- 用户触发的写操作走 `useMutation`（或事件里调用 mutation），不绕道 effect「再观察一个 flag」。
- 日期字段的解析与展示用 **dayjs**，与接口字符串/时间戳的转换集中、可测。

## 5. 样式与 UI（MUI）

- UI 以 **MUI** 为准：优先 `Button`、`TextField`、`Dialog`、`Stack`/`Box` 等既有组件与 `sx` / theme，不平行引入另一套组件库。
- 主题、间距、断点、调色板走 MUI theme；扩展放仓库既有 theme 文件，不散落魔法值。
- 仅在 MUI 覆盖不到的局部补样式；不为此改用 Tailwind / CSS Modules 作为新默认（存量允许，新功能仍优先 MUI）。
- 响应式用 MUI breakpoints / 布局组件；不硬编码仅适用于某一屏幕的假设而不加说明。

## 6. 可访问性（基线）

- 语义化 HTML：按钮用 `button`，链接用 `a`；不要用可点击的 `div` 冒充控件，除非补齐角色、键盘与焦点。
- 表单控件有可见 label（或等价 `aria-label`）；错误信息与控件关联。
- 图标按钮提供无障碍名称；装饰性图标对辅助技术隐藏。
- 焦点可见；弹层 / 对话框遵循焦点陷阱与 Esc 关闭等仓库或组件库约定。
- 图片有恰当 `alt`（装饰图可空 alt）；不要只靠颜色传达状态。
- 交互控件可键盘操作；自定义组件对照既有 a11y 模式实现。

## 7. 性能

- 默认写出正确、清晰的代码；**不要**默认包一层 `useMemo` / `useCallback` / `React.memo`。
- 仓库启用 React Compiler 或文档要求跟随 Compiler 时：按该指导编写，避免与 Compiler 冲突的手写记忆化。
- 仅在实测或明显昂贵计算、不稳定引用导致无谓重渲染时再记忆化；加之前能说清原因。
- 列表使用稳定且有意义的 `key`；禁止用随机数或仅用 index（除非列表静态且永不重排）。
- 大列表 / 重树：跟随仓库虚拟化或拆分方案；不提前过度拆 code split，除非路由或体量已需要。
- 避免在渲染路径里创建巨大临时对象或做重 IO；IO 放事件、effect 或数据层。

## 8. 测试约定（高层）

- 测试目录、命名与命令跟随仓库；没有约定先问。
- **核心组件必须写单元测试**：可复用/承载关键交互或展示规则的组件（含关键自定义 hooks），优先测用户可见行为与边界，少测实现细节。
- **页面级流程 / 页面逻辑必须写 E2E 测试**：路由页、多步用户路径、页面编排与关键成功/失败路径。
- **涉及外部系统时可跳过测试**：流程或逻辑依赖真实第三方、不可替身的外部服务，且无法在隔离环境稳定跑通时，可跳过对应单测或 E2E。须标明跳过原因与依赖的外部系统；未标明不得擅自跳过。
- 能用 mock / MSW / 测试替身隔离的依赖，仍应写测，不算「可跳过」。
- 查询优先按角色、标签文本等无障碍语义（若使用 Testing Library）；避免处处依赖测试专用 id，除非仓库惯例。
- 异步断言等到结果，不靠随意短 `sleep`。

## 9. 安全基线

- 默认把用户与接口内容当不可信；用文本节点 / 安全绑定输出，不把未消毒 HTML 塞进文档。
- **禁止**无必要的 `dangerouslySetInnerHTML`；必须用时：上游消毒、白名单、说明来源，并最小化表面积。
- 不把密钥、token 写进前端源码或提交物；前端可见配置只放公开值。
- 外链 `target="_blank"` 注意 `rel`（如 `noopener noreferrer`）跟随仓库约定。
- 不要在日志或错误上报里带密码、完整支付信息等敏感字段。

## 10. Agent 应做 / 不应做

### 应做

- 改前先读邻近组件、hooks 与 MUI / Query 用法，保持一致。
- 先读 `code-architecture`；重复逻辑按共性抽取，不在本技能里另立一套架构故事。
- 安装与脚本一律用 **pnpm**；只维护 `pnpm-lock.yaml`。
- UI 用 **MUI**；日期用 **dayjs**；HTTP 用 **axios**；远端读写用 **TanStack Query**（`useQuery` / `useMutation`）。
- 组件职责清晰：展示、本地状态、服务端状态边界分明；Props 与关键状态有明确类型。
- 补齐无障碍；**核心组件补单元测试，页面逻辑补 E2E**；因外部系统跳过时写明原因与依赖。
- 不确定路由、客户端全局状态、RSC 边界等未钉死项时：**先问**。

### 不应做

- 不用 npm / yarn；不生成、不提交 `package-lock.json` 或 `yarn.lock`。
- 不引入竞争栈：不用 Ant Design/Chakra 等替代 MUI；不用 moment/date-fns 替代 dayjs（新代码）；不用自研 fetch 层或 ky 等替代 axios 作为默认；不用 SWR/RTK Query 等替代 TanStack Query。
- 不新写 class 组件（除非改存量）；不在条件里调用 Hooks。
- 不默认堆 `useMemo` / `useCallback` / `memo`；不用全局 store 手写远端列表缓存来绕过 Query。
- 不对可隔离的核心组件/页面流程省略测试，或不经说明就以「外部系统」为由跳过。
- 不滥用 `dangerouslySetInnerHTML`；不把密钥塞进前端。
- 不自动提交、不擅自发版；不复制粘贴大段重复 JSX/逻辑代替复用。

## 完成时自检

1. 是否先遵循 `code-architecture`，且本改动无多余复制  
2. 组件 / 文件命名与目录是否符合仓库与本规范  
3. Hooks 用法与状态放置是否合理；远端数据是否走 TanStack Query + axios  
4. Props 与关键类型是否严格；有无 `any` / 危险 HTML  
5. UI 是否用 MUI、日期是否用 dayjs、包管理是否只用 pnpm  
6. 核心组件是否有单测、页面逻辑是否有 E2E；若跳过，是否因外部系统并已标明  
7. 是否无默认记忆化噪音；列表 `key` 是否稳定  
