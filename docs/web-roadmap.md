# Web 端产品目标与技术方案

本文是 CodexMonitor Web 端的规划基准，用于明确最终能力、当前进度和后续落地方案。Web 端的目标不是替代 Tauri 桌面端的全部本地集成，而是在浏览器中提供可远程访问、可多人协作、可运维部署的 CodexMonitor 工作台。

## 目标定位

Web 端最终应支持用户通过浏览器连接 CodexMonitor daemon，完成工作区选择、线程对话、任务执行、实时事件查看、权限控制和基础管理操作。

设计原则：

- **daemon 是后端边界**：浏览器只通过 daemon 暴露的 HTTP/SSE 或后续 WebSocket/API 访问能力，不直接访问本地文件系统或 Codex 子进程。
- **复用现有协议**：优先复用 daemon JSON-RPC 方法和 `app-server-event` 事件，不重新定义一套与桌面端割裂的数据模型。
- **逐步补齐桌面能力**：先完成对话和任务执行闭环，再补齐线程管理、文件查看、设置、队列、权限和部署能力。
- **安全默认开启**：远程访问必须有鉴权；开发环境可以显式使用 insecure 模式，生产环境不能默认裸奔。

## 当前进度

| 模块 | 状态 | 当前能力 | 差距 |
| --- | --- | --- | --- |
| daemon HTTP 网关 | 已有预览版 | 支持 `/api/health`、`/api/rpc`、`/api/events`，可用 `--web-listen` 或 `CODEX_MONITOR_DAEMON_WEB_LISTEN` 启动 | 仍是轻量手写 HTTP；缺少会话、CSRF/Origin 策略、限流、结构化错误码、WebSocket 双向通道 |
| 鉴权 | 已有预览版 | 复用 daemon token，RPC 支持 Bearer token，SSE 支持 query token | 缺少用户/角色、token 生命周期、只读/读写权限、生产级密钥管理 |
| 工作区 | 已有预览版 | Web UI 可加载工作区、选择工作区并发起连接 | 缺少新增/编辑/删除工作区、状态筛选、工作区详情、连接失败诊断 |
| 线程/对话 | 已有预览版 | 可新建线程、发送文本任务、手动填写线程 ID | 缺少线程列表、线程树、历史消息渲染、分支/父子关系、隐藏/恢复、搜索 |
| 实时事件 | 已有预览版 | 通过 SSE 接收 daemon 广播的 `app-server-event`，Web UI 以日志形式展示 | 缺少事件 reducer、消息级 UI、断线重连状态、事件游标/补偿、按线程过滤 |
| 任务执行 | 已有预览版 | 可通过 `send_user_message` 发起文本任务 | 缺少队列/Steer 策略、取消/重试/继续、审批、运行状态和失败恢复 |
| 前端形态 | 已有预览版 | `/web` 或 `/?web=1` 渲染轻量 React 页面 | 缺少正式路由、设计系统复用、状态管理分层、测试覆盖和可部署产物策略 |
| 构建部署 | 部分完成 | daemon-only 构建可用 `--no-default-features` 跳过 Whisper/终端依赖 | 缺少一键开发脚本、生产部署说明、反向代理/TLS 文档、健康检查运维指南 |

## 最终功能清单

### P0：基础对话闭环

- 连接 daemon HTTP 网关并验证健康状态。
- 保存并切换网关地址和访问 token。
- 列出已有工作区，连接一个工作区。
- 新建线程并发送文本任务。
- 实时展示任务输出、状态变化、错误事件和完成状态。
- 支持断线重连并提示当前连接状态。

### P1：线程工作台

- 展示线程列表、当前线程、父子线程关系和处理中的线程。
- 加载历史消息并按消息类型渲染：用户输入、assistant 输出、工具调用、错误、系统状态。
- 支持继续对话、Steer 当前线程、Queue 新任务。
- 支持取消、重试、继续、复制线程信息、隐藏/恢复线程。
- 支持按工作区、线程状态、文本关键词过滤。

### P1：任务执行控制

- 支持 access mode、模型/配置选择、沙箱/审批策略展示与选择。
- 支持任务队列状态、排队中/运行中/完成/失败的可视化。
- 支持操作确认：高风险命令、文件写入、网络访问等审批流。
- 支持任务运行日志、工具调用参数和结果折叠展示。

### P1：工作区管理

- 新增、编辑、删除和刷新工作区。
- 展示工作区路径、类型、连接状态、最近活动线程、配置摘要。
- 支持工作区连接诊断：daemon 状态、Codex 可执行文件、权限、Git 状态。
- 支持多工作区快速切换。

### P2：文件与变更视图

- 浏览工作区文件树和只读文件内容。
- 展示 Git diff、未提交变更和任务产生的文件修改。
- 支持从事件跳转到相关文件、命令或线程消息。
- 支持受控写操作入口；默认不让浏览器直接写文件，所有写入通过 daemon/RPC 审批。

### P2：设置与账户能力

- Web 端设置页：网关、默认模型、默认 follow-up 策略、显示偏好。
- 多用户登录、角色权限、工作区授权范围。
- token 管理、过期、轮换和撤销。
- 审计日志：谁在什么工作区发起了什么任务。

