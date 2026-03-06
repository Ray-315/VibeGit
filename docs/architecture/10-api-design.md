# J. Tauri 命令 API 设计

## 1. API 设计原则

- 所有命令均为 `async fn`，避免阻塞
- 所有命令返回 `Result<T, AppError>`
- `repo_path` 作为第一个参数（区分多仓库场景）
- 输入参数使用驼峰命名（Tauri 自动转换）
- 输出使用 `specta` 派生的类型，保证前后端一致

---

## 2. 仓库管理命令

### `open_repository`

```typescript
// 输入
{ repoPath: string }

// 输出
Result<RepositorySummary, AppError>

// 可能错误
// NOT_A_REPO:       路径不是 git 仓库
// BARE_REPO:        bare 仓库不支持
// INVALID_PATH:     路径不存在或无权限
// PERMISSION_DENIED: 无读取权限

// 调用时机：用户选择文件夹后
// 是否异步：是（打开大仓库可能稍慢）
// 前端缓存：打开成功后更新 repoStore.currentRepoPath，触发其他 Query
```

```rust
#[tauri::command]
pub async fn open_repository(
    state: tauri::State<'_, AppState>,
    repo_path: String,
) -> Result<RepositorySummary, AppError> {
    let path = normalize_path(&repo_path)?;
    let service = Arc::new(GitService::open(&path)?);
    let summary = service.get_summary().await?;
    
    let mut repos = state.repos.lock().await;
    repos.insert(path.clone(), service);
    
    let mut manager = state.repo_manager.lock().await;
    manager.add_recent(&path);
    
    Ok(summary)
}
```

---

### `get_recent_repos`

```typescript
// 输入：无
// 输出：Result<RecentRepo[], AppError>
// 调用时机：欢迎页加载时
// 是否异步：是（验证路径存在性）
// 前端缓存：TanStack Query，key: ['recent_repos']
```

---

### `init_repository`

```typescript
// 输入
{ path: string }
// 输出
Result<RepositorySummary, AppError>
// 可能错误：INVALID_PATH, PERMISSION_DENIED, INTERNAL
// 调用时机：用户点击"新建仓库"
```

---

## 3. 状态与工作区命令

### `get_repo_summary`

```typescript
// 输入
{ repoPath: string }
// 输出
Result<RepositorySummary, AppError>
// 调用时机：仓库打开后、repo:changed 事件后
// 前端缓存：key: ['repo', repoPath, 'summary']，staleTime: 3s
```

---

### `get_status`

```typescript
// 输入
{ repoPath: string }
// 输出
Result<StatusSummary, AppError>
// 可能错误：REPO_NOT_OPENED, GIT_ERROR
// 调用时机：
//   - 进入 Workspace 视图时
//   - repo:changed 事件触发缓存失效后
// 前端缓存：key: ['repo', repoPath, 'status']，staleTime: 3s
// 注意：大仓库（> 5 万文件）可能超过 1s，需骨架屏
```

---

### `stage_files`

```typescript
// 输入
{ repoPath: string; paths: string[] }
// 输出
Result<void, AppError>
// 可能错误：REPO_NOT_OPENED, GIT_ERROR, PERMISSION_DENIED
// 调用时机：用户点击文件旁的"暂存"按钮或"全部暂存"
// 是否异步：是
// 前端行为：成功后 invalidateQueries(['repo', repoPath, 'status'])
```

---

### `unstage_files`

```typescript
// 输入
{ repoPath: string; paths: string[] }
// 输出
Result<void, AppError>
// 调用时机：用户点击"取消暂存"
// 前端行为：同 stage_files
```

---

### `discard_changes`

```typescript
// 输入
{ repoPath: string; paths: string[] }
// 输出
Result<void, AppError>
// 可能错误：REPO_NOT_OPENED, GIT_ERROR
// 调用时机：用户确认"丢弃修改"后（需确认对话框）
// 注意：危险操作，不可撤销。前端必须先弹出确认框。
// 前端行为：成功后 invalidateQueries status
```

---

### `commit_changes`

```typescript
// 输入
{
  repoPath: string;
  message: string;       // subject + 两行换行 + body
  amend?: boolean;       // 第二阶段支持
}
// 输出
Result<CommitSummary, AppError>
// 可能错误：
//   GIT_ERROR:         底层 git commit 失败
//   DETACHED_HEAD:     detached HEAD 状态
//   NOTHING_TO_COMMIT: 暂存区为空
// 调用时机：用户输入消息后点击"提交"或 Ctrl+Enter
// 前端行为：成功后清空输入框，invalidate status + commits
```

---

## 4. 分支管理命令

### `get_branches`

```typescript
// 输入
{ repoPath: string }
// 输出
Result<BranchInfo[], AppError>
// 前端缓存：key: ['repo', repoPath, 'branches']，staleTime: 5s
// 调用时机：侧边栏加载时、branch:changed 事件后
```

