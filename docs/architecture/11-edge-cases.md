# K. 边界情况与风险清单

## 分类说明

| 类别 | 说明 |
|---|---|
| 🔴 高风险 | 可能导致数据丢失或应用崩溃，必须在 MVP 中处理 |
| 🟡 中风险 | 影响功能正常使用，MVP 中至少给出友好提示 |
| 🟢 低风险 | 影响体验，可在第二阶段处理 |

---

## 1. 仓库状态边界

### 1.1 Detached HEAD 🔴

**场景**：`git checkout <sha>` 或 `git rebase` 操作后，HEAD 不指向任何分支。

**风险**：
- 在 detached HEAD 状态下 `commit` 会创建"游离提交"，切换分支后这些提交会被 GC 清除
- 用户可能不理解当前状态

**处理策略**：
```
- 状态栏/Sidebar 明确显示 "Detached HEAD @ abc1234"（橙色警告样式）
- 提交操作：显示警告对话框，说明风险，提供"创建分支后提交"选项
- 允许查看 diff、提交历史（只读操作正常工作）
- RepositorySummary.head 返回 HeadState::Detached { commit_sha }
```

---

### 1.2 Bare Repository 🔴

**场景**：`git init --bare` 创建的仓库，没有工作目录。

**风险**：所有工作区操作（status、stage、diff）都无法执行。

**处理策略**：
```
- 打开时检测：GitService::open() 中调用 repo.is_bare()
- 返回 AppError::BareRepo 错误
- 前端显示友好提示："This is a bare repository and cannot be opened with VibeGit"
- 不尝试打开，直接拒绝
```

---

### 1.3 Ongoing Operation（Merge/Rebase/Cherry-pick 进行中）🔴

**场景**：操作中断（如解决冲突时关闭终端），仓库遗留进行中状态。

**检测方式**（Rust）：
```rust
fn detect_ongoing_operation(repo: &Repository) -> Option<OngoingOperation> {
    let git_dir = repo.path();
    if git_dir.join("MERGE_HEAD").exists()       { return Some(OngoingOperation::Merge); }
    if git_dir.join("rebase-merge").exists()     { return Some(OngoingOperation::Rebase); }
    if git_dir.join("rebase-apply").exists()     { return Some(OngoingOperation::Rebase); }
    if git_dir.join("CHERRY_PICK_HEAD").exists() { return Some(OngoingOperation::CherryPick); }
    if git_dir.join("REVERT_HEAD").exists()      { return Some(OngoingOperation::Revert); }
    None
}
```

**处理策略**：
```
- RepositorySummary 包含 ongoingOperation 字段
- 主界面顶部显示状态横幅：如 "Merge in progress — resolve conflicts and commit, or abort"
- MVP 阶段：仅显示状态，提供"Abort"按钮
- 冲突文件：在状态视图中以特殊颜色（橙色）显示
```

---

### 1.4 Unborn Repository（空仓库） 🟡

**场景**：`git init` 后未进行第一次 commit。

**检测**：`repo.head()` 返回 `UnbornBranch` 错误。

**处理策略**：
```
- HeadState::Unborn 状态
- 提交历史显示空状态："No commits yet — make your first commit!"
- 允许 stage + commit（第一次提交）
- 分支操作禁用（无法 checkout 到不存在的分支）
```

---

## 2. 文件内容边界

### 2.1 二进制文件 🟡

**检测**（Rust）：
```rust
fn is_binary(content: &[u8]) -> bool {
    // 检查前 8000 字节是否含有 \0
    content.iter().take(8000).any(|&b| b == 0)
}
```

**处理策略**：
```
- FileDiff.is_binary = true
- DiffViewer 显示占位符："Binary file — changes not shown"
- 显示文件大小变化（old size / new size）
- 不显示 diff 内容（避免乱码）
```

---

### 2.2 文件编码问题 🟡

**场景**：非 UTF-8 编码文件（GB2312、Shift-JIS 等）

**处理策略**：
```
- 尝试 UTF-8 解码
- 失败时尝试用 chardet 等方式检测编码（第二阶段）
- MVP：UTF-8 解码失败的文件显示为二进制文件
- 保留原始字节，不做修改（git2 操作字节序列，不解析编码）
```

---

### 2.3 超大 Diff 截断 🟡

**阈值**：
- diff 行数 > 5000 行 → 截断
- diff 字节数 > 500 KB → 截断

**处理策略**：
```
- FileDiff.is_truncated = true
- DiffViewer 底部显示："Diff truncated — file too large to display completely"
- 提供"Open in external editor"按钮（第二阶段）
```

---

## 3. 路径与文件系统边界

