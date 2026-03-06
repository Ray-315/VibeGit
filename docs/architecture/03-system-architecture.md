# C. 系统总体架构

## 1. 分层架构总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户界面层 (UI Layer)                         │
│   React + TypeScript + Tailwind CSS + shadcn/ui + Motion            │
│   Pages / Layouts / Components / Hooks                              │
├─────────────────────────────────────────────────────────────────────┤
│                      前端状态层 (State Layer)                        │
│   Zustand (UI状态)  +  TanStack Query (服务端数据缓存)               │
├─────────────────────────────────────────────────────────────────────┤
│                     前端服务层 (Service Layer)                       │
│   tauri-invoke 封装 / API 抽象 / 错误转换 / 类型安全                 │
├────────────────────────┬────────────────────────────────────────────┤
│     Tauri IPC 桥接层   │            Tauri 事件总线                   │
│  (invoke / commands)   │  (emit / listen / once)                    │
├────────────────────────┴────────────────────────────────────────────┤
│                      Rust 命令层 (Commands Layer)                    │
│   #[tauri::command] 函数 / 参数验证 / 错误映射 / 权限检查            │
├─────────────────────────────────────────────────────────────────────┤
│                      Rust 服务层 (Service Layer)                     │
│   GitService / RepoManager / AuthService / GithubApiService         │
├─────────────────────────────────────────────────────────────────────┤
│                      Rust 领域层 (Domain Layer)                      │
│   Domain Models / Error Types / Value Objects                       │
├─────────────────────────────────────────────────────────────────────┤
│                      Rust 基础设施层 (Infra Layer)                   │
│   git2 / reqwest / keyring / tokio / tracing / serde                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 前后端通信机制

### 2.1 Tauri invoke（命令调用）

用于**请求-响应**模式，前端主动调用 Rust 能力：

```
前端                          Rust
  │                             │
  │── invoke("get_status") ──►  │ #[tauri::command]
  │                             │ async fn get_status(...)
  │◄── Ok(StatusSummary) ──────  │
  │       或                     │
  │◄── Err(AppError) ──────────  │
```

特点：
- 每次调用返回 `Promise<T>`
- 错误统一序列化为 `{ code, message, detail }` 格式
- 通过 `tauri-specta` 自动生成 TypeScript 函数签名

### 2.2 Tauri 事件（后台推送）

用于 **Rust → 前端的主动推送**（进度通知、文件变化监听、后台任务状态）：

```
Rust                          前端
  │                             │
  │── emit("repo:status_changed") ──►  │ listen("repo:status_changed")
  │── emit("task:progress", {pct}) ──► │ listen("task:progress")
  │── emit("error:background") ──────► │ listen("error:background")
```

特点：
- 事件名采用 `domain:event_name` 命名约定
- payload 为 JSON 序列化的结构体
- 前端在组件挂载时注册监听，卸载时取消

### 2.3 通信数据格式

所有数据通过 serde_json 序列化，采用 `camelCase`：

```rust
#[derive(Serialize, Deserialize, specta::Type)]
#[serde(rename_all = "camelCase")]
pub struct StatusSummary { ... }
```

---

## 3. 模块边界

```
┌─────────────────────────────────────────────┐
│  src-tauri/src/                             │
│  ├── commands/          ← Tauri command 层  │
│  │   ├── repo.rs                            │
│  │   ├── git.rs                             │
│  │   ├── auth.rs                            │
│  │   └── mod.rs                             │
│  ├── services/          ← 业务逻辑服务层    │
│  │   ├── git_service.rs                     │
│  │   ├── repo_manager.rs                    │
│  │   ├── auth_service.rs                    │
│  │   └── github_api.rs                      │
│  ├── domain/            ← 领域模型与错误    │
│  │   ├── models.rs                          │
│  │   ├── error.rs                           │
│  │   └── mod.rs                             │
│  ├── state.rs           ← AppState 定义     │
│  ├── lib.rs             ← 插件注册入口      │
│  └── main.rs            ← 启动入口          │
└─────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────┐
│  src/ (前端)                                │
│  ├── pages/             ← 路由级页面        │
│  ├── components/        ← 通用 UI 组件      │
│  │   ├── ui/            ← shadcn/ui 基础    │
│  │   └── git/           ← Git 专用组件      │
│  ├── hooks/             ← 自定义 React hooks│
│  ├── store/             ← Zustand stores    │
│  ├── services/          ← invoke 封装层     │
│  ├── types/             ← TS 类型定义       │
│  └── lib/               ← 工具函数         │
└─────────────────────────────────────────────┘
```

---

## 4. 状态管理架构