---

### `checkout_branch`

```typescript
// 输入
{ repoPath: string; branchName: string; createIfNotExists?: boolean }
// 输出
Result<RepositorySummary, AppError>
// 可能错误：
//   GIT_ERROR:          checkout 失败（如工作区有冲突）
//   CHECKOUT_CONFLICT:  工作区有修改且无法自动合并
// 调用时机：用户点击分支名
// 前端行为：成功后更新 summary，invalidate status + branches
```

---

### `create_branch`

```typescript
// 输入
{ repoPath: string; name: string; fromRef?: string; checkout?: boolean }
// 输出
Result<BranchInfo, AppError>
// 可能错误：GIT_ERROR, INVALID_BRANCH_NAME（分支名包含非法字符）
```

---

### `delete_branch`

```typescript
// 输入
{ repoPath: string; name: string; force?: boolean }
// 输出
Result<void, AppError>
// 可能错误：
//   GIT_ERROR:          删除失败
//   BRANCH_NOT_MERGED:  分支未合并（force=false 时）
// 注意：不允许删除当前分支
```

---

## 5. 提交历史命令

### `get_commit_log`

```typescript
// 输入
{
  repoPath: string;
  page: number;          // 从 0 开始
  limit: number;         // 建议 50
  branch?: string;       // null = HEAD
  pathFilter?: string;   // 第二阶段：过滤特定文件路径
}
// 输出
Result<CommitLogResult, AppError>
// CommitLogResult = { commits: CommitSummary[]; hasMore: boolean; total?: number }
// 前端缓存：key: ['repo', repoPath, 'commits', { page, limit, branch }]
// 无限滚动：使用 useInfiniteQuery，加载更多时 page + 1
```

---

### `get_commit_detail`

```typescript
// 输入
{ repoPath: string; sha: string }
// 输出
Result<CommitDetail, AppError>
// 前端缓存：key: ['repo', repoPath, 'commit', sha]，长期缓存（commit 不可变）
// 调用时机：用户点击提交列表中的某个 commit
```

---

## 6. Diff 命令

### `get_diff`

```typescript
// 输入
{
  repoPath: string;
  kind: DiffKind;
  // kind = 'workdir_unstaged' | 'workdir_staged' | 'commit'
  sha?: string;          // kind = 'commit' 时必填
  path?: string;         // 指定单个文件（null = 所有文件的统计）
  contextLines?: number; // 默认 3
}
// 输出
Result<FileDiff, AppError>
// 可能错误：GIT_ERROR, FILE_TOO_LARGE（diff 超过截断阈值）
// 调用时机：用户点击文件列表中的文件
// 前端缓存：key: ['repo', repoPath, 'diff', { kind, sha, path }]
// 注意：stage/unstage 后需失效 staged/unstaged diff 缓存
```

---

## 7. 设置命令

### `get_settings`

```typescript
// 输入：无
// 输出：Result<AppSettings, AppError>
// AppSettings = { theme: 'dark'|'light'; fontSize: number; gitUserName: string; gitUserEmail: string }
```

### `update_settings`

```typescript
// 输入：{ settings: Partial<AppSettings> }
// 输出：Result<AppSettings, AppError>
```

---

## 8. 后台任务事件（Rust → 前端推送）

```typescript
// 事件定义（非 command，通过 tauri::Window::emit 发送）

// 仓库文件系统变化
listen('repo:changed', (event: { repoPath: string }) => void)

// 长耗时任务进度
listen('task:progress', (event: {
  taskId: string;
  percent: number;     // 0-100
  message: string;
}) => void)

// 长耗时任务完成
listen('task:complete', (event: {
  taskId: string;
  result?: unknown;
}) => void)

// 长耗时任务失败
listen('task:error', (event: {
  taskId: string;
  error: AppError;
}) => void)

// 后台错误（如文件监听失败）
listen('app:error', (event: { error: AppError }) => void)
```

---

## 9. 完整命令注册清单

```rust
// commands/mod.rs
tauri::generate_handler![
    // 仓库
    repo::open_repository,
    repo::close_repository,
    repo::get_recent_repos,
    repo::init_repository,
    repo::get_repo_summary,
    // 状态与暂存
    git::get_status,
    git::stage_files,
    git::stage_all,
    git::unstage_files,
    git::unstage_all,
    git::discard_changes,
    git::commit_changes,
    // 分支
    branch::get_branches,
    branch::checkout_branch,
    branch::create_branch,
    branch::delete_branch,
    branch::rename_branch,
    // 历史
    history::get_commit_log,
    history::get_commit_detail,
    // Diff
    diff::get_diff,
    // 设置
    settings::get_settings,
    settings::update_settings,
    // 调试（仅 debug 模式）
    #[cfg(debug_assertions)]
    debug::export_types,
]
```
