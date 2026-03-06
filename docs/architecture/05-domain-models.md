# E. Git 领域模型设计

## 1. 设计原则

- 所有模型在 Rust 端定义，通过 `serde` 序列化为 JSON，再通过 Tauri IPC 传输到前端
- 所有模型实现 `#[derive(Serialize, Deserialize, specta::Type)]`，由 `tauri-specta` 自动生成 TypeScript 类型
- 字段命名使用 `#[serde(rename_all = "camelCase")]` 统一转为 camelCase 供前端使用
- 时间统一使用 Unix timestamp（i64，毫秒），前端负责格式化显示
- 路径在传输中使用 UTF-8 字符串，Rust 端负责规范化（`/` 作为分隔符）

---

## 2. 核心数据模型

### 2.1 RepositorySummary — 仓库摘要

```rust
/// 仓库基本信息，用于侧边栏/欢迎页展示
#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct RepositorySummary {
    /// 仓库根目录绝对路径（规范化，使用 / 分隔）
    pub path: String,
    /// 仓库名称（取路径最后一段）
    pub name: String,
    /// 当前 HEAD 引用
    pub head: HeadState,
    /// 是否有未提交的修改
    pub has_uncommitted_changes: bool,
    /// 是否处于某种 Git 进行中状态
    pub ongoing_operation: Option<OngoingOperation>,
    /// 远端 origin URL（如有）
    pub remote_url: Option<String>,
}

#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(tag = "type", rename_all = "camelCase")]
pub enum HeadState {
    /// 正常分支
    Branch { name: String, upstream: Option<String> },
    /// detached HEAD
    Detached { commit_sha: String },
    /// 未初始化（空仓库）
    Unborn,
}

#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub enum OngoingOperation {
    Merge,
    Rebase,
    CherryPick,
    Revert,
    Bisect,
}
```

**TypeScript 对应类型**（specta 自动生成）：
```typescript
export type RepositorySummary = {
  path: string;
  name: string;
  head: HeadState;
  hasUncommittedChanges: boolean;
  ongoingOperation: OngoingOperation | null;
  remoteUrl: string | null;
};

export type HeadState =
  | { type: 'Branch'; name: string; upstream: string | null }
  | { type: 'Detached'; commitSha: string }
  | { type: 'Unborn' };
```

---

### 2.2 BranchInfo — 分支信息

```rust
#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct BranchInfo {
    /// 分支名称（本地分支如 "main"，远程分支如 "origin/main"）
    pub name: String,
    /// 是否是当前分支
    pub is_current: bool,
    /// 是否是远程跟踪分支
    pub is_remote: bool,
    /// 最新提交 SHA（短格式，7位）
    pub tip_sha: String,
    /// 最新提交消息（第一行）
    pub tip_message: String,
    /// 最新提交时间（Unix 毫秒时间戳）
    pub tip_timestamp: i64,
    /// 与上游的同步状态（仅本地分支）
    pub upstream_status: Option<UpstreamStatus>,
}

#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct UpstreamStatus {
    /// 上游分支名称
    pub upstream_name: String,
    /// 本地领先上游的提交数
    pub ahead: u32,
    /// 本地落后上游的提交数
    pub behind: u32,
}
```

---

### 2.3 CommitSummary — 提交摘要（列表展示用）

```rust
/// 用于提交历史列表，数据量大时需轻量
#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct CommitSummary {
    /// 完整 SHA
    pub sha: String,
    /// 短 SHA（7 位）
    pub short_sha: String,
    /// commit 消息第一行
    pub subject: String,
    /// 作者名称
    pub author_name: String,
    /// 作者邮箱
    pub author_email: String,
    /// 提交时间（Unix 毫秒）
    pub timestamp: i64,
    /// 父 commit SHA 列表（合并提交有多个父）
    pub parent_shas: Vec<String>,
    /// 是否是合并提交
    pub is_merge: bool,
    /// 关联的分支/标签引用名称（用于图谱装饰）
    pub refs: Vec<RefLabel>,
}

#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct RefLabel {
    pub name: String,
    pub kind: RefKind,
}

#[derive(Serialize, Deserialize, specta::Type, Clone)]
pub enum RefKind {
    LocalBranch,
    RemoteBranch,
    Tag,
    Head,
}
```

---

### 2.4 CommitDetail — 提交详情（点击查看）

```rust
/// 点击单个 commit 时获取，包含完整信息
#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct CommitDetail {
    /// 基础摘要信息
    pub summary: CommitSummary,
    /// commit body（消息第一行之后的内容）
    pub body: Option<String>,
    /// 提交者（committer，可能与 author 不同）
    pub committer_name: String,
    pub committer_email: String,
    pub committer_timestamp: i64,
    /// 本次提交变更的文件列表
    pub changed_files: Vec<FileChangeSummary>,
    /// 统计信息
    pub stats: DiffStats,
}

#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct DiffStats {
    pub files_changed: u32,
    pub insertions: u32,
    pub deletions: u32,
}
```

---

### 2.5 FileChange / FileChangeSummary — 文件变更

```rust
/// 文件变更摘要（用于列表展示）
#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct FileChangeSummary {
    /// 文件路径（新路径，重命名时为新路径）
    pub path: String,
    /// 原路径（重命名时有值）
    pub old_path: Option<String>,
    /// 变更状态
    pub status: ChangeStatus,
    /// 新增行数
    pub insertions: u32,
    /// 删除行数
    pub deletions: u32,
    /// 是否是二进制文件
    pub is_binary: bool,
}

#[derive(Serialize, Deserialize, specta::Type, Clone)]
pub enum ChangeStatus {
    Added,
    Modified,
    Deleted,
    Renamed,
    Copied,
    TypeChanged,
    Unmerged,
}
```