### P2：运维与部署

- 生产部署文档：TLS、反向代理、CORS/Origin allowlist、systemd/Windows service。
- 健康检查和 metrics：daemon 存活、活跃连接、队列长度、事件延迟。
- 日志格式化和问题诊断页面。
- 前后端版本兼容检查。

## 技术实现方案

### 后端网关

当前 daemon 已提供轻量 HTTP/SSE 网关。短期继续复用现有 TCP JSON-RPC 处理函数，把 Web 请求转换为 daemon RPC 调用，确保 TCP 客户端、桌面端和 Web 端共享同一套后端行为。

后续建议分两阶段演进：

1. **预览阶段**：保留当前手写 HTTP 实现，补齐必要测试、错误码和事件过滤，避免引入额外框架影响 daemon 体积。
2. **正式阶段**：评估迁移到 `axum` 或 `hyper`，统一路由、中间件、CORS、鉴权、SSE/WebSocket、请求体限制和可观测性。

API 边界：

- `GET /api/health`：返回 daemon 名称、版本、健康状态。
- `POST /api/rpc`：接收 `{ method, params, clientVersion }`，复用 daemon RPC router。
- `GET /api/events`：输出 SSE，事件数据保持 `app-server-event` 兼容。
- 后续新增：`GET /api/schema` 或 `GET /api/capabilities`，用于前端判断 daemon 版本和功能开关。

### 前端架构

当前 Web UI 是一个独立 `WebApp` 预览页。正式化时应按现有前端规则拆分：

- `src/WebApp.tsx` 只保留组合和路由入口。
- Web 状态和副作用迁移到 `src/features/app/hooks/*` 或新增 `src/features/web/*`。
- Web API 客户端保留在 `src/services/webClient.ts`，并与 `src/services/tauri.ts` 对齐方法命名和返回结构。
- 事件订阅和 fanout 复用 `src/services/events.ts` 的模式，避免 Web 端重复实现线程 reducer。
- UI 组件复用既有 design-system primitives 和 tokens，避免重新维护一套 shell 样式。

### 状态模型

Web 端最终应共享桌面端的核心状态模型：

- Workspace：复用 `WorkspaceInfo`。
- Thread：复用线程 summary、active thread、processing state、hidden thread 规则。
- Event：继续使用 `AppServerEvent`，前端通过 reducer 转换为 UI 状态。
- Settings：Web 本地设置和 daemon 持久设置分层；本地只存连接信息和显示偏好。

事件处理原则：

- SSE/WebSocket 事件只作为增量来源。
- 初次进入页面必须通过 RPC 拉取快照。
- 断线重连后必须重新拉快照，避免错过事件导致状态漂移。

### 安全方案

预览版使用 daemon token。正式方案需要：

- 默认要求 token 或登录态；禁止公网部署时使用 `--insecure-no-auth`。
- Bearer token 用于 RPC；SSE 当前因浏览器 `EventSource` 限制使用 query token，正式阶段优先评估 fetch-based SSE 或 WebSocket 以避免 URL 泄露 token。
- 增加 Origin allowlist，替代当前宽松 CORS。
- 增加请求大小限制、速率限制、审计日志和结构化错误。
- 远程危险操作必须通过 daemon 审批策略执行，Web 端只展示和提交用户决策。

### 构建与部署

开发模式：

```bash
cd src-tauri
cargo build --no-default-features --bin codex_monitor_daemon --bin codex_monitor_daemonctl
CODEX_MONITOR_DAEMON_TOKEN=dev-token ./target/debug/codex_monitor_daemon --web-listen 127.0.0.1:4733
```

前端开发模式：

```bash
npm run dev
```

访问：

```text
http://127.0.0.1:5173/web
```

生产模式需要补齐：

- 静态资源构建与托管方式。
- daemon 是否内置静态文件服务。
- 反向代理配置。
- TLS 和 token/登录态管理。

## 里程碑

### M1：预览版可用

- 当前 HTTP/SSE 网关可启动。
- Web UI 可连接 daemon。
- 可加载工作区、新建线程、发送文本任务。
- 可看到实时事件日志。

### M2：对话体验可用

- 线程列表和历史消息加载。
- `app-server-event` reducer 接入真实聊天 UI。
- 断线重连和状态恢复。
- 基础错误提示和空状态。

### M3：任务控制可用

- 队列/Steer 行为接入。
- 取消、重试、继续、审批流。
- 工具调用、命令输出、文件变更展示。

### M4：管理和部署可用

- 工作区管理、设置页、权限模型。
- 生产部署、安全和运维文档。
- 自动化测试覆盖网关、Web client、核心 UI flow。

## 当前下一步

1. 为 `src-tauri/src/bin/codex_monitor_daemon/web.rs` 增加路由、鉴权、SSE 和 RPC 转发测试。
2. 将 `src/WebApp.tsx` 拆成 Web feature 目录，复用现有线程/事件状态模型。
3. 在 Web UI 中实现线程列表和消息级渲染，替代当前日志流展示。
4. 增加连接状态、断线重连和重新拉取快照逻辑。
5. 明确生产安全边界：Origin allowlist、token 传递方式和 insecure 模式警告。
