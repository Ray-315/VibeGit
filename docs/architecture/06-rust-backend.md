# F. Rust 后端设计

## 1. 模块划分总览

```
src-tauri/src/
├── main.rs          — 入口，只启动 Tauri
├── lib.rs           — 插件注册 + command 注册 + AppState 初始化
├── state.rs         — AppState 定义
├── commands/        — IPC 命令层（薄胶水层）
├── services/        — 业务逻辑层
├── domain/          — 领域模型与错误类型
└── utils/           — 纯工具函数
```

**依赖方向**（严格单向）：
```
commands → services → domain ← utils
                    ↑
                  state
```

---

## 2. AppState 设计

```rust
// src-tauri/src/state.rs
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::Mutex;
use crate::services::{GitService, RepoManager};

pub struct AppState {
    /// 已打开的仓库服务实例（key = 规范化路径）
    /// 使用 Arc 允许 clone 到异步任务中
    pub repos: Mutex<HashMap<String, Arc<GitService>>>,
    /// 仓库管理器（处理最近列表、持久化）
    pub repo_manager: Mutex<RepoManager>,
    /// GitHub 认证状态
    pub github_auth: Mutex<Option<GitHubAuthState>>,
}

impl AppState {
    pub async fn get_repo(&self, path: &str) -> Result<Arc<GitService>, AppError> {
        let repos = self.repos.lock().await;
        repos.get(path)
            .cloned()
            .ok_or_else(|| AppError::RepoNotOpened { path: path.to_string() })
    }
}
```

---

## 3. GitService 设计

`GitService` 是核心服务，封装 `git2::Repository`。

```rust
// src-tauri/src/services/git_service.rs
use git2::{Repository, StatusOptions};
use crate::domain::{models::*, error::AppError};

pub struct GitService {
    /// 仓库绝对路径
    path: String,
    /// git2 Repository 实例（内部使用 Mutex 保证线程安全）
    repo: Mutex<Repository>,
}

impl GitService {
    /// 打开仓库，验证是否有效
    pub fn open(path: &str) -> Result<Self, AppError> {
        let repo = Repository::open(path)
            .map_err(|e| match e.code() {
                git2::ErrorCode::NotFound => AppError::NotARepo { path: path.to_string() },
                _ => AppError::GitError { message: e.message().to_string() },
            })?;

        // 拒绝 bare repo
        if repo.is_bare() {
            return Err(AppError::BareRepo);
        }

        Ok(Self {
            path: path.to_string(),
            repo: Mutex::new(repo),
        })
    }

    /// 获取仓库摘要
    pub async fn get_summary(&self) -> Result<RepositorySummary, AppError> { ... }

    /// 获取工作区状态
    pub async fn get_status(&self) -> Result<StatusSummary, AppError> { ... }

    /// 获取分支列表
    pub async fn get_branches(&self) -> Result<Vec<BranchInfo>, AppError> { ... }

    /// 切换分支
    pub async fn checkout_branch(&self, name: &str) -> Result<(), AppError> { ... }

    /// 创建新分支
    pub async fn create_branch(&self, name: &str, from: Option<&str>) -> Result<BranchInfo, AppError> { ... }

    /// 删除本地分支
    pub async fn delete_branch(&self, name: &str) -> Result<(), AppError> { ... }

    /// 获取提交历史
    pub async fn get_commit_log(&self, params: CommitLogParams) -> Result<CommitLogResult, AppError> { ... }

    /// 获取单个 commit 详情
    pub async fn get_commit_detail(&self, sha: &str) -> Result<CommitDetail, AppError> { ... }

    /// 获取文件 diff
    pub async fn get_diff(&self, params: DiffParams) -> Result<FileDiff, AppError> { ... }

    /// Stage 文件
    pub async fn stage_files(&self, paths: &[String]) -> Result<(), AppError> { ... }

    /// Unstage 文件
    pub async fn unstage_files(&self, paths: &[String]) -> Result<(), AppError> { ... }

    /// 提交
    pub async fn commit(&self, message: &str) -> Result<CommitSummary, AppError> { ... }

    /// 丢弃工作区修改
    pub async fn discard_changes(&self, paths: &[String]) -> Result<(), AppError> { ... }
}
```

### 3.1 git2 线程安全说明