---

### 2.6 StatusSummary — 工作区状态

```rust
/// git status 的完整快照
#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct StatusSummary {
    /// 已暂存的文件（将进入下一次 commit）
    pub staged: Vec<WorkingDirEntry>,
    /// 已修改但未暂存的文件
    pub unstaged: Vec<WorkingDirEntry>,
    /// 未跟踪的文件
    pub untracked: Vec<WorkingDirEntry>,
    /// 冲突文件（merge/rebase 进行中）
    pub conflicted: Vec<WorkingDirEntry>,
}

#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct WorkingDirEntry {
    pub path: String,
    pub old_path: Option<String>,
    pub status: ChangeStatus,
    pub is_binary: bool,
}
```

---

### 2.7 FileDiff — 完整 Diff 数据

```rust
/// 单个文件的完整 diff，用于 DiffViewer 渲染
#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct FileDiff {
    pub path: String,
    pub old_path: Option<String>,
    pub status: ChangeStatus,
    pub is_binary: bool,
    /// 文件太大时截断（> 500 KB diff）
    pub is_truncated: bool,
    /// diff hunks 列表
    pub hunks: Vec<DiffHunk>,
    pub stats: DiffStats,
}

#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct DiffHunk {
    /// 原文件起始行号
    pub old_start: u32,
    /// 原文件行数
    pub old_lines: u32,
    /// 新文件起始行号
    pub new_start: u32,
    /// 新文件行数
    pub new_lines: u32,
    /// 标题行（如 "@@ -1,7 +1,6 @@ fn main()"）
    pub header: String,
    /// hunk 内的行列表
    pub lines: Vec<DiffLine>,
}

#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct DiffLine {
    /// 行类型
    pub kind: DiffLineKind,
    /// 行内容（不含换行符）
    pub content: String,
    /// 原文件行号（None 表示新增行）
    pub old_lineno: Option<u32>,
    /// 新文件行号（None 表示删除行）
    pub new_lineno: Option<u32>,
}

#[derive(Serialize, Deserialize, specta::Type, Clone)]
pub enum DiffLineKind {
    Context,   // 上下文行
    Addition,  // 新增行 (+)
    Deletion,  // 删除行 (-)
    HunkHeader, // @@ 行
}
```

---

### 2.8 RemoteInfo — 远端信息

```rust
#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct RemoteInfo {
    /// 远端名称（如 "origin"）
    pub name: String,
    /// fetch URL
    pub url: String,
    /// push URL（可能与 fetch URL 不同）
    pub push_url: Option<String>,
    /// 该远端的分支列表（远程跟踪分支）
    pub branches: Vec<String>,
}
```

---

### 2.9 TagInfo — 标签信息

```rust
#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct TagInfo {
    pub name: String,
    /// 指向的 commit SHA
    pub target_sha: String,
    /// 是否是注释标签（annotated tag）
    pub is_annotated: bool,
    /// 注释标签消息（仅 annotated tag 有）
    pub message: Option<String>,
    /// 创建时间（Unix 毫秒，仅 annotated tag 有精确值）
    pub timestamp: Option<i64>,
    /// 创建者（仅 annotated tag 有）
    pub tagger_name: Option<String>,
}
```

---

### 2.10 RecentRepo — 最近打开的仓库

```rust
/// 持久化到本地配置文件的最近仓库记录
#[derive(Serialize, Deserialize, specta::Type, Clone)]
#[serde(rename_all = "camelCase")]
pub struct RecentRepo {
    pub path: String,
    pub name: String,
    /// 最后打开时间（Unix 毫秒）
    pub last_opened: i64,
    /// 是否仍然存在（打开时验证）
    pub exists: bool,
}
```

---

## 3. 错误类型

```rust
// domain/error.rs
#[derive(Debug, thiserror::Error, Serialize, specta::Type)]
#[serde(tag = "code", rename_all = "SCREAMING_SNAKE_CASE")]
pub enum AppError {
    #[error("Not a git repository: {path}")]
    NotARepo { path: String },

    #[error("Repository not opened: {path}")]
    RepoNotOpened { path: String },

    #[error("Git operation failed: {message}")]
    GitError { message: String },

    #[error("IO error: {message}")]
    IoError { message: String },

    #[error("Invalid path: {path}")]
    InvalidPath { path: String },

    #[error("Detached HEAD state")]
    DetachedHead,

    #[error("Bare repository is not supported")]
    BareRepo,

    #[error("Repository has ongoing operation: {operation}")]
    OngoingOperation { operation: String },

    #[error("Network error: {message}")]
    NetworkError { message: String },

    #[error("Authentication required")]
    AuthRequired,

    #[error("Permission denied: {path}")]
    PermissionDenied { path: String },

    #[error("Repository is too large for this operation")]
    RepoTooLarge,

    #[error("Internal error: {message}")]
    Internal { message: String },
}
```

---

## 4. 类型生成配置

在 `lib.rs` 中配置 specta 自动生成：

```rust
// src-tauri/src/lib.rs
#[cfg(debug_assertions)]
fn export_types() {
    use tauri_specta::{collect_commands, ts};
    ts::builder()
        .commands(collect_commands![
            // 列出所有 command
        ])
        .path("../src/types/bindings.ts")
        .export()
        .unwrap();
}
```

生成的 `src/types/bindings.ts` 包含所有 Rust 类型对应的 TypeScript 定义，以及所有 command 函数的类型签名。