### 3.1 路径分隔符（跨平台）🔴

**问题**：
- Windows：`\` 反斜杠
- macOS/Linux：`/` 正斜杠
- git2 内部使用 `/`

**处理策略**：
```rust
// utils/path.rs
pub fn to_git_path(path: &str) -> String {
    path.replace('\\', "/")
}

pub fn to_display_path(path: &str) -> String {
    #[cfg(target_os = "windows")]
    return path.replace('/', "\\");
    #[cfg(not(target_os = "windows"))]
    return path.to_string();
}
```

- git2 的 path spec 统一使用 `/`
- 显示给用户的路径使用系统分隔符
- 所有 API 传输使用 `/`

---

### 3.2 长路径（Windows）🟡

**问题**：Windows 默认 MAX_PATH = 260 字符，git 仓库可能包含超长路径文件。

**处理策略**：
```
- 在 Tauri 的 manifest 中启用 longPathAware：
  <longPathAware>true</longPathAware>
- git2 的 vendored libgit2 默认支持长路径
- 显示时超长路径用 ... 截断中间部分
```

---

### 3.3 权限不足 🔴

**场景**：
- 仓库目录无读取权限
- `.git` 目录被其他进程锁定（`.git/index.lock` 存在）

**处理策略**：
```
- 捕获 Permission Denied 错误 → AppError::PermissionDenied
- index.lock 存在时：提示"Another git process seems to be running"
- 不自动删除 .git/index.lock（危险操作，需用户手动确认）
```

---

## 4. 仓库规模与性能边界

### 4.1 大仓库（> 5 万文件）🟡

**问题**：`git status` 可能耗时 1–10 秒。

**处理策略**：
```
- 显示骨架屏 loading（不阻塞 UI）
- 异步执行，前端 pending 状态处理
- 长期优化：支持 .gitignore 配置 status 路径过滤
```

---

### 4.2 大历史仓库（> 10 万 commits）🟡

**问题**：`revwalk` 遍历整个历史代价高。

**处理策略**：
```
- 分页加载（每次 50 条），不预加载所有历史
- 前端虚拟列表（不渲染所有行）
- 不预计算 total count（代价高），仅显示"加载更多"
```

---

## 5. 仓库结构边界

### 5.1 Submodule 🟡

**MVP 处理策略**：
```
- submodule 目录在 status 中显示为单个条目（不递归进入）
- 提示："Contains submodules — submodule management coming soon"
- 不允许对 submodule 路径进行 stage/unstage（MVP 排除）
- 第二阶段：基本 submodule 支持（init/update/status 展示）
```

---

### 5.2 Worktree 🟢

**处理策略**：
```
- MVP 忽略 worktree（`git worktree add` 创建的额外工作目录）
- 显示正常，不做特殊处理
- 如果用户打开的是 worktree 路径而非主仓库，能正常工作
```

---

### 5.3 仓库损坏 🔴

**场景**：`.git` 目录文件损坏，git2 无法打开。

**处理策略**：
```
- 捕获所有 git2 打开错误 → AppError::GitError
- 显示错误消息，建议用户运行 `git fsck`
- 不崩溃，不卡死
```

---

## 6. 认证与网络边界（第二阶段相关）

### 6.1 凭证问题（HTTPS / SSH）🟡

**策略**：
```
- 第二阶段实现
- HTTPS：使用系统凭证管理（Windows Credential Manager / macOS Keychain）
- 通过 keyring crate 存储 OAuth token
- SSH：使用系统 SSH agent，不在应用内管理 SSH 密钥
```

### 6.2 网络超时 🟡

**策略**：
```
- reqwest 设置 30s 超时
- 超时后返回 AppError::NetworkError
- 前端显示重试按钮
```

---

## 7. 应用稳定性边界

### 7.1 Rust panic 处理 🔴

```rust
std::panic::set_hook(Box::new(|info| {
    // 记录 panic 到日志文件
    tracing::error!("PANIC: {}", info);
    // 通知前端（如果主线程 panic 则无法通知）
}));
```

### 7.2 前端 React 错误边界 🔴

```tsx
// ErrorBoundary 包裹每个主面板
// 单个面板崩溃不影响整体应用
// 显示"Something went wrong"+ 刷新按钮
```

### 7.3 前端内存泄漏 🟡

**常见场景**：
- 事件监听器未注销
- TanStack Query 长期持有大量 diff 数据

**防范**：
- 所有 `listen()` 在 `useEffect` cleanup 中注销
- TanStack Query `gcTime` 设置为 5 分钟
- diff 数据设置较短的 `gcTime`（1 分钟）