`git2::Repository` 不是 `Send`，因此需要 `Mutex<Repository>` 包装。在每次 async 方法中：

```rust
pub async fn get_status(&self) -> Result<StatusSummary, AppError> {
    // 在同步块中持有锁，避免跨 await 持有
    let repo = self.repo.lock().await;
    let statuses = repo.statuses(Some(&mut StatusOptions::new()))?;
    // 构建结果（纯内存操作）
    let result = build_status_summary(&statuses);
    drop(repo); // 尽早释放锁
    Ok(result)
}
```

对于耗时的 git 操作（fetch、clone），使用 `tokio::task::spawn_blocking`：

```rust
pub async fn fetch_remote(&self, remote: &str) -> Result<(), AppError> {
    let repo_path = self.path.clone();
    tokio::task::spawn_blocking(move || {
        let repo = Repository::open(&repo_path)?;
        // 执行 fetch
        Ok(())
    })
    .await
    .map_err(|e| AppError::Internal { message: e.to_string() })?
}
```

---

## 4. 错误类型设计

### 4.1 使用 thiserror

```rust
// domain/error.rs
use serde::Serialize;

#[derive(Debug, thiserror::Error, Serialize, specta::Type)]
#[serde(tag = "code", rename_all = "SCREAMING_SNAKE_CASE")]
pub enum AppError {
    #[error("Not a git repository: {path}")]
    NotARepo { path: String },
    // ... 见 05-domain-models.md
}

// 让 AppError 可用作 tauri command 的返回错误类型
impl From<AppError> for tauri::ipc::InvokeError {
    fn from(err: AppError) -> Self {
        tauri::ipc::InvokeError::from_anyhow(
            anyhow::anyhow!(serde_json::to_string(&err).unwrap_or_default())
        )
    }
}
```

### 4.2 错误边界划分

| 层级 | 错误类型 | 处理方式 |
|---|---|---|
| git2 | `git2::Error` | 在 GitService 中转为 `AppError` |
| IO | `std::io::Error` | 在 services 中转为 `AppError::IoError` |
| reqwest | `reqwest::Error` | 在 github_api 中转为 `AppError::NetworkError` |
| command | `AppError` | 直接返回，Tauri 序列化为 JSON |
| 前端 | 反序列化后的对象 | 通过 code 字段分类处理 |

### 4.3 不使用 anyhow 的理由

命令层必须返回**类型化错误**（前端需要解析 `code` 字段做不同处理），`anyhow` 的不透明错误类型不适合跨进程传输。`anyhow` 可在**内部临时中间层**用于快速原型，但在服务层边界必须转换为 `AppError`。

---

## 5. 线程模型与异步策略

### 5.1 Tauri 的运行时

Tauri 2 内置 tokio 运行时。所有 `#[tauri::command]` 默认在 tokio 上执行，可以直接使用 `async fn`。

### 5.2 操作分类

| 操作 | 耗时预估 | 执行策略 |
|---|---|---|
| get_status（小/中仓库） | < 200ms | 直接 async，前端 await |
| get_commit_log（首屏 50 条） | < 300ms | 直接 async |
| get_diff（普通文件） | < 100ms | 直接 async |
| stage / unstage / commit | < 500ms | 直接 async |
| checkout（需处理工作区） | < 2s | 直接 async + UI 加载态 |
| get_status（超大仓库 > 10 万文件） | 1–10s | spawn_blocking + 进度事件 |
| fetch / push / pull（网络操作） | 不确定 | spawn_blocking + 进度回调 + 事件推送 |
| clone（大仓库） | 分钟级 | spawn_blocking + 进度事件 + 可取消 |

### 5.3 长耗时操作模式

```rust
// commands/git.rs
#[tauri::command]
pub async fn fetch_remote(
    window: tauri::Window,
    state: tauri::State<'_, AppState>,
    repo_path: String,
    remote_name: String,
) -> Result<(), AppError> {
    let task_id = uuid::Uuid::new_v4().to_string();
    let window_clone = window.clone();
    let task_id_clone = task_id.clone();

    tokio::task::spawn_blocking(move || {
        // 定期发送进度事件
        let _ = window_clone.emit("task:progress", TaskProgress {
            task_id: task_id_clone.clone(),
            percent: 0,
            message: "Connecting...".to_string(),
        });
        // 执行 git fetch
        // ...
        let _ = window_clone.emit("task:complete", TaskComplete {
            task_id: task_id_clone,
        });
    });

    // 立即返回 task_id，前端通过事件跟踪进度
    Ok(())
}
```