### 4.1 状态分层策略

```
TanStack Query（服务端状态）：
  - 仓库摘要、提交历史、分支列表、diff 数据
  - 自动缓存、后台重新获取、失效管理
  - key 格式：['repo', repoPath, 'commits', { page, limit }]

Zustand（客户端 UI 状态）：
  - 当前选中仓库路径
  - 当前选中的 commit SHA
  - 当前选中的文件
  - 侧边栏折叠状态
  - 活跃面板（history / workspace）
  - 全局 toast 队列
  - 模态框状态
```

### 4.2 数据流向

```
用户操作
    │
    ▼
React 组件（调用 hook）
    │
    ▼
自定义 hook（useGitStatus / useBranches 等）
    │
    ├── TanStack Query → invoke("get_status") → Rust
    │
    └── Zustand store（读取/更新 UI 状态）
```

---

## 5. 异步任务与后台刷新机制

### 5.1 短耗时操作（< 500ms）

直接 `invoke` 并等待：stage、unstage、checkout（小仓库）

```typescript
const result = await invoke<StatusSummary>('get_status', { repoPath });
```

### 5.2 长耗时操作（> 500ms）

使用后台任务 + 进度事件：

```
前端 invoke("fetch_remote")
    ↓
Rust: 启动 tokio::spawn 任务
      └─ 每 100ms emit("task:progress", { taskId, percent, message })
      └─ 完成时 emit("task:complete", { taskId, result })
      └─ 失败时 emit("task:error", { taskId, error })

前端: 监听事件更新进度 UI
      任务完成后 invalidate TanStack Query 缓存
```

### 5.3 文件系统变化监听（后台自动刷新）

```
Rust 后台线程（tokio::spawn）：
  使用 notify crate 监听 .git 目录变化
  变化时 emit("repo:changed", { repoPath })

前端：
  listen("repo:changed") → 
  queryClient.invalidateQueries(['repo', repoPath, 'status'])
```

### 5.4 防抖与节流

- 文件系统事件去重：Rust 端 debounce 500ms（避免大批量文件变更时刷洪）
- 前端 TanStack Query：`staleTime: 2000ms`（2 秒内不重复请求）

---

## 6. 错误处理体系

### 6.1 错误流向

```
git2 Error / IO Error / reqwest Error
    │
    ▼
AppError（domain/error.rs，thiserror 派生）
    │
    ▼
命令层 #[tauri::command] 返回 Result<T, AppError>
    │
    ▼
Tauri 序列化为 JSON 错误: { "code": "NOT_A_REPO", "message": "..." }
    │
    ▼
前端 services 层解析为 ApiError TypeScript 类型
    │
    ▼
TanStack Query onError / 自定义 hook 捕获
    │
    ▼
Zustand toast store → 全局 Toast 组件显示
```

### 6.2 错误分类

```
USER_ERROR:    用户操作导致（如提交消息为空），显示友好提示
REPO_ERROR:    仓库状态异常（detached HEAD、bare repo），显示状态说明
GIT_ERROR:     底层 git 操作失败，显示错误消息 + 建议操作
NETWORK_ERROR: GitHub API 网络失败，提示重试
INTERNAL:      内部错误，记录日志，显示通用错误
```

---

## 7. AppState 设计

```rust
// src-tauri/src/state.rs
pub struct AppState {
    // 已打开的仓库注册表（path -> GitService 实例）
    pub repos: Mutex<HashMap<String, Arc<GitService>>>,
    // 最近打开的仓库列表（持久化到磁盘）
    pub recent_repos: Mutex<Vec<RecentRepo>>,
    // GitHub 认证状态
    pub github_auth: Mutex<Option<GitHubAuthState>>,
    // 后台任务注册表
    pub tasks: Mutex<HashMap<String, TaskHandle>>,
}
```

AppState 通过 `tauri::Manager::manage()` 注入，在所有 command 函数中可通过 `State<AppState>` 获取。

---

## 8. 架构约束与原则

1. **单向数据流**：用户操作 → command → Rust → 返回数据 → 前端状态更新 → UI 渲染
2. **命令层只做胶水**：command 函数不含业务逻辑，只做参数验证 + 调用 service
3. **服务层职责单一**：GitService 只管 git 操作，不知道 HTTP；GithubApiService 只管 API，不知道 git
4. **错误在命令层统一转换**：服务层抛 `AppError`，命令层 `map_err` 后返回给前端
5. **前端类型安全**：所有 invoke 调用必须有对应 TypeScript 类型，通过 specta 自动生成
6. **状态最小化**：前端只存 UI 状态，数据全部由 TanStack Query 从 Rust 获取并缓存