---

## 6. 命令层组织方式

```rust
// commands/mod.rs — 统一注册所有命令
pub fn get_commands() -> impl Fn(tauri::Builder<tauri::Wry>) -> tauri::Builder<tauri::Wry> {
    |builder| {
        builder.invoke_handler(tauri::generate_handler![
            // repo
            repo::open_repository,
            repo::close_repository,
            repo::get_recent_repos,
            repo::init_repository,
            // git status & staging
            git::get_repo_summary,
            git::get_status,
            git::stage_files,
            git::unstage_files,
            git::discard_changes,
            git::commit_changes,
            // branches
            branch::get_branches,
            branch::checkout_branch,
            branch::create_branch,
            branch::delete_branch,
            // history
            history::get_commit_log,
            history::get_commit_detail,
            // diff
            diff::get_diff,
            diff::get_file_diff,
        ])
    }
}
```

命令函数规范：
```rust
// commands/git.rs
#[tauri::command]
pub async fn get_status(
    state: tauri::State<'_, AppState>,
    repo_path: String,          // 必须有路径参数
) -> Result<StatusSummary, AppError> {
    // 1. 获取 service 实例
    let service = state.get_repo(&repo_path).await?;
    // 2. 调用 service（业务逻辑在 service 中）
    service.get_status().await
    // 3. 错误自动传播，不在此做处理
}
```

---

## 7. 文件系统访问与权限

### 7.1 Tauri 权限配置

在 `capabilities/default.json` 中精确声明允许访问的路径：
```json
{
  "permissions": [
    "fs:read-all",
    "dialog:open",
    "dialog:save",
    "window:allow-set-title"
  ]
}
```

### 7.2 路径安全

所有路径在传入 GitService 前进行规范化：
```rust
// utils/path.rs
pub fn normalize_path(path: &str) -> Result<String, AppError> {
    let path = std::path::Path::new(path)
        .canonicalize()
        .map_err(|_| AppError::InvalidPath { path: path.to_string() })?;
    // 统一转为正斜杠（跨平台显示一致）
    Ok(path.to_string_lossy().replace('\\', "/"))
}
```

---

## 8. 日志与调试

### 8.1 日志配置

```rust
// lib.rs
fn setup_logging() {
    use tracing_subscriber::{fmt, EnvFilter};
    fmt()
        .with_env_filter(
            EnvFilter::try_from_env("VIBEGIT_LOG")
                .unwrap_or_else(|_| EnvFilter::new("info"))
        )
        .with_target(false)
        .init();
}
```

日志文件位置：
- Windows: `%APPDATA%\com.vibegit.app\logs\`
- macOS: `~/Library/Logs/com.vibegit.app/`
- Linux: `~/.local/share/com.vibegit.app/logs/`

### 8.2 Rust panic 处理

```rust
// main.rs
std::panic::set_hook(Box::new(|info| {
    tracing::error!("PANIC: {}", info);
    // 写入 panic 日志文件
}));
```

### 8.3 开发调试建议

- 使用 `VIBEGIT_LOG=debug cargo tauri dev` 开启详细日志
- Tauri 开发模式自动打开 WebView DevTools
- 使用 `tracing::instrument` 装饰服务方法，记录参数和耗时

---

## 9. RepoManager — 最近仓库管理

```rust
// services/repo_manager.rs
pub struct RepoManager {
    /// 持久化到用户配置目录
    config_path: PathBuf,
    recent_repos: Vec<RecentRepo>,
}

impl RepoManager {
    pub fn load() -> Self { ... }          // 从磁盘加载
    pub fn save(&self) { ... }             // 持久化到磁盘
    pub fn add_recent(&mut self, path: &str) { ... }  // 添加/更新最近记录
    pub fn get_recent(&self) -> Vec<RecentRepo> { ... }
    pub fn remove_invalid(&mut self) { ... }  // 清理不存在的仓库
}
```

配置文件格式：`~/.config/vibegit/config.json`（或 Windows 对应路径）
